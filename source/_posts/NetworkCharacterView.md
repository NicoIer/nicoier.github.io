---
title: NetworkCharacterView
date: 2025-11-03 19:45:27
tags: [Network]
---


- 数据结构的定义
  - NetworkCharacterData：包含位置、旋转、速度、时间戳、动画数据等
  - NetworkAnimationData：包含动画ID、AvatarMask ID、播放时长、权重、播放速度
```csharp
using System;
using MemoryPack;
using UnityEngine;

namespace Character
{
    [MemoryPackable]
    public partial struct NetworkCharacterData
    {
        public Vector3 velocity;
        public Vector3 position;
        public Quaternion rotation;
        
        public long timestamp;
        public int tick;

        /// <summary>
        /// 动画信息
        /// </summary>
        public ArraySegment<NetworkAnimationData> animations;

        public override string ToString()
        {
            return $"Pos:{position} Rot:{rotation} Vel:{velocity} AnimCount:{animations.Count}";
        }
    }

    [MemoryPackable]
    public partial struct NetworkAnimationData
    {
        public int animationId; // 对应的动画 根据id去找到animation文件
        public int avatarMaskId;// 这个动画对应的avatarMask 用于做融合和分层
        public float duration; // 这个动画到目前为止播了多久 用于恢复动画状态
        public float weight;
        public float speed;
    }
}
```


- 主要看Update()
  - 收到网络消息后，生成一个TransformSnapshot，存入transformSnapshotBuffer
  - Update()中根据时间戳进行插值和外推，更新位置和旋转
    - 如果是第一次处理数据，直接设置位置和旋转，并更新动画状态
    - 如果有两个及以上快照，进行插值计算位置和旋转
      - 根据当前的移动状态，计算是否需要立刻切换动画，还是继续播放当前动画（避免网络抖动导致频繁切换动画以及单一动画的卡顿）
    - 如果只有一个快照，根据速度进行外推计算位置和旋转
  - 同时根据插值结果更新动画状态，保证动画的平滑过渡和同步
- 还实现了DeSerializeAnimations()，用于本地玩家将当前动画状态序列化成NetworkAnimationData数组，发送给服务器

```csharp
using System;
using System.Collections.Generic;
using System.Runtime.CompilerServices;
using Animancer;
using MemoryPack;
using UnityEngine;
using UnityEngine.Pool;
using Wepie.Core.CircularBuffer;

namespace Character
{
    [MemoryPackable]
    public partial struct TransformSnapshot
    {
        public long timestamp;
        public Vector3 position;
        public Quaternion rotation;
        public Vector3 velocity;
        public NetworkAnimationData[] animations; // 动画快照
    }

    /// <summary>
    /// 远程玩家的角色，只做表现，没有逻辑
    /// </summary>
    public class NetworkCharacterView : MonoBehaviour
    {
        [field: SerializeField] public CharacterAnimationLibrary characterAnimationLibrary { get; private set; }
        [field: SerializeField] public Animator animator { get; private set; }
        [field: SerializeField] public AnimancerComponent animancer { get; private set; }
        [field: SerializeField] public AnimationCallback animationCallback { get; private set; }
        [field: SerializeField] public Collider remoteCollider { get; private set; }
        [field: SerializeField] public CharacterConfig characterConfig { get; private set; }

        private HashSet<AnimationClip> reusableClipSet = new HashSet<AnimationClip>();

        private bool isLocalPlayer = false;

        private CircularBuffer<TransformSnapshot> transformSnapshotBuffer = new CircularBuffer<TransformSnapshot>(16);

        // 插值参数
        private const float InterpBackTime = 0.1f; // 插值窗口（秒）
        private const float MaxExtrapolationTime = 0.25f; // 最大外推（秒）
        private const float SnapThreshold = 2.0f; // 位置跳跃阈值

        private Vector3 smoothedPosition;
        private Quaternion smoothedRotation;
        private bool initialized = false;

        // 用于判断“运动到静止”时动画不能提前Idle
        private const float MotionThreshold = 0.05f; // 速度阈值

        public void Setup(bool isLocalPlayer)
        {
            characterAnimationLibrary.Setup();
            reusableClipSet = new HashSet<AnimationClip>();
            if (isLocalPlayer)
            {
                Destroy(remoteCollider);
            }
            this.isLocalPlayer = isLocalPlayer;
        }

        [MethodImpl(MethodImplOptions.AggressiveInlining)]
        public void OnNetworkData(in NetworkCharacterData networkData, bool tickImmediately)
        {
            if (isLocalPlayer) return;
            // 存储动画快照
            var snapshot = new TransformSnapshot()
            {
                timestamp = networkData.timestamp,
                position = networkData.position,
                rotation = networkData.rotation,
                velocity = networkData.velocity,
                animations = networkData.animations.Count > 0
                    ? networkData.animations.ToArray()
                    : Array.Empty<NetworkAnimationData>()
            };
            transformSnapshotBuffer.PushBack(snapshot);
        }

        private void Update()
        {
            if (isLocalPlayer) return;

            double now = Time.realtimeSinceStartupAsDouble;
            double renderTimestamp = (now - InterpBackTime) * 1000.0;

            if (transformSnapshotBuffer.Count == 0)
                return;

            if (!initialized)
            {
                var first = transformSnapshotBuffer[0];
                smoothedPosition = first.position;
                smoothedRotation = first.rotation;
                transform.position = smoothedPosition;
                transform.rotation = smoothedRotation;
                UpdateAnimation(first.animations);
                initialized = true;
            }

            // 保证插值区间
            while (transformSnapshotBuffer.Count >= 2 &&
                   transformSnapshotBuffer[1].timestamp <= renderTimestamp)
            {
                transformSnapshotBuffer.PopFront();
            }

            if (transformSnapshotBuffer.Count >= 2)
            {
                var from = transformSnapshotBuffer[0];
                var to = transformSnapshotBuffer[1];
                double t0 = from.timestamp;
                double t1 = to.timestamp;
                float t = t1 > t0 ? Mathf.Clamp01((float)((renderTimestamp - t0) / (t1 - t0))) : 0f;

                Vector3 interpPos = Vector3.Lerp(from.position, to.position, t);
                Quaternion interpRot = Quaternion.Slerp(from.rotation, to.rotation, t);

                if ((smoothedPosition - interpPos).sqrMagnitude > SnapThreshold * SnapThreshold)
                    smoothedPosition = interpPos;
                else
                    smoothedPosition = Vector3.Lerp(smoothedPosition, interpPos, 0.5f);

                smoothedRotation = Quaternion.Slerp(smoothedRotation, interpRot, 0.5f);

                transform.position = smoothedPosition;
                transform.rotation = smoothedRotation;

                // ----- 动画同步修正 -----
                // 判断from->to是否是运动到静止，如果是，则插值期间动画始终用from（运动）动画，只有t==1时才切to（Idle）
                bool fromMoving = from.velocity.sqrMagnitude > MotionThreshold * MotionThreshold;
                bool toIdle = to.velocity.sqrMagnitude <= MotionThreshold * MotionThreshold;
                if (fromMoving && toIdle && t < 1.0f)
                {
                    UpdateAnimation(from.animations);
                }
                else
                {
                    UpdateAnimation(to.animations);
                }
                // --------------------------------
            }
            else
            {
                // 只有一个快照，外推
                var last = transformSnapshotBuffer[0];
                float dt = Mathf.Min((float)((renderTimestamp - last.timestamp) * 0.001f), MaxExtrapolationTime);
                Vector3 extrapolatedPos = last.position + last.velocity * dt;
                Quaternion extrapolatedRot = last.rotation;

                if ((smoothedPosition - extrapolatedPos).sqrMagnitude > SnapThreshold * SnapThreshold)
                    smoothedPosition = extrapolatedPos;
                else
                    smoothedPosition = Vector3.Lerp(smoothedPosition, extrapolatedPos, 0.5f);

                smoothedRotation = Quaternion.Slerp(smoothedRotation, extrapolatedRot, 0.5f);

                transform.position = smoothedPosition;
                transform.rotation = smoothedRotation;

                UpdateAnimation(last.animations);
            }
        }

        private void UpdateAnimation(NetworkAnimationData[] animations)
        {
            reusableClipSet.Clear();
            foreach (var animationData in animations)
            {
                var clip = characterAnimationLibrary.GetClip(animationData.animationId);
                if (clip != null) reusableClipSet.Add(clip);
            }

            // 停止没有用到的动画
            foreach (var animancerState in animancer.States)
            {
                if (!animancerState.IsPlaying) continue;
                bool found = reusableClipSet.Contains(animancerState.Clip);
                if (!found)
                {
                    animancerState.Weight = 0;
                    animancerState.Stop();
                }
            }

            foreach (var animationData in animations)
            {
                var clip = characterAnimationLibrary.GetClip(animationData.animationId);
                if (clip == null) continue;
                if (animancer.States.TryGet(clip, out var state))
                {
                    if (state.IsPlaying) continue;
                    if (state.Speed == 0)
                    {
                        Debug.LogError("nicoier[角色]:反序列化异常 Animation speed is zero for clip: " + clip.name);
                        state.Speed = 1.0f;
                    }

                    state.Play();
                    continue;
                }

                state = animancer.Play(clip);
                state.LayerIndex = animationData.avatarMaskId;
                float speed = animationData.speed;
                if (speed == 0)
                {
                    Debug.LogError("nicoier[角色]:反序列化异常 Animation speed is zero for clip: " + clip.name);
                    speed = 1.0f;
                }

                state.Speed = speed;
                state.Weight = animationData.weight;
                state.Time = animationData.duration;
            }

            // Normalize Weights per layer
            Span<float> totalWeights = stackalloc float[animancer.Layers.Count];
            foreach (var animancerState in animancer.States)
            {
                totalWeights[animancerState.LayerIndex] += animancerState.Weight;
            }

            foreach (var animancerState in animancer.States)
            {
                if (!animancerState.IsPlaying) continue;
                if (animancerState.Weight == 0f) continue;

                var totalWeight = totalWeights[animancerState.LayerIndex];
                if (totalWeight > 0)
                {
                    animancerState.Weight /= totalWeights[animancerState.LayerIndex];
                }
            }
        }

        public ArraySegment<NetworkAnimationData> DeSerializeAnimations(NetworkAnimationData[] reusedAnimations)
        {
            if (!isLocalPlayer)
            {
                throw new InvalidOperationException("nicoier[角色]:Only local player can serialize animations.");
            }

            int count = 0;
            foreach (var animancerState in animancer.States)
            {
                if (animancerState.IsPlaying && animancerState.Weight > 0f && animancerState.Speed > 0f)
                {
                    ++count;
                }
            }

            Span<float> totalWeights = stackalloc float[animancer.Layers.Count];
            foreach (var animancerState in animancer.States)
            {
                totalWeights[animancerState.LayerIndex] += animancerState.Weight;
            }

            ArraySegment<NetworkAnimationData> segment =
                new ArraySegment<NetworkAnimationData>(reusedAnimations, 0, count);
            int index = 0;

            foreach (var animancerState in animancer.States)
            {
                if (!animancerState.IsPlaying) continue;
                if (animancerState.Weight == 0f) continue;

                bool thisLayerGreaterThanOne = totalWeights[animancerState.LayerIndex] > 1.0f;
                float weight = animancerState.Weight;
                if (thisLayerGreaterThanOne)
                {
                    weight = animancerState.Weight / totalWeights[animancerState.LayerIndex];
                }

                var data = new NetworkAnimationData();
                data.animationId = characterAnimationLibrary.GetAnimationId(animancerState.Clip);
                data.avatarMaskId = animancerState.LayerIndex;
                data.duration = animancerState.Duration;
                data.weight = weight;
                float speed = animancerState.Speed;
                if (speed == 0)
                {
                    Debug.LogError("nicoier[角色]: 序列化异常 Animation speed is zero for clip: " + animancerState.Clip.name);
                    speed = 1.0f;
                }

                data.speed = speed;
                reusedAnimations[index] = data;
                ++index;
            }

            return segment;
        }

        public void Dispose()
        {
        }
    }
}

```