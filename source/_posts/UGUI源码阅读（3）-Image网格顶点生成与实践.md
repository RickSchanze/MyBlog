---
title: UGUI源码阅读（3）- Image网格顶点生成与实践
tags:
  - UGUI
  - Unity
categories:
  - UGUI
typora-root-url: ./..
abbrlink: 6b2c390b
date: 2024-05-31 09:03:38
---

# 前言

之前的文章里我们了解到了Image的Rebuild流程，今天我们来看看Image的网格顶点是如何生成了。

本篇文章里你能了解到：

上篇：[UGUI源码阅读(2)--Image的更新流程 | 喜多喜多のBlog (kita-blog.vercel.app)](https://kita-blog.vercel.app/posts/79fdc967.html)

# Image网格顶点生成

上篇文章我们说过Image重写了`OnPopulateMesh`方法，这个方法负责网格顶点生成，其代码如下：

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



在开始之前，我们首先把我们TestImage的RectTransform的值修改为这样：

![image-20240531091921325](/imgs/post/UGUI源码阅读（3）-Image网格顶点生成与实践/image-20240531091921325.png)

将Width和Height设置为整数便于我们对比，此外本次测试中Image的参数如下：

![image-20240531092024645](/imgs/post/UGUI源码阅读（3）-Image网格顶点生成与实践/image-20240531092024645.png)



接下来我们一一查看这些生成的方法。

## 1. sprite为null时

当sprite为null时，就是简单的调用父类的OnPopulateMesh:

```C#
protected virtual void OnPopulateMesh(VertexHelper vh)
{
    var r = GetPixelAdjustedRect();
    var v = new Vector4(r.x, r.y, r.x + r.width, r.y + r.height);

    Color32 color32 = color;
    vh.Clear();
    vh.AddVert(new Vector3(v.x, v.y), color32, new Vector2(0f, 0f));
    vh.AddVert(new Vector3(v.x, v.w), color32, new Vector2(0f, 1f));
    vh.AddVert(new Vector3(v.z, v.w), color32, new Vector2(1f, 1f));
    vh.AddVert(new Vector3(v.z, v.y), color32, new Vector2(1f, 0f));

    vh.AddTriangle(0, 1, 2);
    vh.AddTriangle(2, 3, 0);
}
```

这里的`GetPixelAdjustedRect`会返回Image矩形，我们上面设置的Image参数为(0, 0, 0, 150, 150)(代表(X, Y, Width, Height)后文不再赘述)，则这里的r返回的就是(-75, -75, 150, 150)代表矩形左下角和矩形长宽。

那么这个`GetPixelAdjustRect`是怎么计算呢，这里的计算会转到C++因此不得而知~~（快开源！）~~。

可以看到这里就是简单的增加了四个顶点（左下、左上、右上、右下）并设置了合适的颜色，最后添加了索引。

## 2. type为Simple时

```C#
case Type.Simple:
	if (!useSpriteMesh)
    	GenerateSimpleSprite(toFill, m_PreserveAspect);
	else
    	GenerateSprite(toFill, m_PreserveAspect);
	break;
```

这里有两条分支我们一次讲解

### 2.1 不使用SpriteMesh

```C#
void GenerateSimpleSprite(VertexHelper vh, bool lPreserveAspect)
{
    Vector4 v = GetDrawingDimensions(lPreserveAspect);
    var uv = (activeSprite != null) ? Sprites.DataUtility.GetOuterUV(activeSprite) : Vector4.zero;

    var color32 = color;
    vh.Clear();
    vh.AddVert(new Vector3(v.x, v.y), color32, new Vector2(uv.x, uv.y));
    vh.AddVert(new Vector3(v.x, v.w), color32, new Vector2(uv.x, uv.w));
    vh.AddVert(new Vector3(v.z, v.w), color32, new Vector2(uv.z, uv.w));
    vh.AddVert(new Vector3(v.z, v.y), color32, new Vector2(uv.z, uv.y));

    vh.AddTriangle(0, 1, 2);
    vh.AddTriangle(2, 3, 0);
}
```

函数`GetDrawingDimensions`会返回当前图像矩形的左下角和宽高，假如`lPreserveAspect`为false，那么返回的就是图像矩形，这个值会根据图像的padding进行处理

，假如`lPreserveAspect`为`true`那么返回的将是符合当前图像比例的并且在RectTransform限制下的左下角和宽高，其中用到的调整方法如下：

```C#
private void PreserveSpriteAspectRatio(ref Rect rect, Vector2 spriteSize)
{
    var spriteRatio = spriteSize.x / spriteSize.y;
    var rectRatio = rect.width / rect.height;

    if (spriteRatio > rectRatio)
    {
        var oldHeight = rect.height;
        rect.height = rect.width * (1.0f / spriteRatio);
        rect.y += (oldHeight - rect.height) * rectTransform.pivot.y;
    }
    else
    {
        var oldWidth = rect.width;
        rect.width = rect.height * spriteRatio;
        rect.x += (oldWidth - rect.width) * rectTransform.pivot.x;
    }
}
```

当图像比例大于当前矩形比例时会将调整图像的高度和y，小于时调整图像的宽和x。

随后我们获取了uv坐标（C++），传入`VertexHelper`中，完成了顶点生成。

### 2.2 使用SpriteMesh

使用SpriteMesh时，会使用sprite里记录的顶点和uv来计算，基本上就是将sprite里记录的顶点和索引和uv按比例缩放到要求的rect里面，而spriteMesh顶点如何生成涉及到了C++调用，因此无法查看源码，因此这里仅放一张使用spriteMesh和不使用spriteMesh的对比图：

![image-20240531101454676](/imgs/post/UGUI源码阅读（3）-Image网格顶点生成与实践/image-20240531101454676.png)

可以看到使用SpriteMesh会增加三角形数量，但是相应的需要处理的透明部分就减少了，而对于渲染来说：

+ 使用FullRect时三角形少，但是处理的像素多，对应需要处理的透明部分也就变多，可能会降低性能；

+ 使用SpriteMesh时三角形多，虽然处理的像素少了，但是CPU侧需要的计算多了，也可能会降低性能

因此这两者的使用需要根据自己的程序进行权衡，也可以对Unity生成SpriteMesh的方法进行优化，虽然我们没办法看到Unity生成SpriteMesh的方法，但社区中已有插件来做个这个事情：[TexturePacker Importer | 精灵管理 | Unity Asset Store](https://assetstore.unity.com/packages/tools/sprite-management/texturepacker-importer-16641)

相关的介绍文章在：[Optimizing sprite meshes for Unity (codeandweb.com)](https://www.codeandweb.com/texturepacker/tutorials/optimizing-unity-sprite-meshes)

## 3. type为Sliced时

我们首先将一个Sprite进行九宫格切片：

![image-20240601103125899](/imgs/post/UGUI源码阅读（3）-Image网格顶点生成与实践/image-20240601103125899.png)

这里设置的Border的LTRB值都是50。然后将image的type改为sliced，来看看顶点的生成情况：

![](/imgs/post/UGUI源码阅读（3）-Image网格顶点生成与实践/image-20240601103417439.png)

我这里为关键顶点和区域做了编号，当image横向扩展时，2、5、8区域的横向长度会变化，当image纵向扩展时，4、5、6区域的纵向长度会发生变化，四角（1、3、7、9）的长宽都不变。

让我们深入代码看看这些顶点和三角形是如何生成的：

首先获取了所有必要的信息（代码去除了参数校验部分）：

```C#
private void GenerateSlicedSprite(VertexHelper toFill)
{
    if (!hasBorder)
    {
        GenerateSimpleSprite(toFill, false);
        return;
    }

    Vector4 outer, inner, padding, border;
    outer = Sprites.DataUtility.GetOuterUV(activeSprite);
    inner = Sprites.DataUtility.GetInnerUV(activeSprite);
    padding = Sprites.DataUtility.GetPadding(activeSprite);
    border = activeSprite.border;
    Rect rect = GetPixelAdjustedRect();
```

这里获取了outerUV，innerUV，padding，border和rect，其中border就是我们在Sprite Editor中设置border，rect为image的RectTransform代表的矩形。

随后计算innerRect的信息（即计算1、2、3、4四个顶点的信息）和UV信息：

```C#
    Vector4 adjustedBorders = GetAdjustedBorders(border / multipliedPixelsPerUnit, rect);
	padding = padding / multipliedPixelsPerUnit;

	s_VertScratch[0] = new Vector2(padding.x, padding.y);
	s_VertScratch[3] = new Vector2(rect.width - padding.z, rect.height - padding.w);

	s_VertScratch[1].x = adjustedBorders.x;
	s_VertScratch[1].y = adjustedBorders.y;

	s_VertScratch[2].x = rect.width - adjustedBorders.z;
	s_VertScratch[2].y = rect.height - adjustedBorders.w;

	for (int i = 0; i < 4; ++i)
	{
    	s_VertScratch[i].x += rect.x;
    	s_VertScratch[i].y += rect.y;
	}

	s_UVScratch[0] = new Vector2(outer.x, outer.y);
	s_UVScratch[1] = new Vector2(inner.x, inner.y);
	s_UVScratch[2] = new Vector2(inner.z, inner.w);
	s_UVScratch[3] = new Vector2(outer.z, outer.w);
```

我们会根据image中设置的`Pixel Per Unit Multiplier`参数调整border，随后进行设置，其中`s_VertScratch`记录了必要的信息，它是一个四个元素的`Vector2`数组，第一个和第四个记录了padding的左下角和右上角（即整个矩形的左下角和右下角），第二个和第三个参数记录了border的左下角和右下角，最后将每个元素加上rect的信息就获得了padding和rect实际的左下角和右下角信息。`s_UVScratch`记录了内区域（5）左上角和右下角的UV（第2、3个元素）和外区域（整个矩形）左上角和右下角UV（第1、4个元素）。

最后对VertexHelper进行填充：

```C#
	toFill.Clear();
	for (int x = 0; x < 3; ++x)
	{
    	int x2 = x + 1;
    	for (int y = 0; y < 3; ++y)
    	{
        	if (!m_FillCenter && x == 1 && y == 1)
           	 	continue;
        	int y2 = y + 1;
        	AddQuad(toFill,
                	new Vector2(s_VertScratch[x].x, s_VertScratch[y].y),
                	new Vector2(s_VertScratch[x2].x, s_VertScratch[y2].y),
                	color,
                	new Vector2(s_UVScratch[x].x, s_UVScratch[y].y),
                	new Vector2(s_UVScratch[x2].x, s_UVScratch[y2].y));
    	}
	}
}
```

函数AddQuad声明如下：

```C#
static void AddQuad(VertexHelper vertexHelper, Vector2 posMin, Vector2 posMax, Color32 color, Vector2 uvMin, Vector2 uvMax);
```

以posMin为左下角，posMax为右上角，uvMin为左下角UV，uvMax为右上角构建一个矩形。

例如当x=0,y=0时，就是取最左下角和3号顶点构建矩形，循环下来就构建了九个矩形。

## 4. type为Tiled时

首先我们来看看顶点生成情况：

![image-20240601110557961](/imgs/post/UGUI源码阅读（3）-Image网格顶点生成与实践/image-20240601110557961.png)

如果sprite有border，那么四周就会不断的重复border，中间会不断重复5区域。

在代码中，除了计算和type为Sliced时一样的outerUV，innerUV，padding，border，rect参数，还计算了tiledWidth和tiledHeight(也就是中间重复的部分)：

```C#
void GenerateTiledSprite(VertexHelper toFill) {
    ...
    float tileWidth = (spriteSize.x - border.x - border.z) / multipliedPixelsPerUnit;
    float tileHeight = (spriteSize.y - border.y - border.w) / multipliedPixelsPerUnit;

```

接下来计算了除开border外中间需要重复的范围：

```C#
	var uvMin = new Vector2(inner.x, inner.y);
	var uvMax = new Vector2(inner.z, inner.w);

	// Min to max max range for tiled region in coordinates relative to lower left corner.
	float xMin = border.x;
	float xMax = rect.width - border.z;
	float yMin = border.y;
	float yMax = rect.height - border.w;
```

代码中有许多参数错误处理的代码，包括顶点数不能超过65000个等，也有许多分支，这里我们只分析其中一个分支，下面的代码都是在文章到这里时会执行的代码：

首先我们计算顶点的数量：

```C#
if (activeSprite != null && (hasBorder || activeSprite.packed || activeSprite.texture != null && activeSprite.texture.wrapMode != TextureWrapMode.Repeat))
{
    // Sprite has border, or is not in repeat mode, or cannot be repeated because of packing.
    // We cannot use texture tiling so we will generate a mesh of quads to tile the texture.
    // Evaluate how many vertices we will generate. Limit this number to something sane,
    // especially since meshes can not have more than 65000 vertices.

    long nTilesW = 0;
    long nTilesH = 0;
    if (m_FillCenter)
    {
        nTilesW = (long)Math.Ceiling((xMax - xMin) / tileWidth);
        nTilesH = (long)Math.Ceiling((yMax - yMin) / tileHeight);

        double nVertices = 0;
        if (hasBorder)
        {
            nVertices = (nTilesW + 2.0) * (nTilesH + 2.0) * 4.0; // 4 vertices per tile
        }
        else
        {
            nVertices = nTilesW * nTilesH * 4.0; // 4 vertices per tile
        }
    }
```

当border存在时nTilesW和nTilesH每一个都需要+2，这是因为对于每一个tile都需要多出几个顶点来填充，如下：

![image-20240601112606880](/imgs/post/UGUI源码阅读（3）-Image网格顶点生成与实践/image-20240601112606880.png)

接下来开始填充Center，就是根据nTilesH和nTilesW来不断添加矩形，但是这里TODO也说了，其实多个四边形之间还可以共享顶点，这里可以作为一个优化点。

```C#

    if (m_FillCenter)
    {
        // TODO: we could share vertices between quads. If vertex sharing is implemented. update the computation for the number of vertices accordingly.
        for (long j = 0; j < nTilesH; j++)
        {
            float y1 = yMin + j * tileHeight;
            float y2 = yMin + (j + 1) * tileHeight;
            if (y2 > yMax)
            {
                clipped.y = uvMin.y + (uvMax.y - uvMin.y) * (yMax - y1) / (y2 - y1);
                y2 = yMax;
            }
            clipped.x = uvMax.x;
            for (long i = 0; i < nTilesW; i++)
            {
                float x1 = xMin + i * tileWidth;
                float x2 = xMin + (i + 1) * tileWidth;
                if (x2 > xMax)
                {
                    clipped.x = uvMin.x + (uvMax.x - uvMin.x) * (xMax - x1) / (x2 - x1);
                    x2 = xMax;
                }
                AddQuad(toFill, new Vector2(x1, y1) + rect.position, new Vector2(x2, y2) + rect.position, color, uvMin, clipped);
            }
        }
    }
```

最后计算border的顶点和UV：

```C#
	if (hasBorder)
    {
        clipped = uvMax;
        for (long j = 0; j < nTilesH; j++)
        {
            float y1 = yMin + j * tileHeight;
            float y2 = yMin + (j + 1) * tileHeight;
            if (y2 > yMax)
            {
                clipped.y = uvMin.y + (uvMax.y - uvMin.y) * (yMax - y1) / (y2 - y1);
                y2 = yMax;
            }
            AddQuad(toFill,
                    new Vector2(0, y1) + rect.position,
                    new Vector2(xMin, y2) + rect.position,
                    color,
                    new Vector2(outer.x, uvMin.y),
                    new Vector2(uvMin.x, clipped.y));
            AddQuad(toFill,
                    new Vector2(xMax, y1) + rect.position,
                    new Vector2(rect.width, y2) + rect.position,
                    color,
                    new Vector2(uvMax.x, uvMin.y),
                    new Vector2(outer.z, clipped.y));
        }

        // Bottom and top tiled border
        clipped = uvMax;
        for (long i = 0; i < nTilesW; i++)
        {
            float x1 = xMin + i * tileWidth;
            float x2 = xMin + (i + 1) * tileWidth;
            if (x2 > xMax)
            {
                clipped.x = uvMin.x + (uvMax.x - uvMin.x) * (xMax - x1) / (x2 - x1);
                x2 = xMax;
            }
            AddQuad(toFill,
                    new Vector2(x1, 0) + rect.position,
                    new Vector2(x2, yMin) + rect.position,
                    color,
                    new Vector2(uvMin.x, outer.y),
                    new Vector2(clipped.x, uvMin.y));
            AddQuad(toFill,
                    new Vector2(x1, yMax) + rect.position,
                    new Vector2(x2, rect.height) + rect.position,
                    color,
                    new Vector2(uvMin.x, uvMax.y),
                    new Vector2(clipped.x, outer.w));
        }

        // Corners
        AddQuad(toFill,
                new Vector2(0, 0) + rect.position,
                new Vector2(xMin, yMin) + rect.position,
                color,
                new Vector2(outer.x, outer.y),
                new Vector2(uvMin.x, uvMin.y));
        AddQuad(toFill,
                new Vector2(xMax, 0) + rect.position,
                new Vector2(rect.width, yMin) + rect.position,
                color,
                new Vector2(uvMax.x, outer.y),
                new Vector2(outer.z, uvMin.y));
        AddQuad(toFill,
                new Vector2(0, yMax) + rect.position,
                new Vector2(xMin, rect.height) + rect.position,
                color,
                new Vector2(outer.x, uvMax.y),
                new Vector2(uvMin.x, outer.w));
        AddQuad(toFill,
                new Vector2(xMax, yMax) + rect.position,
                new Vector2(rect.width, rect.height) + rect.position,
                color,
                new Vector2(uvMax.x, uvMax.y),
                new Vector2(outer.z, outer.w));
    }
}
```

其实就是根据具有的nTileW和nTilesH来对border进行重复，最后再加上四个矩形的顶点。

## 5. type为Filled时

这种情况有许多变种，我们挑选其中一种进行查看，参数设置如下：

![image-20240601113347002](/imgs/post/UGUI源码阅读（3）-Image网格顶点生成与实践/image-20240601113347002.png)

其生成的顶点为：

![image-20240601113432700](/imgs/post/UGUI源码阅读（3）-Image网格顶点生成与实践/image-20240601113432700.png)

可以看到将整个矩形划分成了四个小矩形，然后通过某个顶点位置实现Filled的效果。

参数计算和之前差不多，这里不再赘述，同时下面的代码也经过了精简，核心计算就是对四个角落的四边形进行RadialCut

```c#
void GenerateFilledSprite(VertexHelper toFill, bool preserveAspect)
{
    ...
    for (int corner = 0; corner < 4; ++corner)
    {
        // 计算s_xy和s_uv，代表当前corner四个顶点的位置和UV
        // val代表哪个矩形需要改变，比如此时m_FillAmout=0.7，那么
        // 对于第1、2个矩形，Mathf.Clamp01(val)=1
        // 对于第三个矩形，Mathf.Clamp01(val)=0.8
        // 对于第四个矩形，Mathf.Clamp01(val)=0
        float val = m_FillClockwise ?
            m_FillAmount * 4f - ((corner + m_FillOrigin) % 4) :
        m_FillAmount * 4f - (3 - ((corner + m_FillOrigin) % 4));

        if (RadialCut(s_Xy, s_Uv, Mathf.Clamp01(val), m_FillClockwise, ((corner + 2) % 4)))
            AddQuad(toFill, s_Xy, color, s_Uv);
    }
    ...
}

```

其中关键函数`RadialCut`直接修改顶点位置和UV坐标：

```C#
static bool RadialCut(Vector3[] xy, Vector3[] uv, float fill, bool invert, int corner)
{
    // 不进行填充
    if (fill < 0.001f) return false;

    // Even corners invert the fill direction
    if ((corner & 1) == 1) invert = !invert;

    // 不需要进行任何改变
    if (!invert && fill > 0.999f) return true;

    // Convert 0-1 value into 0 to 90 degrees angle in radians
    float angle = Mathf.Clamp01(fill);
    if (invert) angle = 1f - angle;
    angle *= 90f * Mathf.Deg2Rad;

    // Calculate the effective X and Y factors
    float cos = Mathf.Cos(angle);
    float sin = Mathf.Sin(angle);

    RadialCut(xy, cos, sin, invert, corner);
    RadialCut(uv, cos, sin, invert, corner);
    return true;
}
```

这个函数计算了需要划分的角度信息并传给另一个`RadialCut`重载：

```C#
static void RadialCut(Vector3[] xy, float cos, float sin, bool invert, int corner)
{
    int i0 = corner;
    int i1 = ((corner + 1) % 4);
    int i2 = ((corner + 2) % 4);
    int i3 = ((corner + 3) % 4);
    if ((corner & 1) == 1)
    {
        if (sin > cos)
        {
            cos /= sin;
            sin = 1f;

            if (invert)
            {
                xy[i1].x = Mathf.Lerp(xy[i0].x, xy[i2].x, cos);
                xy[i2].x = xy[i1].x;
            }
        }
        else if (cos > sin)
        {
            sin /= cos;
            cos = 1f;

            if (!invert)
            {
                xy[i2].y = Mathf.Lerp(xy[i0].y, xy[i2].y, sin);
                xy[i3].y = xy[i2].y;
            }
        }
        else
        {
            cos = 1f;
            sin = 1f;
        }

        if (!invert) xy[i3].x = Mathf.Lerp(xy[i0].x, xy[i2].x, cos);
        else xy[i1].y = Mathf.Lerp(xy[i0].y, xy[i2].y, sin);
    }
    else
    {
        if (cos > sin)
        {
            sin /= cos;
            cos = 1f;

            if (!invert)
            {
                xy[i1].y = Mathf.Lerp(xy[i0].y, xy[i2].y, sin);
                xy[i2].y = xy[i1].y;
            }
        }
        else if (sin > cos)
        {
            cos /= sin;
            sin = 1f;

            if (invert)
            {
                xy[i2].x = Mathf.Lerp(xy[i0].x, xy[i2].x, cos);
                xy[i3].x = xy[i2].x;
            }
        }
        else
        {
            cos = 1f;
            sin = 1f;
        }

        if (invert) xy[i3].y = Mathf.Lerp(xy[i0].y, xy[i2].y, sin);
        else xy[i1].x = Mathf.Lerp(xy[i0].x, xy[i2].x, cos);
    }
}
```

![3a7f2fe00387976ac86243f8e7b43c9d](/imgs/post/UGUI源码阅读（3）-Image网格顶点生成与实践/3a7f2fe00387976ac86243f8e7b43c9d.jpg)

这是其中一例分析，其他的情况与这里的类似。只是换了一下顺序关系。

# 自定义网格顶点生成实践

上面我们讨论了所有Unity Image自带的网格顶点生成，下面我们进行一个实践，生成一个六边形的Image。

![image-20240601122419540](/imgs/post/UGUI源码阅读（3）-Image网格顶点生成与实践/image-20240601122419540.png)

共六个顶点、四个三角形。代码如下：

```C#
// MyImage.cs
public class MyImage : Image
{
    public bool useUnity;
    public bool clip;
    [Range(0, 0.5f)] public float ratio;

    protected override void OnPopulateMesh(VertexHelper toFill)
    {
        if (useUnity)
        {
            base.OnPopulateMesh(toFill);
            return;
        }

        var rect = GetPixelAdjustedRect();
        var positions = new Vector2[6];
        var uvs = new Vector2[6];
        // @formatter:off
        positions[0].x = 0;         positions[0].y = 0.5f;
        positions[3].x = 1;         positions[3].y = 0.5f;
        positions[1].x = ratio;     positions[1].y = 1;
        positions[2].x = 1 - ratio; positions[2].y = 1;
        positions[5].x = ratio;     positions[5].y = 0;
        positions[4].x = 1 - ratio; positions[4].y = 0;

        for (int i = 0; i < 6; i++)
        {
            positions[i].x = Mathf.Lerp(rect.x, rect.x + rect.size.x, positions[i].x);
            positions[i].y = Mathf.Lerp(rect.y, rect.y + rect.size.y, positions[i].y);
        }
        uvs[0].x = 0f;            uvs[0].y = 0.5f;
        uvs[3].x = 1;             uvs[3].y = 0.5f;
        if (clip) {
            uvs[1].x = uvs[5].x = ratio;
            uvs[2].x = uvs[4].x = 1- ratio;
        } else {
            uvs[1].x = uvs[5].x = 0;
            uvs[2].x = uvs[4].x = 1;
        }
        uvs[1].y = uvs[2].y = 1;
        uvs[5].y = uvs[4].y = 0;
        // @formatter:on
        toFill.Clear();
        for (int i = 0; i < 6; i++)
        {
            toFill.AddVert(positions[i], color, uvs[i]);
        }
        toFill.AddTriangle(0, 1, 2);
        toFill.AddTriangle(0, 2, 3);
        toFill.AddTriangle(0, 3, 4);
        toFill.AddTriangle(0, 4, 5);
    }
}
```

```C#
// MyImageEditor.cs
[CustomEditor(typeof(MyImage))]
public class MyImageEditor : ImageEditor
{
    private SerializedProperty ratioProperty;
    private SerializedProperty useUnityProperty;
    private SerializedProperty clipProperty;

    protected override void OnEnable()
    {
        base.OnEnable();
        ratioProperty = serializedObject.FindProperty("ratio");
        useUnityProperty = serializedObject.FindProperty("useUnity");
        clipProperty = serializedObject.FindProperty("clip");
    }

    public override void OnInspectorGUI()
    {
        base.OnInspectorGUI();
        EditorGUILayout.PropertyField(ratioProperty);
        EditorGUILayout.PropertyField(useUnityProperty);
        EditorGUILayout.PropertyField(clipProperty);
        serializedObject.ApplyModifiedProperties();
    }
}
```

效果如下：

![image-20240601131419483](/imgs/post/UGUI源码阅读（3）-Image网格顶点生成与实践/image-20240601131419483.png)

# 总结

本文总结Image的各种顶点生成方式，并在最后给出了自己的实践。

之后在尽显不规则图像显示时可以自己定义顶点生成方式，与Mask方式相比，目前不知道哪个性能好一点，~~我自己认为顶点生成可能会更好一点~~
