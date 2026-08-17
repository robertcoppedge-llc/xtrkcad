# OOP Architectural Refactoring Plan

## Goal
Migrate the xtrkcad application from a procedural, C-based architecture to a modern, maintainable Object-Oriented Programming (OOP) structure. The primary focus is on improving encapsulation, reducing global state dependency, and creating clear interfaces for modularity.

### 🎯 Core Design Principles
1.  **Encapsulation**: All data structures and their associated manipulation logic must be wrapped within class boundaries, hiding internal details from external components.
2.  **Abstraction**: Interfaces (Abstract Base Classes in C++) will define contracts (e.g., `IGeometryProcessor`) that modules must adhere to, allowing for future changes without affecting the core system.
3.  **Modularity**: The system will be split into three decoupled layers: **Core Logic**, **Utility Library**, and **Presentation Layer**.

### 🔄 Component-by-Component Refactoring Strategy

#### A. Core Business Logic Layer (Bezier & Spiro)
*   **Strategy**: Convert stateful procedural calls into class methods.
*   **`bezctx *` $\rightarrow$ `class BezierContext`**: The context should be managed by a dedicated object that handles its own lifecycle using Resource Acquisition Is Initialization (RAII) principles. All functions that modify the context state become public methods of this class.
*   **`SpiroPathGenerator` Class**: This service class encapsulates the entire spiro calculation. It receives simple, encapsulated inputs and returns a new, self-contained path object (`SpiroSegmentPath`), rather than manipulating global state or requiring explicit cleanup calls.

#### B. Utility & Infrastructure Library Layer (WLib)
*   **Strategy**: Promote all generic functions to static utility methods within a namespace or dedicated utility class (e.g., `UtilityLibrary::MathHelper`).
*   **Goal**: Create a highly stable, stateless library that can be easily accessed by any other module without worrying about global state corruption.
*   **Key Areas**: Math utilities, I/O wrappers, time handling, etc.

### 🔗 Summary of Structural Links & Encapsulation

The core strength of the OOP design will be enforcing clear data flow:
1.  **UI Layer** $\rightarrow$ **Command Object** $\rightarrow$ (Calls method on) **Core Logic Class**.
2.  **Core Logic Class** $ightarrow$ (Uses methods from) **Utility Library**.
3.  **Data Flow**: The application state should be managed by a single central `ApplicationModel` class, which acts as the source of truth and coordinates interactions between the various service objects (`BezierContext`, `SpiroGenerator`).

### ✅ Next Steps for Implementation
1. Establish the C++ project structure using CMake.
2. Implement Abstract Base Classes (interfaces) first.
3. Migrate Utility Library code into static methods, ensuring thread safety.