---
title: 点光源(Specialization Constant)以及变换
tags:
  - Vulkan
  - 渲染
  - 变换
  - Specialization Constant
category:
  - ElbowEngine
typora-root-url: ./..
abbrlink: '539e8922'
date: 2024-08-04 10:52:59
cover: ./imgs/post/点光源-Specialization-Constant-以及变换}/2024-08-04_12-08-23.gif
---

## 一、前言

由于之前ElbowEngine的代码一直都是在做重构（Vulkan官方教程直接写一起了，没有可复用性而言），因此没有增加新东西，也就没写东西，如今重构终于完成，也写了点东西，终于能~~水篇文章了~~。

这次带来的是在Vulkan中实现点光源和变换（也就是说子GameObject的变换会跟着父GameObject的变换一起变），至于为什么讲这么两个相差很大的东西，因为变换这块感觉没法单开一篇文章~~（至少目前为止）~~。

那么这篇文章讲介绍：

+ Vulkan中的Specilization Constant
+ 如何在Vulkan中实现点光源
+ 如何实现一个高效的变换系统

上一篇：[Vulkan应用集成ImGui | 喜多喜多のBlog (kita-blog.vercel.app)](https://kita-blog.vercel.app/posts/8e451ae.html)

本篇：[点光源(Specialization Constant)以及变换 | 喜多喜多のBlog (kita-blog.vercel.app)](https://kita-blog.vercel.app/posts/539e8922.html)

## 一、当前系统组织简介

让我们在正式开始前先介绍一下当前已有的系统。

![image-20240804110926258](./imgs/post/点光源-Specialization-Constant-以及变换}/image-20240804110926258.png)

我将ElbowEngine划分为了五个模块（根据GAMES104）来划分。

+ Core: 负责最最基础的部分，例如最基础的数据结构、序列化和反序列化、反射、日志、字符串、事件系统、数学系统、Object等
+ Platform: 这部分~~计划~~用来处理不同平台的部分，当前只使用了Vulkan一种API，不排除之后会加入其他的图像API，其中的RHI层封装了RenderPass、Texture等实用结构
+ Resource: 这部分用于管理项目中的资产，所有对于模型、纹理等文件的加载与保存都需要这一层暴露的接口进行。
+ Function: 实现功能的地方、例如渲染功能、变换、GameObject、Component等，预计未来可能加入的动画、AI都将在这里实现
+ Tool: 这里主要实现各种Editor UI

此外，还包含一个使用libtooling写的代码生成器，用于自动化生成反射代码。

文件夹结构为大家奉上！

![image-20240804111820275](./imgs/post/点光源-Specialization-Constant-以及变换}/image-20240804111820275.png)

里面细节的东西会在后面的文章慢慢为大家介绍。那么事不宜迟，开启我们今天的重点吧~

## 二、点光源与SpecializationConstant

这部分代码在[RickSchanze/ElbowEngine at 点光源与SpecializationConstant (github.com)](https://github.com/RickSchanze/ElbowEngine/tree/点光源与SpecializationConstant)

有光才能有生机！为了引入光，我们需要定义一系列光源并完成他们的功能，这里选择首先定义点光源，是因为learn opengl中最先介绍的是点光源，也有经典的冯氏光照实现，参考[光照基础 - LearnOpenGL-CN](https://learnopengl-cn.readthedocs.io/zh/latest/02 Lighting/02 Basic Lighting/)，这样我们就只需要关心如何处理Vulkan就行了。

### 2.1 实现

首先我们就遇到了一个问题，我们需要多少个点光源？经过一番查阅资料，我发现Vulkan有个叫做specilization constant，可以让我们在创建管线的时候指定相应的常量，非常好，让我们看看写法怎么写：

我们在shader中定义：

```glsl
layout(constant_id = 0) const int MAX_POINT_LIGHTS = 1;

struct DirectionalLight {
    vec4 position;
    vec4 color;
};

layout(binding = 3) uniform PointLights {
    DirectionalLight lights[MAX_POINT_LIGHTS];
} ubo_point_lights;
```

`MAX_POINT_LIGHTS`就是specialization constant，为了让它生效，我们在C++创建管线的时候需要定义对应的数据：

```C++
    // 配置frag shader的最大光照信息
    TStaticArray<vk::SpecializationMapEntry, 1> specializations;
    specializations[0].constantID = 0;
    specializations[0].size = sizeof(int32_t);
    specializations[0].offset = 0;

    vk::SpecializationInfo specialization_info;
    specialization_info.dataSize = sizeof(int32_t);
    specialization_info.setMapEntries(specializations);
    specialization_info.pData = &g_engine_statistics.graphics.max_point_light_count;

    vk::PipelineShaderStageCreateInfo frag_info{};
    frag_info.setStage(vk::ShaderStageFlagBits::eFragment).setModule(shader_program_->GetFragShaderHandle()).setPName("main");
    frag_info.setPSpecializationInfo(&specialization_info);
    TStaticArray<vk::PipelineShaderStageCreateInfo, 2> shader_stages = {vert_info, frag_info};
```

定义起来很简单，然后我们传入VkPipelineShaderStageCreateInfo即可。

写Buffer的时候与平常用的Buffer没有什么不同，先MapMemory然后Flush即可，这里我介绍一下我如何封装这个Map的过程：

### 2.2 对于Shader、ShaderProgram和Material的封装

首先我们需要集成Platform::RHI::Shader，实现`RegisterShaderVariables`函数，在里面声明所有的Uniform Buffer:

```C++
// StandardForwardShader.h
class StandardForwardVertShader : public RHI::Vulkan::Shader
{
    DECLARE_VERT_SHADER(StandardForwardVertShader)

public:
    void RegisterShaderVariables() override;
};

class StandardForwardFragShader : public RHI::Vulkan::Shader
{
    DECLARE_FRAG_SHADER(StandardForwardFragShader)

public:
    void RegisterShaderVariables() override;
};
```

```c++
// StandardForwardShader.cpp
void StandardForwardVertShader::RegisterShaderVariables()
{
    REGISTER_SHADER_VAR_BEGIN(0)
    REGISTER_VERT_SHADER_VAR_AUTO_StaticUniformBuffer("ubo_view", 3 * sizeof(glm::mat4), 0);
    REGISTER_VERT_SHADER_VAR_AUTO_DynamicUniformBuffer(
        "ubo_instance", sizeof(glm::mat4) * g_engine_statistics.graphics.max_dynamic_model_uniform_buffer_count, sizeof(glm::mat4)
    );
    REGISTER_SHADER_VAR_END
}

void StandardForwardFragShader::RegisterShaderVariables()
{
    REGISTER_SHADER_VAR_BEGIN(2)
    REGISTER_FRAG_SHADER_VAR_AUTO_Sampler2D("texSampler");
    REGISTER_FRAG_SHADER_VAR_AUTO_StaticUniformBuffer("ubo_point_lights", 64 * g_engine_statistics.graphics.max_point_light_count);
    REGISTER_SHADER_VAR_END
}
```

在这里我们声明了StandardForwardShader的顶点Shader和片元Shader，为顶点shader生命了ubo_view这个结构：

```glsl
layout(binding = 0) uniform UboView {
    mat4 projection;
    mat4 view;
    mat4 camera;
} ubo_view;
```

projection和view代表投影矩阵和视图矩阵，camera是我们自定义的数据，目前指引摄像机的位置：

```C++
glm::mat4 Transform::ToGPUMat4()
{
    auto rtn = glm::mat4(1.0f);
    rtn[0]   = Math::ToVector4(GetPosition());
    return rtn;
}
```

它的第一行是摄像机的位置，当然这里在Transform里生命有些不合理，后面重构时会更正，其他位置当前都没有意义，如果需要更多的相机参数会继续往这个Camera里填充，使用mat4主要是为了数据对齐（可见[c++ - Vulkan memory alignment requirements - Stack Overflow](https://stackoverflow.com/questions/45458918/vulkan-memory-alignment-requirements)）。

#### 2.2.1 dynamic uniform buffer

然后我们生命了表示每一个物体的模型矩阵的变量ubo_instance，它是一个dynamic uniform buffer，我们在数据准备阶段填充这个uniform buffer，然后在bind descriptor传入对应的索引就可以：

```C++
// RenderPipeline.cpp
void RenderPipeline::PrepareModelUniformBuffer()
{
    ...
    for (int i = 0; i < meshes.size(); ++i)
    {
        auto& mesh                 = meshes[i];
        auto& mesh_transform       = mesh->GetTransform();
        model_instances_.models[i] = mesh_transform.ToMat4();
    }
}
// LiteForwardRenderPipeline.cpp
void LiteForwardRenderPipeline::Draw(const RenderContextDrawParam& draw_param)
{
    Comp::Camera* main = Comp::Camera::Main;
    // 这里将所有的模型矩阵的信息填充
    Super::Draw(draw_param);
    auto cb = context_->BeginRecordCommandBuffer();
    forward_pass_->Begin(cb, main->background_color);
    auto meshes_to_draw = CollectMeshesWithMaterial();
    for (auto& [material, meshes]: meshes_to_draw)
    {
        material->Use(cb);
        material->SetPostionViewProjection(main);
        // 这里映射到GPU
        material->SetModel(model_instances_.models, model_instances_.size);
        for (int i = 0; i < meshes.size(); i++)
        {
            TArray dynamic_offsets = {i * static_cast<uint32_t>(GetDynamicUniformModelAligment())};
            // 这里传入它的偏移
            material->DrawMesh(cb, *meshes[i], dynamic_offsets);
        }
    }
	...
    Submit(submit_params, draw_param.render_end_fence);
}
// Material.cpp
void Material::DrawMesh(vk::CommandBuffer cb, const Comp::Mesh& mesh, const TArray<uint32_t>& dynamic_offsets)
{
    if (pipeline_ == nullptr)
    {
        return;
    }
    pipeline_->BindPipeline(cb);
    for (auto& mesh_to_draw: mesh.GetSubMeshes())
    {
        pipeline_->BindMesh(cb, *mesh_to_draw.GetRHIResource());
        // 这里绑定了dynamic uniform buffer的偏移
        pipeline_->BindDescriptiorSets(cb, {pipeline_->GetCurrentFrameDescriptorSet()}, vk::PipelineBindPoint::eGraphics, 0, dynamic_offsets);
        pipeline_->DrawIndexed(cb, mesh_to_draw.GetIndices().size());
    }
}
```

这部分的内容可见vulkan示例项目[Vulkan/examples/dynamicuniformbuffer at master · SaschaWillems/Vulkan (github.com)](https://github.com/SaschaWillems/Vulkan/tree/master/examples/dynamicuniformbuffer)或者我项目里分支也可以。

之后再frag shader中我们定义光照的buffer大小和Sampler2d的采样器。

#### 2.2.2 ShaderProgram

ShaderProgram接受一个vert shader和一个frag shader为参数，会自动解析顶点着色器的input数据并绑定到管线，以及为所有的uniform buffer生成对应的CPU buffer，并提供了Map memory的方法

```C++
// ShaderProgram.h
class ShaderProgram {
public:
    ...
   	static ShaderProgram*
    Create(Shader* vert, Shader* frag, const EShaderDestroyTime destroy_time = EShaderDestroyTime::Defered, const AnsiString& debug_name = "")
    {
        Ref   device  = *VulkanContext::Get()->GetLogicalDevice();
        auto* program = new ShaderProgram(device, vert, frag, destroy_time, debug_name);
        return program;
    }
    
        // 设置纹理
    bool SetTexture(const AnsiString& name, const ImageView& view, const Sampler& sampler);
    void SetUniformBuffer(const AnsiString& name, const void* data, size_t size);
private:
    THashMap<AnsiString, TArray<Buffer*>> uniform_buffers_;
    ...
}
// ShaderProgram.cpp
...
void ShaderProgram::SetUniformBuffer(const AnsiString& name, const void* data, size_t size)
{
    if (!uniform_buffers_.contains(name))
    {
        LOG_ERROR_ANSI_CATEGORY(Vulkan, "ShaderProgram::SetStaticUniformBuffer: UniformBuffer {} not found", name);
        return;
    }
    auto& buffers = uniform_buffers_[name];
    for (size_t i = 0; i < g_engine_statistics.graphics.parallel_render_frame_count; i++)
    {
        auto& buffer = buffers[i];
        buffer->MapMemory();
        auto* map_res = buffer->GetMappedCpuMemory();
        memcpy(map_res, data, size);
        buffer->FlushMemory();
    }
}
...
```

有了ShaderProgram就可以供上层Material调用。

#### 2.2.3 Material

Material包含了一个ShaderProgram以及其对应的GraphicsPipeline, 同时还封装了各种实用方法：

```c++
class Material : public Object, public IDetailGUIDrawer
{
public:
    // TODO: 这里应该是传入两个Shader而不是两个路径
    Material(const Path& vert, const Path& frag, const String& name);

    ~Material() override;

    void SetTexture(const AnsiString& name, Resource::Texture* texture);
    void SetTexture(const AnsiString&name, const Path& path);
    void Use(vk::CommandBuffer cb, uint32_t width = 0, uint32_t height = 0, int x = 0, int y = 0) const;
    void SetPostionViewProjection(Comp::Camera* camera);
    void SetModel(glm::mat4* models, size_t size);
    void SetPointLights(void* data, size_t size);
    void DrawMesh(vk::CommandBuffer cb, const Comp::Mesh& mesh, const TArray<uint32_t>& dynamic_offsets);
public:
    // 用于IMGUI绘制
    void OnInspectorGUI() override;
private:
    RHI::Vulkan::ShaderProgram*    shader_program_ = nullptr;
    RHI::Vulkan::GraphicsPipeline* pipeline_       = nullptr;

    // 存储shader里所有的纹理参数
    THashMap<AnsiString, Resource::Texture*> textures_maps_;

    bool texture_map_dirty_ = false;
};
```

有了这些，我们可以继续看RenderPipeline，这是一个用于控制渲染流程的类：

#### 2.2.4 RenderPipeline

```c++
// RenderPipeline.h
class RenderPipeline
{
public:
    RenderPipeline();

    virtual ~RenderPipeline();

    virtual void Draw(const RenderContextDrawParam& draw_param);
    virtual void Build() = 0;

protected:
    size_t GetDynamicUniformModelAligment() const;
    void PrepareFrame();
    vk::Semaphore Submit(const RHI::Vulkan::GraphicsQueueSubmitParams& submit_params, vk::Fence fence_to_trigger = nullptr) const;
    void AddImGuiGraphicsPipeline();
    void DrawImGuiPipeline(vk::CommandBuffer cb) const;
    
    RenderContext*                      context_        = nullptr;
    RHI::Vulkan::ImguiGraphicsPipeline* imgui_pipeline_ = nullptr;

    UBOModelInstance model_instances_;

protected:
    // 每帧处理的需要绘制的mesh
    THashMap<Material*, TArray<Comp::Mesh*>> draw_meshs_;
};
```

我们需要集成这个类，实现Draw和Build,Build用于创建所有需要的RenderPass Draw用于控制RenderPass的绘制流程。

此外这个类还包含了方便的添加ImGui绘制管线的方法，分别在Build和Draw中调用AddImGuiGraphicsPipeline和DrawImGuiPipeline即可。

所有可能需要绘制的Mesh对象都会上传到这个类里，并且以Material分类，也就是说同一Material的多个对象绘制起来会比较快。

后续的剔除将会在PrepareFrame中做，等到剔除完成后我会发一篇文章讲一讲。

下面我们来看看实现RenderPipeline的一个例子：

```C++
// LiteForwardRenderPipeline.h
class LiteForwardRenderPipeline : public RenderPipeline {
public:
    typedef RenderPipeline Super;

    void Draw(const RenderContextDrawParam& draw_param) override;
    void Build() override;

private:
    RHI::Vulkan::RenderPass* forward_pass_ = nullptr;
};

// LiteForwardRenderPipeline.cpp
void LiteForwardRenderPipeline::Draw(const RenderContextDrawParam& draw_param)
{
    Comp::Camera* main = Comp::Camera::Main;
    Super::Draw(draw_param);
    auto cb = context_->BeginRecordCommandBuffer();
    forward_pass_->Begin(cb, main->background_color);

    auto meshes_to_draw = CollectMeshesWithMaterial();

    for (auto& [material, meshes]: meshes_to_draw)
    {
        material->Use(cb);
        material->SetPostionViewProjection(main);
        material->SetModel(model_instances_.models, model_instances_.size);
        for (int i = 0; i < meshes.size(); i++)
        {
            TArray dynamic_offsets = {i * static_cast<uint32_t>(GetDynamicUniformModelAligment())};
            material->DrawMesh(cb, *meshes[i], dynamic_offsets);
        }
    }

    forward_pass_->End(cb);
    DrawImGuiPipeline(cb);
    context_->EndRecordCommandBuffer();

    GraphicsQueueSubmitParams submit_params;
    submit_params.semaphores_to_wait = {draw_param.render_begin_semaphore};
    submit_params.wait_stages        = {vk::PipelineStageFlagBits::eColorAttachmentOutput};

    Submit(submit_params, draw_param.render_end_fence);
}

void LiteForwardRenderPipeline::Build()
{
    forward_pass_ = RenderPassManager::GetOrCreateRenderPass<RenderPass>("SimpleForwardPass");

    AddImGuiGraphicsPipeline();
}
```

其实就是很简单的为每一个对象进行一次draw call，后续可能会加入instancing drawing。

### 2.3 总结

在封装好的基础上添加功能不是很难，这也是我之前要大费周章重构的原因。

让我们来看看效果:

![2024-08-04_12-08-23 (1)](./imgs/post/点光源-Specialization-Constant-以及变换}/2024-08-04_12-08-23 (1).gif)

看起来光照非常粗糙，不过这只是第一歩！

## 三、变换与管道风格的数学运算

### 3.1 变换的实现

我们在变换父对象和子对象时，不能简单的设置一次就更新一次，因为一帧里可能有大量的变换操作，一设置就更新会大量重复操作，因此我们需要将一帧分为多个阶段、在帧结束时对需要进行变换的对象再进行计算。

因此我将GameObject的Tick和Component的Tick分为PreTick、Tick和PostTick三个阶段：

```c++
// Component.h
class REFL Component : public Object
{
    GENERATED_BODY(Component)
public:
    friend class GameObject;
    Component();
    ~Component() override;

    virtual void PreTick(float DeltaTime) {}
    virtual void Tick(float DeltaTime) {}
    virtual void PostTick(float DeltaTime) {}
};

// GameObject.h
class GameObject : public Object
{
public:
    explicit GameObject(GameObject* InParent = nullptr);

    ~GameObject() override;

    // Tick所有Component
    void PreTickComponents(float delta_time);
    void TickComponents(float delta_time);
    void PostTickComponents(float delta_time);

    // GameObject本身做的Tick
    // 对transform的变化的实施会在PostTickObject函数里
    void PreTickObject(float delta_time);
    void TickObject(float delta_time);
    void PostTickObject(float delta_time);

    // 先TickComponent 再TickObject
    // PreTickComponents -> PreTickObject -> TickComponents -> TickObject -> PostTickComponents -> PostTickObject
    void PreTick(float delta_time);
    void Tick(float delta_time);
    void PostTick(float delta_time);
    
     /**
     * Tick所有GameObject
     * @param delta_time
     */
    static void TickObjects(float delta_time);
};
```

对于一帧的Tick，调用静态函数TickObjects对现有所有GameObject进行Tick，因此多个GameObject的Tick顺序无法保证。

```c++
void GameObject::TickObjects(float delta_time)
{
    const auto& objs = ObjectManager::Get()->GetAllObject();
    for (auto* obj: objs | std::views::values)
    {
        if (obj->IsGameObject())
        {
            static_cast<GameObject*>(obj)->PreTick(delta_time);
        }
    }
    for (auto* obj: objs | std::views::values)
    {
        if (obj->IsGameObject())
        {
            static_cast<GameObject*>(obj)->Tick(delta_time);
        }
    }
    for (auto* obj: objs | std::views::values)
    {
        if (obj->IsGameObject())
        {
            static_cast<GameObject*>(obj)->PostTick(delta_time);
        }
    }
}
```

总的来说GameObject的Tick是下面的顺序：

+ PreTick所有GameObject和Componment
+ Tick所有GameObject和Component
+ PostTick所有GameObject和Component

注意一下这里如果有两个GameObject，那么顺序可能是这样的

```C++
PreTick obj1 -> PreTick obj1的Components -> PreTick obj2 -> PreTick obj2的Component ->    // PreTick
Tick obj1 -> Tick obj1的Components -> Tick obj2 -> Tick obj2的Component ->                // Tick
PostTick obj1 -> PostTick obj1的Components -> PostTick obj2 ->PostTick obj2的Component -> // PostTick
```

我们的变换会在PostTick gameobjct阶段执行。

当设置一个transform的值时，会让gameobject的transform_dirty_标记为true, 在post tick时进行更新：

```c++
// Transform.cpp
void Transform::SetPosition(Vector3 pos, bool delay)
{
    position_ = pos;
    if (delay)
    {
        owner_->MarkTransformDirty();
    }
    else
    {
        owner_->ForceUpdateTransform();
    }
}

// GameObject.cpp
void GameObject::MarkTransformDirty()
{
    transform_dirty_ = true;
}

void GameObject::ForceUpdateTransform()
{
    ApplyTransformDeltas();
}

void GameObject::PostTickObject(float delta_time)
{
    if (transform_dirty_)
    {
        ApplyTransformDeltas();
    }
}
```

计算transform的变化时，递归的计算并保存世界坐标

```c++
// 计算
void GameObject::ApplyTransformDeltas()
{
    Transform parent_;
    if (parent_oject_ != nullptr)
    {
        parent_ = parent_oject_->transform_;
    }
    else
    {
        parent_ = Transform::Identity();
    }
    transform_.ApplyModify(parent_.world_position_, parent_.world_rotator_, parent_.world_scale_);
    for (auto* object: sub_game_objects_)
    {
        object->ApplyTransformDeltas();
    }
    transform_dirty_ = false;
}

// Transform.cpp
void Transform::ApplyModify(const Vector3& pos, const Rotator& rot, const Vector3& scale)
{
    world_position_ = position_ + pos;
    world_rotator_ = rotator_ + rot;
    world_scale_ = scale_ * scale;
    composited_mat_dirty_ = true;
}
```

当GPU需要获取变换的模型矩阵时，计算模型矩阵，当然是在有变化后、需要计算再计算：

```c++
Matrix4x4 Transform::GetMat4()
{
    if (composited_mat_dirty_)
    {
        composited_mat_       = GetMatrix4x4Identity();
        composited_mat_       = Math::Translate(composited_mat_, world_position_);
        composited_mat_       = Math::Rotate(composited_mat_, world_rotator_);
        composited_mat_       = Math::Scale(composited_mat_, world_scale_);
        composited_mat_dirty_ = false;
    }
    return composited_mat_;
}
```

我们这里大量使用了脏标记模式，这个模式能帮我们提升大量的性能。

### 3.2 管道风格数学运算

#### 3.2.1 原理

当我们想要让一个向量叉乘一个向量，然后求它的标准化向量，通常情况下应该怎么做？

```c++
Vector3 a;
Vector3 b = Math::Normalize(Math::Cross(a, Constant::ForwardVector));
```

当需要的计算操作较少时这种方法还可以，但是一旦计算操作多起来，我们需要嵌套很多层，要不就是很多个临时变量，很恼火。

刚好C++20容器库引入了ranges库，它的画风是这样的：

```c++
#include <iostream>
#include <ranges>
 
int main()
{
    auto const ints = {0, 1, 2, 3, 4, 5};
    auto even = [](int i) { return 0 == i % 2; };
    auto square = [](int i) { return i * i; };
 
    // the "pipe" syntax of composing the views:
    for (int i : ints | std::views::filter(even) | std::views::transform(square))
        std::cout << i << ' ';
 
    std::cout << '\n';
 
    // a traditional "functional" composing syntax:
    for (int i : std::views::transform(std::views::filter(ints, even), square))
        std::cout << i << ' ';
}
```

使用了 "|" 来连接和进行参数传递，我们之前的代码其实也用过这个，就`container | std::views::valus`可以帮我们过滤出来map的值，舍弃键。

那么我们当然可以对数学运算实现这样的操作！

```c++
template<typename T>
concept MathType = std::is_same_v<std::remove_cvref_t<T>, Vector3>;

class MathRanges
{
public:
    static auto Cross(Vector3 b)
    {
        return [&b](const Vector3& new_a) { return Math::Cross(new_a, b); };
    }

    static float Size(const Vector3 v) { return Math::Size(v); }

    static Vector3 Normalize(const Vector3 v) { return Math::Normalize(v); }
};

template<MathType T, typename F>
auto operator|(T&& value, F&& func)
{
    return func(value);
}
```

目前涉及的数学运算还不是很多，因此只写了一点。

我们重载了 `operator|` 让func以value进行调用，原理其实就这么简单。不过为了不和C++ ranges库里的起冲突，我加了一个concept MathType，当前指定Vector3才能参与重载。这样我们数学运算就可以变成：

```c++
Vector3 a;
float size = a | Cross(Constant::UpVector) | Normaize | Size;
```

瞬间优雅了不少！

注意这里返回了一个lambda函数

```c++
static auto Cross(Vector3 b)
{
    return [&b](const Vector3& new_a) { return Math::Cross(new_a, b); };
}
```

那么让我们来实践一下吧~

#### 3.2.2 实践

我们计划写一个SpaceCircle用于模拟空间圆形轨迹：

```c++
// SpaceCircle.h
class REFL SpaceCircle : public Component
{
    GENERATED_BODY(SpaceCircle)

public:
    SpaceCircle();

    void Tick(float DeltaTime) override'
    void PerformTranslate();

protected:
    PROPERTY(Serialized, Label = "半径")
    float radius_ = 20.f;

    PROPERTY(Serialized, Label = "圆心")
    Vector3 center_ = Vector3(0, 0, 0);

    PROPERTY(Serialized, Label = "速度", Getter = "GetSpeed", Setter = "SetSpeed")
    float speed_ = 1.f;

    PROPERTY(Serialized, Label = "旋转轴", Getter = "GetAxis", Setter = "SetAxis")
    Vector3 axis_ = Constant::UpVector;

    PROPERTY(Serialized, Label = "比例", Getter = "GetScale", Setter = "SetScale")
    float scale_ = 0.f;

    PROPERTY(Serialized, Label = "自动运行")
    bool auto_run_ = true;

    Vector3 a_;
    Vector3 b_;

    bool axis_dirty_ = true;
};
```

利用代码生成器可以生成一些反射代码并在GUI显示，非常好用！

```c++
void SpaceCircle::Tick(float DeltaTime)
{
    if (!auto_run_)
    {
        return;
    }
    scale_ = Math::Mod(scale_ + speed_ * DeltaTime, 1.f);
    PerformTranslate();
}

void SpaceCircle::PerformTranslate()
{
    float theta = 2.0f * Constant::PI * scale_;
    if (axis_dirty_)
    {
        Vector3 ta = Math::Cross(axis_, Constant::RightVector);
        if (Math::IsNearlyZero(ta))
        {
            a_ = axis_ | MathRanges::Cross(Constant::ForwardVector) | MathRanges::Normalize;
        }
        else
        {
            a_ = ta | MathRanges::Normalize;
        }
        b_          = axis_ | MathRanges::Cross(a_) | MathRanges::Normalize;
        axis_dirty_ = false;
    }
    float   cos = Math::Cos(theta);
    float   sin = Math::Sin(theta);
    Vector3 new_pos;
    new_pos.x = center_.x + radius_ * a_.x * cos + radius_ * b_.x * sin;
    new_pos.y = center_.y + radius_ * a_.y * cos + radius_ * b_.y * sin;
    new_pos.z = center_.z + radius_ * a_.z * cos + radius_ * b_.z * sin;
    game_object_->GetTransform().SetPosition(new_pos);
}
```

这里我们用到了MathRanges:

```C++
a_ = axis_ | MathRanges::Cross(Constant::ForwardVector) | MathRanges::Normalize;
a_ = ta | MathRanges::Normalize;
b_ = axis_ | MathRanges::Cross(a_) | MathRanges::Normalize;
```

写好Tick函数我们就可以开始测试了！
```c++
camera_object_ = New<Function::GameObject>(L"摄像机", nullptr);
camera_object_->AddComponent<Function::Comp::Camera>();
auto mesh_obj  = New<Function::GameObject>(L"AK-47_1");
auto mesh_comp = mesh_obj->AddComponent<Function::Comp::StaticMesh>();
mesh_comp->SetMesh(L"Models/AK47/AK47_CS2.fbx");
auto* mat = Function::MaterialManager::CreateMaterials(L"Shaders/vert.spv", L"Shaders/frag.spv", L"AK-47材质");
mesh_comp->SetMaterial(mat);
mat->SetTexture("texSampler", L"Models/AK47/ak47_default_color_psd_5b66a23b.png");
mesh_obj->AddComponent<Function::Comp::SpaceCircle>();

auto obj2 = New<Function::GameObject>(L"AK-47_2", mesh_obj);
obj2->AddComponent<Function::Comp::SpaceCircle>();
auto mesh_cmp = obj2->AddComponent<Function::Comp::StaticMesh>();
mesh_cmp->SetMesh(L"Models/AK47/AK47_CS2.fbx");
mesh_cmp->SetMaterial(mat);

auto* light_obj = New<Function::GameObject>(L"点光源");
light_obj->GetTransform().SetPosition(Vector3(0, 0, 10));
light_obj->AddComponent<Function::Comp::SpaceCircle>();
light_obj->AddComponent<Function::Comp::Light>();
```

我们为两把AK和点光源加上空间圆环轨迹的组件，开始测试！

下面是运行测试结果：

![2024-08-04_12-08-23](./imgs/post/点光源-Specialization-Constant-以及变换}/2024-08-04_12-08-23.gif

![2024-08-04_13-38-29](./imgs/post/点光源-Specialization-Constant-以及变换}/2024-08-04_13-38-29.gif)

两把AK-47以不同的圆心在运动，点光源也在做圆环运动，说明变换实现的还可以。

## 四、总结

本文我们介绍了点光源、变换和ElbowEngine本身的渲染架构，下一篇文章应该是实现点光源的阴影，敬请期待。

引用资料：

+ dynamic uniform buffer: [Vulkan/examples/dynamicuniformbuffer at master · SaschaWillems/Vulkan (github.com)](https://github.com/SaschaWillems/Vulkan/tree/master/examples/dynamicuniformbuffer)
+ specialization constant: [Vulkan/examples/specializationconstants at master · SaschaWillems/Vulkan (github.com)](https://github.com/SaschaWillems/Vulkan/tree/master/examples/specializationconstants)
+ 变换与脏标记模式: [脏标识模式 · Optimization Patterns · 游戏设计模式 (tkchu.me)](https://gpp.tkchu.me/dirty-flag.html)
+ 空间圆环的参数方程：[图解圆的参数方程_空间圆 参数方程-CSDN博客](https://blog.csdn.net/pathfinder1987/article/details/80707812)
+ 点光源在opengl的实现：[基础光照 - LearnOpenGL CN (learnopengl-cn.github.io)](https://learnopengl-cn.github.io/02 Lighting/02 Basic Lighting/)
