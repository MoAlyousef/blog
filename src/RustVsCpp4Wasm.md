# Rust vs C++ for frontend web (wasm) programming

Date: 2022-7-26

Several languages can now target wasm, I'll focus on Rust and C++ as these seem to have the most mature ecosystems, with C++'s Emscripten toolchain and Rust's wasm-bindgen (web-sys, js-sys etc) ecosystem.
Keep in mind that both languages leverage LLVM's ability to generate wasm. Wasm itself has no direct access to the DOM, as such DOM calls pass through javascript. 

Basically when using a language, you're buying into the ecosystem. You can still target Emscripten using Rust via the wasm32-unknown-emscripten target. However it would require that the LLVM version of Rust you're using and the LLVM version Emscripten is using are compatible.
Similarly, you can invoke clang directly with the `--target=wasm32` flag (requires wasm-ld and the std headers), and it should output wasm. However, the non-emscripten wasm ecosystem is barren!

Advantages of using C++:
- Emscripten's headers are C/C++ headers. 
- Emscripten supports CMake (the de jour build system for C++, via both emcmake and a CMake toolchain file). However, the docs refer to raw calls of emcc/em++, which can be difficult to translate to proper CMake scripts:
```cmake
add_executable(index src/main.cpp)
set_target_properties(index PROPERTIES SUFFIX .html LINK_FLAGS "-s WASM=1 -s EVAL_CTORS=2 --bind --shell-file ${CMAKE_CURRENT_LIST_DIR}/my_shell.html")
```
- Emscripten provides Boost, SDL and OpenGL/WebGL support out of the box.
- Emscripten translates OpenGL calls to WebGL.
- vcpkg (a C/C++ package manager) supports building packages for emscripten.
- Qt supports Emscripten (buggy).
- Emscripten provides a virtual file system that simulates the local file system, std::filesystem works out of the box.
- Emscripten supports multithreading. 
- The above means that an existing native game leveraging SDL +/- OpenGL can be recompiled using emscripten, with probably minor tweaks to the build script (and event-loop), and things should run.
- Emscripten bundles the binaryen toolchain as well. For example, compiling with optimizations will automatically run wasm-opt.

Disadvantages of using C++:
- Emscripten requires 800mb of install space. It bundles many tools which might be already installed (like nodejs). If installed in an unusual location, the install would likely be broken!
- Using C++ outside of Emscripten to target wasm/web is complicated. It requires wasm-ld, the std/system headers (maintained in the Emscripten project) and writing the js glue manually.
- Emscripten provides a WebIDL binder, however, bindings to the DOM api are not provided. It can be integrated into a build script, but in any case, it's not ergonomic to generate and use.

It makes targetting the DOM with Emscripten a bit of a chore:
```c++
#include <emscripten/val.h>

using emscripten::val;

int main() {
    auto doc = val::global("document");
    auto body = doc.call<val>("getElementsByTagName", val("body"))[0];
    auto btn = doc.call<val>("createElement", val("BUTTON"));
    body.call<void>("appendChild", btn);
    btn.set("textContent", "Click");
}
```
As you can probably guess, these DOM calls are stingly-typed and aren't checked at compile time, if you pass a wrong type or even a typo, it would error on runtime.

Advantages of using Rust:
- Cargo is agnostic to the target. And installing the wasm32-unknown-unknown target is trivial.
- Even without Emscripten, wasm-bindgen provides bindings to much of the DOM api and other javascript calls.
- wasm-bindgen provides a cli tool which allows generating javascript glue code for loading into web and non-web apps, which can be easily installed using `cargo install wasm-bindgen-cli`.
- The Rust ecosystem provides several tools like wasm-pack and trunk which automatically call wasm-bindgen-cli and create the necessary js and html files needed for web.
- The above means that the calls are checked at compile time, and are easier to program against:
```rust
// The above code translated to Rust
use wasm_bindgen::prelude::*;

fn main() {
    let win = web_sys::window().unwrap();
    let doc = win.document().unwrap();
    let body = doc.body().unwrap();
    let btn = doc.create_element("BUTTON").unwrap();
    body.append_child(&elem).unwrap();
    btn.set_text_content(Some("Click"));
}
```

Disadvantages of using Rust:
- The wasm32-unknown-unknown toolchain doesn't translate filesystem or threading calls. (except for the wasi target which translates std::fs calls into the platform equivalent calls, however, an app targetting wasi might not work in the browser).
- The wasm32-unknown-unknown toolchain can optimize the output when building for release, but further optimization requires installing binaryen.
- The wasm32-unknown-unknown toolchain doesn't translate OpenGL calls to webgl calls.
- The wasm32-unknown-unknown toolchain doesn't support linking C/C++ libs built for wasm.
- wasm-bindgen doesn't support the emscripten wasm target

## Conclusion
Both Rust and C++ can target the browser and perform DOM calls. Rust provides a better api with web-sys. Emscripten's `bind` api is stringly-typed so can be a chore to program against. The wasm32-unknown-unknown target is better geared for DOM calls or graphics via the canvas api, while emscripten is better geared for apps targetting OpenGL/SDL (games). As for client-side computation, both targets can be used.
