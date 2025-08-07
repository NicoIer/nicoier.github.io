---
title: UIFramework
date: 2024-08-30 11:23:58
tags: [Unity, UGUI]
---


这里介绍一下自己DIY的UI框架，功能比较简单，但可以满足大部分需求。

# UIFramework

- 首先在场景中创建一个UIRoot对象，用于管理UI面板
- 创建后会自动生成一个UIDatabase对象，存放在Assets目录下，可以根据自己的需要进行移动
- 可以尝试双击打开UIDatabase(未实现的功能)
- 首先实现一个自定的UI资源加载器(下面分别时基于Addressabels和Resouces的两种IUILoader实现)
    - 三种加载方式分别对应同步加载 ，回调加载 ，Task加载
    - 本质上是加载一个UI面板的Prefab

```csharp
    internal struct AddressablesUILoader : IUILoader
    {
        public GameObject Load<T>() where T : IUIPanel
        {
            string path = $"UI/{typeof(T).Name}/{typeof(T).Name}.prefab";
            return Addressables.LoadAssetAsync<GameObject>(path).WaitForCompletion();
        }

        public void LoadAsync<T>(Action<GameObject> callback) where T : IUIPanel
        {
            string path = $"UI/{typeof(T).Name}/{typeof(T).Name}.prefab";
            var handle = Addressables.LoadAssetAsync<GameObject>(path);
            handle.Completed += operation => { callback(handle.Result); };
        }

        public async Task<GameObject> LoadAsync<T>() where T : IUIPanel
        {
            string path = $"UI/{typeof(T).Name}/{typeof(T).Name}.prefab";
            return await Addressables.LoadAssetAsync<GameObject>(path);
        }

        public void Dispose(GameObject panel)
        {
            Addressables.Release(panel);
        }
    }
```

```csharp
        private struct DefaultLoader : IUILoader
        {
            public GameObject Load<T>() where T : IUIPanel
            {
                return Resources.Load<GameObject>(typeof(T).Name);
            }

            public void LoadAsync<T>(Action<GameObject> callback) where T : IUIPanel
            {
                var handle = Resources.LoadAsync<GameObject>(typeof(T).Name);
                handle.completed += operation => { callback(handle.asset as GameObject); };
            }

            public async Task<GameObject> LoadAsync<T>() where T : IUIPanel
            {
                TaskCompletionSource<GameObject> tcs = new TaskCompletionSource<GameObject>();
                var handle = Resources.LoadAsync<GameObject>(typeof(T).Name);
                tcs.SetResult(handle.asset as GameObject);
                await tcs.Task;
                return tcs.Task.Result;
            }

            public void Dispose(GameObject panel)
            {
                DestroyImmediate(panel);
            }
        }
```

- 然后在代码中初始化UI加载器

```csharp
 UIRoot.Singleton.UIDatabase.Loader = new AddressablesUILoader();
```

- 然后可以按照需要进行UI的加载和销毁

```csharp
UIRoot.Singleton.OpenPanel<TXXXPanel>(); // 打开一个面板
UIRoot.Singleton.ClosePanel<TXXXPanel>(); // 关闭一个面板
UIRoot.Singleton.CloseAllPanel(); // 关闭所有面板
UIRoot.Singleton.Dispose<TXXXPanel>(); // 销毁一个面板
```

- 如何制作一个UIPanel
    - 在UIRoot的Canvas下创建一个Canvas，修改名字，然后拖到Project下，生成一个Prefab
    - 创建一个脚本继承UIPanel，将UIPanel挂载到prefab上
    - 也可以实现IUIPanel接口，实现自己的UIPanel
    - 注意你的prefab路径，需要能够被你的IUILoader加载到

### 无限循环滚动列表

LoopScrollRect: 无限循环滚动列表 ，来自github.com/qiankanglai/LoopScrollRect
