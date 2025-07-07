---
title: FMOD Unity 资源加载
date: 2025-07-07 14:39:49
tags: [Unity, FMOD, 音频]
---


# FMOD Unity 资源加载

FMOD 是一个强大的音频引擎，广泛应用于游戏开发中。Unity 中使用 FMOD 时，资源的加载和管理是一个重要的环节。以下是一些关于 FMOD 在 Unity 中资源加载的基本知识。

## FMOD Unity 资源加载

### 默认资源加载方式
FMOD For Unity的默认资源加载方式是 StreamingAssets。
在Unity Build时会将 ".bank" 文件 自动拷贝到 StreamingAssets 文件夹中。
它在游戏初始化的时候根据路径去SteamingAssets文件夹中加载音频资源。



### 运行时加载
如果需要在运行时动态加载音频资源，可以使用 FMOD 的 API 来实现。
```csharp
RuntimeManager.LoadBank("path/to/bank.bank", true);
RuntimeManager.LoadBank("path/to/bank.bytes", true);

TextAsset textAsset = Resources.Load<TextAsset>("path/to/bank");
RuntimeManager.LoadBank(textAsset, true);

```


## AssetBundle 加载

如果我们的游戏需要资源热更新，那么默认的 StreamingAssets 方式就不适用了。

1. 在FMOD Editor Settings 中 修改加载方式为 AssetBundle
2. FMOD会自动根据.bank文件生成对应的 ".bytes" 文件。
   这些 ".bytes" 文件包含了音频数据和相关信息，可以在 Unity 中使用。
3. 在Unity中创建一个AssetBundle，包含FMOD生成的 ".bytes" 文件。
4. 在游戏运行时，通过 Unity 的 AssetBundle API 加载这些 ".bytes" 文件。

```csharp
AssetBundle assetBundle = AssetBundle.LoadFromFile("path/to/assetbundle");
TextAssetk textAsset = assetBundle.LoadAsset<TextAsset>("bank.bytes");
FMODUnity.RuntimeManager.LoadBank(bank, true);
```

