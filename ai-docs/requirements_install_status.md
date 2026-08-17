# XTrkCAD Build Status - 8/17/2026

## Current Status: PARTIALLY SETUP - NEEDS CLEAN SLATE

### Successfully Installed (Linux/Ubuntu):
- ✅ CMake (via apt)
- ✅ GTK+ 2.0 (`libgtk2.0-dev`)
- ✅ Zlib (`zlib1g-dev`)
- ✅ Libzip (`libzip-dev`)
- ✅ Gettext (`gettext`)
- ✅ MiniXML (`libmxml-dev`)
- ✅ CMocka (`libcmocka-dev`)
- ✅ Doxygen (`doxygen`)
- ✅ Pandoc (`pandoc`)
- ✅ FreeImage (`libfreeimage-dev`)
- ✅ Libxml2 (`libxml2-dev`)

### Successfully Installed (via winget):
- ✅ CMake 4.4.2 at `C:\Program Files\CMake\bin\cmake.exe`
- ✅ LLVM-MinGW-Ucrt-x86_64 v22.1.8 installed via Winget at `C:\Users\rober\AppData\Local\Microsoft\WinGet\Packages\MartinStorsjo.LLVM-MinGW.UCRT_Microsoft.Winget.Source_8wekyb3d8bbwe\llvm-mingw-20260616-ucrt-x86_64`

### MinGW Installation Issue:
The winget install placed MinGW in a Winget package directory rather than the typical `C:\mingw64` location. The actual compiler tools are in:
- `C:\Users\rober\AppData\Local\Microsoft\WinGet\Packages\MartinStorsjo.LLVM-MinGW.UCRT_Microsoft.Winget.Source_8wekyb3d8bbwe\llvm-mingw-20260616-ucrt-x86_64`

### What We Know:
- XTrkCAD requires CMake 3.20+ for building
- Windows builds use MinGW-w64 as the compiler (not MSVC)
- FreeImage library is REQUIRED for bitmap handling
- Zlib and Libzip are needed (usually come with MinGW)

### Next Steps (Reset Required):
2. Install FreeImage library (critical dependency)
3. Create build directory and run CMake configuration with correct compiler paths
4. Build XTrkCAD executable

### Known Working Paths:
- **CMake**: `C:\Program Files\CMake\bin\cmake.exe`
- **MinGW location**: `C:\Users\rober\AppData\Local\Microsoft\WinGet\Packages\MartinStorsjo.LLVM-MinGW.UCRT_Microsoft.Winget.Source_8wekyb3d8bbwe\llvm-mingw-20260616-ucrt-x86_64`
