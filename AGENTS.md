# AGENTS.md

## Cursor Cloud specific instructions

This is **Open CASCADE Technology (OCCT)** — a C++ library for 3D CAD/CAM/CAE. It is a single CMake-based C++ project (not a monorepo).

### Quick reference

| Task | Command |
|------|---------|
| Configure | `cd build && cmake -G Ninja -DCMAKE_C_COMPILER=gcc -DCMAKE_CXX_COMPILER=g++ -DCMAKE_BUILD_TYPE=Release -DUSE_FREETYPE=ON -DUSE_TK=ON -DUSE_OPENGL=ON -DUSE_XLIB=ON -DBUILD_GTEST=ON -DBUILD_INCLUDE_SYMLINK=ON -DBUILD_CPP_STANDARD=C++17 -DBUILD_LIBRARY_TYPE=Shared -DINSTALL_DIR=/workspace/install ..` |
| Build | `cd build && cmake --build . --config Release -- -j$(nproc)` |
| Run GTests | `cd build && source env.sh && ./lin64/gcc/bin/OpenCascadeGTest` |
| Run filtered GTests | `cd build && source env.sh && ./lin64/gcc/bin/OpenCascadeGTest --gtest_filter="*Pattern*"` |
| Run DRAWEXE | `cd build && source env.sh && ./lin64/gcc/bin/DRAWEXE` |
| Lint (clang-format) | `clang-format --dry-run -Werror -style=file <file>` |

### Key caveats

- **Must use GCC, not Clang**: The default `c++` on this VM points to Clang 18 which selects GCC 14's libstdc++ but only GCC 13 dev files are installed. Always pass `-DCMAKE_C_COMPILER=gcc -DCMAKE_CXX_COMPILER=g++` to CMake.
- **Environment setup before running**: Always `source build/env.sh` before running DRAWEXE or OpenCascadeGTest. This sets `LD_LIBRARY_PATH` and OCCT resource paths.
- **Build output location**: Libraries are in `build/lin64/gcc/lib/`, binaries in `build/lin64/gcc/bin/`.
- **glTF not built**: The `libTKXSDRAWGLTF.so` warning in DRAWEXE is harmless — glTF support requires RapidJSON (`-DUSE_RAPIDJSON=ON`) which is not enabled by default without vcpkg.
- **Full build takes ~18 minutes** on a 4-core VM. Use `--gtest_filter` to run targeted tests during development.
- **Coding style**: See `.github/copilot-instructions.md` for OCCT naming conventions, handle usage, and GTest guidelines.
- **Adding files**: After adding new `.hxx`/`.cxx` files, update the corresponding `FILES.cmake` in the package directory.
