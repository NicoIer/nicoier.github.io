---
title: Unity Shader
date: 2024-06-17 18:48:35
tags: [Unity, Shader, 图形学]
---

## 渲染管线

- **渲染管线(Rendering Pipeline)：** 一提到管线，感觉很高大上的样子。说的俗一点就是可以理解为流水线。渲染管线我们可暂时理解为 **从得到模型数据到绘制出图像** 这一过程的称呼。
- **Vertex Shader：** 对顶点数据编程的一段程序。 人类有懒惰的天性，习惯用简化的词汇来表达同一个东西。对 Vertex Shader 也不例外，一般称其为 VS ，但是在本系列文章中会保持全称。
- **Fragment Shader：** 对像素数据编程的一段程序。这里 fragment 可以理解为带有信息（颜色，坐标等）的像素 (Pixel), 一般也简称其为 FS 或者 PS 。 在本系列文章中会保持其全称。
- **FrameBuffer：** 缓存帧数据的存储区，它一般包含的是要显示到显示设备上的位图数据（也就是图片数据）。
- **Fixed Function：** 由于一些硬件支持等历史原因，早期的图形 API **只支持对 GPU 做配置**，这部分只可配置的功能就是 fixed fucntion。这里注意下，fixed function 的功能只能配置，不像 Vertex Shader　和 fragment Shader 可以编程（写自己的算法）。

![2.renderingpipeline.jpg](https://wudixiaop.github.io/images/Shader/2/rendering-pipeline.jpg)



## Shader语言

[Unity - Manual: ShaderLab (unity3d.com)](https://docs.unity3d.com/Manual/SL-Reference.html)

- **CG** ： C For Graphics，是一种专门为图形编程设计的语言，它是一种高级语言，可以编译成汇编语言，也可以编译成高级语言。
- **HLSL** ： High Level Shader Language，是微软公司为DirectX开发的一种高级着色器语言。
- **GLSL** ： OpenGL Shading Language，是OpenGL的着色器语言。



## 坐标系

- 物体坐标系 **ObjectSpace**
- 世界坐标系 **WorldSpace**
- 视口坐标系 **ViewportSpace**
- 屏幕坐标系 **ScreenSpace**



左手和右手坐标系



### MeshRender是如何渲染的

[Unity - Manual: Built-in shader variables (unity3d.com)](https://docs.unity3d.com/Manual/SL-UnityShaderVariables.html)

以立方体为例，Mesh Renderer 组件得到模型数据之后它会执行 vertex shader。vertex shader 里面做了下面这些事：

1. 先把立方体从模型的物体坐标系转换成世界坐标系，**从 物体 到 世界**。这样子，它和摄像机（世界坐标）的位置就用同一个坐标系描述了。
2. 再把立方体从世界坐标转换成视口坐标系，也就是摄像机因为原点的坐标系，**从 世界 到 视口**。这样它是在摄像机的正面，还是在反面了。
3. 最后在投射到屏幕坐标系上， **从 视口 到 屏幕**。这样知道哪些区域需要绘制在屏幕上，哪些不需要。

总结上面一系列变换关系就是： **物体 到 世界 再到 视口 再到 屏幕**。中间经过了三次变换 (transform)。这些变换在数学上通过 **矩阵** 来描述的。



## 颜色

[Photoshop图层混合模式详解 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/94081709)


## Tips

### 平台差异

- DX 和 Open GL 存在屏幕空间坐标差异。 DX中的屏幕坐标系原点在屏幕的左上角，而OpenGL中的屏幕坐标系原点在屏幕的左下角。

### 慎用分支语句

- if else 语句会导致分支预测，分支预测会导致性能下降 
- if else 中使用的变量最好是常量 在编译时就能确定分支的走向
- 分支中的操作尽量简单，不要有复杂的计算
- 嵌套的分支语句会导致性能下降


### 不要除以0
- 除以0不会报错


## Shaderlab

- Shader最精简的骨架

```HLSL
Shader "XXXXName" {

    Subshader {

        Pass { }
    }
}
```

- 通用的Shader模版，包含注释

```HLSL
Shader "#NAME#"
{
    /* Properties区域
     * 
     * Shaderlab 提供的一种用于在Inspector中显示Shader属性的机制
     * 通过Properties区域定义的属性，可以在Inspector中显示，并且可以在Shader中使用
     * 支持的类型查看Unity文档 https://docs.unity3d.com/Manual/SL-Properties.html
     * 
     * 如何定义
     * 属性名("Inspector显示名", 类型) = "默认值" { }
     * 
     * 如何与HLSL关联 
     * 需要在HLSL中声明一个同名的变量，这样Unity会自动将Inspector中的属性值赋值给HLSL中的变量
     */

    Properties
    {
        _BaseMap ("Texture", 2D) = "white" {}// 主贴图
    }
    /* SubShader是Shader的主要部分，包含了一系列的Pass
     * Pass是渲染管线的一个阶段，包含了一个顶点着色器和一个片元着色器
     * 可能存在多个SubShader，Unity会选择当前环境可用的第一个SubShader
     */
    SubShader
    {
        /* Tag 是一个键值对，它的作用是告诉渲染引擎，应该 什么时候 怎么样 去渲染 
         * https://docs.unity.cn/cn/2022.3/Manual/SL-SubShaderTags.html
         */
        Tags
        {
            "RenderType"="Opaque"
            "RenderPipeLine"="UniversalRenderPipeline" //用于指明使用URP来渲染
        }
        /* 通用区域 这里内容可以在多个Pass中共享 */

        HLSLINCLUDE
        #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
        #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"
        #include "Packages/com.unity.render-pipelines.core/ShaderLibrary/UnityInstancing.hlsl"
        #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/DeclareDepthTexture.hlsl"


        // 支持SRP Batcher
        CBUFFER_START(UnityPerMaterial)
            //声明变量
            float4 _BaseMap_ST;
        CBUFFER_END

        TEXTURE2D(_BaseMap); //贴图采样  
        SAMPLER(sampler_BaseMap);

        /* 渲染管线整个流水线都有数据的输入和输出
         * 这些数据是什么? 从哪里输入? 输出到哪里? Shader中的变量如何与这些数据交互?
         */

        /* 常用种类
         * POSITION: 顶点位置
         * NORMAL: 顶点法线
         * TANENT: 顶点切线
         * COLOR: 顶点颜色
         * TXECOORD0: 纹理坐标 UV0 -(UV是一张图 可以有多个通道 UV0表示第一个通道)
         * TEXCOORD1: 纹理坐标 UV1
         * TEXCOORD2: 纹理坐标 UV2
         * TEXCOORD3: 纹理坐标 UV3
         *
         *
         * 输入输出
         * 数据存放在寄存器中，输入从寄存器中读取，输出写入寄存器
         *
         *
         * Shader中的变量如何与这些数据交互
         * 语义绑定
         * void xxFunc(xxxType xxName : XXX) -(作函数参数标记)
         * xxxType xxFunc() : XXX -(作函数返回值标记)
         * struct xxStruct{ 
         *   xxxType xxName : XXX; -(作结构体成员标记)
         * }
         * 输入输出怎么表示
         * in -> 输入
         * out -> 输出
         * inout -> 输入输出
         */

        // a2v -> attribute to vertex
        struct a2v //顶点着色器
        {
            float4 positionOS: POSITION;
            float3 normalOS: TANGENT;
            half4 vertexColor: COLOR;
            float2 uv : TEXCOORD0;
        };

        // v2f -> vertex to fragment
        struct v2f //片元着色器
        {
            float4 positionCS: SV_POSITION;
            float2 uv: TEXCOORD0;
            half4 vertexColor: COLOR;
        };
        ENDHLSL

        Pass
        {
            // Pass的标签
            Tags {}
            HLSLPROGRAM

            // 编译指令 
            #pragma vertex vert // vertex shader 的函数 是 vert
            #pragma fragment frag // fragment shader 的函数 是 frag


            v2f vert(a2v v)
            {
                v2f o;
                o.positionCS = TransformObjectToHClip(v.positionOS.xyz);
                o.uv = TRANSFORM_TEX(v.uv, _BaseMap);
                o.vertexColor = v.vertexColor;
                return o;
            }

            half4 frag(v2f i) : SV_Target /* 注意在HLSL中，fixed4类型变成了half4类型*/
            {
                half4 col = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, i.uv);
                half res = lerp(i.vertexColor, col, i.vertexColor.g).x;
                return half4(res, res, res, 1.0);
            }
            ENDHLSL
        }
    }

    /* Fallback是一个备用Shader，当当前Shader不支持时，会使用Fallback指定的Shader
     * 通常是Unity内置的Shader
     */
    Fallback "Universal Render Pipeline/Unlit"
}
```



## 谁先被渲染

Subshader中的Tags选项用于告诉渲染引擎，什么时候，怎么样去渲染。

支持的Tag可以查阅Unity文档：[ShaderLab：向子着色器分配标签 - Unity 手册](https://docs.unity.cn/cn/2022.3/Manual/SL-SubShaderTags.html)



其中Queue是决定对象渲染顺序的标签，越小越先渲染，但是不能填数值，预先定义了5个词代替数值

- **Background：** 对应数值为 1000，用于需要被最先渲染的对象，如背景什么的。
- **Geometry：** 对应数值为 2000, 用于不透明的物体。这个是默认的选项（如果不指明 Queue 标签的值，自动给你指定为 Geometry）。
- **AlphaTest：** 对应的数值为 2450, 用于需要使用 AlphaTest 的对象来提高性能。AlphaTest 类似于裁剪 (clip) 功能。
- **Transparent：** 对应的数值为 3000， 用于需要使用 alpha blending 的对象，比如粒子，玻璃等。
- **Overlay：** 对应的数值为 4000，用于最后被渲染的对象，比如 UI。

虽然不能直接填数值，但是支持**加减法**

```HLSL
Shader "XXXName" {

    SubShader {
        Tags { "Queue" = "Transparent" }

        Pass {

        }
    }
}
```

## Pargama指令

用于告诉shaderlab编译器，应该这样这样，那样那样。

```HLSL
Shader "XXXName" {

    Subshader {

        pass {
            // CGPROGRAM ... ENDCG 在 Pass 里面
            HLSLPROGRAM

            // vertex shader 的函数是 vert
            #pragma vertex vert

            // fragment shader 的函数是 fragment
            #pragma fragment frag

            void vert() {

            }

            void frag () {

            }

            ENDHLSL
        }
    }
}
```

## Property区域

Shaderlab 提供的一种用于在Inspector中显示Shader属性的机制，通过Properties区域定义的属性，可以在Inspector中显示，并且可以在Sub Shader中使用。

- *属性名("Inspector显示名", 类型) = "默认值" { }*

支持的类型 https://docs.unity3d.com/Manual/SL-Properties.html

需要在HLSL中声明一个同名的变量，这样Unity会自动将Inspector中的属性值赋值给HLSL中的变量

```
Shader "#NAME#"
{
    Properties
    {
        _BaseMap ("Texture", 2D) = "white" {}// 主贴图
    }
    SubShader
    {
        Pass
        {
        }
    }
}
```

## 语义绑定

渲染管线整个流水线都有数据的输入和输出，这些数据是什么? 从哪里输入? 输出到哪里? Shader中的变量如何与这些数据交互?

**常用种类?**

- POSITION: 顶点位置
- NORMAL: 顶点法线
- TANENT: 顶点切线
- COLOR: 顶点颜色
- TXECOORD0: 纹理坐标 UV0
- TEXCOORD1: 纹理坐标 UV1
- TEXCOORD2: 纹理坐标 UV2
- TEXCOORD3: 纹理坐标 UV3

 **输入输出?**

 数据存放在寄存器中，输入从寄存器中读取，输出写入寄存器



 **Shader中的变量如何与这些数据交互?**
语义绑定

```HLSL
void xxFunc(xxxType xxName : XXX) -(作函数参数标记)
  	xxxType xxFunc() : XXX -(作函数返回值标记)
  	struct xxStruct{ 
   	xxxType xxName : XXX; -(作结构体成员标记)
}
```



**输入输出怎么表示**

- in -> 输入
- out -> 输出
- inout -> 输入输出



```HLSL
Shader "Custom/Shader10" {
    SubShader {
        Tags { "RenderType"="Opaque" }

        pass {

            CGPROGRAM

            #pragma vertex vert
            #pragma fragment frag

            // 结构体中使用语义绑定
            struct VertexOutput {
                float4 pos :SV_POSITION;	   	// 转换到投射空间后位置
                float4 texcoord :TEXCOORD0;		// 顶点颜色
            };


            VertexOutput vert(in float4 pos :POSITION /*参数中使用语义绑定*/)
            {
                VertexOutput output;
                output.pos = mul(UNITY_MATRIX_MVP, pos);
                output.texcoord = pos + float4(0.5, 0.5, 0.5, 0);
                return output;
                }

            float4 frag(VertexOutput input) :COLOR // 函数后面使用语义绑定
            {
                return input.texcoord;
            }

            ENDCG
        }

    }
}
```



## 深度缓存

[Unity - Manual: ShaderLab: commands (unity3d.com)](https://docs.unity3d.com/Manual/shader-shaderlab-commands.html)

FrameBuffer时用于存储帧位图的数据存储区域。深度缓存（Depth Buffer）也叫 Z-Buffer，这是用于存储深度数据的存储区。

- 存储的深度数据是什么？
- 有什么用？

**深度是什么？**

描述物体的位置，需要有参考系。深度值的参考系是观察者的视角 - 相机视角。在Vertex Shader之后有一个插值过程，用于生成像素，像素的XY坐标为屏幕坐标，Z坐标就是深度值，存储在深度缓存里面。

**有什么用？**

如果两个物体渲染后的像素的屏幕坐标XY相同，那么应该渲染谁？谁离更近就先渲染谁。

在Shaderlab中用 **ZTest**表示，有以下预设值，默认值是 LEqual ，意思是 渲染小于等于的那一个

```HLSL
ZTest Less | Greater | LEqual | GEqual | Equal | NotEqual | Always
```

如果两个像素的Z也相同呢？深度值一样的情况也叫做 **深度冲突** (Z-fighting)。解决方法是给其中某一个物体设置偏移量。 Shaderlab 中语法是:

```HLSL
Offset Factor, Units
```

Offset 根据一个插值公式来计算出新的深度值 [glPolygonOffset 函数 (Gl.h) - Win32 apps | Microsoft Learn](https://learn.microsoft.com/zh-cn/windows/win32/opengl/glpolygonoffset?redirectedfrom=MSDN)

也可以手动打开和关闭深度写入功能，在 Shaderlab 中用 ZWrite 来控制，它的语法是:

```HLSL
ZWrite On | Off
```

**在Unity渲染中的位置**

在顶点着色器之后，片元着色器之前

![PipelineCullDepth](https://wudixiaop.github.io/images/Shader/11/PipelineCullDepth.png)









