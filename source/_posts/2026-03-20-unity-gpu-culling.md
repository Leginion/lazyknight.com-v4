---
title: Unity GPU Culling 研究笔记
date: 2026-3-20 01:53:48 +0800
tags: [note,unity,unity-gpu,unity-perf]
typora-root-url: ..
typora-copy-images-to: ../img/miscs/
# updated: _______________________________________ +0800
---



本文内容摘录自与Grok讨论GPU Driven的输出，不完全保证准确性，仅供参考。



# 基础

## GPU Culling 定义

Unity中的**GPU Culling**（特别是**GPU Occlusion Culling**）是Unity 6（6000.x 系列）开始大力推广的一项现代渲染优化技术，主要和 **GPU Resident Drawer** （GPU常驻绘制器）紧密配合使用。它把传统上由CPU完成的剔除工作大量迁移到GPU上执行，特别适合 **大规模场景**、**大量动态物体** 或 **高 draw call** 的项目。



2026年初，Unity的GPU Culling体系如下。



## GPU Culling 的类型

| 类型                      | 剔除者      | 需烘焙 | 动态物体支持 | 主要减少               | 典型场景                   |
| ------------------------- | ----------- | ------ | ------------ | ---------------------- | -------------------------- |
| 传统 Frustum Culling      | CPU         | 否     | 支持         | 视锥外物体             | 所有项目（Unity 自动）     |
| 传统 Occlusion Culling    | CPU         | 是     | 基本不支持   | 被遮挡物体 + draw call | 室内、城市、静态场景       |
| **GPU Frustum Culling**   | GPU（间接） | 否     | 支持         | 视锥外物体             | 大量实例化物体（结合 GRD） |
| **GPU Occlusion Culling** | GPU         | 否     | 支持         | 被遮挡物体（三角面）   | 大型开放世界、大量遮挡场景 |

目前 Unity 官方最强、最成熟的实现是 **GPU Occlusion Culling** + **GPU Resident Drawer** 的组合。



## GPU Occlusion Culling 的核心工作原理（2026 当前版本）

**前提**：必须启用 **GPU Resident Drawer**（简称 GRD）

- GRD 把大量物体的绘制数据（位置、LOD、材质属性等）常驻在 GPU 显存中
- 支持 Instancing / SRP Batcher / Mesh GPU Resident 等多种模式

**粗剔除阶段**（通常还是 CPU 做一部分）

- 视锥剔除（Frustum）
- 层级剔除（Layer Cull）
- 阴影剔除等

**进入 GPU 阶段的关键步骤**（简化版）：

- 把所有候选物体的 **bounding sphere**（或Bounds）上传到 GPU（或已经常驻）
- GPU 先渲染一帧 **保守的深度图**（通常选最近/最大的遮挡物先画）
- 使用多级下采样深度缓冲（Hi-Z 风格）+ bounding sphere 对物体进行 occlusion test
- 结果写入一个 **可见性缓冲区**（bitfield / byte buffer）
- 后续渲染管线只绘制被标记为可见的实例

**保守性双帧策略**（很关键）

- 当前帧 + 前一帧 的深度信息都会参考
- 只要任一帧认为可见 → 本帧就渲染
- 这导致延迟一帧的剔除（temporal），但极大地减少 **popping / flickering**



## 实际开启方式（Unity 6 / URP 或 HDRP）

**URP**：

- Project Settings → Graphics → URP 标签 → **关闭 Compatibility Mode**（必须启用 Render Graph）
- 在 **URP Asset** 中：启用 SRP Batcher + 将 **GPU Resident Drawer** 设为 **Instanced Drawing** + BatchRendererGroup Variants = Keep All + Rendering Path = Forward+
- （GRD 必须在 URP Asset 中开启，Occlusion 勾选在 Renderer）

**HDRP**：

- HDRP Asset → 勾选 GPU Occlusion Culling
- 必须支持 Compute Shader 的平台（DX11/12, Vulkan, Metal, PS5, Xbox 等基本都行）

**注意**：不依赖 Static 标记，不需要烘焙 Occlusion 数据！



## 实际能带来多大收益？（社区与官方反馈综合）

| 场景类型                  | Draw Call 减少 | Overdraw/三角面减少 | FPS 提升预期 | 备注                                                         |
| ------------------------- | -------------- | ------------------- | ------------ | ------------------------------------------------------------ |
| 小型室内关卡              | 很小           | 中等                | 0~15%        | CPU Occlusion 可能更好                                       |
| 大型开放世界（静态+动态） | 中等～较大     | 非常显著            | 20~80%+      | 最吃 GPU Occlusion 的场景                                    |
| 几千～上万实例化物体      | 极大（靠 GRD） | 中等～大            | 15~100%+     | 结合 Instancing 才是真正杀手锏<br />若遮挡少反而可能变慢（额外GPU开销） |
| 高 polycount 森林/城市    | 小～中等       | 大                  | 15~60%       | 主要减少 overdraw                                            |

**重要提醒**（很多人踩坑）：

- GPU Occlusion Culling **不减少 Draw Call**！它只减少被光栅化的三角面数量
- 想真正大幅降低 Draw Call → 必须搭配 **GPU Resident Drawer + Instanced Drawing / SRP Batcher**
- 如果你的瓶颈是 **CPU draw call** 而不是 **GPU overdraw**，开启后可能提升不明显甚至变差



## 2026 年当前建议的组合拳（性能最强路径）

1. 开启 **Render Graph**（不再用 Compatibility Mode）

2. 开启 **GPU Resident Drawer** → 选 Instanced Drawing（最强）或 SRP Batcher

3. 开启 **GPU Occlusion**

4. 相机可选开启 **Depth Prepass**（进一步减少overdraw）

5. 大量使用 **GPU Instancing / Material Property Blocks / Entities Graphics**（未来方向）

6. 必要时保留传统 CPU Occlusion 作为补充（混合使用在某些项目可行）



## 一句话总结

**Unity 6+ 的 GPU Culling = “把遮挡判断和大量实例管理交给 GPU，让 CPU 只管发很少的 Draw 命令”**，这是现代引擎（虚幻5 Nanite + UE5.5+、Unity 6.x、Godot 4.3+）共同的演进方向。



# 理解 Culling

**Culling（剔除）**是游戏引擎渲染管线里最核心的优化手段之一。

**本质：**在真正把像素画到屏幕上之前，先把那些“玩家视觉上看不到”的东西尽早扔掉，不让它们占用 CPU 和 GPU 的计算资源。

**为什么需要剔除：**现代游戏场景轻易就有成千上万的物体、几百万到几千万个三角面。如果不做任何优化，把所有物体全都老老实实走一遍渲染流程，绘制所有内容（可见的和不可见的），性能会直接爆炸——CPU 忙着发渲染指令，GPU 忙着计算玩家看不到的物体和画面。



<img src="./img/miscs/13.png" alt="image-20260318165412477" style="zoom:50%;" />

<center>场景中不可见的物体仍然会被绘制</center>



**渲染管线里最常见的几种剔除（按“扔得越早越好”的顺序）**

| 剔除类型                                       | 谁来做                 | 扔掉什么                                   | 扔得有多早               | 节省什么                      | Unity默认开启       |
| ---------------------------------------------- | ---------------------- | ------------------------------------------ | ------------------------ | ----------------------------- | ------------------- |
| **视锥体剔除**<br />Frustum Culling            | 通常CPU                | 完全在摄像机视锥体（镜头视野）之外的物体   | 非常早<br />渲染准备阶段 | CPU发指令、GPU所有工作        | 是（一直都有）      |
| **遮挡剔除**<br />Occlusion Culling            | 以前CPU<br />现在可GPU | 在镜头里，但被别的物体完全挡住看不见的物体 | 比较早<br />几何阶段前   | GPU的三角面、光栅化、着色     | 传统需烘焙，GPU不用 |
| **细节/背面剔除**<br /> LOD + Backface Culling | CPU/GPU混合            | 太远用低细节模型 + 背对摄像机的面          | 中间～晚                 | GPU三角面数量、overdraw       | 部分默认            |
| **Early-Z / Hi-Z剔除**                         | GPU硬件                | 已经被前面物体挡住的像素（甚至还没着色）   | 非常晚<br />光栅化阶段   | GPU overdraw、fragment shader | 硬件自动            |

最关键的两类（常说的 Culling 指代的）就是前两个：

1. **Frustum Culling（视锥剔除）**
   想象摄像机前面有一个金字塔形的“可见范围”（视锥）。完全在金字塔外面的东西直接不处理（比如摄像机背后的山、摄像机左侧很远完全不在拍摄范围内的房子）。
   → 这几乎是所有引擎最基础、最必须的一步，Unity 一直会自动处理。
2. **Occlusion Culling（遮挡剔除）**
   有一座细节极其丰富、规模巨大的城邦就在金字塔（视锥）的极其远方（1000公里以外），但前面有堵巨大的墙、一棵巨型树木、一座庞大的山，把城邦完全挡住了——玩家根本看不见。那为什么还要浪费时间去画它？
   → 这才是真正能大幅省性能的地方，尤其在城市、室内、森林、洞穴这种“有很多互相挡住”的场景。



传统 Unity（2023 及以前）的 Occlusion Culling 是 **CPU + 预烘焙** 的方式：

* 你得把场景标记 Static → 烘焙 → 生成一张“从这个位置能看到哪些东西”的表

* 运行时 CPU 查表决定谁不画

* 缺点：只对静态物体有效，动态物体（如移动的角色、车辆）基本帮不上忙；烘焙时间长；大数据量场景烘焙容易崩
* 此外，静态烘焙出的静态物体，无法接受动态光源、无法接受动态阴影。
  * 这是 Unity 2023 及以前，主要是 Built-in Render Pipeline + 预烘焙 Occlusion Culling 系统 的经典限制和常见误区。
  * 原理：烘焙出的静态物体（Static Occluder / Static Occludee），在运行时被 occlusion culling 完全剔除（cull）掉后，它们不会被渲染（mesh 不 draw），也就无法接受任何动态光源的照明，也无法产生或接收动态阴影。详情见下文《传统 Unity 静态剔除 （Occlusion Culling）的原理和限制》部分。



Unity 6+ 的 **GPU Culling** （特别是 GPU Occlusion Culling）把这个判断挪到了 **GPU 上实时算**：

* 不需要烘焙

* 支持动态物体互相遮挡

* GPU 先快速画一个粗糙的“谁在前谁在后”的深度图

* 然后用计算着色器（Compute Shader）批量检查其他物体的**包围球（bounding sphere / Bounds）**是不是完全被挡住

* 被挡的直接标记“别画了”



**Culling = “提前计算出不需要绘制的内容，并将其剔除”**

越早判断出“这个东西玩家绝对看不到”，就越早把它从流水线里踢出去，后面所有昂贵的计算工作（顶点着色、光栅化、像素着色、overdraw）都不需要再执行。



**而 Unity GPU Culling 的最大意义则在于：**

把传统由 CPU 做的“遮挡判断”的计算过程，转移到 GPU ，配合 GPU Resident Drawer（大量实例一次性发），以实现“让 CPU 尽可能闲置、让 GPU 尽可能只画真正看得见的东西”。



# 部分传统方案特点



## 传统 Unity 静态剔除 （Occlusion Culling）的原理和限制



**定义：**

传统 Unity 静态剔除（Occlusion Culling）主要指代 Built-in Render Pipeline + 预烘焙 Occlusion Culling 系统。



**限制：**

使用这种方式剔除的静态物体，无法接受任何动态光源的照明，也无法产生或接收动态阴影。



**原理：**

烘焙出的静态物体（Static Occluder / Static Occludee），在运行时被 occlusion culling 完全剔除（cull）掉后，它们不会被渲染（mesh 不 draw），因此也就无法接受任何动态光源的照明，也无法产生或接收动态阴影。因为它们已经从动态渲染的流程中被移除了，而动态光源、动态阴影，都需要在动态渲染流程中才能产生。



下面进行更深层阐述。



### 核心机制：Occlusion Culling 是 “渲染级剔除”

* 当一个物体被 Occlusion Culling 判断为 “完全被遮挡” 时，Unity会直接跳过它的 Renderer （MeshRenderer/SkinnedMeshRenderer等）的绘制。
* 不 draw → 意味着没有顶点处理、没有 fragment shader → 意味着没有机会应用任何光照计算（包括动态光源的直接光、实时阴影）。
* 所以：**被 cull 掉的静态物体，对于动态光源来说，等同于“不存在”。**它们既不被动态光照亮，也不投射动态阴影到其他物体上。
  * 注：主渲染 pass 不参与动态光照，但 shadow pass 可能仍贡献阴影（取决于光源类型）




### 动态光源 + 阴影的具体影响

| 场景                                             | 被 cull 的静态物体能否接受动态光？ | 能否投射动态阴影？   | 说明                                                         |
| ------------------------------------------------ | ---------------------------------- | -------------------- | ------------------------------------------------------------ |
| 动态点光/聚光灯照到被 cull 物体                  | 否                                 | 否                   | 物体没渲染 → 没机会采样光源 → 没光照，也没 shadow map 贡献   |
| 动态方向光照到被 cull 物体                       | 否                                 | 否                   | 同上，shadow map 只渲染可见的 caster                         |
| 动态光照亮其他可见物体，但影子落在被 cull 物体上 | —                                  | —                    | 被 cull 物体不渲染 → 即使有影子投射到它表面，也看不到（因为表面没画） |
| 混合光照模式（Mixed / Shadowmask）               | 部分能（baked 部分）               | 部分能（baked 部分） | baked 光照/阴影是预计算贴图，不依赖运行时渲染，所以即使 cull 了也能看到 baked 阴影 |

* **纯实时动态光（Realtime 模式）**：被 cull = 完全没光照 + 没阴影贡献。

* **Baked / Mixed 光**：baked 的间接光、AO、阴影是预烘到 Lightmap / Shadowmask 上的，即使物体被 cull，这些贴图效果理论上还能“残留”在表面（但因为物体本身不渲染，实际也看不到）。

* 很多项目用 Mixed 光 + Shadowmask 时，会发现“被 cull 的墙后面还是有 baked 阴影”，但如果切换到纯 Realtime，就全没了。



### 为什么很多人觉得“静态物体 cull 后还能有动态阴影”？（常见错觉来源）

* Shadowmap 是单独渲染的（从光源视角），Unity 默认会把 **Static Occluder** 也扔进 shadow caster 列表，即使主相机 cull 了它。

* 但如果物体在主相机视口中完全 occluded，很多人误以为“阴影不该画”，其实阴影是独立的——只要光源能“看到”它作为 caster，它就会投阴影到可见区域。

* 老版本有时有 bug（比如 2014-2018 年间多次报告“cull 后灯光/阴影消失”或“反之残留”），但核心逻辑是：**主相机 cull 只影响主渲染 pass，不自动影响 shadow pass**。



### 对比 Unity 6+ 的 GPU Occlusion Culling

* GPU 版 occlusion 是**运行时实时计算**，不依赖 Static 标记。

* 被 cull 的物体（包括动态的）同样不渲染 → 同样不接受动态光、不投动态影。

* 但因为支持动态物体做 occluder，整体更灵活，不再有“只能静态”的限制。



### 总结

**传统预烘焙 Occlusion Culling 的静态物体，一旦在运行时被 cull，就彻底不参与任何实时渲染计算，包括动态光照和动态阴影。只有预烘焙（baked）的光照/阴影部分还能“幸存”（但 cull 后也看不见）。** 这也是为什么大型开放世界项目在 Unity 老版本里特别痛苦：想用 occlusion 省性能，又想有动态车灯/爆炸/角色阴影，就得牺牲 cull，或者自己写额外系统。

