\newpage

# Chapter 23: Engine and Game Loop

Create an `Engine` class that owns the game loop and core subsystems, separating infrastructure from game logic.

---

## What We're Building

| Component | Purpose |
|-----------|---------|
| **Engine** | Singleton that owns Window, Renderer, UIManager |
| **EngineConfig** | Configuration struct for window title, size, vsync |
| **Application lifecycle** | Virtual methods: OnCreate, OnUpdate, OnRender, OnDestroy |

---

## The Problem

Currently, `Application::Run()` contains 377 lines:

```cpp
int Application::Run() {
    // Window creation
    // OpenGL init
    // Scene setup (game-specific!)
    // Game loop
    //   Input, Update, Render, UI (all mixed together)
    // No explicit cleanup
}
```

**Issues:**
- Engine code mixed with game code
- No separation of concerns
- Sandbox is an empty shell
- Hard to reuse for Editor vs Game

---

## The Solution

Split responsibilities:

| Responsibility | Owner |
|----------------|-------|
| Window, Renderer, UIManager | **Engine** |
| Game loop timing | **Engine** |
| Scene, Camera, Assets | **Application** (Sandbox) |
| Game logic | **Application** (Sandbox) |

---

## Engine Class

### EngineConfig

```cpp
struct EngineConfig {
    std::string Title = "VizPsyche";
    uint32_t Width = 800;
    uint32_t Height = 800;
    bool VSync = true;
};
```

### Engine Singleton

```cpp
class Engine {
public:
    static Engine& Get();
    
    void Run(Application* app, const EngineConfig& config = {});
    void Quit();
    
    GLFWManager& GetWindow();
    Renderer& GetRenderer();
    UIManager& GetUIManager();
    float GetDeltaTime() const;
    
private:
    bool Init(const EngineConfig& config);
    void Shutdown();
    
    std::unique_ptr<GLFWManager> m_Window;
    std::unique_ptr<Renderer> m_Renderer;
    std::unique_ptr<UIManager> m_UIManager;
    float m_DeltaTime = 0.0f;
    bool m_Running = false;
};
```

---

## Application Lifecycle

```cpp
class Application {
public:
    virtual ~Application() = default;
    
    virtual void OnCreate() {}
    virtual void OnUpdate(float deltaTime) {}
    virtual void OnRender() {}
    virtual void OnImGuiRender() {}
    virtual void OnDestroy() {}
};
```

| Method | When Called | Purpose |
|--------|-------------|---------|
| `OnCreate()` | Once, before loop starts | Load assets, create scene |
| `OnUpdate(dt)` | Every frame | Game logic, camera controller |
| `OnRender()` | Every frame | Draw calls |
| `OnImGuiRender()` | Every frame | ImGui panels |
| `OnDestroy()` | Once, after loop ends | Cleanup |

---

## The Game Loop

```cpp
void Engine::Run(Application* app, const EngineConfig& config) {
    if (!Init(config)) return;
    
    m_Running = true;
    app->OnCreate();
    
    double prevTime = glfwGetTime();
    
    while (m_Running && !m_Window->WindowShouldClose()) {
        // Delta time
        double now = glfwGetTime();
        m_DeltaTime = static_cast<float>(now - prevTime);
        prevTime = now;
        
        // Input
        m_Window->ProcessInput();
        m_UIManager->BeginFrame();
        
        // Application hooks
        app->OnUpdate(m_DeltaTime);
        app->OnRender();
        app->OnImGuiRender();
        
        // Present
        m_UIManager->Render();
        m_Window->SwapBuffersAndPollEvents();
    }
    
    app->OnDestroy();
    Shutdown();
}
```

---

## Updated Entry Point

> [!NOTE]
> `EntryPoint.h` lives in VizEngine and is included by client applications (Sandbox). This keeps `main()` ownership in the engine while allowing applications to define their own `CreateApplication()` factory.

```cpp
// VizEngine/EntryPoint.h
int main(int argc, char** argv) {
    VizEngine::Log::Init();
    
    VizEngine::EngineConfig config;
    auto app = VizEngine::CreateApplication(config);
    VizEngine::Engine::Get().Run(app, config);
    delete app;
    
    return 0;
}
```

The application factory can customize the config:

```cpp
// Sandbox/SandboxApp.cpp
VizEngine::Application* VizEngine::CreateApplication(VizEngine::EngineConfig& config) {
    config.Title = "My Game";
    config.Width = 1280;
    config.Height = 720;
    return new Sandbox();
}
```

---

## Sandbox Implementation

```cpp
class Sandbox : public VizEngine::Application {
public:
    void OnCreate() override {
        // Create meshes, scene, load models
        // Setup camera, lighting
        // Load shaders, textures
    }
    
    void OnUpdate(float dt) override {
        // Camera controller
        // Object rotation
    }
    
    void OnRender() override {
        // Clear, render scene
    }
    
    void OnImGuiRender() override {
        // Scene Objects panel
        // Lighting panel
        // Scene Controls panel
    }
    
    void OnDestroy() override {
        // Cleanup (automatic via RAII)
    }
    
private:
    Scene m_Scene;
    Camera m_Camera;
    Shader m_LitShader;
    // ...
};
```

---

## Why Singleton?

> [!NOTE]
> The Engine singleton pattern is used by Hazel, Unity, and Unreal. While singletons create global state, a game has exactly one engine instance. The simplicity benefit outweighs the testing drawback for most projects.

---

## Graceful Shutdown

When the window X button is clicked:

1. `WindowShouldClose()` returns true
2. Loop exits
3. `app->OnDestroy()` is called
4. `Engine::Shutdown()` cleans up subsystems

This ensures resources are properly released.

---

## Error Handling

```cpp
bool Engine::Init(const EngineConfig& config) {
    m_Window = std::make_unique<GLFWManager>(...);
    
    if (!gladLoadGLLoader(...)) {
        VP_CORE_ERROR("Failed to initialize GLAD");
        return false;
    }
    
    // Setup OpenGL state, create subsystems
    return true;
}
```

If initialization fails, `Run()` returns early without entering the game loop.

---

## Milestone

After this chapter, you have:

- Engine singleton owning Window, Renderer, UIManager
- Application with virtual lifecycle methods
- Sandbox implementing game-specific logic
- Clean separation of engine vs game code

---

## What's Next

In **Chapter 24**, we'll refactor Application into an abstract base class with proper virtual lifecycle methods.

> **Next:** [Chapter 24: Virtual Lifecycle](24_VirtualLifecycle.md)

> **Previous:** [Chapter 22: Camera Controller](22_CameraController.md)
