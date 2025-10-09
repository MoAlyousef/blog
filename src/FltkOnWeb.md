# FLTK on the Web
<br>
Date: 2025-10-09
<br>

This post is about getting FLTK to build for wasm and run in browsers!
Source code can be found in [a fork of FLTK](https://github.com/MoAlyousef/fltk_wasm32_emscripten) under the emscripten branch.

If you try to `git clone` stock FLTK and build it using the [Emscripten toolchain](https://emscripten.org/), you'll be met with a lot of errors. The first issue is FLTK's CMakeLists.txt. The build process conditionally includes the source files required for the target platform. In FLTK's case, when the build doesn't know what it's building for, it assumes Windows. So it will incorporate the Window's driver files which has win32 calls unidentified by the compiler, and the build fails.

## Step 1: CMake
Extend the build to identify Emscripten as a platform, luckily CMake knows `Emscripten`, so that's as simple as adding:
```cmake
elseif(WIN32)
# add window's driver files
elseif(EMSCRIPTEN)
# empty for now
endif()
```

Some extra changes were also need in the CMake scripts, since apparently `if(UNIX)` in CMake returns true on Emscripten.
I also had to provide a definition for `Window` which points to the underlying handle.
For example on Windows it's an HWND, on macOS it's an NSWindow, on X11 it's a Window. I settled on a `typedef int Window` in FL/platform.H.
This makes the configure step and compilation to succeed. You can silence linker errors by passing the Emscripten shell option `-sERROR_ON_UNDEFINED_SYMBOLS=0`.
Actually trying to get anything to show in the browser won't work since we now need to plug the windowing, drawing and event handling code to something browsers understand.

## Step 2: Driver stubs
I had considered creating a null driver which can be used as a baseline for extra drivers to work. I've also suggested it in the fltk mailing list:
https://groups.google.com/g/fltkcoredev/c/lXqDv3BUW9Q/m/eAJWBVisCQAJ

The issue however was that I found it was necessary to modify FLTK sources outside of drivers to support Emscripten.
So even though I still find that it would be useful eventually if FLTK had a null driver which allows third party developers to create external drivers/backends, it would require more work than what I had set out to do.

So I shifted my focus to adding an Emscripten driver for FLTK.
There are essential classes which are instantiated by FLTK which will handle windowing and graphics:
```c++
Fl_Screen_Driver *Fl_Screen_Driver::newScreenDriver() {
  return new Fl_Emscripten_Screen_Driver();
}

Fl_System_Driver *Fl_System_Driver::newSystemDriver() {
  return new Fl_Emscripten_System_Driver();
}

Fl_Image_Surface_Driver *Fl_Image_Surface_Driver::newImageSurfaceDriver(int w, int h, int highres,
                                                                        Fl_Offscreen off) {
  return new Fl_Emscripten_Image_Surface_Driver(w, h, highres, off);
}

Fl_Graphics_Driver *Fl_Graphics_Driver::newMainGraphicsDriver() {
  return new Fl_Emscripten_Graphics_Driver();
}

Fl_Window_Driver *Fl_Window_Driver::newWindowDriver(Fl_Window *w) {
  return new Fl_Emscripten_Window_Driver(w);
}

Fl_Copy_Surface_Driver *Fl_Copy_Surface_Driver::newCopySurfaceDriver(int w, int h) {
  return new Fl_Emscripten_Copy_Surface_Driver(w, h);
}
```
Basically you have to provide definitions to these class methods (declared in the respective headers). You do that by subclassing each driver and returning an instance of the subclass.

The classes also provide method declarations which need to be defined in the driver sources. Some are already defined in core FLTK, so you can rely on the default implementation or override them when necessary. So there were a lot of stubs in the beginning.

This step will actually build and link correctly, even though it wouldn't show anything in the browser.

## Step 3: Actually deciding what's a Window
I knew from the beginning that I was going to use the canvas for drawing. But the question of "what's a window" in the browser had me stumped for some time. I decided that an FLTK window should map to an HTMLDivElement.
It should contain decorations (borders with a title bar and at least a close button). The decorations will be another `div`, and the client area would be the canvas. The window's handle (the int from before), would be incorporated into the div element's id. In essence my `Fl_Emscripten_Window_Driver::makeWindow()` method would contain this abomination:
```cpp
  EM_ASM(
      {
        let body = document.getElementsByTagName("body")[0];
        let div = document.createElement("DIV");
        div.id = "fltk_div" + $0;
        div.tabIndex = "-1";
        div.addEventListener("contextmenu", (e) => e.preventDefault());
        div.style.position = "absolute";
        div.style.left = $2 + "px";
        div.style.top = $1 ? ($3 - 30) + "px" : $3 + "px";
        div.style.zIndex = 1;
        div.style.backgroundColor = "#f1f1f1";
        div.style.borderRight = "1px solid #555";
        div.style.borderBottom = "1px solid #555";
        div.style.textAlign = "center";
        body.appendChild(div);
        let decor = document.createElement("DIV");
        decor.id = "fltk_decor" + $0;
        decor.style.height = "16px";
        decor.style.font = "14px Arial";
        decor.style.padding = "6px";
        decor.style.cursor = "move";
        decor.style.zIndex = 2;
        decor.style.backgroundColor = "#2196F3";
        decor.style.color = "#fff";
        decor.style.cursor = "pointer";
        div.appendChild(decor);
        let header = document.createElement("DIV");
        header.textContent = UTF8ToString($6);
        header.id = "fltk_decor_header" + $0;
        header.style.font = "14px Arial";
        decor.appendChild(header);
        let close = document.createElement("BUTTON");
        close.id = "closewin";
        close.textContent = "X";
        close.style.font = "bold 14px Arial";
        close.style.position = "absolute";
        close.style.top = "1%";
        close.style.right = "1px";
        close.style.backgroundColor = "#2196F3";
        close.style.border = "none";
        close.style.color = "#fff";
        close.addEventListener("click", () => div.hidden = true);
        decor.appendChild(close);
        let canvas = document.createElement("CANVAS");
        canvas.id = "fltk_canvas" + $0;
        canvas.setAttribute("data-raw-handle", $0.toString());
        canvas.tabIndex = "-1";
        canvas.width = $4;
        canvas.height = $5;
        div.appendChild(canvas);
        canvas.addEventListener("click", () => canvas.focus());
        decor.addEventListener("mousedown", () => canvas.focus());
        div.addEventListener(
            "focusin", () => { canvas.focus(); div.style.zIndex = 1; });
        div.addEventListener(
            "focusout", () => { canvas.blur(); div.style.zIndex = 0; });
        if ($1 === 0) decor.hidden = true;
        // https://www.w3schools.com/HOWTO/howto_js_draggable.asp
        function dragElement(elmnt) {
          var pos1 = 0;
          var pos2 = 0;
          var pos3 = 0;
          var pos4 = 0;
          if (document.getElementById("fltk_decor" + $0)) {
            document.getElementById("fltk_decor" + $0).onmousedown = dragMouseDown;
          } else {
            elmnt.onmousedown = dragMouseDown;
          }

          function dragMouseDown(e) {
            e = e || window.event;
            e.preventDefault();
            pos3 = e.clientX;
            pos4 = e.clientY;
            document.onmouseup = closeDragElement;
            document.onmousemove = elementDrag;
          }

          function elementDrag(e) {
            e = e || window.event;
            e.preventDefault();
            pos1 = pos3 - e.clientX;
            pos2 = pos4 - e.clientY;
            pos3 = e.clientX;
            pos4 = e.clientY;
            elmnt.style.left = (elmnt.offsetLeft - pos1) + "px";
            elmnt.style.top = (elmnt.offsetTop - pos2) + "px";
            _fltk_em_track_div($0, elmnt.offsetLeft|0, elmnt.offsetTop|0);
          }

          function closeDragElement() {
            document.onmouseup = null;
            document.onmousemove = null;
          }
        }

        dragElement(document.getElementById("fltk_div" + $0));
      },
      ID, pWindow->border(), pWindow->x(), pWindow->y(), 
      pWindow->w(), pWindow->h(), pWindow->label() ? pWindow->label(): "");
```
The above code should allow:
- Allow setting a title on the title bar.
- Dragging the div across the screen from its title bar, like you would any window on your desktop.
- Allow Fl_Window::set_border(false) to hide the title bar.

There's a downside to this approach however. FLTK allows embedding windows, while in the browser, you can't embed another canvas or div inside a canvas! So that choice is still questionable!

Plugging the rest of the Fl_Window_Driver methods and wiring them to emscripten methods was quite easy. Mapping FLTK events to browser events, FLTK cursor types to browser cursor types, FLTK fonts to browser fonts was some busy work but not that difficult.
Events required forwarding via `Fl::handle` to the window/div in which they occured.

An intrusive change to the FLTK sources outside of driver code:
```cpp
int Fl::run() {
#ifndef __EMSCRIPTEN__
  while (Fl_X::first) wait(FOREVER);
#else
  emscripten_set_main_loop([]() { Fl::wait(); }, 0, true);
#endif
  return 0;
  return 0;
}
```
Browser environments operate on an event-driven model. An infinite while loop, which is common in native applications, would block the browser's main thread, causing the page to become unresponsive. 

## Step 4: What's a screen
Browsers offer a globalThis.screen, and by default a browser page has only one screen. so that might be a no brainer!
However FLTK needs to know in which confines it's working. Some panels, the taskbar and browser menu actually part of the screen.
I decided to go with availWidth and availHeight:
```cpp
using namespace emscripten;
int Fl_Emscripten_Screen_Driver::w() {
  val screen = val::global("screen");
  return screen["availWidth"].as<int>();
}
int Fl_Emscripten_Screen_Driver::h() {
  val screen = val::global("screen");
  return screen["availHeight"].as<int>();
}
```
Other screen driver methods on the FLTK side included getting mouse coordinates, handling text composition since you need to translate key presses inside text-accepting widgets into actual text, handling any special keys pressed with character keys. 
The screen driver also covers handling cut/copy/paste which required me venturing into some new territories.
I never had to deal with the browser's clipboard for example. Getting the clipboard text for example requires:
```cpp
std::string clipText =
          emscripten::val::global("navigator")["clipboard"].call<val>("readText").await().as<std::string>();
```
Writing to the clipboard is also done via navigator.clipboard.writeText.

Similarly getting an image from the clipboard isn't trivial either:
```cpp
EM_ASYNC_JS(EM_VAL, get_clipboard_image, (), {
  const itemList = await navigator.clipboard.read();
  let imageType;
  const item = itemList.find(item => item.types.some(type => type.startsWith('image/')));
  if (item) {
    const imageBlob = await item.getType(imageType = item.types.find(type => type.startsWith('image/')));
    const imageBitmap = await createImageBitmap(imageBlob);
    const canvas = new OffscreenCanvas(imageBitmap.width, imageBitmap.height);
    const context = canvas.getContext('2d');
    context.drawImage(imageBitmap, 0, 0);
    const imageData = context.getImageData(0, 0, canvas.width, canvas.height);
    return imageData;
  } else {
    return null;
  }
});
```

## Step 5: Graphics
FLTK on linux/bsd uses Cairo for drawing (default on wayland, requires FLTK_GRAPHICS_CAIRO on X11).
The browser's canvas api is similar enough to cairo's graphics api, so that was very helpful in translating graphics calls to canvas calls. For example compare the following:
```cpp
void Fl_Cairo_Graphics_Driver::loop(int x0, int y0, int x1, int y1, int x2, int y2) {
  cairo_save(cairo_);
  cairo_new_path(cairo_);
  cairo_move_to(cairo_, x0, y0);
  cairo_line_to(cairo_, x1, y1);
  cairo_line_to(cairo_, x2, y2);
  cairo_close_path(cairo_);
  cairo_stroke(cairo_);
  cairo_restore(cairo_);
  surface_needs_commit();
}
```
to:
```cpp
void Fl_Emscripten_Graphics_Driver::loop(int x0, int y0, int x1, int y1, int x2, int y2) {
  EM_ASM(
      {
        let ctx = Emval.toValue($0);
        ctx.save();
        ctx.beginPath();
        ctx.moveTo($1, $2);
        ctx.lineTo($3, $4);
        ctx.lineTo($5, $6);
        ctx.closePath();
        ctx.stroke();
        ctx.restore();
      },
      ctxt, x0, y0, x1, y1, x2, y2);
}
```
Drawing images was tricky since FLTK supports L8, LA8, RGB8 and RGBA8 formats, so FLTK images needed conversion to an RGBA8 format before sending them to the canvas for drawing.

Building this would finally show a window with whatever widget you put in it. The implementation was buggy from some mismatches in event handling but that was easily worked out since there was something I can actually test and see!

Adding offscreen support, FLTK supports offscreen drawing via Fl_Offscreen and the Image Surface driver, and browsers support an Offscreen canvas. This requires enabling the Emscripten flag OFFSCREENCANVAS_SUPPORT. Which if enabled, should work out of the box.

## Step 6: Reworking events
It's not enough to translate a browser event to an FLTK event:
```cpp
static int match_mouse_event(int eventType) {
  switch (eventType) {
    case EMSCRIPTEN_EVENT_MOUSEDOWN:
      return FL_PUSH;
    case EMSCRIPTEN_EVENT_MOUSEUP:
      return FL_RELEASE;
    case EMSCRIPTEN_EVENT_MOUSEMOVE:
      return FL_MOVE;
    case EMSCRIPTEN_EVENT_MOUSEENTER:
      return FL_ENTER;
    case EMSCRIPTEN_EVENT_MOUSELEAVE:
      return FL_LEAVE;
    case EMSCRIPTEN_EVENT_DBLCLICK:
      return FL_PUSH;
    case EMSCRIPTEN_EVENT_CLICK:
      return FL_PUSH;
    default:
      return 0;
  }
}
```

You would also need to handle state:
```cpp
int flev = match_mouse_event(eventType);
  if (flev == FL_PUSH) {
    if (eventType == EMSCRIPTEN_EVENT_DBLCLICK)
      Fl::e_clicks = 1;
    else
      Fl::e_clicks = 0;
    Fl::e_is_click = 1;
    px = Fl::e_x_root;
    py = Fl::e_y_root;
    if (event->button == 0)
      state |= FL_BUTTON1;
    if (event->button == 1)
      state |= FL_BUTTON2;
    if (event->button == 2)
      state |= FL_BUTTON3;
    Fl::e_keysym = FL_Button + event->button + 1;
// same for other mouse events
```
Same for key presses! 
It also turns out that you have to unregister events from closed windows (divs)!
This step was probably the least interesting to work on and took a disproportionatly long time.

## Step 7: Font support
By default, FLTK has a set of 15 fonts which it supports automatically. Luckily these can be mapped to web-safe fonts which are available on all browsers. Some browsers (chrome and related) additionally provide a queryLocalFonts method which allows you to set a font from the host system:
```cpp
  bool has_query_fonts_api = EM_ASM_INT({
    let has_api = false;
    if ("queryLocalFonts" in window) {
      navigator.permissions.query({ name: "local-fonts" }).then((result) => {
      if (result.state === "granted" || result.state === "prompt") {
        has_api = true;
      }});
    }
    return has_api;
  });
  // clang-format on
  if (!has_query_fonts_api) {
    Fl::set_font((Fl_Font)(FL_FREE_FONT + 1), name);
    built_in_table.push_back({name});
    return FL_FREE_FONT + 1;
  } else {
    val window = val::global("window");
    val availablefonts = window.call<val>("queryLocalFonts").await();
    std::vector<val> vec = vecFromJSArray<val>(availablefonts);
    int count = 0;
    for (const val &font : vec) {
      std::string familyname0 = font["family"].as<std::string>();
      int lfont = familyname0.size() + 2;
      const char *familyname = familyname0.c_str();
      char *fname = new char[lfont];
      snprintf(fname, lfont, " %s", familyname);
      char *regular = strdup(fname);
      Fl::set_font((Fl_Font)(count++ + FL_FREE_FONT), regular);
      built_in_table.push_back({regular});

      snprintf(fname, lfont, "B%s", familyname);
      char *bold = strdup(fname);
      Fl::set_font((Fl_Font)(count++ + FL_FREE_FONT), bold);
      built_in_table.push_back({bold});

      snprintf(fname, lfont, "I%s", familyname);
      char *italic = strdup(fname);
      Fl::set_font((Fl_Font)(count++ + FL_FREE_FONT), italic);
      built_in_table.push_back({italic});

      snprintf(fname, lfont, "P%s", familyname);
      char *bi = strdup(fname);
      Fl::set_font((Fl_Font)(count++ + FL_FREE_FONT), bi);
      // The returned fonts are already sorted.
      built_in_table.push_back({bi});
      delete[] fname;
    }
    return FL_FREE_FONT + count;
  }
```

## Step 8: Supporting file dialogs
FLTK provides an Fl_Native_File_Chooser which wraps the native file picker on all of its backends.
Browsers provide a File Sytem API, and some support extensions which support showOpenFilePicker, showSaveFilePicker and showDirectoryPicker, that's how some sites allow you to open a file dialog and upload a file for example.
This has limited availability unfortunately. Firefox for example doesn't support showing dialogs via the above-mentioned api. You can use an HTMLInputElement and set the type to `file` (equivalent to `<input type="file" />`), but after experimenting with that I found it quite limited. No writable handles, no directory traversal and other minor ones!
Where it's supported, spawning a File dialog from FLTK in the browser should work:
```cpp
// This translates the chooser type to a browser picker. We have 3 main types:
// 1- showOpenFilePicker
// 2- showSaveFilePicker
// 3- showDirectoryPicker
// clang-format off
EM_ASYNC_JS(EM_VAL, showChooser, (int type, const char *filter, const char *dir, const char *preset), {
  if (!window.showOpenFilePicker) {
    return null;
  }
  let multiple = false;
  let files = false;
  let save = false;
  if (type === 0 || type === 2 || type === 4) {
    files = true;
  }
  if (type === 2 || type === 3) {
    mutliple = true;
  }
  if (type > 3) {
    save = true;
  }
  let func;
  if (files) {
    if (save) {
      func = window.showSaveFilePicker;
    } else {
      func = window.showOpenFilePicker;
    }
  } else {
    func = window.showDirectoryPicker;
  }
  let dir1 = dir ? UTF8ToString(dir) : 'desktop';
  let filt = UTF8ToString(filter).split(' ');
  // I use application/x-abiword since I don't think it's widely used as a mime type!
  const openPickerOpts = {
    types: [
      {
        accept: {
          "application/x-abiword": filt,
        },
      },
    ],
    startIn: dir1,
    excludeAcceptAllOption: true,
    multiple: multiple,
  };
  const savePickerOpts = {
    types: [
      {
        accept: {
          "application/x-abiword": filt,
        },
      },
    ],
    suggestedName: UTF8ToString(preset),
    startIn: dir1,
    excludeAcceptAllOption: true,
  };
  const directoryOpts = {
    mode: "readwrite",
    startIn: dir1,
  };
  if (files) {
    return Emval.toHandle(func(save ? savePickerOpts : openPickerOpts));
  } else {
    return Emval.toHandle(func(directoryOpts));
  }
});
```
Accessing the selected files is done via:
- fl_read_to_string
- fl_read_to_binary
- fl_write_to_file

That's because the standard C/C++ file functions work only for Emscripten's virtual filesystem. The above functions utilise browser functions to carry reads and writes:
```cpp
  // writing to a file using the browsers' file system api
  val data1 = val(typed_memory_view(len, data));
  val file = filehandles[idx];
  val writable = file.call<val>("createWritable").await();
  writable.call<val>("write", data1).await();
  writable.call<val>("close").await();
```

## Step 9: Working around limitations
- Mitigating the reentrancy issue
As mentioned previously, since browsers are event-driven, you should avoid long while loops since these would block your page. FLTK's menu windows would loop until something is selected. The blocking can be (partially) mitigated by using `emscripten_sleep` in the ` Fl_Emscripten_System_Driver::wait` method and enabling Asyncify support in Emscripten.
Full mitigation would require removing while loops in menu code but that would be too intrusive!

- Virtual keyboard support for text accepting widgets on mobile
When you use a mobile browser and tap into an HTMLInputElement (input) or a textarea element, you automatically get a virtual keyboard that you can type into. However, an Fl_Input or Fl_Text_Editor aren't the above mentioned elements. They're drawings in the canvas. While on Android you can manually show the keyboard and that would work, on ios it's just not possible.
Some browsers provide a [Virtual Keyboard api](https://developer.mozilla.org/en-US/docs/Web/API/VirtualKeyboard_API), alas safari isn't one of them, nor is WebView on iOS, which is used by chrome and firefox on iOS.
To actually support this, I would have to modify FLTK to make Fl_Input be backed by an input element, and Fl_Text_Editor be backed by a textarea element. The change was too intrusive so I dropped it. If eventually iOS browsers support the Virtual Keyboard API, I might implement this!

## Step 10: Getting emscripten support for fltk-rs
All the work was done in [a fork of FLTK](https://github.com/MoAlyousef/fltk_wasm32_emscripten) under the emscripten branch. Adding support in fltk-rs would require cloning the fork and building it as part of the fltk-rs build. By default we build in single-threaded mode to avoid the extra policy permission required for SharedArrayBuffer. The only changes required apart from the build system is exposing file reads/writes for the File System API due to the same limitation for the standard file operations.

## Conclusion
Overall, I wouldn’t recommend building a web UI that relies heavily on canvas-based rendering for its visuals and widgets. The limitations described in Step 9 are part of this. Additionally this approach sacrifices many of the benefits offered by native HTML, CSS, and JavaScript (accessibility and the sheer amount of man-years put into improving things). Development can also be cumbersome, as even small changes often require a full rebuild. Additionally, binary size becomes a concern—while a simple interface built with standard web technologies might only take a few kilobytes, FLTK wasm builds compiled with Emscripten can easily reach several hundred kilobytes, even with release builds, stripping and lto.
Finally I can say, I learned a lot in the process, and I have 2 nice demos to show:
- C++ demo
https://moalyousef.github.io/fltk_emscripten/
[(source code)](https://moalyousef.github.io/fltk_emscripten/)
- Rust demo
https://moalyousef.github.io/fltk-rs-emscripten-example/
[(source code)](https://github.com/MoAlyousef/fltk-rs-emscripten-example)

Thank you for reading!