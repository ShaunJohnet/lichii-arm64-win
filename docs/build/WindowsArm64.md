# Building Lichii for Windows on ARM64

The release workflow ships no Windows ARM64 build. This branch carries the changes needed to
build one:

- `CMakeLists.txt`: the bundled Qt 6.11 and sirit packages for Windows ARM64 are built against the
  DLL runtime (`/MD`), so the build uses `/MD` on ARM64 as on x64. A static runtime failed the
  link's `RuntimeLibrary` check against Qt.
- `src/common/nx2_sampler.cpp`: the optional `EDEN_NX2_SAMPLE` sampler reads x64 thread contexts,
  so it is compiled on x64 only. Other targets get the existing stub.
- `src/shader_recompiler/ir_opt/global_memory_to_storage_buffer_pass.cpp`: MSVC STL 14.51 sends a
  12-byte struct into its ARM64 vectorized `std::find`, which fails a `static_assert`. This file
  uses the scalar algorithms on ARM64.
- `src/input_common/CMakeLists.txt`: the bundled ARM64 Qt was compiled with C++/WinRT
  2.0.250303.1, and the link fails (`/failifmismatch: C++/WinRT version`) against the Windows SDK's
  newer copy used by `switch2_ble.cpp`. Configure downloads that version's `cppwinrt.exe` from
  NuGet (SHA-256 pinned) and generates matching headers from the installed SDK into the build tree.
- `src/dynarmic/.../backend/arm64/a64_address_space.cpp`: under the MSVC ABI, `MemoryRead128`
  returns its `Vector` through a pointer passed after `this`, not in x0 and x1, so the JIT's
  trampolines pass it a stack slot. Without this a game crashes as soon as it runs.
- `cpmfile.json`, `externals/ffmpeg/CMakeLists.txt`: Windows ARM64 uses
  [ShaunJohnet/lichii-ffmpeg-av1](https://github.com/ShaunJohnet/lichii-ffmpeg-av1)
  `9.0.2-av1-arm64-1`, crueter-ci's FFmpeg build with libdav1d 1.5.4 (NEON assembly) merged into
  `avcodec.lib`. crueter-ci's own package has no AV1 decoder, so Switch 2 titles' AV1 movies
  were black.

Compile with **clang-cl**, as the Windows CI does. MSVC's `cl.exe` rejects dynarmic's ARM64 backend
(`error C2719`, a 4096-byte-aligned parameter passed by value) and the x64-only `_umul128`.

OpenGL is disabled on Windows ARM64 by the existing CMake options, so the renderer is Vulkan.

## Requirements

- Visual Studio 2022 or later with the C++ workload and the **ARM64 build tools** (MSVC v14.4x+ ARM64)
- [LLVM](https://github.com/llvm/llvm-project/releases) 22 (`clang-cl`, `lld-link`, `llvm-lib`)
- CMake 3.31+, Ninja, Git, Python 3
- [Vulkan SDK](https://vulkan.lunarg.com/) (for `glslangValidator`)

Qt, Boost, FFmpeg, OpenSSL, SDL3 and the other dependencies are downloaded at configure time.

## Native build on an ARM64 Windows machine

Run from a `cmd` prompt in the repository root:

```bat
call "C:\Program Files\Microsoft Visual Studio\18\Community\VC\Auxiliary\Build\vcvarsall.bat" arm64
cmake -S . -B build\arm64 -G Ninja -DCMAKE_BUILD_TYPE=Release ^
  -DCMAKE_C_COMPILER=clang-cl -DCMAKE_CXX_COMPILER=clang-cl ^
  -DLICHII_TESTS=OFF -DBUILD_TESTING=OFF -DLICHII_ROOM=OFF -DLICHII_ROOM_STANDALONE=OFF ^
  -DUSE_DISCORD_PRESENCE=OFF -DENABLE_WERROR=OFF -DLICHII_BUILD_PLATFORM=windows-arm64
cmake --build build\arm64 --target lichii -- -j 4
```

Adjust the Visual Studio path to your edition and version. `-j 4` keeps memory use down: each
compile can take 1-2 GB, and the default job count ran a 16 GB machine out of memory.

The executable is `build\arm64\bin\lichii.exe`.

## Cross build from an x64 Windows machine

The ARM64 Qt package's `moc`, `rcc` and `uic` are ARM64 executables, so a cross build also needs
the x64 Qt package for its host tools. Configure once for x64 so CPMUtil downloads it to
`.cache/cpm/qt6/6.11.1-windows-amd64`:

```bat
call "C:\Program Files\Microsoft Visual Studio\18\Community\VC\Auxiliary\Build\vcvarsall.bat" x64
cmake -S . -B build\hostqt -G Ninja -DCMAKE_BUILD_TYPE=Release -DLICHII_TESTS=OFF
```

Then configure and build for ARM64 in a fresh prompt:

```bat
call "C:\Program Files\Microsoft Visual Studio\18\Community\VC\Auxiliary\Build\vcvarsall.bat" x64_arm64
set H=%CD:\=/%/.cache/cpm/qt6/6.11.1-windows-amd64
cmake -S . -B build\arm64 -G Ninja -DCMAKE_BUILD_TYPE=Release ^
  -DCMAKE_SYSTEM_NAME=Windows -DCMAKE_SYSTEM_PROCESSOR=ARM64 ^
  "-DCMAKE_C_COMPILER=C:/Program Files/LLVM/bin/clang-cl.exe" ^
  "-DCMAKE_CXX_COMPILER=C:/Program Files/LLVM/bin/clang-cl.exe" ^
  -DCMAKE_C_COMPILER_TARGET=aarch64-pc-windows-msvc -DCMAKE_CXX_COMPILER_TARGET=aarch64-pc-windows-msvc ^
  "-DCMAKE_LINKER=C:/Program Files/LLVM/bin/lld-link.exe" ^
  "-DCMAKE_AR=C:/Program Files/LLVM/bin/llvm-lib.exe" ^
  "-DCMAKE_RC_COMPILER=C:/Program Files/LLVM/bin/llvm-rc.exe" ^
  -DQT_HOST_PATH=%H% -DQT_HOST_PATH_CMAKE_DIR=%H%/lib/cmake ^
  -DQt6CoreTools_DIR=%H%/lib/cmake/Qt6CoreTools -DQt6GuiTools_DIR=%H%/lib/cmake/Qt6GuiTools ^
  -DQt6WidgetsTools_DIR=%H%/lib/cmake/Qt6WidgetsTools -DQt6LinguistTools_DIR=%H%/lib/cmake/Qt6LinguistTools ^
  -DQt6ToolsTools_DIR=%H%/lib/cmake/Qt6ToolsTools -DQt6UiTools_DIR=%H%/lib/cmake/Qt6UiTools ^
  -DLICHII_TESTS=OFF -DBUILD_TESTING=OFF -DLICHII_ROOM=OFF -DLICHII_ROOM_STANDALONE=OFF ^
  -DUSE_DISCORD_PRESENCE=OFF -DENABLE_WERROR=OFF -DLICHII_BUILD_PLATFORM=windows-arm64
cmake --build build\arm64 --target lichii -- -j 4
```

Without the `Qt6*Tools_DIR` overrides, the ARM64 Qt package finds its own tools and the configure
step fails with `Test run of "moc" executable ... failed`.

## Status

The cross build produces an ARM64 `lichii.exe` (clang-cl 22.1.8, MSVC 14.51, Windows SDK
10.0.26100). Qt is linked statically, so a package needs only `lichii.exe`, `nx2_switches.txt` and
the ARM64 Visual C++ runtime DLLs from `VC\Redist\MSVC\<version>\arm64\Microsoft.VC145.CRT`.

The native ARM64 build and running the emulator on Snapdragon hardware are untested.
