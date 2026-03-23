# ONScripter-RU (WebAssembly Fork)

Fork of [ONScripter-RU](https://github.com/umineko-project/onscripter-ru) with modifications to compile and run in the browser via [Emscripten](https://emscripten.org/) and WebAssembly.

This fork powers [Umineko Web](https://github.com/VictoriqueMoe/umineko-web) - the full Umineko no Naku Koro ni (PS3/Umineko Project build) running entirely in a browser tab.

## What This Fork Changes

The upstream engine targets Windows, macOS, Linux, iOS, and Android. It relies on multithreading, synchronous file I/O, and native GPU rendering - none of which translate directly to the browser. This fork adds an `__EMSCRIPTEN__` compile target that replaces threaded subsystems with single-threaded synchronous equivalents, rewires file I/O to fetch assets over HTTP on demand, and adapts the graphics and audio pipelines for WebGL and the Web Audio API.

All changes are gated behind `#ifdef __EMSCRIPTEN__` so the native builds remain unaffected.

## Modified Files

18 source files contain Emscripten-specific changes:

### File I/O and Classic Art Swap

| File                         | Change                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
|------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `Support/FileIO.cpp`         | `EM_ASYNC_JS` hook that intercepts `fopen()` on lazy-loaded files. In local/production mode, fetches the real data over HTTP via `await fetch()`. In remote mode, reads from the user's local game folder via `window.readLocalFile()` (backed by the File System Access API or `<input webkitdirectory>`). Writes the data into the Emscripten virtual filesystem and returns the file handle to the caller. The engine sees synchronous file access regardless of the source. |
| `Support/FileIO.cpp`         | Classic art sprite swap. When `window.classicMode` is true, sprite fetch requests are redirected from `/game/sprites/` to `/classic/sprites/` where pre-processed Ryukishi07 original art is served. Lip overlay (`/2/`) fetches are blocked in classic mode (classic sprites have no lip layers). Includes `classicModeToggle()` exported function that clears the engine's image cache and resets VFS sprite stubs to force re-fetching on mode change.                       |
| `Engine/Core/ONScripter.hpp` | Added public `clearImageCache()` method to expose the private image cache for the classic mode toggle.                                                                                                                                                                                                                                                                                                                                                                          |

### Video Playback

| File                            | Change                                                                                                                                                                                                                                                                                                  |
|---------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `Engine/Media/Controller.cpp`   | `pumpSynchronous()` - replaces the multi-threaded demuxer/decoder pipeline with a single-threaded synchronous loop. Reads packets via `av_read_frame()`, decodes video and audio frames inline, and pushes them into a bounded frame queue (capped at 6 frames to stay within WASM's 2GB memory limit). |
| `Engine/Media/Controller.cpp`   | Subtitle blending added to the synchronous decode path. On native, `applySubtitles()` is called from the decoder thread. On Emscripten, it is called inline after each video frame is decoded.                                                                                                          |
| `Engine/Media/Controller.cpp`   | `finish()` adapted for single-threaded teardown (no threads to join).                                                                                                                                                                                                                                   |
| `Engine/Media/Controller.hpp`   | `pumpSynchronous()` declaration.                                                                                                                                                                                                                                                                        |
| `Engine/Media/VideoDecoder.cpp` | Colour space conversion adjusted - skips `sws_setColorspaceDetails()` tuning that is not needed for browser rendering.                                                                                                                                                                                  |
| `Engine/Layers/Media.cpp`       | Synchronous frame pumping in the media layer's update loop. Calls `pumpSynchronous()` each frame instead of relying on background threads to fill the queue.                                                                                                                                            |

### Subtitle Rendering

| File                         | Change                                                                                                                                                                                                                                                                                                      |
|------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `Engine/Layers/Subtitle.cpp` | `SubtitleLayer` rewritten for single-threaded operation. `startDecoding()` skips mutex/semaphore creation and background thread launch. `update()` calls `subtitleDriver.extractFrame()` directly instead of reading from a thread-fed frame queue. `endDecoding()` skips semaphore wait and mutex cleanup. |

### Threading and Async

| File                          | Change                                                                                                                             |
|-------------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| `Engine/Components/Async.cpp` | `SDL_CreateThread` calls skipped. The async controller's queue mechanism runs inline instead of dispatching to background threads. |

### Event Loop and Persistence

| File                         | Change                                                                                                                                                                           |
|------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `Engine/Core/Event.cpp`      | Periodic IDBFS sync added to the main event loop. Calls `FS.syncfs()` at regular intervals to flush save data from the Emscripten virtual filesystem to the browser's IndexedDB. |
| `Engine/Core/ONScripter.cpp` | Startup path adjustments for the browser environment.                                                                                                                            |
| `Engine/Core/File.cpp`       | File handling adjustments for the Emscripten virtual filesystem.                                                                                                                 |

### Graphics

| File                        | Change                                                                                                             |
|-----------------------------|--------------------------------------------------------------------------------------------------------------------|
| `Engine/Graphics/GPU.cpp`   | WebGL-compatible GPU initialisation. Adjusts context creation and feature detection for the GLES2-over-WebGL path. |
| `Engine/Graphics/GPU.hpp`   | Related declarations.                                                                                              |
| `Engine/Graphics/GLES2.cpp` | GLES2 shader compatibility adjustments for WebGL.                                                                  |

### Input and Other

| File                             | Change                                                                  |
|----------------------------------|-------------------------------------------------------------------------|
| `Engine/Core/Image.cpp`          | Frame queue management adjustments for the single-threaded decode path. |
| `Engine/Core/Animation.cpp`      | Minor adjustments for Emscripten compatibility.                         |
| `Engine/Core/CommandExt.cpp`     | Command handling adjustments for browser environment.                   |
| `Engine/Core/Sound.cpp`          | Audio path adjustments for Web Audio API backend.                       |
| `Engine/Components/Joystick.hpp` | Joystick detection disabled (not applicable in browser).                |
| `Engine/Core/ONScripter.hpp`     | Related declarations.                                                   |

## Building

This fork is not built standalone. It is cloned and compiled as part of the [Umineko Web](https://github.com/VictoriqueMoe/umineko_web_asm) Docker build, which provides the Emscripten SDK, cross-compiled dependencies (FFmpeg, libass, HarfBuzz, FriBidi, SDL2_gpu), and the CMake configuration that ties everything together.

For native platform builds (Windows, macOS, Linux), refer to the upstream [Compilation.md](https://github.com/umineko-project/onscripter-ru/blob/master/Resources/Docs/Compilation.md).

## Credits

### Fork
- [VictoriqueMoe](https://github.com/VictoriqueMoe) - Emscripten/WebAssembly port

### Original
- Ogapee
- "Uncle" Mion Sonozaki
- Umineko Project
- All third-party library authors
