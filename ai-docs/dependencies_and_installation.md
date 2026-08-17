# Dependencies and Installation Guide

This document outlines the dependencies required to build and run XTrackCAD, categorized by their necessity for core functionality versus optional features.

## 1. Core Build Dependencies

The following packages are required to compile the base application and its primary features.

### System Libraries (Linux/Unix)
* **CMake**: The primary build system.
* **GTK+ 2.0**: Provides the graphical user interface backend.
* **Zlib**: Used for data compression.
* **Libzip**: Used for handling compressed archives.
* **gettext**: Required for internationalization (i18n) and localized strings.

### Build Tools
* **GCC/G++**: C/C++ compiler.
* **Make**: Build automation tool.

---

## 2. Optional Dependencies

These packages enable additional features or tools. If they are not installed, the relevant features will be disabled during the build process.

| Feature | Dependency | Description |
| :--- | :--- | :--- |
| **SVG Export** | `MiniXML` | Enables exporting layouts to SVG format. |
| **Unit Testing** | `CMocka` | Framework used for running automated unit tests. |
| **Documentation** | `Doxygen` | Generates technical API documentation. |
| **Doc Conversion** | `Pandoc` | Used for converting documentation formats. |
| **Windows Graphics**| `FreeImage` | Required on Windows for handling bitmap exports. |

---

## 3. Installation Instructions (Ubuntu/Debian)

To install all dependencies (including optional ones) on Ubuntu or Debian-based systems, run the following command:

```bash
sudo apt-get update
sudo apt-get install -y \
    cmake \
    libgtk2.0-dev \
    zlib1g-dev \
    libzip-dev \
    gettext \
    libmxml-dev \
    cmocka \
    doxygen \
    pandoc \
    libfreeimage-dev \
    libxml2-dev
```

## 4. Build Process Overview

1. **Configure**: Use `cmake` to generate the build files.
   ```bash
   mkdir build && cd build
   cmake ..
   ```
2. **Compile**: Use `make` to build the application.
   ```bash
   make -j$(nproc)
   ```
3. **Test (Optional)**: If `CMocka` was found, you can run tests:
   ```bash
   make test
   ```
