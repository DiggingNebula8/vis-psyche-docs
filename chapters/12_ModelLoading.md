\newpage

# Chapter 12: Model Loading (glTF)

## The Problem: Hardcoded Geometry

Look at our current mesh factory methods:

```cpp
auto pyramid = Mesh::CreatePyramid();
auto cube = Mesh::CreateCube();
auto plane = Mesh::CreatePlane();
```

These are fine for learning, but real games have:
- Characters with thousands of vertices
- Detailed environments
- Props, weapons, vehicles
- All created by artists in 3D modeling software

We need to **load external 3D model files**.

---

## Why glTF?

There are many 3D file formats. Here's why we chose glTF:

| Format | Pros | Cons |
|--------|------|------|
| **OBJ** | Simple, text-based, widely supported | No materials, no animations, outdated |
| **FBX** | Industry standard, full-featured | Proprietary (Autodesk), complex to parse |
| **COLLADA** | Open standard, XML-based | Verbose, slow to parse |
| **glTF** | Modern, PBR materials, compact binary | Newer (less legacy content) |

### The "JPEG of 3D"

glTF (GL Transmission Format) was designed by Khronos Group (the same people behind OpenGL) specifically for real-time rendering:

- **Efficient** - Designed to be fast to load and render
- **Complete** - Geometry, materials, textures, animations, scenes
- **PBR** - Physically-based rendering materials built-in
- **Two formats** - `.gltf` (JSON + binary) or `.glb` (single binary file)

---

## glTF File Structure

Understanding glTF's structure is key to loading it correctly.

### The Hierarchy

```
glTF File
├── scenes[]           ← Root of the scene graph
│   └── nodes[]        ← References to node indices
├── nodes[]            ← Transform + mesh/camera/light reference
│   ├── translation, rotation, scale
│   ├── mesh           ← Index into meshes[]
│   └── children[]     ← Child node indices
├── meshes[]           ← Geometry containers
│   └── primitives[]   ← Actual geometry (one mesh can have multiple primitives)
│       ├── attributes ← Position, normal, texcoord accessors
│       ├── indices    ← Index accessor
│       └── material   ← Material index
├── materials[]        ← PBR material definitions
│   └── pbrMetallicRoughness
│       ├── baseColorFactor
│       ├── metallicFactor
│       ├── roughnessFactor
│       └── baseColorTexture
├── textures[]         ← Texture definitions
│   ├── source         ← Image index
│   └── sampler        ← Sampler index
├── images[]           ← Image data (embedded or URI)
├── accessors[]        ← Typed views into buffer data
├── bufferViews[]      ← Byte ranges within buffers
└── buffers[]          ← Raw binary data (vertices, indices, etc.)
```

### Accessors and Buffer Views

This is the trickiest part of glTF. Data flows like this:

```
Buffer (raw bytes)
    ↓
BufferView (byte offset + length, stride)
    ↓
Accessor (component type, count, type like VEC3)
    ↓
Your vertex array
```

For example, to get positions:

```cpp
// 1. Find the accessor
const auto& accessor = model.accessors[primitive.attributes.at("POSITION")];

// 2. Find the buffer view
const auto& bufferView = model.bufferViews[accessor.bufferView];

// 3. Find the buffer
const auto& buffer = model.buffers[bufferView.buffer];

// 4. Calculate the pointer
const float* positions = reinterpret_cast<const float*>(
    buffer.data.data() + bufferView.byteOffset + accessor.byteOffset
);

// 5. Read 'accessor.count' elements of type VEC3 (3 floats each)
```

### .gltf vs .glb

| Format | Structure | Use Case |
|--------|-----------|----------|
| `.gltf` | JSON file + separate `.bin` + image files | Development, debugging |
| `.glb` | Single binary file (JSON + binary chunks) | Distribution, faster loading |

Both contain the same data, just packaged differently. tinygltf handles both transparently.

---

## The tinygltf Library

We use [tinygltf](https://github.com/syoyo/tinygltf) - a header-only glTF loader.

### Why tinygltf?

- **Header-only** - Same pattern as stb_image
- **Uses stb_image** - We already have it for texture loading
- **Well-tested** - Used by many projects
- **Complete** - Supports glTF 2.0 fully

### Basic Usage

```cpp
#include "tiny_gltf.h"

tinygltf::Model model;
tinygltf::TinyGLTF loader;
std::string err, warn;

// Load .glb (binary)
bool success = loader.LoadBinaryFromFile(&model, &err, &warn, "model.glb");

// Or load .gltf (JSON)
// bool success = loader.LoadASCIIFromFile(&model, &err, &warn, "model.gltf");

if (!warn.empty()) {
    VP_CORE_WARN("glTF warning: {}", warn);
}

if (!err.empty()) {
    VP_CORE_ERROR("glTF error: {}", err);
    return nullptr;
}

if (!success) {
    VP_CORE_ERROR("Failed to load glTF: {}", filepath);
    return nullptr;
}

// Now 'model' contains all the data!
```

---

## The Material Struct

Before loading models, we need a way to represent materials:

```cpp
// VizEngine/Core/Material.h

struct VizEngine_API Material
{
    std::string Name = "Unnamed";
    
    // PBR properties
    glm::vec4 BaseColor = glm::vec4(1.0f);    // RGBA
    float Metallic = 0.0f;                     // 0 = dielectric, 1 = metal
    float Roughness = 0.5f;                    // 0 = smooth, 1 = rough
    
    // Textures (nullptr if not present)
    std::shared_ptr<Texture> BaseColorTexture = nullptr;
    std::shared_ptr<Texture> MetallicRoughnessTexture = nullptr;
    std::shared_ptr<Texture> NormalTexture = nullptr;
    
    // Helper to check if material has textures
    bool HasBaseColorTexture() const { return BaseColorTexture != nullptr; }
};
```

### PBR: Metallic-Roughness Workflow

glTF uses the **metallic-roughness** PBR workflow:

| Property | Range | Meaning |
|----------|-------|---------|
| **Base Color** | RGBA | Albedo/diffuse color |
| **Metallic** | 0-1 | 0 = plastic/wood, 1 = metal |
| **Roughness** | 0-1 | 0 = mirror-smooth, 1 = completely rough |

Why PBR?
- **Physically accurate** - Materials look correct under any lighting
- **Artist-friendly** - Intuitive parameters
- **Consistent** - Same material looks good in any engine

---

## The Model Class

A container for loaded model data:

```cpp
// VizEngine/Core/Model.h

class VizEngine_API Model
{
public:
    // Factory method - the main way to create models
    static std::unique_ptr<Model> LoadFromFile(const std::string& filepath);
    
    // Access loaded data
    const std::vector<std::shared_ptr<Mesh>>& GetMeshes() const { return m_Meshes; }
    const std::vector<Material>& GetMaterials() const { return m_Materials; }
    
    // Get material for a specific mesh
    const Material& GetMaterialForMesh(size_t meshIndex) const;
    
    // Model info
    const std::string& GetName() const { return m_Name; }
    size_t GetMeshCount() const { return m_Meshes.size(); }
    
private:
    Model() = default;  // Only LoadFromFile can create
    
    std::string m_Name;
    std::string m_Directory;  // For resolving relative texture paths
    
    std::vector<std::shared_ptr<Mesh>> m_Meshes;
    std::vector<Material> m_Materials;
    std::vector<size_t> m_MeshMaterialIndices;  // Which material for each mesh
    
    // Texture cache to avoid reloading
    std::unordered_map<std::string, std::shared_ptr<Texture>> m_TextureCache;
};
```

---

## Loading Geometry

The core of model loading - extracting vertex data from glTF:

```cpp
std::unique_ptr<Model> Model::LoadFromFile(const std::string& filepath)
{
    tinygltf::Model gltfModel;
    tinygltf::TinyGLTF loader;
    std::string err, warn;

    // Determine file type and load
    bool success = false;
    if (filepath.ends_with(".glb")) {
        success = loader.LoadBinaryFromFile(&gltfModel, &err, &warn, filepath);
    } else {
        success = loader.LoadASCIIFromFile(&gltfModel, &err, &warn, filepath);
    }

    if (!success) {
        VP_CORE_ERROR("Failed to load model: {}", filepath);
        return nullptr;
    }

    auto model = std::unique_ptr<Model>(new Model());
    model->m_Name = filepath;
    model->m_Directory = GetDirectoryFromPath(filepath);

    // Load all materials first
    model->LoadMaterials(gltfModel);

    // Load all meshes
    model->LoadMeshes(gltfModel);

    return model;
}
```

### Extracting Vertex Data

```cpp
void Model::LoadMeshes(const tinygltf::Model& gltfModel)
{
    for (const auto& gltfMesh : gltfModel.meshes)
    {
        for (const auto& primitive : gltfMesh.primitives)
        {
            std::vector<Vertex> vertices;
            std::vector<unsigned int> indices;

            // Get accessors
            const auto& posAccessor = gltfModel.accessors[
                primitive.attributes.at("POSITION")
            ];
            
            // Position data
            const float* positions = GetBufferData<float>(
                gltfModel, posAccessor
            );

            // Normal data (may not exist)
            const float* normals = nullptr;
            if (primitive.attributes.count("NORMAL")) {
                const auto& normAccessor = gltfModel.accessors[
                    primitive.attributes.at("NORMAL")
                ];
                normals = GetBufferData<float>(gltfModel, normAccessor);
            }

            // Texture coordinates (may not exist)
            const float* texCoords = nullptr;
            if (primitive.attributes.count("TEXCOORD_0")) {
                const auto& uvAccessor = gltfModel.accessors[
                    primitive.attributes.at("TEXCOORD_0")
                ];
                texCoords = GetBufferData<float>(gltfModel, uvAccessor);
            }

            // Build vertices
            for (size_t i = 0; i < posAccessor.count; i++)
            {
                Vertex v;
                
                // Position (always present)
                v.Position = glm::vec4(
                    positions[i * 3 + 0],
                    positions[i * 3 + 1],
                    positions[i * 3 + 2],
                    1.0f
                );

                // Normal (default to up if missing)
                if (normals) {
                    v.Normal = glm::vec3(
                        normals[i * 3 + 0],
                        normals[i * 3 + 1],
                        normals[i * 3 + 2]
                    );
                } else {
                    v.Normal = glm::vec3(0.0f, 1.0f, 0.0f);
                }

                // Texture coordinates (default to 0,0 if missing)
                if (texCoords) {
                    v.TexCoords = glm::vec2(
                        texCoords[i * 2 + 0],
                        texCoords[i * 2 + 1]
                    );
                } else {
                    v.TexCoords = glm::vec2(0.0f);
                }

                // Color (white by default)
                v.Color = glm::vec4(1.0f);

                vertices.push_back(v);
            }

            // Load indices
            if (primitive.indices >= 0) {
                const auto& indexAccessor = gltfModel.accessors[primitive.indices];
                LoadIndices(gltfModel, indexAccessor, indices);
            }

            // Create mesh and store
            auto mesh = std::make_shared<Mesh>(vertices, indices);
            m_Meshes.push_back(mesh);
            m_MeshMaterialIndices.push_back(
                primitive.material >= 0 ? primitive.material : 0
            );
        }
    }
}
```

### Helper: Getting Buffer Data

```cpp
template<typename T>
const T* Model::GetBufferData(
    const tinygltf::Model& model, 
    const tinygltf::Accessor& accessor)
{
    const auto& bufferView = model.bufferViews[accessor.bufferView];
    const auto& buffer = model.buffers[bufferView.buffer];
    
    return reinterpret_cast<const T*>(
        buffer.data.data() + bufferView.byteOffset + accessor.byteOffset
    );
}
```

### Handling Different Index Types

glTF can use different integer types for indices:

```cpp
void Model::LoadIndices(
    const tinygltf::Model& model,
    const tinygltf::Accessor& accessor,
    std::vector<unsigned int>& indices)
{
    const auto& bufferView = model.bufferViews[accessor.bufferView];
    const auto& buffer = model.buffers[bufferView.buffer];
    const void* dataPtr = buffer.data.data() + bufferView.byteOffset + accessor.byteOffset;

    indices.reserve(accessor.count);

    switch (accessor.componentType)
    {
        case TINYGLTF_COMPONENT_TYPE_UNSIGNED_SHORT:
        {
            const uint16_t* buf = static_cast<const uint16_t*>(dataPtr);
            for (size_t i = 0; i < accessor.count; i++)
                indices.push_back(buf[i]);
            break;
        }
        case TINYGLTF_COMPONENT_TYPE_UNSIGNED_INT:
        {
            const uint32_t* buf = static_cast<const uint32_t*>(dataPtr);
            for (size_t i = 0; i < accessor.count; i++)
                indices.push_back(buf[i]);
            break;
        }
        case TINYGLTF_COMPONENT_TYPE_UNSIGNED_BYTE:
        {
            const uint8_t* buf = static_cast<const uint8_t*>(dataPtr);
            for (size_t i = 0; i < accessor.count; i++)
                indices.push_back(buf[i]);
            break;
        }
    }
}
```

---

## Loading Materials and Textures

### Extracting Material Properties

```cpp
void Model::LoadMaterials(const tinygltf::Model& gltfModel)
{
    for (const auto& gltfMat : gltfModel.materials)
    {
        Material material;
        material.Name = gltfMat.name;

        // PBR Metallic-Roughness
        const auto& pbr = gltfMat.pbrMetallicRoughness;
        
        material.BaseColor = glm::vec4(
            pbr.baseColorFactor[0],
            pbr.baseColorFactor[1],
            pbr.baseColorFactor[2],
            pbr.baseColorFactor[3]
        );
        
        material.Metallic = static_cast<float>(pbr.metallicFactor);
        material.Roughness = static_cast<float>(pbr.roughnessFactor);

        // Base color texture
        if (pbr.baseColorTexture.index >= 0)
        {
            material.BaseColorTexture = LoadTexture(
                gltfModel, 
                pbr.baseColorTexture.index
            );
        }

        // Metallic-roughness texture (R=unused, G=roughness, B=metallic)
        if (pbr.metallicRoughnessTexture.index >= 0)
        {
            material.MetallicRoughnessTexture = LoadTexture(
                gltfModel,
                pbr.metallicRoughnessTexture.index
            );
        }

        // Normal map
        if (gltfMat.normalTexture.index >= 0)
        {
            material.NormalTexture = LoadTexture(
                gltfModel,
                gltfMat.normalTexture.index
            );
        }

        m_Materials.push_back(material);
    }

    // Ensure at least one default material
    if (m_Materials.empty())
    {
        m_Materials.push_back(Material{});
    }
}
```

### Loading Textures

glTF textures can be embedded in the file or referenced externally:

```cpp
std::shared_ptr<Texture> Model::LoadTexture(
    const tinygltf::Model& gltfModel,
    int textureIndex)
{
    const auto& texture = gltfModel.textures[textureIndex];
    const auto& image = gltfModel.images[texture.source];

    // Check cache first
    if (!image.uri.empty() && m_TextureCache.count(image.uri))
    {
        return m_TextureCache[image.uri];
    }

    std::shared_ptr<Texture> tex;

    if (!image.image.empty())
    {
        // Embedded texture - data is already loaded
        tex = std::make_shared<Texture>(
            image.image.data(),
            image.width,
            image.height,
            image.component  // Number of channels
        );
    }
    else if (!image.uri.empty())
    {
        // External texture - load from file
        std::string fullPath = m_Directory + "/" + image.uri;
        tex = std::make_shared<Texture>(fullPath);
        m_TextureCache[image.uri] = tex;
    }

    return tex;
}
```

---

## Integration with Scene

### Using Model with SceneObject

```cpp
// Load a model
auto helmetModel = Model::LoadFromFile("assets/helmet.glb");

if (helmetModel)
{
    // Add each mesh to the scene
    for (size_t i = 0; i < helmetModel->GetMeshCount(); i++)
    {
        auto& obj = scene.AddObject(helmetModel->GetMeshes()[i]);
        obj.ObjectTransform.Position = glm::vec3(0.0f, 2.0f, 0.0f);
        
        // Store material reference for rendering
        // (You might extend SceneObject to include material)
    }
}
```

### Extended SceneObject (Optional)

To properly support materials, you might extend `SceneObject`:

```cpp
struct SceneObject
{
    std::shared_ptr<Mesh> MeshPtr;
    Transform ObjectTransform;
    glm::vec4 Color = glm::vec4(1.0f);
    bool Active = true;
    
    // NEW: Material support
    Material ObjectMaterial;  // Copy of material properties
    // Or: const Material* MaterialPtr;  // Reference to shared material
};
```

---

## Updating the Shader

To use material properties, update your shader:

### Vertex Shader (unchanged from lighting)

```glsl
#version 460 core

layout (location = 0) in vec4 aPos;
layout (location = 1) in vec3 aNormal;
layout (location = 2) in vec4 aColor;
layout (location = 3) in vec2 aTexCoord;

out vec3 v_FragPos;
out vec3 v_Normal;
out vec2 v_TexCoord;

uniform mat4 u_Model;
uniform mat4 u_MVP;

void main()
{
    v_FragPos = vec3(u_Model * aPos);
    v_Normal = mat3(transpose(inverse(u_Model))) * aNormal;
    v_TexCoord = aTexCoord;
    gl_Position = u_MVP * aPos;
}
```

### Fragment Shader (with material uniforms)

```glsl
#version 460 core

out vec4 FragColor;

in vec3 v_FragPos;
in vec3 v_Normal;
in vec2 v_TexCoord;

// Light
uniform vec3 u_LightDirection;
uniform vec3 u_LightColor;

// Camera
uniform vec3 u_ViewPos;

// Material (from glTF)
uniform vec4 u_BaseColor;
uniform float u_Metallic;
uniform float u_Roughness;
uniform sampler2D u_BaseColorTex;
uniform bool u_HasBaseColorTex;

void main()
{
    // Get base color
    vec4 baseColor = u_BaseColor;
    if (u_HasBaseColorTex) {
        baseColor *= texture(u_BaseColorTex, v_TexCoord);
    }
    
    // Simple Blinn-Phong (could upgrade to full PBR later)
    vec3 norm = normalize(v_Normal);
    vec3 lightDir = normalize(-u_LightDirection);
    vec3 viewDir = normalize(u_ViewPos - v_FragPos);
    vec3 halfDir = normalize(lightDir + viewDir);
    
    // Ambient
    vec3 ambient = 0.1 * baseColor.rgb;
    
    // Diffuse
    float diff = max(dot(norm, lightDir), 0.0);
    vec3 diffuse = diff * baseColor.rgb * u_LightColor;
    
    // Specular (roughness affects shininess)
    float shininess = mix(256.0, 8.0, u_Roughness);
    float spec = pow(max(dot(norm, halfDir), 0.0), shininess);
    // Metals have colored specular, dielectrics have white
    vec3 specColor = mix(vec3(0.04), baseColor.rgb, u_Metallic);
    vec3 specular = spec * specColor * u_LightColor;
    
    vec3 result = ambient + diffuse + specular;
    FragColor = vec4(result, baseColor.a);
}
```

### Setting Material Uniforms

```cpp
void RenderWithMaterial(Shader& shader, const Material& mat)
{
    shader.SetVec4("u_BaseColor", mat.BaseColor);
    shader.SetFloat("u_Metallic", mat.Metallic);
    shader.SetFloat("u_Roughness", mat.Roughness);
    
    bool hasBaseTex = mat.HasBaseColorTexture();
    shader.SetBool("u_HasBaseColorTex", hasBaseTex);
    
    if (hasBaseTex) {
        mat.BaseColorTexture->Bind(0);
        shader.SetInt("u_BaseColorTex", 0);
    }
}
```

---

## Where to Find Test Models

Free glTF models for testing:

| Source | URL | Notes |
|--------|-----|-------|
| **Khronos Sample Models** | github.com/KhronosGroup/glTF-Sample-Models | Official test models |
| **Sketchfab** | sketchfab.com | Many free models (look for CC license) |
| **Poly Haven** | polyhaven.com | Free HDRIs, textures, and models |
| **glTF Viewer** | gltf-viewer.donmccurdy.com | Test models render correctly |

### Recommended Test Models

1. **Box** - Simplest possible model
2. **BoxTextured** - Box with a texture
3. **Suzanne** - Blender monkey (good for testing normals)
4. **DamagedHelmet** - PBR showcase model
5. **FlightHelmet** - Complex multi-mesh model

---

## Common Issues

### Model Appears Black

**Causes:**
1. Normals not loaded or inverted
2. Material not applied
3. Light not set up

**Debug:**
```cpp
// Temporarily use normal as color
FragColor = vec4(v_Normal * 0.5 + 0.5, 1.0);
```

### Model Has Wrong Scale

glTF uses meters. If your model is tiny or huge:
```cpp
transform.Scale = glm::vec3(0.01f);  // If model is in centimeters
transform.Scale = glm::vec3(100.0f); // If model is too small
```

### Textures Not Loading

**Causes:**
1. Wrong texture path (relative vs absolute)
2. Image format not supported
3. Texture index out of bounds

**Debug:**
```cpp
VP_CORE_INFO("Loading texture: {} ({}x{})", 
    image.uri, image.width, image.height);
```

### Performance Issues

**Symptoms:** Slow loading, stuttering

**Solutions:**
1. Use `.glb` instead of `.gltf` (single file, no extra I/O)
2. Cache textures across models
3. Load models asynchronously (future topic)

---

## Key Takeaways

1. **glTF is the modern standard** - PBR materials, compact, well-supported
2. **Accessors → BufferViews → Buffers** - The data access chain
3. **Materials are PBR** - BaseColor, Metallic, Roughness
4. **tinygltf makes it easy** - Header-only, handles both .gltf and .glb
5. **Handle missing data gracefully** - Not all models have normals, textures, etc.
6. **Cache textures** - Avoid reloading the same texture multiple times
7. **Test with sample models** - Khronos provides official test assets

---

## Exercise

1. **Load a simple model** - Try the Box or BoxTextured sample
2. **Display model info** - Show mesh count, material names in ImGui
3. **Handle multi-mesh models** - Load FlightHelmet (multiple meshes)
4. **Implement texture caching** - Share textures across models
5. **Add model browser** - File picker to load any .glb file

---

> **Reference:** For the complete class diagram and file locations, see [Appendix A: Code Reference](A_Reference.md).


