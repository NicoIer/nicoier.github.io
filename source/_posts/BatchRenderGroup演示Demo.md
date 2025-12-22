---
title: BatchRenderGroup演示Demo
date: 2025-12-18 13:21:59
tags: [ Unity, BatchRendererGroup ]
---

```text
本文实现了一个最小的BatchRendererGroup演示Demo，本例子将会绘制一个可配置位置、旋转、缩放和颜色的Mesh实例。
参考了brg-shooter项目中的数据布局和使用方式。
```

![brg_demo.gif](../img/brg_demo.gif)

# BatchRendererGroup是什么

Batch Renderer Group (BRG) 是Unity在2020年推出的一个重要组件，旨在优化渲染流程并提升渲染效率。
它主要解决了多线程Batch处理的问题，并作为Hybrid Renderer V2[^1]的底层基础发布。
实际上，Batch Renderer Group是一套暴露给业务层进行Batch处理的API。
另外，BatchRendererGroup的开发者看到了这套API对性能提升带来的巨大潜力，在Unity 2022中对BRG进行了进一步的易用性重构。
现在，BRG可以脱离Hybrid Renderer V2独立使用。

[^1]: Entity Graphics的前生

# BatchRendererGroup的使用方法

```csharp
//1、创建BatchRendererGroup，注册Mesh和Material资源，以及回调函数OnPerformCulling
m_BRG = new BatchRendererGroup(this.OnPerformCulling, IntPtr.Zero);
m_MeshID = m_BRG.RegisterMesh(mesh);
m_MaterialID = m_BRG.RegisterMaterial(material);
        
//2、为batch 准备数据. 
var metadata = new NativeArray<MetadataValue>(3, Allocator.Temp);
metadata[0] = new MetadataValue { NameID = Shader.PropertyToID("unity_ObjectToWorld"), Value = 0x80000000 | byteAddressObjectToWorld, };
metadata[1] = new MetadataValue { NameID = Shader.PropertyToID("unity_WorldToObject"), Value = 0x80000000 | byteAddressWorldToObject, };
metadata[2] = new MetadataValue { NameID = Shader.PropertyToID("_BaseColor"), Value = 0x80000000 | byteAddressColor, };

//3、最后AddBatch
m_BatchID = m_BRG.AddBatch(metadata, m_InstanceData.bufferHandle, (uint)BufferOffset, (uint)BufferWindowSize);
        
//4、OnPerformCulling回调，在回调中处理每个相机的裁剪，裁剪之后组织自己的渲染数据等
public unsafe JobHandle OnPerformCulling( BatchRendererGroup rendererGroup,
        BatchCullingContext cullingContext,BatchCullingOutput cullingOutput, IntPtr userContext)
{...}
```

# 细节说明

- 数据对齐：float3x4是按照列存储的，而Matrix4x4是按照行存储的，BRG使用的数据为前者，所以当我们表示一个Vector3
  position时，对应的float3x4如下
    ```csharp
    float3x4(
        1, 0, 0, 0,
        1, 0, 0, 0,
        1, x, y, z 
    );
    ```

- 数据布局：也就是如何在GraphicsBuffer中计算数组的GPU地址，一个float3x4对应的字节数是sizeof(float) * 3 * 4 = 48
  ，一个float4对应的字节数是16字节。
  例子中渲染一个物体需要 objectToWorld + worldToObject + color 三个属性，也就是 48 * 2 + 16 = 112 字节。


- UBO[^2] vs SSBO[^3]：BatchRendererGroup支持两种不同的缓冲区类型，分别是Uniform Buffer Object (UBO) 和 Shader Storage
  Buffer Object (SSBO)，
  通过BatchRendererGroup.BufferTarget可以查询当前使用的缓冲区类型。之所以不同是因为不同的平台对UBO和SSBO的支持程度不同。我们需要当前BatchRendererGroup使用的缓冲区类型来决定我们创建GraphicsBuffer的Target类型。
  如果目标缓冲区类型是UBO，那么意味我们要手动计算窗口大小
  ```csharp
  _alignedWindowSizeBytes = BatchRendererGroup.GetConstantBufferMaxWindowSize();
  _maxInstancePerWindow = Math.Max(1, _alignedWindowSizeBytes / kGpuItemSizeBytes);
  ```
  如果目标缓冲区类型是SSBO，那么我们可以直接创建一个足够大的缓冲区 把一个instace的数据都放进去
  ```csharp
  _alignedWindowSizeBytes = (kGpuItemSizeBytes + 15) & ~15; // 16-byte align
  _maxInstancePerWindow = 1; // single window for one instance
  ```

- 创建GraphicsBuffer时的Target类型
  ```csharp
  _windowSizeInFloat4 = _alignedWindowSizeBytes / kFloat4Size;
  
  if (useUBO)
      _gpuBuffer = new GraphicsBuffer(GraphicsBuffer.Target.Constant, _windowSizeInFloat4, kFloat4Size);
  else
      _gpuBuffer = new GraphicsBuffer(GraphicsBuffer.Target.Raw, _alignedWindowSizeBytes / 4, 4);
  ```

- 组织Batch的Metadata信息
  ```csharp
  var meta = new NativeArray<MetadataValue>(3, Allocator.Temp, NativeArrayOptions.UninitializedMemory);
  meta[0] = CreateMetadataValue(ObjectToWorldID, 0, true); // obj2world
  meta[1] = CreateMetadataValue(WorldToObjectID, _maxInstancePerWindow * 3 * kFloat4Size, true); // world2obj
  meta[2] = CreateMetadataValue(BaseColorID, _maxInstancePerWindow * 3 * 2 * kFloat4Size, true); // color
  
  private static MetadataValue CreateMetadataValue(int nameID, int gpuOffset, bool isPerInstance)
  {
      const uint kIsPerInstanceBit = 0x80000000;
      return new MetadataValue
      {
          NameID = nameID,
          Value = (uint)gpuOffset | (isPerInstance ? kIsPerInstanceBit : 0u) 
      };
  }
  ```

- 组织Batch
    ```csharp
    _batchID = _brg.AddBatch(meta, _gpuBuffer.bufferHandle, 0u, useUBO ? (uint)_alignedWindowSizeBytes : 0u);
    ```

- 更新Buffer
   ```csharp
  int iInWindow = 0;
  int windowOffset = 0; // only one window

  // Build rotation (uniform scale applied)
  float3x3 rot = math.float3x3(rotation) * uniformScale;
  float3 bpos = worldPos;

  // obj2world (SoA layout used in brg-shooter debris)
  _sysmemBuffer[windowOffset + iInWindow * 3 + 0] = new float4(rot.c0.x, rot.c0.y, rot.c0.z, rot.c1.x);
  _sysmemBuffer[windowOffset + iInWindow * 3 + 1] = new float4(rot.c1.y, rot.c1.z, rot.c2.x, rot.c2.y);
  _sysmemBuffer[windowOffset + iInWindow * 3 + 2] = new float4(rot.c2.z, bpos.x, bpos.y, bpos.z);

  // world2obj (packed the same way as brg-shooter)
  _sysmemBuffer[windowOffset + _maxInstancePerWindow * 3 * 1 + iInWindow * 3 + 0] =
                new float4(rot.c0.x, rot.c1.x, rot.c2.x, rot.c0.y);
  _sysmemBuffer[windowOffset + _maxInstancePerWindow * 3 * 1 + iInWindow * 3 + 1] =
                new float4(rot.c1.y, rot.c2.y, rot.c0.z, rot.c1.z);
  _sysmemBuffer[windowOffset + _maxInstancePerWindow * 3 * 1 + iInWindow * 3 + 2] =
                new float4(rot.c2.z, -bpos.x, -bpos.y, -bpos.z);

  // color
  _sysmemBuffer[windowOffset + _maxInstancePerWindow * 3 * 2 + iInWindow] =
                new float4(color.r, color.g, color.b, color.a); 
  ```

- 上传数据到GPU
    ```csharp
    _gpuBuffer.SetData(_sysmemBuffer, 0, 0, kGpuItemFloat4Count);
    ```
  
- 组织DrawCommand
    ```csharp
    var drawCommands = new BatchCullingOutputDrawCommands
    {
        drawCommandCount = 1,
        drawRangeCount = 1,
        drawRanges = (BatchDrawRange*)UnsafeUtility.Malloc(sizeof(BatchDrawRange), 16, Allocator.TempJob),
        visibleInstances = (int*)UnsafeUtility.Malloc(sizeof(int), 16, Allocator.TempJob),
        drawCommands = (BatchDrawCommand*)UnsafeUtility.Malloc(sizeof(BatchDrawCommand), 16, Allocator.TempJob),
        instanceSortingPositions = null,
        instanceSortingPositionFloatCount = 0
    };

    drawCommands.drawRanges[0] = new BatchDrawRange
    {
        drawCommandsBegin = 0,
        drawCommandsCount = 1,
        filterSettings = new BatchFilterSettings
        {
            renderingLayerMask = 1,
            layer = 0,
            motionMode = MotionVectorGenerationMode.Camera,
            shadowCastingMode = castShadows ? ShadowCastingMode.On : ShadowCastingMode.Off,
            receiveShadows = true,
            staticShadowCaster = false,
            allDepthSorted = false
        }
    };

    drawCommands.visibleInstances[0] = 0;

    drawCommands.drawCommands[0] = new BatchDrawCommand
    {
        visibleOffset = 0,
        visibleCount = 1,
        batchID = _batchID,
        materialID = _matID,
        meshID = _meshID,
        submeshIndex = 0,
        splitVisibilityMask = 0xff,
        flags = BatchDrawCommandFlags.None,
        sortingPosition = 0
    };

    cullingOutput.drawCommands[0] = drawCommands;
    ```


[^2]: UBO（Uniform Buffer Object）/ ConstantBuffer：
各图形 API 的“常量缓冲区”（D3D 的 Constant Buffer、Vulkan 的 Uniform Buffer、Metal 的 Constant buffer）。
小且快速，有严格的大小/对齐限制（常见单次绑定窗口上限 64KB，具体平台不同）。
只能在着色器里只读（渲染阶段），由驱动高效缓存。

[^3]: SSBO（Shader Storage Buffer Object）/ Structured/Raw Buffer：
大容量、灵活的“存储缓冲区”（D3D 的 StructuredBuffer/ByteAddressBuffer，Vulkan 的 Storage Buffer，Metal 的 device buffer）。
限制更少，容量可远大于 UBO；可在计算着色器中读写（图形管线阶段一般读）。
某些平台上访问成本可能略高于 UBO，但更通用

# 源代码

```csharp
using System;
using Unity.Burst;
using Unity.Collections;
using Unity.Collections.LowLevel.Unsafe;
using Unity.Jobs;
using Unity.Mathematics;
using UnityEngine;
using UnityEngine.Rendering;

namespace Assembly
{
    // Self-contained minimal BRG demo:
    // - Single instance
    // - SoA layout identical to brg-shooter: obj2world (3 float4) + world2obj (3 float4) + color (1 float4)
    // - Uses BatchRendererGroup directly (no external helpers)
    public unsafe class BRGDemo : MonoBehaviour
    {
        [Header("Instance Params")] public Vector3 worldPos = Vector3.zero;
        public Quaternion rotation = Quaternion.identity;
        public float uniformScale = 1.0f;
        public Color color = Color.white;

        [Header("Render Resources")] public Mesh mesh;
        public Material material;
        public bool castShadows = true;

        // BRG core
        private BatchRendererGroup _brg;
        private BatchID _batchID;
        private BatchMeshID _meshID;
        private BatchMaterialID _matID;
        private GraphicsBuffer _gpuBuffer;
        private NativeArray<float4> _sysmemBuffer;

        // Constants aligned with brg-shooter
        private const int kFloat4Size = 16;
        private const int kGpuItemFloat4Count = 7; // 3 (obj2world) + 3 (world2obj) + 1 (color)
        private const int kGpuItemSizeBytes = kGpuItemFloat4Count * kFloat4Size;

        // Cached layout info
        private int _alignedWindowSizeBytes;
        private int _maxInstancePerWindow;
        private int _windowSizeInFloat4;

        // Shader property IDs
        private static readonly int ObjectToWorldID = Shader.PropertyToID("unity_ObjectToWorld");
        private static readonly int WorldToObjectID = Shader.PropertyToID("unity_WorldToObject");
        private static readonly int BaseColorID = Shader.PropertyToID("_BaseColor");

        private void Awake()
        {
            InitBRG();
            UploadGPUData(); // initial upload
        }

        private void LateUpdate()
        {
            UploadGPUData();
        }

        private void OnDestroy()
        {
            _brg.RemoveBatch(_batchID);
            _brg.UnregisterMesh(_meshID);
            _brg.UnregisterMaterial(_matID);
            _brg.Dispose();

            _gpuBuffer?.Dispose();
            if (_sysmemBuffer.IsCreated) _sysmemBuffer.Dispose();
        }

        private void InitBRG()
        {
            // Determine buffer mode (SSBO vs UBO)
            bool useUBO = BatchRendererGroup.BufferTarget == BatchBufferTarget.ConstantBuffer;

            // Compute window sizes
            if (useUBO)
            {
                _alignedWindowSizeBytes = BatchRendererGroup.GetConstantBufferMaxWindowSize();
                _maxInstancePerWindow = Math.Max(1, _alignedWindowSizeBytes / kGpuItemSizeBytes);
            }
            else
            {
                _alignedWindowSizeBytes = (kGpuItemSizeBytes + 15) & ~15; // 16-byte align
                _maxInstancePerWindow = 1; // single window for one instance
            }

            _windowSizeInFloat4 = _alignedWindowSizeBytes / kFloat4Size;

            // Allocate system buffer (one window is enough for a single instance)
            _sysmemBuffer = new NativeArray<float4>(_windowSizeInFloat4, Allocator.Persistent,
                NativeArrayOptions.ClearMemory);

            // Allocate GPU buffer
            if (useUBO)
                _gpuBuffer = new GraphicsBuffer(GraphicsBuffer.Target.Constant, _windowSizeInFloat4, kFloat4Size);
            else
                _gpuBuffer = new GraphicsBuffer(GraphicsBuffer.Target.Raw, _alignedWindowSizeBytes / 4, 4);

            // Create BRG
            _brg = new BatchRendererGroup(OnPerformCulling, IntPtr.Zero);

            // Metadata
            var meta = new NativeArray<MetadataValue>(3, Allocator.Temp, NativeArrayOptions.UninitializedMemory);
            meta[0] = CreateMetadataValue(ObjectToWorldID, 0, true); // obj2world
            meta[1] = CreateMetadataValue(WorldToObjectID, _maxInstancePerWindow * 3 * kFloat4Size, true); // world2obj
            meta[2] = CreateMetadataValue(BaseColorID, _maxInstancePerWindow * 3 * 2 * kFloat4Size, true); // color

            _batchID = _brg.AddBatch(meta, _gpuBuffer.bufferHandle, 0u, useUBO ? (uint)_alignedWindowSizeBytes : 0u);
            meta.Dispose();

            _meshID = _brg.RegisterMesh(mesh);
            _matID = _brg.RegisterMaterial(material);

            // Huge bounds to disable frustum cull for this demo
            _brg.SetGlobalBounds(new Bounds(Vector3.zero, Vector3.one * 100000f));
        }

        [BurstCompile]
        private void UploadGPUData()
        {
            // Single instance index = 0
            int iInWindow = 0;
            int windowOffset = 0; // only one window

            // Build rotation (uniform scale applied)
            float3x3 rot = math.float3x3(rotation) * uniformScale;
            float3 bpos = worldPos;

            // obj2world (SoA layout used in brg-shooter debris)
            _sysmemBuffer[windowOffset + iInWindow * 3 + 0] = new float4(rot.c0.x, rot.c0.y, rot.c0.z, rot.c1.x);
            _sysmemBuffer[windowOffset + iInWindow * 3 + 1] = new float4(rot.c1.y, rot.c1.z, rot.c2.x, rot.c2.y);
            _sysmemBuffer[windowOffset + iInWindow * 3 + 2] = new float4(rot.c2.z, bpos.x, bpos.y, bpos.z);

            // world2obj (packed the same way as brg-shooter)
            _sysmemBuffer[windowOffset + _maxInstancePerWindow * 3 * 1 + iInWindow * 3 + 0] =
                new float4(rot.c0.x, rot.c1.x, rot.c2.x, rot.c0.y);
            _sysmemBuffer[windowOffset + _maxInstancePerWindow * 3 * 1 + iInWindow * 3 + 1] =
                new float4(rot.c1.y, rot.c2.y, rot.c0.z, rot.c1.z);
            _sysmemBuffer[windowOffset + _maxInstancePerWindow * 3 * 1 + iInWindow * 3 + 2] =
                new float4(rot.c2.z, -bpos.x, -bpos.y, -bpos.z);

            // color
            _sysmemBuffer[windowOffset + _maxInstancePerWindow * 3 * 2 + iInWindow] =
                new float4(color.r, color.g, color.b, color.a);

            // Upload exactly 7 float4s
            _gpuBuffer.SetData(_sysmemBuffer, 0, 0, kGpuItemFloat4Count);
        }

        private static MetadataValue CreateMetadataValue(int nameID, int gpuOffset, bool isPerInstance)
        {
            const uint kIsPerInstanceBit = 0x80000000;
            return new MetadataValue
            {
                NameID = nameID,
                Value = (uint)gpuOffset | (isPerInstance ? kIsPerInstanceBit : 0u)
            };
        }

        [BurstCompile]
        public JobHandle OnPerformCulling(BatchRendererGroup rendererGroup, BatchCullingContext cullingContext,
            BatchCullingOutput cullingOutput, IntPtr userContext)
        {
            // Single draw command, single visible instance
            var drawCommands = new BatchCullingOutputDrawCommands
            {
                drawCommandCount = 1,
                drawRangeCount = 1,
                drawRanges = (BatchDrawRange*)UnsafeUtility.Malloc(sizeof(BatchDrawRange), 16, Allocator.TempJob),
                visibleInstances = (int*)UnsafeUtility.Malloc(sizeof(int), 16, Allocator.TempJob),
                drawCommands = (BatchDrawCommand*)UnsafeUtility.Malloc(sizeof(BatchDrawCommand), 16, Allocator.TempJob),
                instanceSortingPositions = null,
                instanceSortingPositionFloatCount = 0
            };

            drawCommands.drawRanges[0] = new BatchDrawRange
            {
                drawCommandsBegin = 0,
                drawCommandsCount = 1,
                filterSettings = new BatchFilterSettings
                {
                    renderingLayerMask = 1,
                    layer = 0,
                    motionMode = MotionVectorGenerationMode.Camera,
                    shadowCastingMode = castShadows ? ShadowCastingMode.On : ShadowCastingMode.Off,
                    receiveShadows = true,
                    staticShadowCaster = false,
                    allDepthSorted = false
                }
            };

            drawCommands.visibleInstances[0] = 0;

            drawCommands.drawCommands[0] = new BatchDrawCommand
            {
                visibleOffset = 0,
                visibleCount = 1,
                batchID = _batchID,
                materialID = _matID,
                meshID = _meshID,
                submeshIndex = 0,
                splitVisibilityMask = 0xff,
                flags = BatchDrawCommandFlags.None,
                sortingPosition = 0
            };

            cullingOutput.drawCommands[0] = drawCommands;
            return default;
        }
    }
}
```

```text
references：
- https://unity.com/cn/blog/engine-platform/batchrenderergroup-sample-high-frame-rate-on-budget-devices
- https://gamedev.center/trying-out-new-unity-api-batchrenderergroup/
```