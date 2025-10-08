# Interfacing with the Objective-C runtime
<br>
Date: 2022-07-18
<br>

I recently released a proof-of-concept [library](https://github.com/MoAlyousef/floui) wrapping several native widgets on Android and iOS. It's written in C++, and I've also released [Rust bindings](https://github.com/MoAlyousef/floui-rs) to it. In my post on the Rust subreddit announcing the release, a fellow redditor validly remarked "I'm a little surprised you wrapped a floui-rs around the Floui C++ project rather than just writing rust and calling into objc or the jni". I wasn't satisfied with my succinct answer, but I thought a Reddit reply wouldn't provide enough context to many reading it. So I decided to write this post. Just a note before diving in, floui's iOS code implementation is in Objective-C++ and requires a `#define FLOUI_IMPL` macro in at least one Objective-C++ source file, the rest of the gui code can be written in cpp files or .mm files since the interface is in C++. Regarding the JNI part, it's equally painful to write in C++ or Rust. So no point in discussing that.

Most Apple frameworks expose an Objective-C api, except for some which expose a C++ (DriverKit) or a Swift api (StoreKit2). That means that Objective-C is Apple's system's language par excellence, and other languages will need to be able to interface with it for any functionality provided by Apple in its frameworks. Interfacing with Objective-C isn't straightforward, and apart from Swift (and only on Apple platforms, it can't interface with gnustep for example), no other language can directly interface with it. Luckily, the objc runtime offers C functions which allow other languages to interface with Objective-C frameworks. And any language that can call C, can -via the objc runtime- interface with Objective-C.

In this post, I'll show how this can be done using C++ and Rust. The C++ version can be modified to C by just replacing `auto` with a concrete type.

As an example, we'll be creating an iOS app purely in C++ and then in Rust. I say pure in that there's no visible Objective-C code as far as the developer is concerned, however, this still calls into the UIKit framework which is an ObjC framework.
The app will be the equivalent of the following Objective-C app:
```objc
// main.m or main.mm
#import <UIKit/UIKit.h>

@interface AppDelegate : UIResponder <UIApplicationDelegate>
@property(strong, nonatomic) UIWindow *window;
@end

@interface ViewController :UIViewController
@end

@implementation AppDelegate
- (BOOL)application:(UIApplication *)application
    didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    CGRect frame = [[UIScreen mainScreen] bounds];
    self.window = [[UIWindow alloc] initWithFrame:frame];
    [self.window setRootViewController:[ViewController new]];
    self.window.backgroundColor = UIColor.whiteColor;
    [self.window makeKeyAndVisible];
    return YES;
}
@end

@implementation ViewController
- (void)clicked {
    NSLog(@"clicked");
}
- (void)viewDidLoad {
    [super viewDidLoad];
    UIButton *btn = [UIButton buttonWithType:UIButtonTypeCustom];
    btn.frame = CGRectMake(100, 100, 80, 30);
    [btn setTitle:@"Click" forState:UIControlStateNormal];
    [btn setTitleColor:UIColor.blueColor forState:UIControlStateNormal];
    [btn addTarget:self
                  action:@selector(clicked)
        forControlEvents:UIControlEventPrimaryActionTriggered];
    [self.view addSubview:btn];
}
@end

int main(int argc, char *argv[]) {
    NSString *appDelegateClassName;
    @autoreleasepool {
        appDelegateClassName = NSStringFromClass([AppDelegate class]);
    }
    return UIApplicationMain(argc, argv, nil, appDelegateClassName);
}
```
Simple enough, creates a view with a button which prints to the console when clicked. Note that Objective-C can seamlessly incorporate C++ code into what's called Objective-C++, and it requires changing the file extension from .m to .mm. This already speaks volumes about the flexibility and extendibility of Objective-C. Generally I find Objective-C++ is less verbose than Objective-C since it can benefit from modern C++ features like type inference:
```cpp
UISomeBespokenlyLongTypeName *t = [UISomeBespokenlyLongTypeName new];
// becomes
auto t = [UISomeBespokenlyLongTypeName new];
```
Modern C++ also offers some other niceties like lambdas, namespaces, metaprogramming capabilities, optional, variant, containers and algorithms.
Objective-C itself is considered verbose, espacially when compared to Swift. However, calling into the ObjC runtime from other languages, as we'll see, is even more verbose.

To get things working in pure C++, we need to include several headers, those of the Objective-C runtime and CoreFoundation and CoreGraphics. The Objective-C runtime headers provide several methods like objc_msgSend and others which allow us to create Objective-C classes and add/override methods etc.
```cpp
// main.cpp
#include <CoreFoundation/CoreFoundation.h>
#include <CoreGraphics/CoreGraphics.h>
#define OBJC_OLD_DISPATCH_PROTOTYPES 1
#include <objc/objc.h>
#include <objc/runtime.h>
#include <objc/message.h>

extern "C" int UIApplicationMain(int, ...);

extern "C" void NSLog(objc_object *, ...);

BOOL didFinishLaunching(objc_object *self, SEL _cmd, void *application, void *options) {
    auto mainScreen = objc_msgSend((id)objc_getClass("UIScreen"), sel_getUid("mainScreen"));
    CGRect (*boundsFn)(id receiver, SEL operation);
    boundsFn = (CGRect(*)(id, SEL))objc_msgSend_stret;
    CGRect frame = boundsFn(mainScreen, sel_getUid("bounds"));
    auto win = objc_msgSend((id)objc_getClass("UIWindow"), sel_getUid("alloc"));
    win = objc_msgSend(win, sel_getUid("initWithFrame:"), frame);
    auto viewController = objc_msgSend((id)objc_getClass("ViewController"), sel_getUid("new"));
    objc_msgSend(win, sel_getUid("setRootViewController:"), viewController);
    objc_msgSend(win, sel_getUid("makeKeyAndVisible"));
    auto white = objc_msgSend((id)objc_getClass("UIColor"), sel_getUid("whiteColor"));
    objc_msgSend(win, sel_getUid("setBackgroundColor:"), white);
    object_setIvar(self, class_getInstanceVariable(objc_getClass("AppDelegate"), "window"), win);
    return YES;
}

void didLoad(objc_object *self, SEL _cmd) {
    objc_super _super = {
         .receiver = self,
         .super_class = objc_getClass("UIViewController"),
    };
    objc_msgSendSuper(&_super, sel_getUid("viewDidLoad"));
    auto btn = objc_msgSend((id)objc_getClass("UIButton"), sel_getUid("buttonWithType:"), 0);
    objc_msgSend(btn, sel_getUid("setFrame:"), CGRectMake(100, 100, 80, 30));
    auto title = objc_msgSend((id)objc_getClass("NSString"), sel_getUid("stringWithUTF8String:"), "Click");
    objc_msgSend(btn, sel_getUid("setTitle:forState:"), title, 0);
    auto blue = objc_msgSend((id)objc_getClass("UIColor"), sel_getUid("blueColor"));
    objc_msgSend(btn, sel_getUid("setTitleColor:forState:"), blue, 0);
    objc_msgSend(btn, sel_getUid("addTarget:action:forControlEvents:"), self, sel_getUid("clicked"), 1 << 13);
    auto view = objc_msgSend(self, sel_getUid("view"));
    objc_msgSend(view, sel_getUid("addSubview:"), btn);
}

void clicked(objc_object *self, SEL _cmd) {
    auto msg = objc_msgSend((id)objc_getClass("NSString"), sel_getUid("stringWithUTF8String:"), "clicked");
    NSLog(msg);
}

int main(int argc, char *argv[]) {
    auto AppDelegateClass = objc_allocateClassPair(objc_getClass("UIResponder"), "AppDelegate", 0);
    class_addIvar(AppDelegateClass, "window", sizeof(id), 0, "@");
    class_addMethod(AppDelegateClass, sel_getUid("application:didFinishLaunchingWithOptions:"), (IMP) didFinishLaunching, "i@:@@");
    objc_registerClassPair(AppDelegateClass);

    auto ViewControllerClass = objc_allocateClassPair(objc_getClass("UIViewController"), "ViewController", 0);
    class_addMethod(ViewControllerClass, sel_getUid("viewDidLoad"), (IMP) didLoad, "v@");
    class_addMethod(ViewControllerClass, sel_getUid("clicked"), (IMP) clicked, "v@");
    objc_registerClassPair(ViewControllerClass);
    
    auto frame = objc_msgSend((id)objc_getClass("UIScreen"), sel_getUid("mainScreen"));
    auto name = objc_msgSend((id)objc_getClass("NSString"), sel_getUid("stringWithUTF8String:"), "AppDelegate");
    id autoreleasePool = objc_msgSend((id)objc_getClass("NSAutoreleasePool"), sel_registerName("new"));
    UIApplicationMain(argc, argv, nil, name);
    objc_msgSend(autoreleasePool, sel_registerName("drain"));
}
```
You can create a new Objective-C project, delete all source files and replace them with this single C++ file, and XCode will happily build it and run the binary on your simulator.
If you would like to compile this from the command-line:
```
clang++ -std=c++11 -arch x86_64 -isysroot $(xcrun --sdk iphonesimulator --show-sdk-path) main.cpp -fobjc-arc -lobjc -framework UIKit
# You can install it directly on an iOS simulator if you prepare an appropriate info.plist
xcrun simctl install booted path/to/bundle.app # assumes you have a simulator booted
```
You can also use CMake (with a [toolchain file](https://github.com/leetal/ios-cmake)) with certain bundle info predefined in your CMakeLists.txt:
```cmake
cmake_minimum_required(VERSION 3.14)
project(app)

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

set(MACOSX_BUNDLE_BUNDLE_NAME "Minimal Uikit Application")
set(MACOSX_BUNDLE_BUNDLE_VERSION 0.1.0)
set(MACOSX_BUNDLE_COPYRIGHT "Copyright © 2022 moalyousef.github.io. All rights reserved.")
set(MACOSX_BUNDLE_GUI_IDENTIFIER com.neurosrg.cpure)
set(MACOSX_BUNDLE_ICON_FILE app)
set(MACOSX_BUNDLE_LONG_VERSION_STRING 0.1.0)
set(MACOSX_BUNDLE_SHORT_VERSION_STRING 0.1)

add_executable(main main.cpp)
target_compile_features(main PUBLIC cxx_std_11)
target_link_libraries(main PUBLIC "-framework UIKit" "-framework CoreFoundation" "-framework Foundation" objc)
```
Then:
```
cmake -Bbin -GNinja -DPLATFORM=OS64COMBINED -DCMAKE_TOOLCHAIN_FILE=ios.toolchain.cmake # just to get the compile commands for clangd auto-completion on vscode
rm -rf bin
cmake -Bbin -GXcode -DPLATFORM=OS64COMBINED -DCMAKE_TOOLCHAIN_FILE=ios.toolchain.cmake 
cd bin
xcodebuild build -configuration Debug -sdk iphonesimulator -arch x86_64 CODE_SIGN_IDENTITY="" CODE_SIGNING_REQUIRED=NO
xcrun simctl install booted Debug-iphonesimulator/main.app
```

Returning to our C++ example, you can notice how the verbosity of Objective-C became 3-fold in the C++ example. This is also a stringly-typed api, meaning if you type the selector wrong, the compiler won't catch it, and you'll be hit with a runtime exception. You can argue that Objective-C also will gladly let you write whatever selector you want, and still throw. However, with tooling like XCode, this would be caught. You can even compile your code with `-Wno-objc-method-access` to catch such problems in Objective-C at compile time.

We'll now move to Rust, which is a relatively younger programming language. It's not an Apple officially-supported language, however, it has much better crossplatform support than for example Swift. And more importantly, it can target Apple's ObjcC runtime. To do that, we'll use the [objc crate](https://github.com/SSheldon/rust-objc), which offers some convenient wrappers around the runtime functions:
```rust
extern crate objc; // remember to add it to your Cargo.toml!

use objc::declare::ClassDecl;
use objc::runtime::{Object, Sel, BOOL, YES};
use objc::{class, msg_send, sel, sel_impl};
use std::os::raw::c_char;
use std::ptr;

#[repr(C)]
struct Frame(pub f64, pub f64, pub f64, pub f64);

extern "C" fn did_finish_launching_with_options(
    obj: &mut Object,
    _: Sel,
    _: *mut Object,
    _: *mut Object,
) -> BOOL {
    unsafe {
        let frame: *mut Object = msg_send![class!(UIScreen), mainScreen];
        let frame: Frame = msg_send![frame, bounds];
        let win: *mut Object = msg_send![class!(UIWindow), alloc];
        let win: *mut Object = msg_send![win, initWithFrame: frame];
        let vc: *mut Object = msg_send![class!(ViewController), new];
        let _: () = msg_send![win, setRootViewController: vc];
        let _: () = msg_send![win, makeKeyAndVisible];
        let white: *mut Object = msg_send![class!(UIColor), whiteColor];
        let _: () = msg_send![win, setBackgroundColor: white];
        obj.set_ivar("window", win as usize);
    }
    YES
}

extern "C" fn did_load(obj: &mut Object, _: Sel) {
    unsafe {
        let _: () = msg_send![super(obj, class!(UIViewController)), viewDidLoad];
        let view: *mut Object = msg_send![&*obj, view];
        let btn: *mut Object = msg_send![class!(UIButton), buttonWithType:0];
        let _: () = msg_send![btn, setFrame:Frame(100., 100., 80., 30.)];
        let title: *mut Object = msg_send![class!(NSString), stringWithUTF8String:"Click\0".as_ptr()];
        let _: () = msg_send![btn, setTitle:title forState:0];
        let blue: *mut Object = msg_send![class!(UIColor), blueColor];
        let _: () = msg_send![btn, setTitleColor:blue forState:0];
        let _: () = msg_send![btn, addTarget:obj action:sel!(clicked) forControlEvents:1<<13];
        let _: () = msg_send![view, addSubview: btn];
    }
}

extern "C" fn clicked(_obj: &mut Object, _: Sel) {
    println!("clicked");
}

fn main() {
    unsafe {
        let ui_responder_cls = class!(UIResponder);
        let mut app_delegate_cls = ClassDecl::new("AppDelegate", ui_responder_cls).unwrap();

        app_delegate_cls.add_method(
            sel!(application:didFinishLaunchingWithOptions:),
            did_finish_launching_with_options
                as extern "C" fn(&mut Object, Sel, *mut Object, *mut Object) -> BOOL,
        );

        app_delegate_cls.add_ivar::<usize>("window");

        app_delegate_cls.register();

        let ui_view_controller_cls = class!(UIViewController);
        let mut view_controller_cls =
            ClassDecl::new("ViewController", ui_view_controller_cls).unwrap();

        view_controller_cls.add_method(
            sel!(viewDidLoad),
            did_load as extern "C" fn(&mut Object, Sel),
        );

        view_controller_cls.add_method(sel!(clicked), clicked as extern "C" fn(&mut Object, Sel));

        view_controller_cls.register();

        let name: *mut Object =
            msg_send![class!(NSString), stringWithUTF8String:"AppDelegate\0".as_ptr()];

        extern "C" {
            fn UIApplicationMain(
                argc: i32,
                argv: *mut *mut c_char,
                principalClass: *mut Object,
                delegateName: *mut Object,
            ) -> i32;
        }

        let autoreleasepool: *mut Object = msg_send![class!(NSAutoreleasePool), new];
        // Anything needing the autoreleasepool
        let _: () = msg_send![autoreleasepool, drain];

        UIApplicationMain(0, ptr::null_mut(), ptr::null_mut(), name);
    }
}
```
This can be built with cargo:
```
cargo build --target=x86_64-apple-ios # targetting a simulator
# you can move the generated binary to a prepared bundle folder with an appropriate info.plist
```
Similarly, you can use cargo-bundle, and define bundle metadata in your Cargo.toml:
```toml
[package.metadata.bundle]
name = "myapp"
identifier = "com.neurosrg.myapp"
category = "Education"
short_description = "A pure rust app"
long_description = "A pure rust app"
```
And with cargo-bundle:
```
cargo bundle --target x86_64-apple-ios
xcrun simctl install booted target/x86_64-apple-ios/debug/bundle/ios/pure.app
```
A nice thing about cargo-bundle, even though its iOS bundle support is experimental, is that it's still faster than xcodebuild!

Back to our Rust example, it's not as verbose as the pure C/C++ version, this is thanks to the [objc](https://github.com/SSheldon/rust-objc) crate doing a lot of the heavy lifting such as encoding and other niceties, it requires however explicit types when using the msg_send! macro. Also returning structs works with msg_send, whereas in C/C++, you'd want to use objc_msgSend_stret(). Although you don't see a lot of quotes like in the C++ version, it's still a stringly-typed api, also meaning a wrong typo won't be caught at compile time, instead it'll throw a runtime exception. One downside is that since Rust isn't an officially-supported language by Apple, projects like the objc crate and others wrapping other Apple frameworks are made by members of the Rust community, and can fall into issues like lack of maintainance (which appears to have happened to the objc crate).

## Conclusion
C/C++/Rust can easily target the Objective-C runtime, less so when it comes to targetting Swift, but that's not as important. C/C++ have an extra advantage in that they're officially supported by Apple, for example you can create an XCode project and create both C/C++ source files and headers, and you'll get automatic integration in the build in addition to code completion etc. The system compiler on Apple is clang (AppleClang) which is a C/C++/ObjC/ObjCpp compiler. The default buildsystem, xcodebuild, supports creating universal binaries out of the box, so does CMake, the de-facto C++ buildsystem (which is actually unsupported by XCode, however it can generate xcodeproj files). Rust comes with its own buildsystem/package manager, Cargo. Although Cargo is great, like Rust, it's not directly supported in XCode. Also as the time of writing this, it can't generate MacOS or iOS bundles, nor can it produce universal binaries. Luckily, you can use other packages like cargo-bundle and cargo-lipo to create your bundles and universal libraries. Using the ObjC runtime functions like objc_msgSend/msg_send, apart from allow a develper to write in their preferred programming language, add no advantage whatsoever to the codebase. When it comes to api, it's exceedingly verbose, it's stringly-typed (like the JNI), most of it is unsafe to use (when it comes to Rust) that it's just more convenient to wrap it all in `unsafe`. Essentially, writing Objective-C/C++ can be the least painful path.  