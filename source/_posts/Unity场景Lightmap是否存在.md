---
title: Unity场景Lightmap是否存在
date: 2025-07-22 14:19:35
tags: [Unity, Lightmap]
---

## Unity场景Lightmap是否存在

判断当前激活的场景是否存在Lightmap，可以通过以下代码实现：
```csharp
var dataAsset = Lightmapping.lightDataAsset;
if(dataAsset != null && dataAsset.lightmaps != null && dataAsset.lightmaps.Length > 0)
{
    Debug.Log("Lightmap exists in the scene.");
}
else
{
    Debug.Log("No Lightmap found in the scene.");
}
```

通常，在美术对场景编辑后，会有一系列处理流程，通过检查Lightmap的存在，可以确保场景的光照效果已经被正确烘焙。
结合自动化测试，可以在场景加载时自动检查Lightmap的存在性，确保场景的光照效果符合预期。