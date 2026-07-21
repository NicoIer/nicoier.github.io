---
title: Unity UI深度遮挡 UIDepthOccluder
date: 2026-07-08 00:00:00
tags: [Unity, URP, RenderGraph, UI, Shader, 深度, 游戏开发]
categories: [Unity]
---

https://github.com/NicoIer/UnityToolkit/blob/main/Runtime/Renderer/UIDepthOccluder.cs

https://github.com/NicoIer/UnityToolkit/blob/main/Runtime/Renderer/UIDepthOccluderFeature.cs

# UIDepthOccluder

大块不透明 UI 最后肯定会盖住场景，但 UI 绘制之前，它背后的 Opaque 物体还是会正常经过光栅化和片元着色。

这里的方案是在 `BeforeRenderingOpaques` 阶段，先按 UI 的屏幕范围写一块近裁剪面深度。后面的 Opaque 像素会尽量在深度测试阶段被挡掉。

它不会减少 CPU 提交、DrawCall 或顶点处理，主要省的是被 UI 覆盖区域的片元开销，而且后续 Shader 必须正常使用深度测试。UI 是半透明的、面积很小或者背后的 Shader 很便宜时，多画一次深度可能反而更亏。

## 流程

- `UIDepthOccluder` 收集当前完全不透明的 UI
- 把 `RectTransform` 转成 viewport 范围
- `UIDepthOccluderFeature` 在 Opaque 前插入 RenderPass
- Shader 只写深度，不写颜色

`UIDepthOccluder` 挂在 UI Image 上，Renderer Feature 挂在 URP Renderer Data 上。

Shader 使用：

```text
Hidden/UnityToolkit/UIDepthOccluder
```

默认用一个矩形 Quad。如果是圆形头像或异形边框，可以传自定义 `occluderMesh`。Mesh 顶点使用 normalized `[0,1]` 坐标，`(0,0)` 是左下角，`(1,1)` 是右上角。

## 收集遮挡体

组件启用时检查 Image 的继承 alpha，完全不透明才注册：

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

淡入淡出过程中 alpha 会变化，所以 `LateUpdate` 还要继续同步注册状态：

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

alpha 小于 1 时不能写这块深度，否则 UI 还没完全显示，后面的场景已经被挡掉了。

## 转换到 Viewport

默认 Mesh 在 `[0,1]` 范围内，绘制前用 `RectTransform` 的世界角点算出 viewport 范围。

下面截取的是 `ScreenSpaceOverlay` 分支，世界角点可以直接除以屏幕尺寸：

```csharp
_rectTransform.GetWorldCorners(_worldCorners);

viewMin = new Vector2(
    _worldCorners[0].x / Screen.width,
    _worldCorners[0].y / Screen.height);
viewMax = new Vector2(
    _worldCorners[2].x / Screen.width,
    _worldCorners[2].y / Screen.height);

var sx = viewMax.x - viewMin.x;
var sy = viewMax.y - viewMin.y;
```

然后构造 `localToViewport`：

```csharp
return new Matrix4x4(
    new Vector4(sx, 0, 0, 0),
    new Vector4(0, sy, 0, 0),
    new Vector4(0, 0, 1, 0),
    new Vector4(viewMin.x, viewMin.y, 0, 1)
);
```

其他 Canvas 模式会先取 `canvas.worldCamera` 或当前相机，再用 `WorldToScreenPoint` 转到屏幕坐标。最后得到的矩阵会直接传给 `cmd.DrawMesh`。

当前算法只取左下角和右上角，适合不旋转的矩形 UI。UI 有旋转时范围会不准。

## 在 Opaque 前写深度

Pass 的执行时机：

```csharp
_pass = new UIDepthOccluderPass(_material)
{
    renderPassEvent = RenderPassEvent.BeforeRenderingOpaques
};
```

RenderGraph 路径直接写当前激活的深度纹理：

```csharp
using (var builder = renderGraph.AddRasterRenderPass<PassData>(
    "UIDepthOccluder", out var passData))
{
    builder.SetRenderAttachmentDepth(
        resourceData.activeDepthTexture,
        AccessFlags.Write);
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

每个遮挡体是一次 `DrawMesh`。没有配置 Mesh 时使用默认 Quad：

```csharp
vertices = new[]
{
    new Vector3(0, 0, 0),
    new Vector3(0, 1, 0),
    new Vector3(1, 1, 0),
    new Vector3(1, 0, 0)
};
```

当前 Feature 只处理带 `MainCamera` tag 的相机：

```csharp
if (!cameraData.camera.CompareTag("MainCamera")) return;
```

项目里的主相机没有这个 tag 时，Pass 不会执行。

## Shader

关键状态只有几个：

```shaderlab
ColorMask 0
ZWrite On
ZTest Always
Cull Off
```

顶点阶段把 viewport 坐标转成 clip space，并把深度放到近裁剪面：

```hlsl
float3 viewportPos =
    mul(UNITY_MATRIX_M, float4(input.positionOS.xyz, 1.0)).xyz;
float2 clipXY = viewportPos.xy * 2.0 - 1.0;

#if UNITY_UV_STARTS_AT_TOP
clipXY.y = -clipXY.y;
#endif

output.positionCS = float4(
    clipXY,
    UNITY_NEAR_CLIP_VALUE,
    1.0);
```

`ColorMask 0` 不写颜色，`ZTest Always` 保证这块区域的深度被覆盖。后续落在这里的 Opaque 像素会被深度测试剔除。

## 异形 Mesh

矩形 UI 直接用默认 Quad。

如果要按 Sprite Alpha 生成异形遮挡，可以在 Editor 中离线处理：

- 采样 Sprite alpha，得到 bool 网格
- 合并连续区域
- 输出 normalized `[0,1]` 空间的 Mesh

这个 Mesh 只描述遮挡范围。UI 本身仍然由原来的 Image 绘制。

## 限制

- Pass 在 `BeforeRenderingOpaques`，主要优化 Opaque；透明物体是否被挡取决于它自己的深度状态
- 只支持继承 alpha 为 1 的 Image
- 当前 viewport 矩阵不支持旋转矩形
- 每个遮挡体都会增加一次深度绘制
- 只处理 `MainCamera`

兼容模式还有一个命名问题。分支里读取的是：

```csharp
UIDepthOccluder.ActiveOccluders
```

运行时代码实际字段是：

```csharp
UIDepthOccluder.s_ActiveOccluders
```

启用 `URP_COMPATIBILITY_MODE` 前需要把名字统一。

## 没有效果时

- 检查相机的 `MainCamera` tag
- 检查 Renderer Data 是否添加 `UIDepthOccluderFeature`
- 检查 Shader 是否为 `Hidden/UnityToolkit/UIDepthOccluder`
- 检查 Image 的继承 alpha
- 用 Frame Debugger 确认 Pass 在 Opaque 前执行
- 临时去掉 `ColorMask 0`，观察遮挡 Mesh 的位置

先确认 Pass 有没有执行，再查矩阵和深度。
