# Migrating to SDL 3.0

This guide provides useful information for migrating applications from SDL 2.0 to SDL 3.0.

We have provided a handy Python script to automate some of this work for you [link to script], and details on the changes are organized by SDL 2.0 header below.

SDL headers should now be included as `#include <SDL3/SDL.h>`. Typically that's the only header you'll need in your application unless you are using OpenGL or Vulkan functionality.

CMake users should use this snippet to include SDL support in their project:
```
find_package(SDL3 REQUIRED CONFIG REQUIRED COMPONENTS SDL3)
target_link_libraries(mygame PRIVATE SDL3::SDL3)
```

Autotools users should use this snippet to include SDL support in their project:
```
PKG_CHECK_MODULES([SDL3], [sdl3])
```
and then add $SDL3_CFLAGS to their project CFLAGS and $SDL3_LIBS to their project LDFLAGS

Makefile users can use this snippet to include SDL support in their project:
```
CFLAGS += $(shell pkg-config sdl3 --cflags)
LDFLAGS += $(shell pkg-config sdl3 --libs)
```

The SDL3test library has been renamed SDL3_test.

There is no static SDLmain library anymore, it's now header-only, see below in the SDL_main.h section.


begin_code.h and close_code.h in the public headers have been renamed to SDL_begin_code.h and SDL_close_code.h. These aren't meant to be included directly by applications, but if your application did, please update your `#include` lines.


## SDL_cpuinfo.h

The following headers are no longer automatically included, and will need to be included manually:
- immintrin.h
- mm3dnow.h
- mmintrin.h
- xmmintrin.h
- emmintrin.h
- pmmintrin.h
- arm_neon.h
- arm64_neon.h
- armintr.h
- arm64intr.h
- altivec.h
- lsxintrin.h
- lasxintrin.h

## SDL_events.h

The `timestamp` member of the `SDL_Event` structure now represents nanoseconds, and is populated with `SDL_GetTicksNS()`

The `timestamp_us` member of the sensor events has been renamed `sensor_timestamp` and now represents nanoseconds. This value is filled in from the hardware, if available, and may not be synchronized with values returned from `SDL_GetTicksNS()`.

You should set the `event.common.timestamp` field before passing an event to `SDL_PushEvent()`. If the timestamp is 0 it will be filled in with `SDL_GetTicksNS()`.

`SDL_GetEventState` used to be a macro, now it's a real function, but otherwise functions identically.

The `SDL_DISPLAYEVENT_*` events have been moved to top level events, and `SDL_DISPLAYEVENT` has been removed. In general, handling this change just means checking for the individual events instead of first checking for `SDL_DISPLAYEVENT` and then checking for display events. You can compare the event >= `SDL_DISPLAYEVENT_FIRST` and <= `SDL_DISPLAYEVENT_LAST` if you need to see whether it's a display event.

The `SDL_WINDOWEVENT_*` events have been moved to top level events, and `SDL_WINDOWEVENT` has been removed. In general, handling this change just means checking for the individual events instead of first checking for `SDL_WINDOWEVENT` and then checking for window events. You can compare the event >= `SDL_WINDOWEVENT_FIRST` and <= `SDL_WINDOWEVENT_LAST` if you need to see whether it's a window event.


## SDL_gamecontroller.h

Removed SDL_GameControllerGetSensorDataWithTimestamp(), if you want timestamps for the sensor data, you should use the sensor_timestamp member of SDL_CONTROLLERSENSORUPDATE events.

## SDL_gesture.h

The gesture API has been removed. There is no replacement planned in SDL3.
However, the SDL2 code has been moved to a single-header library that can
be dropped into an SDL3 or SDL2 program, to continue to provide this
functionality to your app and aid migration. That is located in the
[SDL_gesture GitHub repository](https://github.com/libsdl-org/SDL_gesture).

## SDL_hints.h

The following hints have been removed:
* SDL_HINT_IDLE_TIMER_DISABLED (use SDL_DisableScreenSaver instead)
* SDL_HINT_VIDEO_X11_FORCE_EGL (use SDL_HINT_VIDEO_FORCE_EGL instead)
* SDL_HINT_VIDEO_X11_XINERAMA (Xinerama no longer supported by the X11 backend)
* SDL_HINT_VIDEO_X11_XVIDMODE (Xvidmode no longer supported by the X11 backend)

* Renamed hints 'SDL_HINT_VIDEODRIVER' and 'SDL_HINT_AUDIODRIVER' to 'SDL_HINT_VIDEO_DRIVER' and 'SDL_HINT_AUDIO_DRIVER'
* Renamed environment variables 'SDL_VIDEODRIVER' and 'SDL_AUDIODRIVER' to 'SDL_VIDEO_DRIVER' and 'SDL_AUDIO_DRIVER'

## SDL_main.h

SDL3 doesn't have a static libSDLmain to link against anymore.  
Instead SDL_main.h is now a header-only library **and not included by SDL.h anymore**.

Using it is really simple: Just `#include <SDL3/SDL_main.h>` in the source file with your standard
`int main(int argc, char* argv[])` function.

The rest happens automatically: If your target platform needs the SDL_main functionality,
your `main` function will be renamed to `SDL_main` (with a macro, just like in SDL2),
and the real main-function will be implemented by inline code from SDL_main.h - and if your target
platform doesn't need it, nothing happens.  
Like in SDL2, if you want to handle the platform-specific main yourself instead of using the SDL_main magic,
you can `#define SDL_MAIN_HANDLED` before `#include <SDL3/SDL_main.h>` - don't forget to call `SDL_SetMainReady()`!

If you need SDL_main.h in another source file (that doesn't implement main()), you also need to
`#define SDL_MAIN_HANDLED` there, to avoid that multiple main functions are generated by SDL_main.h

There is currently one platform where this approach doesn't always work: WinRT.  
It requires WinMain to be implemented in a C++ source file that's compiled with `/ZW`. If your main
is implemented in plain C, or you can't use `/ZW` on that file, you can add another .cpp
source file that just contains `#include <SDL3/SDL_main.h>` and compile that with `/ZW` - but keep
in mind that the source file with your standard main also needs that include!
See [README-winrt.md](./README-winrt.md) for more details.

Furthermore, the different `SDL_*RunApp()` functions (SDL_WinRtRunApp, SDL_GDKRunApp, SDL_UIKitRunApp)
have been unified into just `int SDL_RunApp(int argc, char* argv[], void * reserved)` (which is also
used by additional platforms that didn't have a SDL_RunApp-like function before).

## SDL_platform.h

The preprocessor symbol __MACOSX__ has been renamed __MACOS__, and __IPHONEOS__ has been renamed __IOS__

## SDL_pixels.h

SDL_CalculateGammaRamp has been removed, because SDL_SetWindowGammaRamp has been removed as well due to poor support in modern operating systems (see [SDL_video.h](#sdl_videoh)).

## SDL_render.h

SDL_GetRenderDriverInfo() has been removed, since most of the information it reported were
estimates and could not be accurate before creating a renderer. Often times this function
was used to figure out the index of a driver, so one would call it in a for-loop, looking
for the driver named "opengl" or whatnot. SDL_GetRenderDriver() has been added for this
functionality, which returns only the name of the driver.

Additionally, SDL_CreateRenderer()'s second argument is no longer an integer index, but a
`const char *` representing a renderer's name; if you were just using a for-loop to find
which index is the "opengl" or whatnot driver, you can just pass that string directly
here, now. Passing NULL is the same as passing -1 here in SDL2, to signify you want SDL
to decide for you.

## SDL_rwops.h

SDL_RWread and SDL_RWwrite (and SDL_RWops::read, SDL_RWops::write) have a different function signature in SDL3.

Previously they looked more like stdio:

```c
size_t SDL_RWread(SDL_RWops *context, void *ptr, size_t size, size_t maxnum);
```

But now they look more like POSIX:

```c
Sint64 SDL_RWread(SDL_RWops *context, void *ptr, Sint64 size);
```

Previously they tried to read/write `size` objects of `maxnum` bytes each. Now they try to read/write `size` bytes, which solves
concerns about what should happen to the file pointer if only a fraction of an object could be read, etc. The return value is
different, too. For reading:

- SDL_RWread returns the number of bytes read, which will be less than requested on error or EOF.
- If there was an error but some bytes were read, it will return the number of bytes read.
- On error when no bytes were read, it returns -1.
- For non-blocking RWops (new to SDL3!), if we are neither at an error or EOF but it would require blocking to read more data, it returns -2.

For writing:

- SDL_RWwrite returns the number of bytes written, which might be less on error or if the RWops is non-blocking.
- If there was an error but some bytes were written, it will return the number of bytes written.
- On error when no bytes were written, it returns -1.
- For non-blocking RWops (new to SDL3!), if we are not at an error but it would require blocking to write more data, it returns -2.


As you can see, RWops can now be non-blocking! There is no API in SDL to
toggle a RWops to (non-)blocking mode, they must be created as such. The
existing SDL_RWFrom* functions do not create non-blocking objects, so existing
code (and much of the code you would care to write by default) does not have
to contend with this behavior.


SDL_RWFromFP has been removed from the API, due to issues when the SDL library uses a different C runtime from the application.

You can implement this in your own code easily:
```c
#include <stdio.h>


static Sint64 SDLCALL
stdio_size(SDL_RWops * context)
{
    Sint64 pos, size;

    pos = SDL_RWseek(context, 0, RW_SEEK_CUR);
    if (pos < 0) {
        return -1;
    }
    size = SDL_RWseek(context, 0, RW_SEEK_END);

    SDL_RWseek(context, pos, RW_SEEK_SET);
    return size;
}

static Sint64 SDLCALL
stdio_seek(SDL_RWops * context, Sint64 offset, int whence)
{
    int stdiowhence;

    switch (whence) {
    case RW_SEEK_SET:
        stdiowhence = SEEK_SET;
        break;
    case RW_SEEK_CUR:
        stdiowhence = SEEK_CUR;
        break;
    case RW_SEEK_END:
        stdiowhence = SEEK_END;
        break;
    default:
        return SDL_SetError("Unknown value for 'whence'");
    }

    if (fseek((FILE *)context->hidden.stdio.fp, (fseek_off_t)offset, stdiowhence) == 0) {
        Sint64 pos = ftell((FILE *)context->hidden.stdio.fp);
        if (pos < 0) {
            return SDL_SetError("Couldn't get stream offset");
        }
        return pos;
    }
    return SDL_Error(SDL_EFSEEK);
}

static Sint64 SDLCALL
stdio_read(SDL_RWops * context, void *ptr, Sint64 size)
{
    size_t nread;

    nread = fread(ptr, 1, (size_t) size, (FILE *)context->hidden.stdio.fp);
    if (nread == 0 && ferror((FILE *)context->hidden.stdio.fp)) {
        return SDL_Error(SDL_EFREAD);
    }
    return (Sint64) nread;
}

static Sint64 SDLCALL
stdio_write(SDL_RWops * context, const void *ptr, Sint64 size)
{
    size_t nwrote;

    nwrote = fwrite(ptr, 1, (size_t) size, (FILE *)context->hidden.stdio.fp);
    if (nwrote == 0 && ferror((FILE *)context->hidden.stdio.fp)) {
        return SDL_Error(SDL_EFWRITE);
    }
    return (Sint64) nwrote;
}

static int SDLCALL
stdio_close(SDL_RWops * context)
{
    int status = 0;
    if (context) {
        if (context->hidden.stdio.autoclose) {
            /* WARNING:  Check the return value here! */
            if (fclose((FILE *)context->hidden.stdio.fp) != 0) {
                status = SDL_Error(SDL_EFWRITE);
            }
        }
        SDL_FreeRW(context);
    }
    return status;
}

SDL_RWops *
SDL_RWFromFP(void *fp, SDL_bool autoclose)
{
    SDL_RWops *rwops = NULL;

    rwops = SDL_AllocRW();
    if (rwops != NULL) {
        rwops->size = stdio_size;
        rwops->seek = stdio_seek;
        rwops->read = stdio_read;
        rwops->write = stdio_write;
        rwops->close = stdio_close;
        rwops->hidden.stdio.fp = fp;
        rwops->hidden.stdio.autoclose = autoclose;
        rwops->type = SDL_RWOPS_STDFILE;
    }
    return rwops;
}
```


## SDL_sensor.h

Removed SDL_SensorGetDataWithTimestamp(), if you want timestamps for the sensor data, you should use the sensor_timestamp member of SDL_SENSORUPDATE events.


## SDL_stdinc.h

The standard C headers like stdio.h and stdlib.h are no longer included, you should include them directly in your project if you use non-SDL C runtime functions.
M_PI is no longer defined in SDL_stdinc.h, you can use the new symbols SDL_PI_D (double) and SDL_PI_F (float) instead.


## SDL_surface.h

Removed unused 'flags' parameter from SDL_ConvertSurface and SDL_ConvertSurfaceFormat.

SDL_CreateRGBSurface() and SDL_CreateRGBSurfaceWithFormat() have been combined into a new function SDL_CreateSurface().
SDL_CreateRGBSurfaceFrom() and SDL_CreateRGBSurfaceWithFormatFrom() have been combined into a new function SDL_CreateSurfaceFrom().

You can implement the old functions in your own code easily:
```c
SDL_Surface *SDL_CreateRGBSurface(Uint32 flags, int width, int height, int depth, Uint32 Rmask, Uint32 Gmask, Uint32 Bmask, Uint32 Amask)
{
    return SDL_CreateSurface(width, height,
            SDL_MasksToPixelFormatEnum(depth, Rmask, Gmask, Bmask, Amask));
}

SDL_Surface *SDL_CreateRGBSurfaceWithFormat(Uint32 flags, int width, int height, int depth, Uint32 format)
{
    return SDL_CreateSurface(width, height, format);
}

SDL_Surface *SDL_CreateRGBSurfaceFrom(void *pixels, int width, int height, int depth, int pitch, Uint32 Rmask, Uint32 Gmask, Uint32 Bmask, Uint32 Amask)
{
    return SDL_CreateSurfaceFrom(pixels, width, height, pitch,
            SDL_MasksToPixelFormatEnum(depth, Rmask, Gmask, Bmask, Amask));
}

SDL_Surface *SDL_CreateRGBSurfaceWithFormatFrom(void *pixels, int width, int height, int depth, int pitch, Uint32 format)
{
    return SDL_CreateSurfaceFrom(pixels, width, height, pitch, format);
}

```

But if you're migrating your code which uses masks, you probably have a format in mind, possibly one of these:
```c
// Various mask (R, G, B, A) and their corresponding format:
0xFF000000 0x00FF0000 0x0000FF00 0x000000FF => SDL_PIXELFORMAT_RGBA8888
0x00FF0000 0x0000FF00 0x000000FF 0xFF000000 => SDL_PIXELFORMAT_ARGB8888
0x0000FF00 0x00FF0000 0xFF000000 0x000000FF => SDL_PIXELFORMAT_BGRA8888
0x000000FF 0x0000FF00 0x00FF0000 0xFF000000 => SDL_PIXELFORMAT_ABGR8888
0x0000F800 0x000007E0 0x0000001F 0x00000000 => SDL_PIXELFORMAT_RGB565
```


## SDL_syswm.h

This header no longer includes platform specific headers and type definitions, instead allowing you to include the ones appropriate for your use case. You should define one or more of the following to enable the relevant platform-specific support:
* SDL_ENABLE_SYSWM_ANDROID
* SDL_ENABLE_SYSWM_COCOA
* SDL_ENABLE_SYSWM_KMSDRM
* SDL_ENABLE_SYSWM_UIKIT
* SDL_ENABLE_SYSWM_VIVANTE
* SDL_ENABLE_SYSWM_WAYLAND
* SDL_ENABLE_SYSWM_WINDOWS
* SDL_ENABLE_SYSWM_X11

The structures in this file are versioned separately from the rest of SDL, allowing better backwards compatibility and limited forwards compatibility with your application. Instead of calling `SDL_VERSION(&info.version)` before calling SDL_GetWindowWMInfo(), you pass the version in explicitly as `SDL_SYSWM_CURRENT_VERSION` so SDL knows what fields you expect to be filled out.

### SDL_GetWindowWMInfo

This function now returns a standard int result instead of SDL_bool, returning 0 if the function succeeds or a negative error code if there was an error. You should also pass `SDL_SYSWM_CURRENT_VERSION` as the new third version parameter. The version member of the info structure will be filled in with the version of data that is returned, the minimum of the version you requested and the version supported by the runtime SDL library.


## SDL_timer.h

SDL_GetTicks() now returns a 64-bit value. Instead of using the `SDL_TICKS_PASSED` macro, you can directly compare tick values, e.g.
```c
Uint32 deadline = SDL_GetTicks() + 1000;
...
if (SDL_TICKS_PASSED(SDL_GetTicks(), deadline)) {
    ...
}
```
becomes:
```c
Uint64 deadline = SDL_GetTicks() + 1000
...
if (SDL_GetTicks() >= deadline) {
    ...
}
```

If you were using this macro for other things besides SDL ticks values, you can define it in your own code as:
```c
#define SDL_TICKS_PASSED(A, B)  ((Sint32)((B) - (A)) <= 0)
```

## SDL_version.h

SDL_GetRevisionNumber() has been removed from the API, it always returned 0 in SDL 2.0


## SDL_video.h

SDL_SetWindowBrightness and SDL_SetWindowGammaRamp have been removed from the API, because they interact poorly with modern operating systems and aren't able to limit their effects to the SDL window.

Programs which have access to shaders can implement more robust versions of those functions using custom shader code rendered as a post-process effect.


Removed 'SDL_GL_CONTEXT_EGL' from OpenGL configuration attributes
You can instead use 'SDL_GL_SetAttribute(SDL_GL_CONTEXT_PROFILE_MASK, SDL_GL_CONTEXT_PROFILE_ES);'

