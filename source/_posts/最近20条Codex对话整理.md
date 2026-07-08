---
title: 最近20条Codex对话整理
date: 2026-06-17 11:24:27
tags: [Codex, AI, 开发记录]
categories: [工具]
---

最近让 Codex 总结了一下本地记录里的最近20条对话。

这里的口径不是当前窗口里的20句聊天，而是 `~/.codex/session_index.jsonl` 里的最近20个会话。时间范围大概是 2026-06-11 到 2026-06-16。

看下来基本都在围绕几个东西转：JoltPhysics、BattleProcess、projectjinn、QTE、LOD、技能表现、构建资源。

很明显，最近不是在写新系统，就是在修各种边界问题。

# 怎么使用这份整理

这份整理不是流水账，主要用来看最近的工作重心和反复出现的问题。

比较有用的读法是：

- 哪些模块反复出现
- 哪些问题是语义没对齐
- 哪些问题是跨平台资源或 native binding 引起的
- 哪些问题适合沉淀成测试或工具

如果后面要继续让 Codex 处理同类任务，可以先把这些规律喂给它。比如 JoltPhysics 相关任务要优先看 native 结构体布局、资源路径、导出函数和跨平台差异；BattleProcess 相关任务要先确认状态切换和服务器数据语义。

# 最近20条

## 1. 实现萨拉扫描技能

给 `SarahScanCapability` 补了萨拉大扫描逻辑，参考的是狮吼技能。

主要是监听 `CommonDataTrack` 和 `EventTrack`，在对应帧触发扫描、屏幕变暗、HUD显示，并且给 `ScanTrigger` 增加了单次显示时长覆盖，避免修改普通扫描按钮的默认时长。

这个任务的重点不是难，而是要和已有的 Timeline/Track 体系接上。

## 2. 鞭子QTE显示时禁用操作

鞭子QTE UI出现时，需要禁止玩家操作，但是 UI 本身不能隐藏。

处理方式是通过 `CanvasGroup.interactable` 控制交互，并把阻塞逻辑集中到 `UseItemQTEUI` 自己持有的 `UIVisibilityBlocker` 列表里。

后面又顺手处理了 `BattleAttackView` 快进时粒子系统也要同步进度的问题，以及服务器 Combo 应用时要跳到下一段 Transition 起点。

这类问题的核心是：表现和服务器数据必须对齐，不然看起来就会晚一点。

## 3. 修复QTE目标死亡按失败处理

鞭子QTE过程中，如果目标怪物死了，也要按QTE失败处理。

后面又讨论了 QTE 是否应该抽成一个和 `Charge`、`Combo` 同级的 State，以及 QTE 成功/失败后如何回到 `Charge`，最后怎么走到 `TakeBack`。

这个对话更像是在整理状态机语义。先把“什么时候切状态”说清楚，再写代码会稳很多。

## 4. 查找OS X键盘移动问题

Jolt viewer 在 macOS 上鼠标有用，但是键盘不能移动。

最开始怀疑输入法，后来发现不是。问题集中在 macOS 的窗口焦点和事件分发上。

最后的方案是按 `NSWindow*` 分桶保存输入状态，每个 viewer 只取自己窗口的键鼠输入。多窗口时也会清掉其他窗口残留的按键状态，避免 WASD 卡在旧窗口。

这种问题很典型：单窗口能跑，多窗口就露馅。

## 5. 排查 Windows 字体报错

Windows 上 Jolt viewer 能开窗口，但是某台电脑找不到字体文件并 Fatal Error。

最后改成条件编译式资源输出：Windows 下资源放到 `Assets/...`，Mac/Linux 保持原来的 `Fonts/...`、`Shaders/...` 布局。

这里不能只修 Windows，因为前一次改资源路径把 Mac 搞坏了。

跨平台资源路径就是这样，改一边最好马上想一下另外两边。

## 6. 补充LOD加载与屏占比逻辑

角色组件表里新增了 LOD1、LOD2、LOD3，需要补加载逻辑和类似 Unity LOD Group 的屏占比逻辑。

后面继续补了 Inspector 绘制和 Editor 模式预览。

中间遇到 `SkinnedMeshRenderer` mesh 数据不匹配的问题，最后定位到骨骼路径解析的兜底逻辑太宽，会从过大的 root 下面找同名节点。

修法是只从当前 `CharacterViewLOD` 实例根节点解析 prefab path，别去全局乱找第一个同名节点。

## 7. 检查 DrawBodyGL 旋转问题

Unity 里的 `DrawBodyGL` 和物理引擎自带调试器看到的旋转不一致。

问题一路查到服务端生成 Box 的逻辑，最后把 `BoxCollider.position` 的含义改成几何中心。

这件事本质是统一语义：服务端、编辑器预览、运行时 drawer 都要认同同一个 center/offset 定义。

否则每一层看起来都“有道理”，合起来就是错的。

## 8. 分离牵手交互按钮

牵手交互原来和机关交互共用按钮逻辑，现在要拆成独立按钮。

同时要求移除 `Init()`，改走 `Bind/UnBind`，引用通过 prefab 挂载，不到处做空判。

最后 `InteractTrigger` 里普通交互和牵手交互分别保存状态，可以并行触发，不再互相抢 `curInteractable`。

UI 逻辑拆开之后，代码反而更直。

## 9. 新增输入应用区间

鞭子轻攻击新增了“输入应用区间”。

以前玩家点击后立刻跳阶段，现在改成连击检测区间只读取输入，等到输入应用区间进入时才真正应用。

这能解决手感和战斗数据不同步的问题。点击是输入，应用是战斗时间线上的事件，两者不应该混在一起。

## 10. 移除 NetworkReqRsp 的 Unity 依赖

把 `NetworkReqRsp` 从 Unity 依赖里拆出来。

重点是不能再依赖 `UnityEngine`、`MonoBehaviour`、`UniTask`、`Cysharp` 这些东西，要变成更干净的请求响应逻辑。

这类拆分主要是为了让底层网络代码可以在非 Unity 环境复用。

## 11. Unity下排除Viewer编译

`Viewer.cs` 在 Unity 环境下不应该参与编译。

处理方式是在文件外层加条件编译，Unity 下排除 Viewer 相关类型和实现，非 Unity 环境保持原样。

这个属于很小的改动，但是能避免 Unity 项目被 native viewer 代码拖下水。

## 12. 检查 Mac 键盘相机控制

另一次 Mac viewer 输入问题，方向是把相机控制放到 C# 层做。

最后做了统一的 `ViewerKeyboardInput`：macOS 用 CoreGraphics 判断窗口焦点和键盘状态，Windows 用 `GetForegroundWindow`、`GetAsyncKeyState`。

然后在 `PhysicsViewService` 里统一应用相机移动。

这比继续追 native viewer 的输入细节更容易控制。

## 13. Fix Metal library not found

JoltPhysicsApp 在 Mac 上报 `Metal error returned: library not found`，后面又变成 `library format is not supported`。

最终发现不是单纯路径问题，而是 `Shaders.metallib` 格式太旧。

修法是从 `joltc` 的 macOS 构建产物同步新的 `Shaders.metallib`，同时保留 shader 源文件，方便以后重新生成。

资源文件有时候比代码更容易坑人。

## 14. 实现 CollideShape 方法

给 `PhysicsMgr` 实现 `CollideShape`，同时补了一批 JoltPhysics 相关测试。

测试重点放在 `Body.WorldTransform`、`CenterOfMassTransform`、`InverseCenterOfMassTransform`、`JPH_Mat4` 布局这些接口上。

这里其实是在给 PInvoke 层兜底。native 结构体布局不对，上层逻辑再干净也没用。

## 15. 修正 JPH_Mat4 结构生成

ClangSharp 生成 `JPH_Mat4` 时，把四个 `JPH_Vec4` 生成成了 fixed buffer。

期望结果是直接生成：

```csharp
internal partial struct JPH_Mat4
{
    public JPH_Vec4 Column0;
    public JPH_Vec4 Column1;
    public JPH_Vec4 Column2;
    public JPH_Vec4 Column3;
}
```

这类问题看着像代码生成细节，实际上会直接影响矩阵传递和物理结果。

## 16. 添加 CastShape 单元测试

给 `JoltPhysicsSharp` 补 `CastShape` 单元测试。

重点是命中比例和接触点位置，不能只是跑一下不崩。

测试里如果关键断言被注释了，那就不是测试，只是示例。

## 17. 排查 CollideShape 交叠检测不稳定

`CollideShape_DetectsOverlap` 有时候能过，有时候不能过。

排查方向包括 `JPH_CollideShapeSettings_Init` 的初始化作用、`NarrowPhaseQuery` 里类似问题，以及 `CastShape` 命中结果为0的情况。

中间还顺便确认了 Vulkan 缺失一般不影响普通物理查询，它主要影响 viewer 或 Vulkan 后端。

这个问题的教训是：native settings 结构体一定要初始化，不要赌默认内存状态。

## 18. 查询 libjoltc.dll 导出函数

用 `llvm-readobj --coff-exports` 查询 Windows 下 `libjoltc.dll` 的导出函数。

最后把完整导出表输出成 txt，方便后面查 PInvoke 是否对得上。

这一步很朴素，但是查 native binding 问题时很好用。

## 19. 修复JoltPhysics构建插件复制

Windows 构建 JoltPhysics 时，对应 Plugins 没有复制。

处理方向是让 item 自带 `Plugins/...` 相对目标路径，构建目标再用项目目录和 `OutDir` 解析绝对路径。

MSBuild 的相对路径很容易受当前目录影响，最好少绕几层。

## 20. 查找 dotnet 单元测试命令

一开始只是问 `dotnet test` 怎么跑，以及怎么指定类。

后来变成 Rider 测试报 `-1073741819`，命令行也复现了。

这个错误是 Windows 的 `0xC0000005 Access Violation`，基本不是 Rider 的锅，更像 native DLL、PInvoke、回调生命周期、dispose 顺序的问题。

最后给出的方向是单独跑相关测试、加详细日志、用 `--blame-crash` 抓 dump。

# 总结

这20条对话有几个很明显的特点。

第一，大部分问题都不是孤立文件修改，而是客户端、服务器、编辑器预览、native 插件之间的语义对齐。

比如 Box 的中心点定义、Combo 的应用时间、QTE 失败回退、粒子快进，这些都不是“改一行就好”的问题。只要两边认知不一致，表现就会错。

第二，JoltPhysics 最近占比很高。

从 `JPH_Mat4`、`CastShape`、`CollideShape`，到 Metal shader、Windows 字体、Mac 键盘输入、DLL 导出表，基本把 C# 到 native 的几层都摸了一遍。

第三，“不需要你跑构建”出现得很多。

这也说明现在更依赖局部检查，比如 `git diff --check`、文件格式检查、导出表检查、资源 hash 检查。构建不是不能跑，而是不要每次都把反馈周期拉长。

第四，最近在逐渐把一些隐式逻辑显式化。

牵手按钮独立、QTE 状态独立、输入检测和输入应用拆开、`NetworkReqRsp` 移除 Unity 依赖，这些改动都在减少混在一起的状态。

说到底，Codex 在这些任务里更像一个执行很快的代码助手。真正重要的还是把语义说清楚：谁拥有状态，什么时候切换，数据以哪一边为准。

这几个问题想清楚了，代码通常就不会太离谱。
