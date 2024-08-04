---
title: UGUI源码阅读(2)--Image的更新流程
tags:
  - UGUI
  - Unity
categories:
  - UGUI
typora-root-url: ./..
abbrlink: 79fdc967
date: 2024-05-27 08:56:26
cover: ./imgs/post/UGUI源码阅读-2-Image的更新流程/image-20240527094154320.png
---

# 前言

大家好！本篇文章是UGUI源码阅读文章的第二篇。

本篇文章主要介绍组件`Image`的更新流程，在本篇文章，我们将会了解：脏标记模式（设计模式）、Graphics类、GraphicsRegistry类、UI元素Rebuild流程等。

上一篇为：[UGUI源码阅读(1)--从Button开始 | 喜多喜多のBlog (kita-blog.vercel.app)](https://kita-blog.vercel.app/posts/5f2caef2.html)

下一篇为：[UGUI源码阅读（3）- Image网格顶点生成与实践 | 喜多喜多のBlog (kita-blog.vercel.app)](https://kita-blog.vercel.app/posts/6b2c390b.html)

# Image简单介绍

`Image`组件是UGUI中重要的用于显示图片的一个组件，如下：

![image-20240527094154320](/imgs/post/UGUI源码阅读-2-Image的更新流程/image-20240527094154320.png)

一个Image由不同的顶点构成，上图坐下为Simple情况下生成的顶点情况，右下为Tiled情况下生成的顶点情况。

# 当我们设置sprite时发生了什么？

我们设置图像时调用了下面的代码：

```C#
public Sprite sprite
{
    get
    {
        return m_Sprite;
    }
    set
    {
        if (m_Sprite != null)
        {
            if (m_Sprite != value)
            {
                m_SkipLayoutUpdate = m_Sprite.rect.size.Equals(value ? value.rect.size : Vector2.zero);
                m_SkipMaterialUpdate = m_Sprite.texture == (value ? value.texture : null);
                m_Sprite = value;

                ResetAlphaHitThresholdIfNeeded();
                SetAllDirty();
                TrackSprite();
            }
        }
        else if (value != null)
        {
            m_SkipLayoutUpdate = value.rect.size == Vector2.zero;
            m_SkipMaterialUpdate = value.texture == null;
            m_Sprite = value;

            ResetAlphaHitThresholdIfNeeded();
            SetAllDirty();
            TrackSprite();
        }

        void ResetAlphaHitThresholdIfNeeded()
        {
            if (!SpriteSupportsAlphaHitTest() && m_AlphaHitTestMinimumThreshold > 0)
            {
                Debug.LogWarning("Sprite was changed for one not readable or with Crunch Compression. Resetting the AlphaHitThreshold to 0.", this);
                m_AlphaHitTestMinimumThreshold = 0;
            }
        }

        bool SpriteSupportsAlphaHitTest()
        {
            return m_Sprite != null && m_Sprite.texture != null && !GraphicsFormatUtility.IsCrunchFormat(m_Sprite.texture.format) && m_Sprite.texture.isReadable;
        }
    }
}
```

调用set时，只有在Spirte发生变化时，才会对所有需要的数据标记为脏数据，而对于`SetAllDirty`，这是父类`Graphics`的成员函数，它主要根据一些标志来标记脏数据：

```C#
public virtual void SetAllDirty()
{
    // Optimization: Graphic layout doesn't need recalculation if
    // the underlying Sprite is the same size with the same texture.
    // (e.g. Sprite sheet texture animation)

    if (m_SkipLayoutUpdate)
    {
        m_SkipLayoutUpdate = false;
    }
    else
    {
        SetLayoutDirty();
    }

    if (m_SkipMaterialUpdate)
    {
        m_SkipMaterialUpdate = false;
    }
    else
    {
        SetMaterialDirty();
    }

    SetVerticesDirty();
    SetRaycastDirty();
}
```

可以看到无论如何顶点数据和射线数据都会被标记为脏，我们进入`SetVerticesDirty`函数中，可以看到：

```C#
/// <summary>
/// Mark the vertices as dirty and needing rebuilt.
/// </summary>
/// <remarks>
/// Send a OnDirtyVertsCallback notification if any elements are registered. See RegisterDirtyVerticesCallback
/// </remarks>
public virtual void SetVerticesDirty()
{
    if (!IsActive())
        return;

    m_VertsDirty = true;
    CanvasUpdateRegistry.RegisterCanvasElementForGraphicRebuild(this);

    if (m_OnDirtyVertsCallback != null)
        m_OnDirtyVertsCallback();
}
```

标记为脏时，还调用了`CanvasUpdateRegistry.RegisterCanvasElementForGraphicRebuild`来为Canvas下一次更新时注册，表示自己需要进行重建。

对其他数据例如Raycast、Layout的脏标记进行类似，不同的是设置Layout时使用的是`LayoutRebuilder.MarkLayoutForRebuild`，标记Raycast时使用的是`GraphicRegistry.RegisterRaycastGraphicForCanvas`。这里我们主要关心`CanvasUpdateRegistry`里的内容。

# CanvasUpdateRegistry

这个类有两个成员：

```C#
private readonly IndexedSet<ICanvasElement> m_LayoutRebuildQueue = new IndexedSet<ICanvasElement>();
private readonly IndexedSet<ICanvasElement> m_GraphicRebuildQueue = new IndexedSet<ICanvasElement>();
```

保存了需要进行Rebuild的UI元素，而上文中`SetVerticesDirty()`函数调用的`CanvasUpdateRegistry.RegisterCanvasElementForGraphicRebuild(this)`将图像组件自身加入了`m_GraphicsRebuildQueue`以便进行更新。

我们继续在sprite的get函数上打上断点进行调试，可以看到关键调用栈：

![image-20240528090346762](/imgs/post/UGUI源码阅读-2-Image的更新流程/image-20240528090346762.png)

UI的更新将从`Canvas.SendWillRenderCanvases()`函数开始，而在`CanvasUpdateRegistry`的构造函数中就将`PerformUpdate`函数加入了`Canvas.willRenderCanvases()`中：

```C#
protected CanvasUpdateRegistry()
{
    Canvas.willRenderCanvases += PerformUpdate;
}
```

而对于`PerformUpdate()`函数来说，其流程经过了如下几个阶段：

清除无效Graphic和Layout重建队列中的无效元素 -> 对Layout重建队列进行排序并重建 -> 裁剪UI元素 -> 对Graphic重建队列元素进行重建，下面我们一一查看这些阶段：

## 1. 清除无效Graphic和Layout重建队列中的无效元素

这个阶段做的事情很简单，就是看两个队列中的元素是不是null以及是否被摧毁（`ICanvasElement.IsDestroyed()`）。

```C#
private void CleanInvalidItems()
{
    // So MB's override the == operator for null equality, which checks
    // if they are destroyed. This is fine if you are looking at a concrete
    // mb, but in this case we are looking at a list of ICanvasElement
    // this won't forward the == operator to the MB, but just check if the
    // interface is null. IsDestroyed will return if the backend is destroyed.

    var layoutRebuildQueueCount = m_LayoutRebuildQueue.Count;
    for (int i = layoutRebuildQueueCount - 1; i >= 0; --i)
    {
        var item = m_LayoutRebuildQueue[i];
        if (item == null)
        {
            m_LayoutRebuildQueue.RemoveAt(i);
            continue;
        }

        if (item.IsDestroyed())
        {
            m_LayoutRebuildQueue.RemoveAt(i);
            item.LayoutComplete();
        }
    }
	// Graphics队列类似
    ...
}
```

## 2. 对Layout重建队列进行排序并重建

代码如下：

```C#
m_LayoutRebuildQueue.Sort(s_SortLayoutFunction);
for (int i = 0; i <= (int)CanvasUpdate.PostLayout; i++)
{
    for (int j = 0; j < m_LayoutRebuildQueue.Count; j++)
    {
         rebuild.Rebuild((CanvasUpdate)i);
    }
}
for (int i = 0; i < m_LayoutRebuildQueue.Count; ++i)
    m_LayoutRebuildQueue[i].LayoutComplete();
```

上面的代码经过了简化，去除了一些错误处理和性能采样的部分。

可以看到重建队列以`s_SortLayoutFunction`进行排序，那么这个排序函数是怎么样的，翻阅源码可以看到排序函数最后调用到了：

```C#
private static int SortLayoutList(ICanvasElement x, ICanvasElement y)
{
    Transform t1 = x.transform;
    Transform t2 = y.transform;

    return ParentCount(t1) - ParentCount(t2);
}

private static int ParentCount(Transform child)
{
    if (child == null)
        return 0;

    var parent = child.parent;
    int count = 0;
    while (parent != null)
    {
        count++;
        parent = parent.parent;
    }
    return count;
}
```

也就是说对Layout重建元素的排序是通过判断每个元素父元素个数来判断的。如果元素A的父级元素比B的父级元素多，那么它的Layout会首先被重建。

而在队列重建时，调用了`ICanvasElement.Rebuild(CanvasUpdate executing)`函数，这个函数针对不同阶段进行实现。例如这里我们研究的`Image`组件继承了`Graphics`类，它重写了`ICanvasElemnet.Rebuild`函数，只在`CanvasUpdate.PreRender`时起作用：

```C#
public virtual void Rebuild(CanvasUpdate update)
{
    if (canvasRenderer == null || canvasRenderer.cull)
        return;

    switch (update)
    {
        case CanvasUpdate.PreRender:
            if (m_VertsDirty)
            {
                UpdateGeometry();
                m_VertsDirty = false;
            }
            if (m_MaterialDirty)
            {
                UpdateMaterial();
                m_MaterialDirty = false;
            }
            break;
    }
}
```

换句话说，对于组件`Image`，不会进行Layout重建而只有Graphics重建。

## 3. UI元素裁剪

这一部分的关键代码就一句：

```C#
ClipperRegistry.instance.Cull();
```

裁剪和Mask等有关，我们之后单开一篇文章进行叙述。

## 4. 对Graphic重建队列元素进行重建

如下，代码去除了错误处理、参数验证和性能采样的部分：

```C#
for (var i = (int)CanvasUpdate.PreRender; i < (int)CanvasUpdate.MaxUpdateValue; i++)
{
    for (var k = 0; k < m_GraphicRebuildQueue.Count; k++)
    {
         element.Rebuild((CanvasUpdate)i);
    }
}

for (int i = 0; i < m_GraphicRebuildQueue.Count; ++i)
    m_GraphicRebuildQueue[i].GraphicUpdateComplete();
```

这一部分和Layout重建的差别不大，区别仅仅是不同的CanvasUpdate阶段。接下来函数的调用将会转到`Canvas.Rebuild()`中。

# Canvas.Rebuild()

这部分对应的代码如下：

```C#
public virtual void Rebuild(CanvasUpdate update)
{
    if (canvasRenderer == null || canvasRenderer.cull)
        return;
    switch (update)
    {
        case CanvasUpdate.PreRender:
            if (m_VertsDirty)
            {
                UpdateGeometry();
                m_VertsDirty = false;
            }
            if (m_MaterialDirty)
            {
                UpdateMaterial();
                m_MaterialDirty = false;
            }
            break;
    }
}
```

这里可以看到只有被标记为脏时才会对材质和Geometry进行更新。

这里的`UpdateGeometry`会调用到`DoMeshGeneration`来对顶点进行更新。

```C#
private void DoMeshGeneration()
{
    if (rectTransform != null && rectTransform.rect.width >= 0 && rectTransform.rect.height >= 0)
        OnPopulateMesh(s_VertexHelper);
    else
        s_VertexHelper.Clear(); // clear the vertex helper so invalid graphics dont draw.

    var components = ListPool<Component>.Get();
    GetComponents(typeof(IMeshModifier), components);

    for (var i = 0; i < components.Count; i++)
        ((IMeshModifier)components[i]).ModifyMesh(s_VertexHelper);

    ListPool<Component>.Release(components);

    s_VertexHelper.FillMesh(workerMesh);
    canvasRenderer.SetMesh(workerMesh);
}
```

这里会调用`OnPopulateMesh(s_VertexHelper)`来填充需要的顶点，`Image`组件正是重载此组件来进行顶点生成的。之后会获取当前object所以实现了`IMeshModifier`的函数来对顶点进行修改，比如`Outline`组件就实现了这个接口。最后再将CanvasRenderer的Mesh设置成生成的Mesh。

而对于材质的更新只是将canvasRenderer的材质进行更新而已：

```C#
protected virtual void UpdateMaterial()
{
    if (!IsActive())
        return;

    canvasRenderer.materialCount = 1;
    canvasRenderer.SetMaterial(materialForRendering, 0);
    canvasRenderer.SetTexture(mainTexture);
}
```



虽然`Image`重写了`UpdateMaterial`方法但也只是加一些`Alpha`相关的东西而已。

# Image的顶点生成方法

这个部分对应上面代码中的`OnPopulateMesh`部分，`Image`对其的重载如下：

```C#
protected override void OnPopulateMesh(VertexHelper toFill)
{
    if (activeSprite == null)
    {
        base.OnPopulateMesh(toFill);
        return;
    }

    switch (type)
    {
        case Type.Simple:
            if (!useSpriteMesh)
                GenerateSimpleSprite(toFill, m_PreserveAspect);
            else
                GenerateSprite(toFill, m_PreserveAspect);
            break;
        case Type.Sliced:
            GenerateSlicedSprite(toFill);
            break;
        case Type.Tiled:
            GenerateTiledSprite(toFill);
            break;
        case Type.Filled:
            GenerateFilledSprite(toFill, m_PreserveAspect);
            break;
    }
}
```

可以看到根据`ImageType`的不同会有不同的顶点生成方法，我们拿最简单的`GenerateSimpleSprite`为例：

```C#
void GenerateSimpleSprite(VertexHelper vh, bool lPreserveAspect)
{
    Vector4 v = GetDrawingDimensions(lPreserveAspect);
    var uv = (activeSprite != null) ? Sprites.DataUtility.GetOuterUV(activeSprite) : Vector4.zero;

    var color32 = color;
    vh.Clear();
    // 添加了顶点、颜色和UV
    vh.AddVert(new Vector3(v.x, v.y), color32, new Vector2(uv.x, uv.y));
    vh.AddVert(new Vector3(v.x, v.w), color32, new Vector2(uv.x, uv.w));
    vh.AddVert(new Vector3(v.z, v.w), color32, new Vector2(uv.z, uv.w));
    vh.AddVert(new Vector3(v.z, v.y), color32, new Vector2(uv.z, uv.y));

    vh.AddTriangle(0, 1, 2);
    vh.AddTriangle(2, 3, 0);
}
```

可以看到就只是简单的添加了四个顶点和对应的索引而已。~~更细节的部分之后新开一篇讨论（挖坑）~~

# 总结

今天我们讨论`Image`的重建流程，讲解了`Image`的顶点生成，`Image`的更新流程，了解掉了`Image`中使用的脏标记模式，同时对`CanvasUpdateRegistry`和`Graphics`类有了一定的了解。但也留下了一些疑问，例如裁剪是如何进行的？这里只描述了UI如何进行重建，那么是什么时候进行渲染的呢？~~有待我继续看源码填坑~~

经过这篇文章我们就可以自定义自己的Image的顶点生成方法，可以根据此来生成不规则图片，比如一个菱形图片，只要重写`OnPopulateMesh`即可生成不同形状的网格了。

