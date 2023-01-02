# Cargo as a tool to distribute C/C++ executables

Date: 2021-05-04

If you didn't know already, Cargo, Rust's package manager, can be used to install executable binaries via cargo-install. The downside is that they have to be Rust binaries. That makes sense since it does build binaries from source, and naturally Cargo knows how to build Rust projects!

However, this functionality is too good to be exclusive to Rust projects. We'll see how we can also use cargo-install to install C/C++ executables.

In short, this works by redefining the C/C++ executable's main() function.

The technique described here is useful when an application lacks an official package, when the official package is outdated or when you don't wish to conflict with an already installed official package. 

A similar technique was used to wrap [Fluid](https://crates.io/crates/fltk-fluid) (FLTK’s gui designer). However with C++, the redefined main function needs also to be preceded by `extern "C"` to avoid name mangling.

Back to the topic. We'll start with a simpler project.
We'll create our Rust project:
```
$ cargo new cbin
$ cd cbin
```
We'll create a C binary which takes command-line arguments:
```
$ touch src/main.c
$ cat > src/main.c << EOF
> #include <stdio.h>
> int main(int argc, char **argv) {
>   if (argc > 1)
>     printf("Hello %s\n", argv[1]);
>   return 0;
> }
> EOF
```

To build the C binary, we'll need a build script, and we can use the cc crate along with it to make our lives a bit easier!
```toml
# Cargo.toml
[build-dependencies]
cc = "1.0"
```

Our build.rs file (which we create at the root of our project) would look something like:
```rust
// build.rs
fn main() {
    cc::Build::new()
        .file("src/main.c")
        .define("main", "cbin_main") // notice our define here!
        .compile("cbin");
}
```

Check everything builds by running `cargo build`, you'll notice cargo built a static library `libcbin.a` (or cbin.lib if using the msvc toolchain) in the OUT_DIR.

Lets now wrap the library call `cbin_main` which we had redefined in our build. Our src/main.rs should look like:

```rust
// src/main.rs
use std::env;
use std::ffi::CString;
use std::os::raw::*;

extern "C" {
    pub fn cbin_main(argc: c_int, argv: *mut *mut c_char) -> c_int;
}

fn main() {
    let mut args: Vec<_> = env::args()
        .into_iter()
        .map(|s| CString::new(s).unwrap().into_raw())
        .collect();
    let _ret = unsafe { cbin_main(args.len() as i32, args.as_mut_ptr()) };
}
```
This basically takes all command-line args passed to the Rust binary and passes them to the C binary (that we turned into a library!).

You can check by running:
```
$ cargo run -- world
```

Now you can run `cargo install --path .` to install your C binary wrapper. Or you can publish your crate to crates.io and your crate