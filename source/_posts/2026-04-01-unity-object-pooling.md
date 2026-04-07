---
title: Unity 对象池的设计
date: 2026-4-1 18:40:14 +0800
tags: [note,unity,unity-perf]
typora-root-url: ..
typora-copy-images-to: ../img/miscs/
# updated: _______________________________________ +0800
---

Object Pooling 是一种非常常用且有效的降低GC、优化性能的手段。

然而这个技术在实际落地中存在一些痛点，本文聚焦以下场景进行讨论：

1. 纯GO情况下的Pooling
2. 同类GO+不同Prefab(Mono)
3. 不同类GO+不同Prefab(Mono)
4. 动态扩容的情况

## 纯GO情况下的Pooling

在理想情况下，Pooling可以只针对纯粹的GO。

即该GO不包含任何Component、MonoBehaviour。

这种情况下：
* 当我们需要GO时，可以直接从池中获取对象。
* 当我们不需要某个GO时，可以直接返还到池中。

这是最简单的情况，不需要做任何策略，也可以最快实现Pooling的落地。

## 同类GO+不同Prefab(Mono)

以割草游戏为例：
* 我们场景上频繁出现约1000左右同类GO的创建和释放，假设为：骷髅兵
* 我们场景上偶尔出现1~10左右的精英GO的创建和释放，假设为：骷髅大法师
* 我们场景上罕见出现1~3左右的领主GO的创建和释放，假设为：骷髅领主

### 案例A1：1000骷髅兵+10骷髅大法师+1骷髅领主

在该案例中，骷髅兵会频繁地创建、删除。

如果不对骷髅兵进行池化处理，大量的GameObject快速创建和删除，必然会导致大量的GC压力。

由于骷髅大法师和骷髅领主的数量较少，在性能相对不敏感场合下，我们可以暂时忽略对大法师和领主进行处理。

此时我们只需要针对骷髅兵做处理。（暂不考虑动态扩容）

伪代码：
```
Pool:
    Queue<GameObject> queue; // 骷髅兵池
    Pop() => queue.Pop(); obj.Activate();
    Release(obj) => queue.Add(obj); obj.Deactivate();
```

通过以上思路，可以简单实现在该案例情况下的池化。

### 案例A2：500骷髅兵+100骷髅骑兵+200骷髅射手

在该案例中，3种怪物类型均为小怪，且均会频繁创建、删除。

由于每一个怪物类型都有不同的数值、行动逻辑组件、动画组件，它们通常不能直接Pop/Release。

和案例A1的差异主要在于 `取用/回收` 时的处理。

我们必须实现以下目标：
* 明确提取目标怪物类型 - 是骷髅兵？骷髅骑士？骷髅射手？
* 针对被取用/回收的目标怪物类型，使用与之适配的Activate/Deactivate逻辑。
* 回收时，能够正确回收到对应的池子中。

伪代码：
```
Pool:
    Queue<GameObject>[] queues = new();
    // id: 1 骷髅兵 2 骷髅骑士 3 骷髅射手

    Pop(id) => queues[id].Pop(); obj.Activate();
    Release(obj) => queues[obj.PoolId].Add(obj); obj.Deactivate();

GameObject (Event Listener):
    OnActivate():
        ActivateMonster(this.PoolId); // 在这里做针对特定类型怪物的激活逻辑
    OnDeactivate():
        DeactivateMonster(this.PoolId); // 在这里做针对特定类型怪物的失活逻辑
```

在该案例情况下，我们需要能够通过GameObject获取到其应当所属的池子ID，同时当从池子中取出时，也需要能运行正确的激活/失活逻辑。

**潜在问题：**
* 针对未知ID，我们需要使用懒加载方式创建新的Queue，确保策划可以放心添加不同的怪物而不影响池子的正确运行。
* 当怪物类型过多——假设游戏存在1000种不同怪物类型，但玩家通常只会同时遭遇10~20中不同怪物类型。  
此时我们需要建立对池子本身的回收策略，并在合适的时候释放缓存。

## 不同类GO+不同Prefab(Mono)

### 案例B1：A2情况+10种不同子弹+10种不同爆炸粒子，每秒发射共200发子弹

* 怪物Prefab：GO + 适配怪物的各类MonoBehaviour、Component。
* 子弹Prefab：GO + 适配子弹的各类MonoBehaviour、Component。
* 子弹爆炸粒子：GO + 各类粒子发射器配置和组件。

在案例A2中，我们管理了同一大类（怪物）下的多个变体。  
现在扩展为多个完全不相关的对象类型（怪物、子弹、粒子），每种类型下又有各自的变体。

这时对象池的管理需要类型路由能力，即根据对象的种类和具体变体，找到对应的池子。

伪代码：
```
Pool:
    Dictionary<string, Queue<GameObject>> queues = new();
    // key: Mst_Skull_1 骷髅怪1 Mst_Skull_2 骷髅怪2 Mst_Skull_3 骷髅怪3
    // Blt_Normal_1 普通子弹1 Blt_Normal_2 普通子弹2 Blt_Normal_3 普通子弹3
    // Ptc_1 粒子1 Ptc_2 粒子2 Ptc_3 粒子3
    // ..

    Pop(key) => queues[key].Pop(); obj.Activate();
    Release(obj) => queues[obj.PoolKey].Add(obj); obj.Deactivate();

GameObject (Event Listener):
    OnActivate():
        ActivateMonster(this.PoolId); // 在这里做针对特定类型怪物的激活逻辑
    OnDeactivate():
        DeactivateMonster(this.PoolId); // 在这里做针对特定类型怪物的失活逻辑

IPoolable:
    void OnSpawn();
    void OnDespawn();

Entity_Mst_Skull_1 : MonoBehaviour, IPoolable:
    void OnSpawn() { .. }
    void OnDespawn() { .. }
Entity_Mst_Skull_2 : MonoBehaviour, IPoolable: ..
Entity_Mst_Skull_3 : MonoBehaviour, IPoolable: ..
Entity_Blt_Normal_1 : MonoBehaviour, IPoolable: ..
```

Key的建立很容易，但针对每一种支持池化的Entity类型对应编写OnSpawn/OnDespawn规则很容易演变为大量的重复代码。

可以通过配置驱动来缓解这个问题。

**针对Mst_Skull：**

* 给定ScriptableObject
* 在SO中设定Poolable相关处理细节
    * 是否需要处理Animator？
    * 是否需要处理Health、Movement？
    * 是否需要处理SpriteFrameRenderer？（2D序列帧播放）
    * 所属类型 - 2D敌人，需要重设SpriteRenderer, etc..
    * 机制类型 - 是否需要变更移动模式？直线移动、环形移动、随机移动，..

## 动态扩容的情况

可以预定各池子初始为10个，当创建的怪物数量达到一定阈值，自动按0.5进行增量扩容。  
当池子一定时间未处于饱和状态（比如 >= 70%），自动按0.5进行收缩释放。  
当池子超出一定时间没有任何对象，自动清空该池子并移除引用。

池子的动态扩容和收缩策略需要根据项目实际情况和目标平台等条件处理，不存在万能的解决方案。

## 总结与最佳实践

* 分层设计：根据对象的使用频率和类型数量，选择合适的池化粒度。对于高频对象（如子弹），必须池化；对于低频对象，可酌情池化或直接销毁。
* 组件复用优先于组件增删：避免在运行时 AddComponent / Destroy，而是预先挂载所有可能用到的组件，通过 enabled 控制。
* 明确对象生命周期：每个池化对象都应实现标准的 OnSpawn / OnDespawn 方法，重置所有状态（位置、旋转、速度、动画、粒子等）。
* 监控与调优：在开发阶段打印池的使用情况（命中率、扩容次数），根据实际数据调整初始容量和最大容量。
* 内存与性能平衡：池化并非越大越好，过大的池会占用更多内存，要根据目标平台（移动端 vs PC）合理设置。
