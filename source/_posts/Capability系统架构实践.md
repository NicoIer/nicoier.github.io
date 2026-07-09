---
title: Capability系统架构实践
date: 2026-07-09 10:00:00
description: 参考 GDC2025《双影奇境》的 Capabilities 思路，记录一个 Unity Capability 框架的基础结构、生命周期、通信方式、Tag 阻塞和实现代价。
tags: [Unity, CSharp, Capability, 架构]
categories: [编程]
---

# Capability 系统架构实践

这里记录一下 `Capability` 模式的框架实现。

`Capability` 是挂在对象上的细粒度行为单元。它自己判断什么时候激活，什么时候退出，激活期间每帧做什么。

个人认为，Capability 可以理解为 ECS 的一种变体：把 System 的生命周期和行为单元拆分到对象上，形成一个个独立的小行为。

`Component` 不负责决策，只保存数据和工具函数。

`CapabilitySystem` 不关心业务，只负责按顺序推进所有 `Capability` 的生命周期。

这套思路参考了 GDC 2025《双影奇境》里对 Capabilities 的介绍。这里不带具体项目玩法，只从框架和架构层面整理。

## 核心结论

- `Capability` 是行为单元，不是数据容器。
- `Component` 是共享状态，不做行为决策。
- `Config` 是参数，不参与运行时决策。
- `Holder` 持有能力、组件、配置和 Tag 阻塞表。
- `CapabilitySystem` 是 TickManager，统一处理激活、退出和 Tick。
- 能力之间不要互相调用，通信尽量通过 `Component`。
- 特殊情况不要塞进通用能力，用 Tag + Instigator 阻塞。

说白了就是：

```text
GameObject / Owner
  -> Holder
      -> Capabilities  行为
      -> Components    状态
      -> Configs       参数
      -> TagBlockers   阻塞来源

CapabilitySystem
  -> 按 TickGroup 和 Order 推进所有 Capability
```

![Capability 整体结构](../img/capability-architecture-overview.svg)

## 为什么需要 Capability

传统 `GameObject + Component` 写法很容易出现一个问题：谁都想改状态，谁都想做决策。

一开始只是一个组件里多写一点逻辑。

后来组件之间互相调用，组件和对象互相调用，最后行为散在各处。出 Bug 时很难判断“这个状态到底是谁改的”。

`Capability` 的目标是把行为从组件里拿出来。

组件只提供状态和工具函数。

行为决策集中到一个个小能力里。

比如一个能力应该能自己回答这些问题：

- 我现在能不能激活。
- 我激活时要做什么。
- 我激活期间每帧要做什么。
- 我什么时候退出。
- 我退出时要清理什么。

这样行为的生命周期就比较清楚。

## 它不是状态机

`Capability` 很像状态，但它不是传统状态机。

传统状态机通常强调状态切换。一个主状态退出，另一个主状态进入。

`Capability` 可以并行激活。

也就是说，一个对象身上可以同时运行多个能力。

这些能力之间没有显式的连线，也不需要知道彼此存在。它们只根据输入数据、组件状态、配置和 Tag 阻塞来判断自己是否应该运行。

所以它更像是很多个小生命周期并行运行，而不是一个大状态机。

## 和 ECS 的关系

从职责上看，`Capability` 有点像 ECS 里的 `System`。

两者都负责行为。

但它们的组织方式不同。

ECS 的 `System` 通常面向一批 Entity 执行。

`Capability` 则挂在某个对象的 Holder 上，描述这个对象自己的一个行为。

所以可以把它理解成：

```text
ECS:
  System -> 扫一批 Entity + Component

Capability:
  Owner -> Holder -> 一组小型 System
```

它保留了 `GameObject + Component` 的直观性，又把行为拆到了更细的单元里。

## 和 GAS 的区别

它和 GAS 这类 Ability 系统也有相似点。

都是细粒度行为，都可以用 Tag 控制能不能执行。

区别主要在触发方向。

GAS 更偏事件驱动。外部输入或事件尝试激活 Ability，再由标签和条件决定是否允许。

`Capability` 更偏轮询。每帧由能力自己执行 `ShouldActivate` / `ShouldDeactivate`。

轮询的好处是排查直接。

一个能力没激活，就去 `ShouldActivate` 里逐行看条件。

代价是每帧都要检查，能力数量多之后，需要关注性能和调试工具。

## 框架结构

框架核心代码在：

```text
Assets/Framework/UnityToolkit/Capabilities
```

主要类型：

| 类型 | 作用 |
| --- | --- |
| `CapabilitySystem` | 管理所有能力，按 TickGroup 和 Order 更新 |
| `CapabilityBase<TTag, TOwner>` | 能力基类，封装注册、Owner、Holder、生命周期 |
| `ICapability` | 能力运行时接口 |
| `ICapabilityHolder<TTag, TOwner>` | 能力拥有者接口 |
| `CapabilityHolderBase<TTag, TOwner>` | 非 Unity 场景下的通用 Holder |
| `MonoBehaviorCapabilityHolder<TTag>` | Unity `GameObject` 侧 Holder |
| `IComponent` | 运行时状态接口 |
| `IConfig` | 配置接口 |
| `Instigator` | 阻塞来源 |
| `ETickGroup` | Tick 分组 |

整体关系如下：

```text
CapabilitySystem
  Register(ICapability)
  Unregister(ICapability)
  Update(deltaTime)
  FixedUpdate(fixedDeltaTime)

CapabilityBase
  Setup(holder, system)
  ShouldActivate()
  OnActivated()
  TickActive(deltaTime)
  ShouldDeactivate()
  OnDeactivated()
  OnOwnerDestroyed(system)

Holder
  GetOwner()
  TryGetComp<T>()
  TryGetConfig<T>()
  BlockCapabilities(tag, instigator)
  UnblockCapabilities(tag, instigator)
  HasBlockedTag(tag)
```

## 生命周期

一个能力的生命周期基本是：

```text
Setup
  -> 每帧检查 ShouldActivate
  -> OnActivated
  -> 每帧 TickActive
  -> 每帧检查 ShouldDeactivate
  -> OnDeactivated
```

框架里的 `CapabilitySystem.Update` 大概是这个逻辑：

```text
foreach tickGroup:
  foreach capability:
    currentActive = capability.active

    if currentActive && ShouldDeactivate:
      active = false
      OnDeactivated

    if !currentActive && ShouldActivate:
      active = true
      OnActivated

    if active:
      activeDuration += deltaTime
      deActiveDuration = 0
      TickActive(deltaTime)
    else:
      activeDuration = 0
      deActiveDuration += deltaTime
```

这里有一个细节：`currentActive` 是本轮开始时的状态。

所以一个能力如果这一帧从激活变成非激活，不会在同一轮里立刻再次激活。

但一个能力如果这一帧从非激活变成激活，会在同一帧执行 `TickActive`。

![Capability 生命周期](../img/capability-lifecycle.svg)

## Setup

`CapabilityBase.Setup` 做三件事：

```csharp
system.Register(this);
Owner = holder.GetOwner();
capabilityComp = holder;
```

也就是说，能力在 Setup 时注册到全局 `CapabilitySystem`，并拿到自己的 Owner 和 Holder。

能力内部如果需要组件或配置，一般在 `Setup` 中从 Holder 拿：

```csharp
capabilityComp.TryGetComp(out SomeComponent component);
capabilityComp.TryGetConfig(out SomeConfig config);
```

注意，`tickGroup` 在 Setup 后不能再改。

`CapabilityBase.tickGroup` 的 setter 里会检查 `_baseSetupDone`。Setup 后再改会抛异常。

所以 Tick 分组应该在构造或初始化阶段确定。

## TickGroup 和 Order

能力不是简单地按注册顺序更新。

框架提供两个排序维度：

- `ETickGroup`
- `tickGroupOrder`

当前 `ETickGroup` 有：

```text
NetworkEarly
Input
Gameplay
AfterGameplay
NetworkLate
```

`CapabilitySystem.OnInit` 会取出所有 TickGroup。

`Register` 时会把能力放进对应组，然后按 `tickGroupOrder` 排序：

```csharp
list.Sort((a, b) => a.tickGroupOrder.CompareTo(b.tickGroupOrder));
```

所以顺序控制分两层：

1. 先按 TickGroup。
2. 同组里按 Order。

这个设计是为了解决更新顺序问题。

比如架构上可以约定：

- 输入读取放在 `Input`。
- 常规逻辑放在 `Gameplay`。
- 依赖前面结果的收口逻辑放在 `AfterGameplay`。
- 网络同步前后分别放在 `NetworkEarly` 和 `NetworkLate`。

具体组怎么用取决于项目，但框架提供了这个顺序插槽。

![Capability Tick 顺序](../img/capability-tick-order.svg)

## FixedUpdate

`Update` 负责普通 Tick。

`FixedUpdate` 只处理激活中的物理能力。

框架判断方式很简单：

```text
if capability.active && capability is IPhysicsTick:
  PhysicsTickActive(fixedDeltaTime)
```

也就是说，不是所有能力都会进物理 Tick。

只有实现了 `IPhysicsTick` 的能力，且当前处于 active 状态，才会执行 `PhysicsTickActive`。

## Holder

`Holder` 是能力和对象之间的连接点。

它负责持有：

```text
capabilities
components
configs
tagBlockers
```

能力不应该到处找依赖。

统一从 Holder 取：

```csharp
TryGetComp<T>()
TryGetConfig<T>()
TryGetCapability<T>()
```

其中 `TryGetCapability<T>()` 虽然存在，但架构上不要把它作为主要通信方式。

如果能力之间经常互相拿引用调用方法，最后还是会回到互相耦合。

更推荐的方式是：

```text
Capability A -> 写 Component 状态
Capability B -> 读 Component 状态
```

Component 是共享状态，Capability 是行为决策。

## Component 和 Config

框架里 `IComponent` 和 `IConfig` 都是空接口。

这说明框架不约束具体数据结构。

一般可以按这个规则区分：

- `Component`：运行时状态，会被能力读写。
- `Config`：配置参数，运行时一般只读。

比如架构层面可以有：

```text
InputComponent
RuntimeStateComponent
ActionComponent
```

也可以有：

```text
ActionConfig
CooldownConfig
ViewConfig
```

这些名字只是示例，重点是职责分开。

不要把决策逻辑塞进 Component。

Component 可以有工具函数，但不应该决定“现在是否进入某个行为”。

## Tag Blocker

特殊情况如果都写进 `ShouldActivate`，能力会很快变脏。

比如某个能力想临时禁止另一个类别的能力。

比较直接的做法是：

```csharp
capabilityComp.BlockCapabilities(SomeTag.Action, new Instigator(this));
```

结束时再解除：

```csharp
capabilityComp.UnblockCapabilities(SomeTag.Action, new Instigator(this));
```

Holder 内部用的是：

```text
Dictionary<TTag, List<Instigator>>
```

同一个 Tag 可以被多个来源阻塞。

只有所有来源都解除之后，这个 Tag 才算没有被阻塞。

这比简单计数更方便排查，因为可以知道阻塞是谁加的。

注意一点：框架不会自动阻止某个能力激活。

`CapabilitySystem` 不知道某个能力属于哪个 Tag，也不会自动查 `HasBlockedTag`。

所以能力自己要在 `ShouldActivate` 或 `ShouldDeactivate` 里检查：

```csharp
if (capabilityComp.HasBlockedTag(SomeTag.Action))
{
    return false;
}
```

换句话说，Tag Blocker 是基础设施，不是自动规则系统。

![Tag Blocker 和 Instigator](../img/capability-tag-blocker.svg)

## Instigator

`Instigator` 是阻塞来源。

当前实现很简单：

```csharp
public struct Instigator : IEquatable<Instigator>
{
    public object reference;
}
```

它用 `reference` 做相等比较。

所以 Block 和 Unblock 必须用同一个来源语义。

如果激活时用能力实例做 Instigator，退出时也应该用同一个能力实例。

如果激活时用某个系统对象做 Instigator，退出时也应该用同一个系统对象。

否则就会出现 Block 加上了，但 Unblock 解不掉的问题。

## ScriptableObject 资产层

Unity 侧还提供了三个资产基类：

```text
CapabilityAsset
ComponentAsset
ConfigAsset
```

它们的职责是声明依赖：

```csharp
public abstract ICapability[] GetDependencies();
public abstract IEnumerable<IComponent> GetDependencies();
public abstract IConfig[] GetDependencies();
```

这层不是运行时系统本身，而是装配辅助。

可以用它把能力、组件、配置做成可配置资源，再由具体 Holder 在初始化时展开。

如果项目需要更大的组合粒度，也可以在这层之上做 Sheet。

Sheet 本质上就是一组 `CapabilityAsset / ComponentAsset / ConfigAsset` 的集合。

![Capability 资产装配](../img/capability-asset-assembly.svg)

## MonoBehaviour Holder

`MonoBehaviorCapabilityHolder<TTag>` 用来把框架接进 Unity。

它继承 `MonoBehaviour`，并实现：

```csharp
ICapabilityHolder<TTag, GameObject>
```

所以 Owner 就是：

```csharp
gameObject
```

它内部同样维护：

```text
capabilityAssets
componentAssets
configAssets
tagBlockers
capabilities
components
configs
```

这层只解决 Unity 对象和框架之间的桥接。

具体什么时候展开 Asset、什么时候调用能力的 Setup，仍然由上层初始化流程决定。

## 销毁流程

Owner 销毁时需要清理能力。

`CapabilityBase.OnOwnerDestroyed` 做了两件事：

```csharp
if (active) OnDeactivated();
system.Unregister(this);
```

这个细节很重要。

如果能力还处于 active 状态，销毁前会先走一次 `OnDeactivated`。

这样能力可以释放状态、解除 Tag、停止表现或清理临时资源。

然后再从 `CapabilitySystem` 里反注册。

## 一个最小能力

伪代码大概这样：

```csharp
public class SomeCapability : CapabilityBase<SomeTag, GameObject>
{
    SomeComponent component;
    SomeConfig config;

    public override void Setup(ICapabilityHolder<SomeTag, GameObject> holder, ICapabilitySystem system)
    {
        base.Setup(holder, system);
        capabilityComp.TryGetComp(out component);
        capabilityComp.TryGetConfig(out config);
    }

    public override bool ShouldActivate()
    {
        if (capabilityComp.HasBlockedTag(SomeTag.SomeAction)) return false;
        return component.requested;
    }

    public override void OnActivated()
    {
        component.running = true;
    }

    public override void TickActive(in float deltaTime)
    {
        component.elapsed += deltaTime;
    }

    public override bool ShouldDeactivate()
    {
        return component.elapsed >= config.duration;
    }

    public override void OnDeactivated()
    {
        component.running = false;
        component.elapsed = 0;
    }
}
```

这段代码里没有业务含义，只体现框架约定。

能力自己判断激活和退出。

运行时状态放在 Component。

参数放在 Config。

打断条件走 Tag。

## 好处

第一，行为边界清楚。

每个能力只看自己的生命周期，不需要一个大类处理所有分支。

第二，特殊情况容易局部处理。

如果某个行为要临时禁止另一类行为，只需要 Block 对应 Tag。

不需要把所有特殊情况都写进被禁止能力的判断里。

第三，调试路径直接。

能力为什么不激活，通常就看 `ShouldActivate`。

能力为什么不退出，通常就看 `ShouldDeactivate`。

第四，扩展成本低。

新增一个行为通常是新增一个 Capability，再补它需要的 Component / Config。

不会天然影响其他能力。

## 代价

第一，轮询有成本。

能力数量多时，`ShouldActivate` 和 `ShouldDeactivate` 会被大量调用。

所以这些函数里不要做复杂查询。

第二，顺序很重要。

TickGroup 和 Order 如果设计不好，就会出现读旧数据、重复写状态、前后帧表现不一致的问题。

第三，Tag 需要规范。

Tag 太粗会误伤能力。

Tag 太细又会失去统一阻塞的价值。

第四，Block / Unblock 必须成对。

尤其是有多个 Instigator 同时阻塞同一个 Tag 时，来源必须稳定。

第五，需要工具。

能力少时靠断点还行。

能力多时，最好有运行时面板或时间线工具，能看到：

- 当前有哪些能力 active。
- 哪些能力 inactive。
- 哪些 Tag 被 Block。
- Block 来源是谁。
- activeDuration / deActiveDuration 是多少。

否则排查会越来越靠经验。

第六，不要过度抽象。

Capability 强调细粒度和局部内聚。

两个能力长得像，不代表一定要抽一个通用父类。

如果抽完之后到处是参数和分支，还不如让它们保持独立。

## 适合什么场景

`Capability` 适合有生命周期的行为。

比如：

- 输入驱动的行为。
- 持续一段时间的行为。
- 可以被其他行为打断的行为。
- 需要按 TickGroup 排序的行为。
- 需要和其他能力共享状态但不直接耦合的行为。

不太适合这些东西：

- 纯工具函数。
- 纯配置数据。
- 全局服务。
- 一次性无状态计算。
- 大批量数据并行处理。

如果只是一个函数，直接写函数即可。

如果只是配置，放 Config。

如果只是状态，放 Component。

不要什么东西都做成 Capability。

## 新增能力的流程

一般按这个顺序做：

1. 确定它是不是一个有生命周期的行为。
2. 定义需要的 Tag。
3. 定义需要的 Component。
4. 定义需要的 Config。
5. 继承 `CapabilityBase<TTag, TOwner>`。
6. 在构造或初始化阶段设定 `tickGroup` 和 `tickGroupOrder`。
7. 在 `Setup` 里从 Holder 拿依赖。
8. 在 `ShouldActivate` 里写进入条件。
9. 在 `OnActivated` 里写进入逻辑。
10. 在 `TickActive` 里写持续逻辑。
11. 在 `ShouldDeactivate` 里写退出条件。
12. 在 `OnDeactivated` 里清理状态和解除阻塞。
13. 通过 Holder 或 Asset 装配进对象。

注意，`CapabilitySystem` 只负责调度。

它不会帮你判断业务条件，也不会自动处理 Tag。

这些规则要在能力里显式写清楚。

## 实现时的几个原则

`ShouldActivate` 里只做判断，不做副作用。

`OnActivated` 里做一次性进入逻辑。

`TickActive` 里做每帧逻辑。

`ShouldDeactivate` 里只做退出判断。

`OnDeactivated` 里做清理。

Component 只保存共享状态，不主动驱动行为。

Config 只放参数，不塞运行时状态。

Tag Blocker 用来处理外部限制，不要用一堆全局 boolean 代替。

Instigator 要稳定，谁 Block 谁 Unblock。

## 总结

Capability 模式的核心不是某个具体玩法，而是行为组织方式。

它把一个对象身上的行为拆成很多小能力。

每个能力自己管理生命周期。

能力之间通过 Component 共享状态，通过 Tag Blocker 互相限制。

`CapabilitySystem` 只做调度，`Holder` 只做装配和查询。

这套方案的收益是开发速度快、行为边界清楚、特殊情况容易局部扩展。

代价是轮询、顺序、Tag 规范和调试工具都要认真设计。

## 参考

- [Capability 系统实践](https://zhuanlan.zhihu.com/p/1927489591159521310)
- `Assets/Framework/UnityToolkit/Capabilities/CapabilitySystem.cs`
- `Assets/Framework/UnityToolkit/Capabilities/CapabilityBase.cs`
- `Assets/Framework/UnityToolkit/Capabilities/CapabilityHolderBase.cs`
- `Assets/Framework/UnityToolkit/Capabilities/interface.cs`
- `Assets/Framework/UnityToolkit/Capabilities/Instigator.cs`
- `Assets/Framework/UnityToolkit/Capabilities/ETickGroup.cs`
- `Assets/Framework/UnityToolkit/Capabilities/Runtime/MonoBehaviorCapabilityHolder.cs`
- `Assets/Framework/UnityToolkit/Capabilities/Runtime/CapabilityAsset.cs`
- `Assets/Framework/UnityToolkit/Capabilities/Runtime/ComponentAsset.cs`
- `Assets/Framework/UnityToolkit/Capabilities/Runtime/ConfigAsset.cs`
