---
title: 如何让Shader支持DOTS
date: 2025-11-19 11:04:11
tags: [Unity,DOTS,Shader,SRP Batch,GPU Instancing]
categories: [Unity]
---

DOTS Shader 需要同时支持SRP Batch 和 GPU Instacing
参考Unity官方文档修改即可
- https://zhuanlan.zhihu.com/p/82102092
- https://docs.unity3d.com/2022.3/Documentation/Manual/dots-instancing-shaders.html

# 示例Shader
```shaderlab

Shader "DOTS/Example"
{
Properties
{
_BaseMap ("Texture", 2D) = "white" {}// 主贴图
_Color ("Color", Color) = (1,1,1,1) //颜色
}
SubShader
{
Tags
{
"RenderType"="Opaque"
"RenderPipeLine"="UniversalRenderPipeline" //用于指明使用URP来渲染
}
Pass
{
// Pass的标签
Tags {}
HLSLPROGRAM
// 编译指令
#pragma vertex vert // vertex shader 的函数 是 vert
#pragma fragment frag // fragment shader 的函数 是 frag

            #pragma multi_compile_instancing
            #include_with_pragmas "Packages/com.unity.render-pipelines.universal/ShaderLibrary/DOTS.hlsl"


            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            

            // 支持SRP Batcher
            CBUFFER_START(UnityPerMaterial)
                //声明变量
                float4 _BaseMap_ST;
                float4 _Color;
            CBUFFER_END

            TEXTURE2D(_BaseMap); //贴图采样  
            SAMPLER(sampler_BaseMap);


            #ifdef UNITY_DOTS_INSTANCING_ENABLED
            UNITY_DOTS_INSTANCING_START(MaterialPropertyMetadata)
                UNITY_DOTS_INSTANCED_PROP(float4, _Color)
            UNITY_DOTS_INSTANCING_END(MaterialPropertyMetadata)
            #endif


            //        a2v -> attribute to vertex
            struct a2v //顶点着色器
            {
                float4 positionOS: POSITION;
                float2 uv : TEXCOORD0;
                UNITY_VERTEX_INPUT_INSTANCE_ID
            };

            // v2f -> vertex to fragment
            struct v2f //片元着色器
            {
                float4 positionCS: SV_POSITION;
                float2 uv: TEXCOORD0;
                UNITY_VERTEX_INPUT_INSTANCE_ID
            };


            v2f vert(a2v v)
            {
                v2f o;
                UNITY_SETUP_INSTANCE_ID(v);
                UNITY_TRANSFER_INSTANCE_ID(v, o);
                UNITY_INITIALIZE_VERTEX_OUTPUT_STEREO(o);
                o.positionCS = TransformObjectToHClip(v.positionOS.xyz);
                o.uv = TRANSFORM_TEX(v.uv, _BaseMap);
                return o;
            }

            half4 frag(v2f i) : SV_Target /* 注意在HLSL中，fixed4类型变成了half4类型*/
            {
                UNITY_SETUP_INSTANCE_ID(i);
                half4 col = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, i.uv);
                return col * _Color;
            }
            ENDHLSL
        }
    }

    //    Fallback "Universal Render Pipeline/Unlit"
}
```
