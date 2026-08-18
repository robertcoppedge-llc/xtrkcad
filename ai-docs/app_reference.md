# Application Reference Documentation

## 🏗️ Architecture Overview

XTrackCAD follows a layered, modular architecture designed for cross-platform compatibility and high mathematical precision. The application separates high-level user interface concerns from low-level geometric calculations and system-specific abstractions.

### 📂 Directory Structure & Modules

#### 1. Presentation Layer (`app/bin/`)
The entry point and UI orchestration module.
*   **Core UI Logic**: Manages the main window, menu systems, toolbars, and user input handling.
*   **Command Pattern**: Implements an undo/redo system (via `cundo.c`) to manage complex drawing operations.
*   **UI Components**: Contains specialized widgets and dialogs for selecting paths, parameters, and views.
*   **Resource Management**: Handles icon loading, menu definition, and interface localization.

#### 2. Logic Layer (`app/cornu/`)
The geometric "brain" of the application. This module is platform-agnostic and focuses purely on mathematical modeling.
*   **Bezier Context (`bezctx`)**: Manages the state and lifecycle of Bezier curve geometry, handling creation, manipulation, and conversion.
*   **Spiro Spiral Generator (`spiro`)**: Implements algorithms for generating spiro spiral paths using point-based data.
*   **Geometric Kernels**: Contains the mathematical primitives required for path generation and shape manipulation.

#### 3. Abstraction Layer (`app/wlib/`)
Provides a cross-platform hardware and OS abstraction layer.
*   **`gtklib/`**: GTK+ 2.0 abstraction for Linux/Unix-based systems.
*   **`mswlib/`**: Microsoft Windows-specific implementation of the windowing and UI abstraction.
*   **`include/`**: Platform-independent headers and utility definitions used by both GTK and MSW implementations.

#### 4. Utility & Foundation Layer
Low-level, reusable modules that support the entire application stack.
*   **`cJSON/`**: A lightweight, embedded JSON parsing library for data interchange.
*   **`dynstring/`**: Dynamic string management utilities.
*   **`app/lib/`**: Application-wide resources, including configuration (`.ini`), parameter templates (`.xtp`), and desktop integration files.
*   **`app/tools/`**: Development and auxiliary tools (e.g., Halibut for documentation, image converters).

---

## 📚 Library Reference

### 🛠️ Internal Libraries

| Library | Scope | Purpose |
| :--- | :--- | :--- |
| **`cornu`** | Logic | Core geometric algorithms (Bezier, Spiro, etc.). |
| **`wlib`** | Abstraction | Cross-platform UI and system abstraction (GTK/Win32). |
| **`cJSON`** | Utility | Embedded JSON processing. |
| **`dynstring`** | Utility | Dynamic memory-managed string operations. |
| **`i18n`** | Localization | Internationalization and translation management. |

### 🌐 External Libraries

| Library | Category | Usage in XTrackCAD |
| :--- | :--- | :--- |
| **GTK+ 2.0** | UI Toolkit | Primary toolkit for the Linux/Unix graphical interface. |
| **Zlib** | Compression | Used for data compression and decompression. |
| **Libzip** | Archiving | Handles compressed archive files. |
| **MiniXML** | Serialization | Enables SVG export functionality. |
| **CMocka** | Testing | Unit testing framework for verifying geometric and logic components. |
| **Doxygen** | Documentation | Automates technical API documentation generation. |
| **Pandoc** | Documentation | Converts Markdown documentation to other formats. |
| **FreeImage** | Graphics | Cross-platform bitmap/image export (primarily on Windows). |
| **Gettext** | Localization | Standard toolset for handling translated strings. |

---

## ✨ Developer Guidelines

*   **Geometry First**: When adding new shapes or paths, prioritize implementation in the `cornu/` module to ensure platform independence.
*   **Platform Agnosticism**: Any new UI component should be implemented in `wlib/include/` first, then specialized in `gtklib/` (Linux) and `mswlib/` (Windows).
*   **Testing**: New logic modules should include corresponding unit tests using the `CMocka` framework.
*   **Build System**: All dependencies and linking rules are managed via `CMake`. Modifications to the build system should be reflected in the `CMakeLists.txt` files throughout the tree.
