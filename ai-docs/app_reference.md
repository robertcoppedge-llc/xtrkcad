# Application Reference Documentation (Current State)

## Overview

The xtrkcad application is a robust, procedural C-based geometry tool designed for CAD drawing and path generation. The codebase demonstrates a highly modular structure, allowing specific functionalities to be developed and maintained in isolation.

### 📂 Directory Structure Analysis

The source code is separated into key functional modules:
*   **`cornu/`**: Core business logic. Contains domain-specific algorithms like Bezier Curve context management and Spiro Spiral path generation. This is the geometric brain of the application.
*   **`wlib/`**: Utility Library. Houses fundamental building blocks, abstract data types, system utilities, and general math helper functions. This module should be treated as a shared infrastructure layer.
*   **`lib/`**: Application glue code, containing primary resources (`*.desktop`, `xtrkcad.ini`) and the main build targets that assemble all other modules into a runnable application.

### 🧱 Key Functional Components

#### 1. Core Geometry Processing (Cornu)
*   **Bezier Context (`bezctx`)**: Manages the state and lifecycle of Bezier curve geometry, handling creation, manipulation, and conversion to the final format (`xtrkcad_to_xtrkcad`). This object maintains the context needed for drawing complex paths.
*   **Spiro Spiral Generator (`spiro`)**: Implements complex mathematical algorithms to generate spiro spiral paths. It manages point data (`spiro_cp`, `spiro_seg`) and can convert its output into a format usable by Bezier contexts.

#### 2. Utility Library (WLib)
*   Provides general-purpose, reusable utilities across the application. These include math functions, resource loaders, and cross-platform helper routines. This library is foundational to the entire system's operation.

### ✨ Usage Notes for New Developers
*   **Build Process**: The project relies on CMake (`CMakeLists.txt`) to link these modules together correctly. Understanding the dependency graph defined by the `CMake` directory is essential before modifying any source code.
*   **Execution Flow**: A typical drawing operation begins in the UI layer, passes a request (e.g., 'draw Bezier curve') to the `BezContext` module, which then utilizes functions from the `UtilityLibrary` and potentially the `SpiroGenerator` to calculate the final geometry.