---
title: Unity Shader LOD
date: 2025-05-22 14:54:20
tags: [Unity, Shader, 性能优化]
---


# Shader LOD



在移动端游戏上，常常会做很多优化工作，来让游戏在设备上流畅运行。
一般的方案有：降低模型的复杂度，降低纹理的分辨率，降低模型的LOD（Level of Detail）等。

针对不同的设备还有有画质分级


这里要讲的是Shader LOD:
https://docs.unity.cn/cn/2022.3/Manual/SL-ShaderLOD.html

## 示例代码

```csharp
Shader myShader = Shader.Find("MyShader"); // 任何可以获得Shader对象的方法
myShader.maximumLOD = -1; // 设置LOD为-1，表示能多大用多大

myShader.maximumLOD = 100; // 设置LOD为100，表示只能使用LOD <=100的Shader

myShader.maximumLOD = 10; // 设置LOD为10，表示只能使用LOD <=10的Shader
```

```shaderlab
Shader "MyShader"
{
    // ... 其他的Shader内容...
    SubShader
    {
        LOD 100
        
    // ... 其他的SubShader内容...    
    }
    SubShader
    {
        LOD 10
        
    // ... 其他的SubShader内容...    
    }
```

## 用途

Shader LOD（Level of Detail）用于在不同的设备上使用不同的Shader版本，以优化性能和视觉效果。通过设置LOD值，Unity可以根据设备的性能自动选择合适的Shader版本，从而提高渲染效率。
