# fltk-rs in 2022

Date: 2023-01-02

## Looking back
Looking back into 2022, [fltk-rs](https://github.com/fltk-rs/fltk-rs) saw its 1.0 release in April 2022. On October 2022, the project finished its 3rd year. 2022 also saw the publication of the fltk-rs [book](https://fltk-rs.github.io/fltk-book/). And a rewrite of [fl2rust](https://github.com/fltk-rs/fl2rust), which is a FLUID to Rust transpiler. Fluid is a RAD FLTK application which is similar to Gtk's glade and QtCreator. 

Looking back further, fltk-rs was started for a specific requirement, to easily deploy statically-linked gui applications on Windows 7 PCs in my university hospital's simulation center, it also had to be crossplatform for those using mac laptops! 

At the time, most pure Rust toolkits lacked many functionalities I needed (menus, tables, multiline text input, custom graph drawing, multiwindows ...etc), and I was just starting to use Rust so I felt incapable of contributing to the budding gui ecosystem. Gtk and Qt bindings existed at the time, but required dynamic linking. It was also during covid lockdown, so I had some extra time since teaching duties and elective cases decreased. So instead of just writing the project in another language, I started learning Rust and applying that knowledge into creating the bindings to FLTK.

That's to say that as a novice, I made many mistakes, most of which I consider fixed with the 1.0 release, however, there are some which were pointed out later, namely the timeout api, escpecially when it comes to cancellation. The older functions were deprecated, but the newer ones like `app::add_timeout3` and `app::remove_timeout3` stand out like a sore thumb.

Maybe releasing a 1.0 was a bit hasty. It taught me however more things to mitigate api breakage. Another aspect was targetting FLTK 1.4, which if you don't know is a yet to be released version of FLTK. That means it's a moving target. And even though FLTK is considered quite conservative when it comes to C++ codebases (it's still using C++98 and without the std library!), it's actively developend and several of the added functionality have changed their function signatures, which required some workarounds in fltk-rs. Some things were out of my hand, such as the upstream removal of the FLTK android driver since it was considered experimental and difficult to integrate on the C++ side, especially in preparation for the 1.4 release, so to avoid managing forks and such, it was subsequently removed from fltk-rs. 

On the other hand, FLTK itself had nice improvements. Drawing on Linux/BSD now uses Cairo for anti-aliased drawing, and on Windows, it uses GDI+ to the same effect. A wayland backend was added which allows targetting wayland directly, i.e. not through xwayland. And the OpenGL backend was extended to allow drawing widgets using OpenGL. That means GlWindow can now display widgets, if you need hardware acceleration or need to display widgets on top of 3D graphics!

## Looking forward
The current plan is that once FLTK 1.4 is shipped, to release the last version of fltk-rs version 1, and continue working on fltk-rs 2.0 (work on that has started in version2 branch in the fltk-rs repo). This would be using a 0.20.x version (if possible), until an FLTK 1.5 is released, and only then to release version 2.

I'm also planning to see if AccessKit can be retrofitted to fltk-rs, and maybe provide that functionality in a different crate. Even though FLTK handles keyboard navigation and input method editors, it still lacks in screen reader support.

I also plan to try out the newer Rust gui toolkits, since I feel far removed from where I once was. I've only tried egui in the past year and a half, and that was to add an [fltk-rs integration](https://github.com/fltk-rs/fltk-egui) to it.

I'm already excited to see the ecosystem maturing. If you frequent the Rust subreddit, you would notice a recurring question on what gui framework to use, and there would always be a few who would say that Rust isn't gui yet. Maybe not a few years back, but if you compare the situation to other programming languages (apart from C/C++), Rust already provides many gui crates that you can already use.