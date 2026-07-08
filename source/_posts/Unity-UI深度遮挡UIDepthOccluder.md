---
title: Unity UI深度遮挡 UIDepthOccluder
date: 2026-07-08 00:00:00
tags: [Unity, URP, RenderGraph, UI, Shader, 深度, 游戏开发]
categories: [Unity]
---

https://github.com/NicoIer/UnityToolkit/blob/main/Runtime/Renderer/UIDepthOccluder.cs

https://github.com/NicoIer/UnityToolkit/blob/main/Runtime/Renderer/UIDepthOccluderFeature.cs



先说结论：这个方案适合优化大块不透明 UI 背后的 Opaque 场景绘制。

它不是让 UI 参与场景渲染，也不是解决 UI 和场景物体的显示层级问题。它只是提前把 UI 覆盖区域写入深度，让后面的不透明物体在深度测试阶段被挡掉。

如果 UI 是半透明的、经常旋转缩放，或者遮挡区域很小，这个方案收益会比较有限，甚至可能因为多一次深度绘制变得更慢。

## 为什么需要 UI 深度遮挡

正常情况下，UI 通常是在场景物体之后绘制的。

所以从最终画面看，UI 一定可以盖住场景物体。

问题不在显示结果，而在渲染过程。

UI 虽然最后会盖住画面，但它没有提前写入场景深度。

对 Opaque 阶段来说，这块 UI 还不存在，UI 背后的场景物体仍然会正常经过深度测试、光栅化和片元着色。

如果这块 UI 是完全不透明的，那么这些被 UI 覆盖的场景像素最终一定不可见。

这里的方案是：

在 Opaque 绘制之前，根据 UI 的屏幕区域先写一遍深度。

这样后面的不透明物体在经过深度测试时，就会被这块 UI 深度提前挡住。

本质上是把不透明 UI 当成一个屏幕空间的深度遮挡体，用最终必然遮挡的 UI 区域，提前剔除后面不会被看见的场景像素。

## 从性能优化角度看

这个方案也可以从性能优化角度理解。

有些 UI 区域本来就是完全不透明的，比如固定的底部技能栏、侧边面板、头像框、全屏 HUD 边框。

这些区域最终一定会盖住后面的场景画面。

如果仍然正常绘制这些区域背后的不透明物体，那么这些像素虽然最后不可见，但前面的顶点处理、光栅化和片元着色仍然可能已经发生。

所以可以在这些不透明 UI 区域先写入一块靠近相机的深度。

后续 Opaque 物体落到这块屏幕区域时，会在深度测试阶段被挡掉。

这样 UI 本身不仅是显示层上的遮挡，也变成了场景渲染阶段的提前遮挡。

这种做法更适合下面几类场景：

- UI 遮挡区域固定，或者变化频率很低
- UI 是完全不透明的，不存在半透明叠加需求
- UI 背后经常有大量不透明场景、角色、特效承载物
- 被遮挡区域的片元成本比较高，比如复杂 Shader、密集模型、远景地形

它不是免费优化，因为每个遮挡体也需要额外一次深度绘制。

如果 UI 很小、背后几乎没有复杂场景，或者主要瓶颈在 CPU / 顶点阶段，收益就会比较有限。

## 整体方案

整体流程分成四步：

- 收集当前可见的 `UIDepthOccluder`
- 把 UI 的 `RectTransform` 区域转换到 viewport 空间
- 在 URP 的 `BeforeRenderingOpaques` 阶段插入一个 RenderPass
- Shader 只写深度，不写颜色

`UIDepthOccluder` 挂在 UI Image 上，用来描述这块 UI 是否要参与深度遮挡。

`UIDepthOccluderFeature` 挂到 URP Renderer 上，负责在渲染流程里插入真正的深度写入。

`UIDepthOccluder.shader` 不输出颜色，只把指定区域写进深度。

## 使用方式

首先在 URP Renderer Data 上添加 `UIDepthOccluderFeature`。

然后给它指定 Shader：

```text
Hidden/UnityToolkit/UIDepthOccluder
```

接着在需要作为遮挡体的 UI Image 上挂 `UIDepthOccluder`。

如果不指定 `occluderMesh`，代码会使用一个默认 Quad，遮挡范围就是整个 RectTransform。

如果 UI 不是矩形，比如圆形头像、特殊边框、异形遮罩，就可以传一个自定义 Mesh。

这个 Mesh 的顶点需要在 normalized `[0,1]` 空间里：

- `(0, 0)` 表示 UI 左下角
- `(1, 1)` 表示 UI 右上角

这样同一个 Mesh 可以跟随 RectTransform 缩放，不需要关心真实屏幕尺寸。

需要注意的是，当前 Feature 只处理带 `MainCamera` tag 的相机：

```csharp
if (!cameraData.camera.CompareTag("MainCamera")) return;
```

如果项目里主相机没有这个 tag，这个 Pass 不会执行。

## 关键代码

### 收集可见的 UI 遮挡体

`UIDepthOccluder` 内部维护了一个静态列表：

```csharp
public static readonly List<UIDepthOccluder> s_ActiveOccluders = new List<UIDepthOccluder>();
```

组件启用时，如果当前 Image 的继承透明度为 1，就加入列表。

```csharp
void OnEnable()
{
    if (_image.canvasRenderer.GetInheritedAlpha() >= 1f)
    {
        s_ActiveOccluders.Add(this);
        _registered = true;
    }
}
```

`LateUpdate` 里会继续检查透明度。

```csharp
void LateUpdate()
{
    bool visible = _image.canvasRenderer.GetInheritedAlpha() >= 1f;
    if (visible && !_registered)
    {
        s_ActiveOccluders.Add(this);
        _registered = true;
    }
    else if (!visible && _registered)
    {
        s_ActiveOccluders.Remove(this);
        _registered = false;
    }
}
```

这里的判断比较直接：只有完全不透明的 UI 才参与深度遮挡。

如果 UI 正在做淡入淡出，alpha 小于 1 时不会写入深度，避免画面还没完全显示就开始遮挡场景。

### 从 UI 空间转换到 Viewport 空间

`occluderMesh` 的顶点在 `[0,1]` 空间里。

真正绘制前，需要把它转换到屏幕 viewport。

核心就是取 `RectTransform` 的世界角点，然后换算成屏幕比例。

```csharp
_rectTransform.GetWorldCorners(_worldCorners);

viewMin = new Vector2(_worldCorners[0].x / Screen.width, _worldCorners[0].y / Screen.height);
viewMax = new Vector2(_worldCorners[2].x / Screen.width, _worldCorners[2].y / Screen.height);

var sx = viewMax.x - viewMin.x;
var sy = viewMax.y - viewMin.y;
```

最后构造一个矩阵，把 Mesh 从 `[0,1]` 映射到 `[viewMin, viewMax]`。

```csharp
return new Matrix4x4(
    new Vector4(sx, 0, 0, 0),
    new Vector4(0, sy, 0, 0),
    new Vector4(0, 0, 1, 0),
    new Vector4(viewMin.x, viewMin.y, 0, 1)
);
```

这个矩阵会作为 `cmd.DrawMesh` 的 model matrix 传给 Shader。

Shader 里拿到的 `UNITY_MATRIX_M` 实际上就是这个 localToViewport 矩阵。

### 在 Opaque 前写入深度

RendererFeature 创建 Pass 时，把执行时机放在 `BeforeRenderingOpaques`。

```csharp
_pass = new UIDepthOccluderPass(_material)
{
    renderPassEvent = RenderPassEvent.BeforeRenderingOpaques
};
```

这样 UI 遮挡体会先写入深度，后面的 Opaque 才会被影响。

RenderGraph 路径中，Pass 直接写当前激活的深度纹理：

```csharp
using (var builder = renderGraph.AddRasterRenderPass<PassData>("UIDepthOccluder", out var passData))
{
    builder.SetRenderAttachmentDepth(resourceData.activeDepthTexture, AccessFlags.Write);
    builder.AllowPassCulling(false);

    passData.material = _passData.material;
    passData.drawCalls = _passData.drawCalls;

    builder.SetRenderFunc((PassData data, RasterGraphContext context) =>
    {
        var cmd = context.cmd;
        for (int i = 0; i < data.drawCalls.Count; i++)
        {
            var (mesh, matrix) = data.drawCalls[i];
            cmd.DrawMesh(mesh, matrix, data.material, 0, 0);
        }
    });
}
```

每个遮挡体最终就是一次 `DrawMesh`。

如果没有配置 `occluderMesh`，就使用默认 Quad。

```csharp
var mesh = occluder.occluderMesh != null ? occluder.occluderMesh : GetDefaultQuad();
```

默认 Quad 也是 `[0,1]` 空间：

```csharp
vertices = new[]
{
    new Vector3(0, 0, 0),
    new Vector3(0, 1, 0),
    new Vector3(1, 1, 0),
    new Vector3(1, 0, 0)
}
```

### Shader 只写深度

Shader 的关键状态很少：

```shaderlab
ColorMask 0
ZWrite On
ZTest Always
Cull Off
```

`ColorMask 0` 表示不写颜色。

`ZWrite On` 表示写深度。

`ZTest Always` 表示不管当前深度是什么，都把这个 UI 区域写进去。

顶点阶段把 viewport 坐标转成 clip space：

```hlsl
float3 viewportPos = mul(UNITY_MATRIX_M, float4(input.positionOS.xyz, 1.0)).xyz;
float2 clipXY = viewportPos.xy * 2.0 - 1.0;

#if UNITY_UV_STARTS_AT_TOP
clipXY.y = -clipXY.y;
#endif

output.positionCS = float4(clipXY, UNITY_NEAR_CLIP_VALUE, 1.0);
```

这里的 `UNITY_NEAR_CLIP_VALUE` 会把深度写到近裁剪面。

也就是说，这块 UI 区域会成为非常靠近相机的深度。

后面的 Opaque 物体只要落在这个屏幕区域内，就会被深度测试挡掉。

片元阶段虽然返回了颜色，但因为 `ColorMask 0`，所以不会真的写进颜色缓冲。

```hlsl
half4 frag(Varyings input) : SV_Target
{
    return half4(1, 1, 1, 1);
}
```

## Mesh 怎么生成

如果只是矩形遮挡，不需要生成 Mesh，默认 Quad 就够了。

如果要根据 Sprite Alpha 生成异形遮挡，可以在 Editor 工具里做一次离线生成。

思路是：

- 读取 Sprite 的 alpha
- 按固定分辨率采样成一个 bool 网格
- 把连续区域合并成矩形
- 输出一个 normalized `[0,1]` 空间的 Mesh

这个 Mesh 只负责描述 UI 的遮挡形状，不参与真实 UI 绘制。

UI 显示还是由原来的 Image 完成，深度写入由 `UIDepthOccluder` 完成。

## 代价和限制

这个方案很直接，但是有几个限制：

- 它主要影响 Opaque 阶段，因为 Pass 插在 `BeforeRenderingOpaques`
- 透明物体是否会被挡住，取决于后续透明 Shader 的深度测试和渲染顺序
- 当前矩阵适合不旋转的矩形 UI，旋转 UI 只用左下角和右上角会不准确
- Image 的继承 alpha 必须大于等于 1，半透明 UI 不会注册
- Feature 只处理 `MainCamera`

还有一个源码细节需要注意。

兼容模式分支里使用的是：

```csharp
var occluders = UIDepthOccluder.ActiveOccluders;
```

但是当前运行时代码里实际字段是：

```csharp
UIDepthOccluder.s_ActiveOccluders
```

如果要启用 `URP_COMPATIBILITY_MODE`，需要把这两个名字统一一下。

## 排查方式

如果接入后没有效果，可以按这个顺序查：

- 确认当前相机是否带 `MainCamera` tag
- 确认 URP Renderer Data 上已经挂了 `UIDepthOccluderFeature`
- 确认 Feature 使用的是 `Hidden/UnityToolkit/UIDepthOccluder`
- 确认 UI Image 的继承 alpha 是否为 1
- 确认 `RectTransform` 的世界角点是否落在屏幕内
- 用 Frame Debugger 看 `UIDepthOccluder` Pass 是否在 Opaque 前执行
- 临时把 Shader 的 `ColorMask 0` 去掉，看遮挡 Mesh 是否画在预期位置

这类问题最好先确认 Pass 有没有执行，再看深度是否写对。不要一上来就怀疑深度测试。

## 小结

`UIDepthOccluder` 的核心不是让 UI 参与正常场景绘制。

它只是借用 UI 的屏幕区域，在 Opaque 之前写一块深度。

这样落在 UI 覆盖区域里的场景像素，就会在深度测试阶段提前被剔除。

这个思路适合一些明确的屏幕空间不透明覆盖区域，比如底部技能栏、侧边面板、头像框、固定 HUD 边框。

如果项目里有大块固定的不透明 UI，它也可以作为一种性能优化手段：提前把这些 UI 区域写入深度，让后续场景物体通过深度测试自然剔除。

只要把它理解成“屏幕空间深度写入器”，整个实现就比较清楚了。
