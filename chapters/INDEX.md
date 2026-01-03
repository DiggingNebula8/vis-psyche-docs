\newpage

# VizPsyche Engine Technical Book

A hands-on guide to building a 3D rendering engine from scratch.

## Table of Contents

### Part 1: Foundation
1. **[Introduction](00_Introduction.md)** - What we're building and why
2. **[Build System](01_BuildSystem.md)** - CMake, project structure, building
3. **[DLL Architecture](02_DLLArchitecture.md)** - Exports, namespaces, entry points
4. **[Third-Party Libraries](03_ThirdPartyLibraries.md)** - GLFW, GLAD, GLM, ImGui, spdlog, stb_image, tinygltf

### Part 2: Infrastructure
5. **[Logging System](04_LoggingSystem.md)** - spdlog wrapper, log levels, macros
6. **[Window & Context](05_WindowAndContext.md)** - GLFWManager, OpenGL context, input

### Part 3: Graphics I
7. **[OpenGL Fundamentals](06_OpenGLFundamentals.md)** - Pipeline, buffers, shaders, coordinates
8. **[Abstractions](07_Abstractions.md)** - RAII wrappers, Rule of 5, clean APIs
9. **[Textures](08_Textures.md)** - Image loading, GPU textures, UV mapping

### Part 4: Editor I
10. **[Dear ImGui](09_DearImGui.md)** - Immediate mode GUI, widgets, UIManager wrapper

### Part 5: Engine Architecture
11. **[Transform & Mesh](10_TransformAndMesh.md)** - Position, rotation, scale, geometry factories
12. **[Camera System](11_CameraSystem.md)** - View/projection matrices, camera movement
13. **[Scene Management](12_SceneManagement.md)** - SceneObject, shared resources, object selection

### Part 6: Graphics II
14. **[Lighting](13_Lighting.md)** - Blinn-Phong model, normals, directional lights

### Part 7: Assets
15. **[Model Loading](14_ModelLoading.md)** - glTF format, tinygltf, PBR materials

### Part 8: Input
16. **[Input System](15_InputSystem.md)** - Keyboard, mouse, polling vs events, edge detection
17. **[Camera Controller](16_CameraController.md)** - WASD movement, mouse look, scroll zoom

### Part 9: Graphics III *(planned)*
18. **[Advanced OpenGL](17_AdvancedOpenGL.md)** - Framebuffers, depth/stencil, cubemaps
19. **[Advanced Lighting](18_AdvancedLighting.md)** - Shadows, PBR, HDR, bloom

### Part 10: Engine II *(planned)*
20. **[Entity Component System](19_ECS.md)** - Components, systems, queries

### Appendices
- **[Appendix A: Code Reference](A_Reference.md)** - Class diagrams, file reference, debugging tips

---

## Reading Order

Read chapters in order. Each builds on the previous:

```
00 Introduction
      ↓
01 Build System ←── Understand how we compile
      ↓
02 DLL Architecture ←── Understand how engine/app interact
      ↓
03 Third-Party Libraries ←── Know our building blocks
      ↓
04 Logging System ←── Track what's happening
      ↓
05 Window & Context ←── Create window, OpenGL context
      ↓
06 OpenGL Fundamentals ←── Understand graphics basics
      ↓
07 Abstractions ←── Understand our C++ patterns
      ↓
08 Textures ←── Add images to geometry
      ↓
09 Dear ImGui ←── Debug UI for development
      ↓
10 Transform & Mesh ←── Geometry and positioning
      ↓
11 Camera System ←── View the 3D world
      ↓
12 Scene Management ←── Manage multiple objects
      ↓
13 Lighting ←── Make it look 3D
      ↓
14 Model Loading ←── Load external 3D models
      ↓
15 Input System ←── Handle user interaction
      ↓
16 Camera Controller ←── WASD movement, mouse look
      ↓
Appendix A ←── Reference material
```

---

## How to Use This Book

### While Coding
Keep the book open alongside the code. When you see a class, find its section.

### To Learn
Work through exercises at the end of each chapter.

### To Review
Use [Appendix A](A_Reference.md) as a quick reference for class diagrams and file locations.

---

## Prerequisites

- Basic C++ (classes, templates, pointers)
- Basic linear algebra (vectors, matrices)
- Visual Studio 2022 or later
- CMake 3.16+

---

## Updates

This is a **living document**. As the engine grows, new chapters will be added:

- [x] Model Loading
- [x] Editor I (Dear ImGui)
- [x] Input System
- [x] Camera Controller
- [ ] Advanced OpenGL *(next)*
- [ ] Advanced Lighting
- [ ] Editor II (UI Framework)
- [ ] Entity Component System
- [ ] Animation
- [ ] Physics
- [ ] Audio

