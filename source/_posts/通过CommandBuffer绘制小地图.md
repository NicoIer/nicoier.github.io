---
title: 通过CommandBuffer绘制小地图
date: 2025-08-07 11:10:40
tags: [ Unity, CommandBuffer, 小地图, Shader ,Mask]
---

最近在做小地图的优化方案， 原先的方案很常见：

为目标物体在场景内很远的位置生成一个SpriteRenderer代理，使用一个小地图相机在对应位置拍摄，

小地图相机的渲染目标是一张正方形的RT，RT会作为UI上一个RawImage的纹理。

通过Camera->RT->RawImage的方式，最终在UI上显示小地图。

这样的做法有几个问题：

- 需要额外一个Camera，需要完整走Camera的渲染流程，有一些流程无法跳过，比如剔除、渲染排序等，这些都会影响性能
- 需要额外一张RT，增加了带宽开销，这在低端机几乎是不可接受的

然后是新的方案，将小地图绘制插入到UI绘制流程中，使用CommandBuffer在UI绘制目标上绘制小地图（通常是直接绘制到屏幕）。
这样就可以避免额外的Camera和RT开销。

- 首先要收集需要绘制的物体
- 然后收集投影矩阵
- 进行Viewport转换，计算绘制位置
- 在限制的区域内绘制

第一步直接使用原方案中的SpriteRenderer代理

第二步直接使用原方案中的小地图相机的投影矩阵（只不过这次要禁用掉Camera，只使用矩阵数据）

第三步因为我们的绘制目标是UI的绘制目标，这里以1920*1080为例，实际上我们要绘制的目标是1920*1080中的一小部分区域，因此需要做Viewport转换。

假设小地图的尺寸是Vector2 size，小地图正方形的左上角距离屏幕左上角是Vector2 offset,那么Viewport转换就是：

```csharp
var viewport = new Rect(0,Screen.height - size.x, size.x, size.y);
viewport.x += offset.x;
viewport.y -= offset.y;

float targetAspect = size.x / size.y;
float screenAspect = Screen.width / (float)Screen.height;

if (Math.Abs(screenAspect - targetAspect)>0.01f)`
{
    // 屏幕宽度大于小地图宽高比，计算缩放比例
    float scale = screenAspect / targetAspect;
    viewport.width *= scale;
}
   
```

第四步需要重写一下Sprite-Default的Shader，在Frag里限制绘制区域（这一步也可以Stencil实现)
只考虑Screen.x > Screen.y的情况，圆形中心在屏幕中心uv(0.5,0.5)，半径的像素数量为Screen.y /2，直接用0.5uv作为半径会导致结果并不是圆形，而是椭圆，因为Screen.x!=Screen.y，所以需要对X坐标进行缩放。

```shaderlab

            Varyings Vert(Attributes v)
            {
                // ......
                o.screenPos =  ComputeScreenPos(o.positionCS);
                o.color = v.color * _Color * _RendererColor;
                // ......
                return o;
            }
            half4 Frag(Varyings i) : SV_Target
            {
                // ......
                float2 center = float2(0.5, 0.5);
                float aspect = _ScreenParams.x / _ScreenParams.y;
                float2 pixelUv = i.screenPos.xy / i.screenPos.w;
                pixelUv.x = (pixelUv.x - center.x) * aspect + center.x; // Adjust for aspect ratio
                if (length(pixelUv - center) > 0.5)
                {
                    // If the pixel is outside the circle, discard it.
                    discard;
                }
                // ......
            }
```