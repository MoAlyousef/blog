# Forays into the Wasm Component Model

The initial title of this post was "The wasm component model isn't real, it can't hurt you"! Along with an image I planned to add with terms like wasi, wit, wac, wkg, jco and a few other confusing wasm-related terms. Eventually reverted since the component model obviously exists, albeit support for it is still fragmentary. Browsers and javascript runtimes don't support it (yet) and it's still a [Phase 1 proposal](https://github.com/WebAssembly/proposals?tab=readme-ov-file#phase-1---feature-proposal-cg).

However, where wasi is concerned, the component model appears to be officially endorsed. The [wasi-sdk](https://github.com/WebAssembly/wasi-sdk) (under the official WebAssembly org) will generate a wasm component when building a C/C++ binary targeting wasm32-wasip2. The Rust toolchain's [wasm32-wasip2 target](https://doc.rust-lang.org/nightly/rustc/platform-support/wasm32-wasip2.html) (experimental tier 2 since Nov 2024) will similarly generate a wasm component. The wasm-component-ld (wrapper around wasm-ld) is automatically run on the generated wasm core module (unless you opt out by passing the `--skip-wit-component` linker flag).

I had to recently port a library which supported freestanding wasm32, wasip1 and emscripten to wasip2, without having understood the component model, and that was a painful experience. So this post aims to shed light into what I learned in the process.
I'll preface by saying that LLMs didn't help much. I'm guessing since it's all too new and there aren't many resources on the subject.

If you're new to the wasm ecosystem, you might be wondering how a wasm component differs from whatever was before it, which was a core module. You can read more about it [here](https://component-model.bytecodealliance.org/design/why-component-model.html).Without going into much detail on the differences, core modules limited data exchange to basic types, namely integers and floats. If you needed to pass a string from a core module to javascript for example, you would pass the address (an integer) of that string in wasm's linear memory, that along with its length, unless nul-terminated in which case you would need to account for that. The component model aims at remedying this by allowing the exchange of higher level types (generic lists, variants, records, enums, strings etc) without concerning yourself with your wasm binary's memory or __indirect_function_table. As such these things are hidden from you, in exchange, you get higher level abstractions. You also no longer have to fiddle with a myriad of linker flags like `--import-memory --export-memory --export-table --export-dynamic --export-if-defined=whatever`.
This is done by declaring your types and interfaces in WIT (Wasm Interface Type language) in wit files in your wit directory!

The idea is that higher-level wit interfaces will be distributed, devs will program against the APIs they declare, from any programming language which supports the component model (currently 8 languages), without meddling with low-level details or a C ABI. Components should give us better modularity, language interop and portability across languages and runtimes.

Before going into that, lets see how things worked prior to the component model.

## Before components

### Importing an extern function (from javascript)

Let's say you wanted to console.log a Rust string:
```rust
unsafe extern "C" {
    fn console_log_string(s: *const u8, len: usize);
}

fn main() {
    let s = "Hello, world!";
    unsafe {
        console_log_string(s.as_ptr(), s.len());
    }
}
```

On the javascript side, you would define `console_log_string`:
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>
<body>
    <script>
        window.onload = async () => {
            const response = await fetch("./target/wasm32-unknown-unknown/debug/blog.wasm");
            const { instance } = await WebAssembly.instantiateStreaming(response, {
                env: {
                    console_log_string: (ptr, len) => {
                        const memory = new Uint8Array(instance.exports.memory.buffer, ptr, len);
                        // Typically you would instantiate your TextDecoder once instead of with every call
                        const string = new TextDecoder('utf-8').decode(memory);
                        console.log(string);
                    }
                }
            });
            instance.exports.main();
        };
    </script>
</body>
</html>
```
When targeting a javascript runtime you would just do away with the html part and window.onload.

The wasip1 model is practically the same, with a slight difference in that you would pass a `wasi_snapshot_preview1` object alongside `env`. An npm library that I would recommend is [@bjorn3/browser_wasi_shim](https://github.com/bjorn3/browser_wasi_shim):
```javascript
    let wasi = new WASI([], [], []);
    const response = await fetch("./target/wasm32-wasip1/debug/blog.wasm");
    const { instance } = await WebAssembly.instantiateStreaming(response, {
        wasi_snapshot_preview1: wasi.wasiImport
        env: {
            console_log_string: (ptr, len) => {
                const memory = new Uint8Array(instance.exports.memory.buffer, ptr, len);
                const string = new TextDecoder('utf-8').decode(memory);
                console.log(string);
            }
        }
    });
```

Similarly with emscripten, you would typically define the function in your C/C++ source code using the EM_JS macro:
```cpp
    EM_JS(void, console_log_string, (const char *ptr, size_t len), {
        const str = UTF8ToString(ptr, len);
        console.log(str);
    });
```
You can also pass the definition of `console_log_string` if you build with the shell option `-sMODULARIZE`:
```javascript
    import initModule from "./bin/main.mjs";
    window.onload = async () => {
        const mymain = await initModule({
            console_log_string: /* definition goes here */
        });
    };
```

### Exporting a function (to javascript)
Let's say we want to use a native function in our javascript. Before the component model, similarly to how we imported the function, we'll have to export native functions as extern "C" (or in the case of Zig, extern "env") function for it to be callable from javascript. 

```rust
#[unsafe(no_mangle)]
extern "C" fn my_strlen(s: *const u8) -> usize {
    unsafe {
        let mut len = 0;
        while *s.add(len) != 0 {
            len += 1;
        }
        len
    }
}

#[unsafe(no_mangle)]
extern "C" fn greet(s: *const u8, len: usize) -> *const u8 {
    unsafe {
        let greeting = format!("Hello {}\0", 
            std::str::from_utf8(std::slice::from_raw_parts(s, len)).unwrap()
        );
        let ptr = std::alloc::alloc(std::alloc::Layout::from_size_align(greeting.len(), 1).unwrap());
        std::ptr::copy_nonoverlapping(greeting.as_ptr(), ptr, greeting.len());
        ptr
    }
}
```

In emscripten you would use the EMSCRIPTEN_KEEPALIVE macro along with specifying it as an extern "C" function.

Which can be used from js:
```javascript
    const enc = new TextEncoder();
    const dec = new TextDecoder("utf-8");
    const txt = enc.encode("World!");
    // __rust_alloc & __rust_dealloc are automatically exported in wasm32 core module compiled by the rust toolchain
    const ptr = instance.exports.__rust_alloc(txt.length, 1);
    new Uint8Array(instance.exports.memory.buffer).set(txt, ptr);
    const msg = instance.exports.greet(ptr, txt.length);
    let len = instance.exports.my_strlen(msg);
    console.log(dec.decode(new Uint8Array(instance.exports.memory.buffer, msg, len)));
    instance.exports.__rust_dealloc(ptr, len, 1);
```

## With components

### Importing an extern function (from javascript)
Now when it comes to wasip2, unless you pass the `--skip-wit-component` flag to the linker (wasm-component-ld), you would end up with a wasm component. So how can we declare our `console_log_string` function for usage within Rust, and how can we define it in javascript. Well we will have to do it in WIT. Luckily for simple cases, you can use a macro `wit_bindgen::generate!` if you add wit-bindgen as a dependency to your project. And since we're building a runnable program, we'll also use the `wasip2` crate:
```toml
[package]
name = "blog"
version = "0.1.0"
edition = "2024"

[lib]
crate-type = ["cdylib"]

[dependencies]
wit-bindgen = "0.44"
wasip2 = "1"
```
For C/C++ code, you would have to run `wit-bindgen` manually or as part of your build via CMake for example. It will generate a header, source file and an object file!
These should be added to your build, unless you're creating a library, in which case the object file should be exposed as a target (in CMake parlance!), otherwise you risk losing it in the final link step. That's actually easier than telling your dependents to manually pass --whole-archive/--no-whole-archive to the linker. Exposing the object as a target can be easily done in CMake:
```cmake
  # code from the library I'm working on!
  # can be used by consumers using target_link_libraries(myapp PRIVATE emcore::component_type)
  install(FILES ${CMAKE_CURRENT_LIST_DIR}/src/env_component_type.o
          DESTINATION ${CMAKE_INSTALL_LIBDIR})

  add_library(emcore_component_type INTERFACE)
  target_link_libraries(emcore_component_type INTERFACE
    "$<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/src/env_component_type.o>"
    "$<INSTALL_INTERFACE:${CMAKE_INSTALL_LIBDIR}/env_component_type.o>"
  )
  add_library(emcore::component_type ALIAS emcore_component_type)
  install(TARGETS emcore_component_type EXPORT emcoreTargets)
```

Back to Rust, notice in the above Cargo.toml how we change this from an executable binary to a cdylib. That's because runnable wasip2 components (or those using the `-mexec-model=command` in C/C++), would define a `wasi:cli/run` interface:
```rust
wit_bindgen::generate!({inline: "
package my:app@0.1.0;

interface logger {
    console-log-string: func(s: string);
}

world app {
    import logger;
}
"});

struct App;

impl wasip2::exports::cli::run::Guest for App {
    fn run() -> Result<(), ()> {
        crate::my::app::logger::console_log_string("Hello, world!");
        Ok(())
    }
}

wasip2::cli::command::export!(App);
```

Since browsers don't support wasip2 as of yet, we can use jco by the bytecodealliance org to generate the necessary core modules and javascript glue code. After installing jco, we run the `transpile` command:
```bash
npm i --save-dev @bytecodealliance/jco
npx jco transpile ./target/wasm32-wasip2/release/blog.wasm -O -o bin/app --instantiation async --no-nodejs-compat --tla-compat --no-typescript
```
The `-O` flag tells jco to optimize the generated wasm modules. This might not be necessary for Rust wasm components, however if you try to generate C/C++ wasm components in Release mode, you'll be hit with an error. Basically binaryen can't read the wasm component format. More on that later!

The transpile step will generate a directory `bin/app` with the generated core wasm modules and js glue files.
We can then instantiate the generated wasm modules, then pass our definition of console-log-string as part of my:app/logger interface:
```javascript
import { WASIShim } from "@bytecodealliance/preview2-shim/instantiation";
// generated by jco
import { instantiate as initApp } from "../bin/app/blog.js";

async function main() {
  const getAppCore = async (p) => {
    const bytes = await fetch(
      new URL(`../bin/app/${p}`, import.meta.url)
    );
    return WebAssembly.compileStreaming(bytes);
  };

  const wasiShim = new WASIShim({});
  const wasi = wasiShim.getImportObject();

  const app = await initApp(getAppCore, {
    ...wasi,
    "my:app/logger": {
      consoleLogString: (s) => {
        console.log(s);
      },
    },
  });
  app.run.run();
}

await main();
```

The above code requires a bundler like webpack to resolve the node_modules paths etc.

### Exporting a function (to javascript)
With the component model, we would simply define the function and export it:
```rust
wit_bindgen::generate!({inline: "
package my:app@0.1.0;

interface greeter {
    greet: func(s: string) -> string;
}

world app {
    export greeter;
}
"});

struct App;

impl crate::exports::my::app::greeter::Guest for App {
    fn greet(s: String) -> String {
        format!("Hello, {}!", s)
    }
}

export!(App);
```

And we can use it from our javascript:
```javascript
import { WASIShim } from "@bytecodealliance/preview2-shim/instantiation";
// generated by jco
import { instantiate as initApp } from "../bin/app/blog.js";

async function main() {
  const getAppCore = async (p) => {
    const bytes = await fetch(
      new URL(`../bin/app/${p}`, import.meta.url)
    );
    return WebAssembly.compileStreaming(bytes);
  };

  const wasiShim = new WASIShim({});
  const wasi = wasiShim.getImportObject();

  const app = await initApp(getAppCore, {
    ...wasi,
  });
  console.log(app.greeter.greet("World!"));
}

await main();
```

Actually even the instantiation code is simpler since we don't import any javascript exports, but I went for the manual instiation code for symmetry with the previous section!
For example if you don't pass --instantion:
```bash
# no --instantiation
npx jco transpile ./target/wasm32-wasip2/release/blog.wasm -O -o bin/app --no-nodejs-compat --tla-compat --no-typescript
```

You would load using:
```javascript
import { $init, greeter } from "../bin/app/blog.js";

async function main() {
  await $init;
  console.log(greeter.greet("World!"));
}

await main();
```

## Downsides
I like the idea behind the component model and would like for it to succeed. It would greatly simplify working with wasm. However it's not all moonlight and roses. Especially for those of us more interested in wasm in the browser.

- Currently the binaries are larger when targeting wasip2, but that's irrespective of whether we're building a wasip2 core module or component.

- It's unclear whether browsers will support the component model if and when wasi_snapshot_preview2 lands. That means we might still need the jco-transpile step for longer.

- No centralised wit registry as of yet. wit files need to be vendored in a wit/deps directory or pulled via wkg from non-centralized registries.

- Things still haven't settled so interfaces are prone to changes.

- Outside the browser, support is fragmentary and lagging across most wasm runtimes. Currently only wasmtime supports it.

- Adding wit interfaces might feel like effort duplication.

- Tools galore! Working with components requires more tools than what you would typically require from your default toolchain:
    - wit-bindgen
    - jco
    - wasm-tools
    - wac
    - wkg

- Ye olde tools don't work (well) with wasm components: Binaryen, (wasm)objdump, (wasm)strip.

- Linking components isn't done with your usual linker, you can use wac to compose and plug components.

- Some of the above mentioned tools are early in development and are not yet stable.

- Debugging components can be a bit difficult. Lots of trampolines!!

## Conclusion
I like the value proposition of the component model, however, things are still cooking. Starting out, you might run into a steep learning curve, mostly because it's different from what you might be used to. WIT isn't difficult to learn. The tooling in my opinion needs to become more streamlined and part of the toolchain.
The bigger picture however, is that once the component model is widely supported, it should make wasm programming much easier since you're programming against higher level abstractions, while previously you had to deal with lower level C-like interfaces. The underlying language would be irrelevant to developers consuming those interfaces. WIT will become the new ABI.