# Swift (HandBrake) Comprehensive Architecture & Task Distribution Reference

This reference document provides a detailed layout of the **Swift** video transcoding repository. It catalogs folders, subfolders, and key files, explaining what each component does and how they are linked across the workspace.

---

## 1. Directory Tree & Folder Roles

Here is the exact folder structure of the Swift project:

```
Swift/
├── contrib/             # 3rd-party dependency build definitions (recipes)
│   ├── ffmpeg/          # FFmpeg decoder/encoder engine configuration
│   ├── x264/            # H.264/AVC software encoder recipe
│   ├── x265/            # H.265/HEVC software encoder recipe
│   ├── svt-av1/         # SVT-AV1 video encoder recipe
│   ├── libvpl/          # Intel Video Processing Library (QSV) build recipe
│   └── [31+ others]     # Libraries like libass, zimg, fribidi, jansson, etc.
├── gtk/                 # Linux user interface (GTK4/C)
├── macosx/              # macOS native user interface (Cocoa/ObjC/Swift)
├── win/                 # Windows native user interface (.NET C#/WPF)
├── make/                # Python scripts and Makefile templates for build setup
│   ├── cross/           # Meson configurations for cross-compiling (e.g. MinGW)
│   ├── include/         # GNU Make definitions for compiler flags, rules, and contribs
│   └── lib/             # Helper libraries for Python configure scripts
├── test/                # Command-line interface client (HandBrakeCLI) code
└── libhb/               # Core transcoding engine library (C)
    ├── handbrake/       # Header files containing shared definitions and API contracts
    ├── platform/        # Hardware acceleration adapter files
    │   └── macosx/      # Apple VideoToolbox and Metal GPU shader adapters
    └── templates/       # Frame-processing templates for parallel filters
```

---

## 2. Comprehensive Directory & File Catalog

### A. The Build System: `make/`
This folder handles configuration, system checks, and compiles the Makefile templates.
- **[`make/configure.py`](file:///home/harshit/Pending/Swift/make/configure.py)**: Main entry point for the build configuration. It runs platform checks (checking compiler versions, tools like `nasm`, `ninja`, `meson`), registers user-facing options (such as `--disable-gtk`, `--enable-qsv`), and outputs the final `./build/GNUmakefile` and configuration headers.
- **[`make/df-fetch.py`](file:///home/harshit/Pending/Swift/make/df-fetch.py)** / **[`make/df-verify.py`](file:///home/harshit/Pending/Swift/make/df-verify.py)**: Downloader and integrity checker for package source tarballs configured under `contrib/`.
- **`make/include/` subfiles**:
  - **`contrib.defs`**: Defines standard compilation recipes and variables for external packages.
  - **`gcc.defs`**: Sets up compilation/linker arguments (`-O3`, `-mfpmath=sse`, LTO, debug modes). It defines the executable generation commands like `TEST.GCC.EXE++`.
  - **`main.defs`** / **`main.rules`**: Root variables and targets (`make build`, `make install`) for HandBrake.

---

### B. The Command-Line interface: `test/`
This module compiles into `HandBrakeCLI` and runs user-facing parameters.
- **[`test/test.c`](file:///home/harshit/Pending/Swift/test/test.c)**: Command line parser and wrapper. It parses input flags (`-i`, `-o`, `--preset`), tracks progress updates, registers event logging callbacks, and fires backend calls like `hb_init` and `hb_start`.
- **[`test/parsecsv.c`](file:///home/harshit/Pending/Swift/test/parsecsv.c)** / **`parsecsv.h`**: Internal parser for loading batch lists and CSV jobs into transcoder tasks.
- **`test/module.defs`**: Contains target compiler flags (`TEST.GCC.pkgconfig`) and defines what external dependencies are linked (e.g. `libass`, `SvtAv1Enc`, `dvdnav`).

---

### C. The Core Transcoder Engine: `libhb/`

#### 1. API Headers (`libhb/handbrake/`)
Provides types, structs, and function prototypes shared across filters, encoders, and the UI.
- **[`libhb/handbrake/handbrake.h`](file:///home/harshit/Pending/Swift/libhb/handbrake/handbrake.h)**: Declares the main external control APIs: `hb_init()`, `hb_start()`, `hb_stop()`, `hb_close()`.
- **[`libhb/handbrake/common.h`](file:///home/harshit/Pending/Swift/libhb/handbrake/common.h)**: Defines core internal structures, primarily:
  - `hb_job_t`: Holds transcode pipeline configs (vcodec, width, frame rate, crop info, filters, audio list).
  - `hb_buffer_t`: The wrapper for video/audio frames passed through FIFO queues.
  - `hb_work_object_t`: The base class representing pipeline steps (containing pointers to `init`, `work`, and `close` callbacks).
- **[`libhb/handbrake/ports.h`](file:///home/harshit/Pending/Swift/libhb/handbrake/ports.h)**: System-level interfaces for OS-neutral functions: thread initialization (`hb_thread_init`), lock primitives (`hb_lock_init`), and condition variables (`hb_cond_init`).
- **[`libhb/handbrake/taskset.h`](file:///home/harshit/Pending/Swift/libhb/handbrake/taskset.h)**: Declares structures for `taskset_t`, worker arguments, and sync variables used by parallel filters.

#### 2. GPU Hardware Acceleration & Metal (`libhb/platform/macosx/`)
Contains macOS-specific video encoding/decoding and GPU filters.
- **`encvt.c`**: VideoToolbox encoder integration (H.264, HEVC, ProRes hardware acceleration).
- **`comb_detect_vt.m`** / **`deinterlace_vt.m`** / **`rotate_vt.c`**: Hardware-accelerated Apple Metal filters.
- **`metal_utils.m`**: Utility functions compiling shaders and managing GPU texture buffers.

#### 3. Core Engine Pipeline Files (`libhb/`)
The main C files that control execution, data queuing, and transcoding algorithms.

* **[`libhb/hb.c`](file:///home/harshit/Pending/Swift/libhb/hb.c)**:
  - **Role**: Coordinates the overall library state.
  - **APIs**: Exposes `hb_init()` to initialize dependencies and `hb_start()` to spawn the job orchestrator thread.

* **[`libhb/work.c`](file:///home/harshit/Pending/Swift/libhb/work.c)**:
  - **Role**: The main job controller. Inside `do_job()`, it checks codec settings, configures filter pipelines, constructs `hb_fifo_t` queues, and instantiates each pipeline task (`hb_work_object_t`) as an independent system thread.
  - **Execution Loop**: Implements `hb_work_loop()`, which runs continuously for each thread, pulling data from `fifo_in`, calling the codec processing function, and outputting to `fifo_out`.

* **[`libhb/fifo.c`](file:///home/harshit/Pending/Swift/libhb/fifo.c)**:
  - **Role**: Implements thread-safe ring-buffers (`hb_fifo_t`) that connect pipeline stages.
  - **APIs**: Exposes `hb_fifo_push()` and `hb_fifo_get()`. When a FIFO queue exceeds its threshold (e.g. `FIFO_SMALL`), the pushing thread is blocked using condition variables. If the FIFO is empty, the consumer thread is blocked.

* **[`libhb/taskset.c`](file:///home/harshit/Pending/Swift/libhb/taskset.c)**:
  - **Role**: Worker pool manager for parallel video filters.
  - **Flow**: Spawns $N$ worker threads during `taskset_init()`. In `taskset_cycle()`, it sets a start flag (`begin = 1`) and calls `hb_cond_signal()` to wake the worker threads. Workers call the registered filter functions on their allocated horizontal frame slices and signal `complete_cond` when finished. The calling thread waits until all workers are done before outputting the frame.

* **[`libhb/sync.c`](file:///home/harshit/Pending/Swift/libhb/sync.c)**:
  - **Role**: Integrates video and audio streams (`WORK_SYNC_VIDEO`). Tracks presentation timestamps (PTS), drop rates, duplicate rates, and maintains frame counters (`common->st_counts`).
  - **APIs**: Implements the rate calculator `p.rate_avg = 1000.0 * processed_frames / elapsed_time`.

* **[`libhb/ports.c`](file:///home/harshit/Pending/Swift/libhb/ports.c)**:
  - **Role**: Maps platform-independent wrappers (`hb_thread_init()`, `hb_lock()`) to native OS calls (POSIX threads `pthread_create`/`pthread_mutex_lock` on Linux/macOS and Win32 threads/critical sections on Windows).

* **[`libhb/reader.c`](file:///home/harshit/Pending/Swift/libhb/reader.c)** / **`stream.c`**:
  - **Role**: Opens the file, extracts streams, and parses container metadata. It runs the input packet loop, distributing demuxed packets to decoder input queues.

* **[`libhb/decavcodec.c`](file:///home/harshit/Pending/Swift/libhb/decavcodec.c)**:
  - **Role**: Wrapper for FFmpeg decoders. Decodes compressed video/audio packets into raw image surfaces (`AVFrame`) and audio buffers.

* **[`libhb/encx264.c`](file:///home/harshit/Pending/Swift/libhb/encx264.c)** / **`encx265.c`** / **`encsvtav1.c`**:
  - **Role**: Codec wrappers. They map filter settings to encoder configuration parameters, initialize software encoders (x264, x265, SVT-AV1), and handle multithreading parameters for encoders.

* **[`libhb/muxavformat.c`](file:///home/harshit/Pending/Swift/libhb/muxavformat.c)**:
  - **Role**: Wraps FFmpeg container writing tools. Integrates encoded packet flows and writes them to the output file (MKV, MP4, WebM).

* **[`libhb/comb_detect.c`](file:///home/harshit/Pending/Swift/libhb/comb_detect.c)** / **`nlmeans.c`** / **`avfilter.c`**:
  - **Role**: Video filtering filters. They call `taskset_cycle()` in [`libhb/taskset.c`](file:///home/harshit/Pending/Swift/libhb/taskset.c) to parallelize pixel-heavy analysis and filtering algorithms across all CPU threads.

---

## 3. Linkage Matrix (Function and Data Connections)

This matrix maps out which code files import, call, or configure other files in the project:

```
┌─────────────┐       (1) Spawns work thread        ┌──────────────┐
│  test/test.c│ ──────────────────────────────────> │  libhb/work.c│
└──────┬──────┘                                     └──────┬───────┘
       │                                                   │
       │ (2) Invokes                                       │ (3) Creates buffers
       ▼                                                   ▼
┌─────────────┐                                     ┌──────────────┐
│  libhb/hb.c │                                     │ libhb/fifo.c │
└─────────────┘                                     └──────┬───────┘
                                                           │
                                                           │ (4) Interfaces via queue
                                                           ▼
                                                    ┌──────────────┐
                                                    │libhb/sync.c  │
                                                    └──────┬───────┘
                                                           │
                                                           │ (5) Loops parallel filter slices
                                                           ▼
                                                    ┌──────────────┐
                                                    │libhb/taskset.│
                                                    └──────────────┘
```

### Detailed Linkage Map:

1. **[`test/test.c`](file:///home/harshit/Pending/Swift/test/test.c)** ➔ **[`libhb/hb.c`](file:///home/harshit/Pending/Swift/libhb/hb.c)**:
   - CLI setup runs `hb_init()` and `hb_start()` inside `hb.c`.
2. **[`libhb/hb.c`](file:///home/harshit/Pending/Swift/libhb/hb.c)** ➔ **[`libhb/work.c`](file:///home/harshit/Pending/Swift/libhb/work.c)**:
   - `hb_start()` calls `hb_work_init()` to spawn the job thread (`work_func`), which runs the transcoder.
3. **[`libhb/work.c`](file:///home/harshit/Pending/Swift/libhb/work.c)** ➔ **[`libhb/fifo.c`](file:///home/harshit/Pending/Swift/libhb/fifo.c)**:
   - Configures the pipeline. It instantiates the reader, decoder, sync, filter, and encoder threads, linking them together with `hb_fifo_init()`.
4. **[`libhb/reader.c`](file:///home/harshit/Pending/Swift/libhb/reader.c)** ➔ **[`libhb/decavcodec.c`](file:///home/harshit/Pending/Swift/libhb/decavcodec.c)** ➔ **[`libhb/sync.c`](file:///home/harshit/Pending/Swift/libhb/sync.c)**:
   - `reader.c` extracts packets, pushing them to the decoder (`decavcodec.c`) via packet queues. The decoder decodes them into raw frames, pushing them into the sync queues of `sync.c`.
5. **[`libhb/sync.c`](file:///home/harshit/Pending/Swift/libhb/sync.c)** ➔ **`libhb/filters`** (e.g. **[`libhb/comb_detect.c`](file:///home/harshit/Pending/Swift/libhb/comb_detect.c)**):
   - Once synced, frames are passed to the video filters. High-computational filters leverage `taskset_cycle()` in [`libhb/taskset.c`](file:///home/harshit/Pending/Swift/libhb/taskset.c) to run pixel processing routines in parallel on slice worker threads.
6. **`libhb/filters`** ➔ **[`libhb/encx264.c`](file:///home/harshit/Pending/Swift/libhb/encx264.c)** ➔ **[`libhb/muxavformat.c`](file:///home/harshit/Pending/Swift/libhb/muxavformat.c)**:
   - Processed frames are put into the video encoder queue. The encoder compresses them and outputs packets to the muxer queue. `muxavformat.c` writes the packets to the output file container.
7. **All Threads** ➔ **[`libhb/ports.c`](file:///home/harshit/Pending/Swift/libhb/ports.c)**:
   - All components utilize mutexes, allocations, and threading hooks defined in `ports.c` to maintain cross-platform compatibility.
