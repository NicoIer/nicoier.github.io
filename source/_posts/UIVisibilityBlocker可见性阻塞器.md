---
title: UIVisibilityBlocker 可见性阻塞器
date: 2026-07-09 16:00:00
description: 多个系统同时控制一个 UI 时，用来源表合并显隐和交互状态，避免互相覆盖。
tags: [Unity, UGUI, UIFramework, CSharp]
categories: [Unity]
---

# UIVisibilityBlocker

源码：

```text
Runtime/UIFramework/UIVisibilityBlocker.cs
```

这个组件用来处理多个地方同时控制一个 UI 的情况，其实很类似之前写的Capability中的发起者模式。

比如 Loading 和新手引导都隐藏了按钮。Loading 先结束时不能直接 `SetActive(true)`，因为新手引导还没结束。

`UIVisibilityBlocker` 会记录每个隐藏来源，只有相关来源全部解除后才恢复。

![UIVisibilityBlocker 流程](../img/ui-visibility-blocker-flow.svg)

## 数据结构

`VisibilityMode` 决定怎么应用状态：

```csharp
public enum VisibilityMode
{
    GameObject,
    CanvasGroup,
}
```

`GameObject` 模式用 `SetActive` 控制显示，`CanvasGroup` 模式修改透明度和交互。

具体阻塞哪些内容由一个 `Flags` 枚举控制：

```csharp
[Flags]
public enum VisibilityModeEnum
{
    Alpha = 1 << 0,
    Interactable = 1 << 1,
    BlockRaycasts = 1 << 2,

    All = Alpha | Interactable | BlockRaycasts
}
```

所以可以只隐藏、只禁用点击，也可以全部阻塞。

每个来源和它的阻塞内容保存在字典里：

```csharp
private readonly Dictionary<string, VisibilityModeEnum> _blockSources;
```

## Block 和 Unblock

```csharp
public void Block(string source, VisibilityModeEnum mode = VisibilityModeEnum.All)
{
    _blockSources[source] = mode;
    UpdateVisibility();
    UpdateDebugInfo();
}

public void Unblock(string source)
{
    _blockSources.Remove(source);
    UpdateVisibility();
    UpdateDebugInfo();
}
```

同一个 `source` 再次调用 `Block` 会覆盖旧的 mode，不是增加一次引用计数。解除时也必须传同一个字符串，最好直接定义常量：

```csharp
const string SourceLoading = "Loading";
const string SourceGuide = "Guide";

blocker.Block(SourceLoading);
blocker.Unblock(SourceLoading);
```

所有来源的 mode 通过按位 OR 合并：

```csharp
private VisibilityModeEnum GetBlockedMode()
{
    var mode = 0;
    foreach (var blockSource in _blockSources)
    {
        mode |= (int)blockSource.Value;
    }

    return (VisibilityModeEnum)mode;
}
```

例如：

```text
Loading -> Alpha
Guide   -> Interactable | BlockRaycasts

最终状态 = Alpha | Interactable | BlockRaycasts
```

只要还有一个来源包含某个 flag，这部分状态就不会恢复。

## 两种应用方式

### GameObject

`GameObject` 模式下，`Alpha` 对应 `Target.SetActive`：

```csharp
if (ShouldApplyMode(blockedMode, VisibilityModeEnum.Alpha))
    Target.SetActive(!blockedMode.HasFlag(VisibilityModeEnum.Alpha));
```

如果节点上还有 `CanvasGroup`，`Interactable` 和 `BlockRaycasts` 仍然会应用到它。

这种方式会真的禁用节点。UI 隐藏后不需要继续参与布局、Update 和子节点逻辑时可以用它。

### CanvasGroup

`CanvasGroup` 模式不关闭 GameObject：

```csharp
private void ApplyCanvasGroupMode(CanvasGroup cg, VisibilityModeEnum blockedMode)
{
    if (ShouldApplyMode(blockedMode, VisibilityModeEnum.Alpha))
        cg.alpha = blockedMode.HasFlag(VisibilityModeEnum.Alpha) ? 0f : 1f;

    if (ShouldApplyMode(blockedMode, VisibilityModeEnum.Interactable))
        cg.interactable = !blockedMode.HasFlag(VisibilityModeEnum.Interactable);

    if (ShouldApplyMode(blockedMode, VisibilityModeEnum.BlockRaycasts))
        cg.blocksRaycasts = !blockedMode.HasFlag(VisibilityModeEnum.BlockRaycasts);
}
```

需要保留布局和脚本生命周期，或者只想禁用点击时，用这个模式比较合适。

如果找不到 `CanvasGroup`，当前实现会输出 Warning，然后退回 `GameObject` 模式。

## 为什么需要 _appliedBlockMode

`ShouldApplyMode` 不只检查当前状态：

```csharp
private bool ShouldApplyMode(VisibilityModeEnum currentMode, VisibilityModeEnum mode)
{
    return currentMode.HasFlag(mode) || _appliedBlockMode.HasFlag(mode);
}
```

例如上一帧的状态是 `Alpha`，UI 已经被隐藏。下一帧来源解除，当前 mode 变成 0。

如果只检查当前 mode，代码就不会再处理 `Alpha`，UI 也就不会恢复。`_appliedBlockMode` 记录上一次应用过的 flag，让解除阻塞时对应字段再执行一次。

## 使用

全部隐藏：

```csharp
blocker.Block("Loading");
blocker.Unblock("Loading");
```

只禁用点击：

```csharp
blocker.Block(
    "Guide",
    VisibilityModeEnum.Interactable | VisibilityModeEnum.BlockRaycasts);
```

只隐藏画面：

```csharp
blocker.Block("FadeOut", VisibilityModeEnum.Alpha);
```

组件还提供了 `ClearAll()`，用于面板关闭或状态整体重置。这个方法会清掉其他系统加进来的来源，不能当成普通的 `Unblock` 使用。

## 调试

打开 `showDebugInfo` 后，Inspector 会显示当前的来源：

```text
Loading: All
Guide: Interactable, BlockRaycasts
```

`UpdateDebugInfo` 带有 `Conditional("UNITY_EDITOR")`，不会进入 Player 逻辑。UI 没有恢复时，先看还有谁没 `Unblock`。

## 注意

- `source` 不能为空，也不要用时间戳、随机数或临时 hash 拼接。
- `Unblock` 删除不存在的来源时不会报错，清理阶段可以直接调用。
- `CanvasGroup` 模式恢复的是 `alpha = 1`、`interactable = true`、`blocksRaycasts = true`，不会保存阻塞前的值。
- `OnDestroy` 只清理内部字典，不会再恢复目标状态。Blocker 和目标生命周期不一致时需要自己处理。
- 只有一个地方控制显隐时直接处理即可，没必要再套一层 Blocker。
