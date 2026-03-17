---
title: Unity GPU Driven 研究笔记
date: 2026-3-18 01:39:43 +0800
tags: [note,unity,unity-gpu,unity-perf]
typora-root-url: ..
# updated: _______________________________________ +0800
---



本文内容摘录自与Grok讨论GPU Driven的输出，不完全保证准确性，仅供参考。



## GPU Driven / 定义



**GPU Driven**

ComputeBuffer + DrawMeshInstancedIndirect + ComputeShader culling



GPU Driven 的核心机制是把 Culling、LOD、绘制 ，尽可能全部推到 GPU 侧进行运算，CPU只发一次 Draw 调用。

产生的限制：传统 Animator / Animation / SkinnedMeshRenderer 彻底无法使用。



注意：

WebGL 目前基本不支持 GPU Driven 。

WebGPU 虽有进展，但Unity支持不完整。



## GPU Driven / 基础原理深入理解



### GPU Driven 的三个核心组件



* ComputeBuffer

* DrawMeshInstancedIndirect

* ComputeShader culling



### 为什么以上三者几乎总是绑定在一起使用？



先确认核心目标。



#### 三者的核心目标

让CPU尽可能只做 **一次初始化 + 极少更新** ，把 **“哪些物体要画、画多少个、在哪里画”** 的所有决策全部交给GPU。

实现CPU侧只需要发 1~少数几次 Draw Call 就能渲染 几万~几百万 个物体，以释放CPU的运算压力。



#### 为什么必须使用 DrawMeshInstancedIndirect/RenderMeshIndirect ？



**DrawMeshInstancedIndirect 的关键特性（也是其存在的唯一理由）：**

* 实现从 GPU Buffer （args buffer）直接读取 **实例数量、起始索引、索引数量** 参数，无需CPU传递。
* CPU 只需要调用一次 DrawMeshInstancedIndirect ，此后 GPU 自行读取 buffer 决定画多少。
* 如果没有这个“间接“机制， GPU 就无法自己决定 ”画多少个“，culling 运算结果无法获取 → 导致 GPU Driven 链条断裂。



**对比其他绘制API：**

* Graphics.DrawMeshInstanced(Matrix4x4[] arrays)：需要CPU每帧把所有可见实例的矩阵数组 SetData() 到 GPU → 意味着 CPU 开销爆炸（拷贝数万~数百万矩阵）。
* 普通 Graphics.DrawMesh（一个一个画）：数万 ~ 数百万 Draw Call，CPU直接死机。
* 普通 GameObject + MeshRenderer：每个物体1个GO + 1个Draw Call，百万级直接OOM + 崩溃。



#### DrawMeshInstancedIndirect 是怎么实现这个“间接”机制的？

Unity 的 `Graphics.DrawMeshInstancedIndirect` API事实上是对GPU（硬件）驱动API的薄层封装。

真正实现“间接渲染”机制的，是GPU硬件本身功能。

详述见《关键核心概念 / DrawMeshInstancedIndirect》。



#### 为什么必须使用 ComputeBuffer ，而非普通数组 / Texture / 其它？



**如果不用ComputeBuffer，该用什么？**

| 替代品                         | 后果 / 问题                                                  |
| ------------------------------ | ------------------------------------------------------------ |
| Matrix4x4[] + SetData 每帧传递 | CPU 拷贝开销巨大，10w+ 实例情况下压力巨大                    |
| Texture2D（动画纹理 / VAT）    | 适合顶点动画，但无法高效储存“任意数据结构”（如float4x4、uint flags、LOD level 等） |
| GraphicsBuffer（新API）        | 可以使用，但 ComputeBuffer 更通用，ComputeShader 读写最自然  |



#### ComputeBuffer 的关键作用（在 GPU Driven 中）：

1. 存**原始实例数据**（位置、旋转、scale、颜色、动画状态、bounds等）——所有物体共用一份大缓冲。
2. 存**culling结果**（可见实例列表，或间接绘制参数args）。
3. 存**间接绘制参数**（args buffer：instanceCount、indexCount等5个uint）。
4. ComputeShader可以**读+写**它（RWStructuredBuffer），实现动态修改。



如果没有 ComputeBuffer （或 GraphicsBuffer ），ComputeShader 就无法把 culling 结果写回 GPU，也无法让DrawMeshInstancedIndirect读取动态的 instanceCount → 直接导致间接绘制机制失效。



#### 为什么需要 ComputeShader 来做 Culling ？



**如果不用 ComputeShader culling，用什么？**

| 替代品                                    | 后果 / 问题                                                 |
| ----------------------------------------- | ----------------------------------------------------------- |
| CPU 做 frustum culling (Jobs + Burst)     | 10w+ 物体时 CPU 仍然成为瓶颈（尤其是动态物体）              |
| 不做任何 culling                          | 画所有物体 → overdraw + 顶点/像素压力激增，帧率降低到个位数 |
| 只用Unity内置culling (renderer.isVisible) | 对 InstancedIndirect 完全无效（它不走 GameObject 管线）     |



**ComputeShader culling 的关键价值：**

* **并行度极高**：GPU 上万个线程同时判断一个实例是否在视锥/遮挡/LOD 范围内。

* **结果直接写回 GPU buffer**：可见实例追加到 AppendBuffer，或标记 visibility bit，或直接改写 argsBuffer 的 instanceCount。

* **零 CPU 拷贝**：culling 完 → 直接用修改后的 argsBuffer 绘制，无需 CPU 读回结果再 SetData。



没有 ComputeShader，culling 结果就只能：在 CPU 计算 → 把可见列表传回 GPU。这会回到“每帧 SetData 大数组”的老路，GPU Driven 的“让 CPU 几乎闲置”目标直接破产。



#### 三者协同使用的逻辑链条（缺一不可）

```
CPU（初始化/很少更新）
   ↓ 只发一次或极少次
   ↓
ComputeShader dispatch（每帧或每几帧）
   ↓ 并行判断几万～百万实例
   ↓ 结果写到 ComputeBuffer（可见列表 / args buffer / instanceCount）
   ↓
DrawMeshInstancedIndirect（1 次调用）
   ↓ GPU 自己从 args ComputeBuffer 读 instanceCount、indexCount 等
   ↓ 根据 instanceCount 从另一个 ComputeBuffer 读每个实例的真正数据（位置/旋转/颜色）
   ↓ 顶点着色器用 SV_InstanceID 索引数据 → 画出所有可见实例
```



**缺少任何一个的后果：**

| 缺少                      | 退化为                                | CPU负担 | 支持的实例数预估           | 是否还算GPU Driven |
| ------------------------- | ------------------------------------- | ------- | -------------------------- | ------------------ |
| DrawMeshInstancedIndirect | DrawMeshInstanced + CPU Set Data      | 高      | ~1k-10k                    | 否                 |
| ComputeBuffer             | 无法动态修改 instanceCount 和实例数据 | 高      | ~数千                      | 否                 |
| ComputeShader culling     | ①全部画 / ②CPU culling + SetData      | 中~高   | ~1w-10w（CPU性能影响）     | 半吊子             |
| 三者全有                  | 完整的 GPU Driven                     | 极低    | 10w-100w+（显存/带宽影响） | 是                 |



#### 为什么三者必须一起使用？（三者的核心价值）

* **DrawMeshInstancedIndirect 提供了“让 GPU 自己决定画多少”的入口**

* **ComputeBuffer 提供了 GPU 可读可写的“共享内存”** 

* **ComputeShader 提供了在 GPU 上高效“计算出要画多少”的能力**

三者缺了任何一个，**“让 GPU 完全自主决定绘制内容”** 这个闭环就断掉，CPU 就不得不重新介入，性能直接掉数量级。



## 关键核心概念 / Material.SetBuffer



### Material.SetBuffer 做了什么？



`instanceMaterial.SetBuffer("instanceMatrices", matrixBuffer);`

该API将matrixBuffer（一个ComputeBuffer）的GPU内存地址/绑定点，注册到该Material的shader属性系统中。

Unity在渲染这个Material时，会在Shader的常量缓冲区（或资源绑定槽）中，把这个buffer绑定为一个可访问的StructuredBuffer（或RWStructuredBuffer，如果是ComputeShader）。



其中 `"instanceMatrices"` 名字是开发者在 shader 中自行声明的，比如：

```hlsl
StructuredBuffer<float4x4> instanceMatrices;
```

这个名字必须完全匹配 `SetBuffer` 的字符串。



### Graphics.DrawMeshInstancedIndirect 的调用时机？

当你调用 `Graphics.DrawMeshInstancedIndirect(mesh, submesh, instanceMaterial, bounds, argsBuffer, ...)` 时：

- Unity 会使用 `instanceMaterial`（或你传入的 MaterialPropertyBlock，如果有）来设置当前渲染状态。
- 这包括：
  - 绑定 shader
  - 设置 pass
  - 绑定所有已通过 Material.SetBuffer / MaterialPropertyBlock.SetBuffer 设置的 ComputeBuffer / Texture 等资源

* 产生结果：GPU 在执行这个 draw call 时，会自动获得 instanceMaterial 所关联的所有 buffer 绑定（包括你 SetBuffer 的 matrixBuffer）。



### GPU 如何读取到 matrixBuffer 数据？

在 vertex shader（或 procedural setup 函数）中，通过 `SV_InstanceID`（或 `unity_InstanceID`）拿到当前实例的 ID（0 ~ instanceCount-1）。



然后直接索引：

```hlsl
uint id = unity_InstanceID;               // 或 SV_InstanceID
float4x4 objectToWorld = instanceMatrices[id];
```



随后，GPU直接从显存中读取对应位置的矩阵数据，进行变换。



### 关键概念澄清



**传递给 DrawMeshInstancedIndirect 的，不是 Material 的内存地址 。**

Material 只是一个“配置容器”。

DrawMeshInstancedIndirect 真正传入的是 Material 的渲染状态快照（包括所有绑定的 buffer、纹理、常量等）。

GPU 拿到的是这些绑定的资源视图 / 槽位，而不是 Material 对象本身。



**绑定是 per-Material 的（或 per-PropertyBlock 的）。**

如果你对同一个 Material 调用多次 `SetBuffer` ，最后一次会覆盖前面的。

如果你想在同一个 Material 上为不同 draw call 用不同 buffer，通常需要用 **MaterialPropertyBlock** ：

```c#
var block = new MaterialPropertyBlock();
block.SetBuffer("instanceMatrices", differentMatrixBuffer);
Graphics.DrawMeshInstancedIndirect(..., instanceMaterial, bounds, argsBuffer, 0, block);
```

（这样每次 draw call 都能用独立的 buffer 绑定，而不污染全局 Material。）



**必须在 Draw 调用前完成 SetBuffer 。**

SetBuffer 是立即生效的，但如果在 draw 之后再 SetBuffer，这一帧的绘制不会看到更新（下一帧才生效）。



**如果 shader 里没声明对应的 StructuredBuffer 。**

即使调用了 SetBuffer ，GPU 也读不到（绑定失败，数据为垃圾或 0）。



### 小结

`instanceMaterial.SetBuffer("instanceMatrices", matrixBuffer);`

**该语句的作用是：**

* 把 ComputeBuffer 的 GPU 绑定信息注册到 Material 的属性系统中。
* 调用 `Graphics.DrawMeshInstancedIndirect(..., instanceMaterial, ...)` 时，Unity 把 Material 的所有绑定（包括这个 buffer）设置到当前渲染管线。
* GPU 在 shader 执行时，通过实例 ID 直接从该 buffer 的显存地址读取对应矩阵数据，实现 per-instance 变换。



## 关键核心概念 / DrawMeshInstancedIndirect

针对 DrawMeshInstancedIndirect（以下简称 DMII） 的概念说明。



### Unity 6 的现状

注意：在 Unity 6+ 中， `Graphics.DrawMeshInstancedIndirect` 已标记为 obsolete，推荐迁移到 `Graphics.RenderMeshIndirect` （底层机制相同，兼容性更好）。



### 为什么必须调用这个API？

DMII 是 Unity 向 GPU 发出 “实例化间接绘制命令” 的**唯一入口**。

只有调用它时，Unity（Runtime）才会把绘制请求（包含mesh、material、argsBuffer等），打包成一个合法的 draw call 提交给底层图形 API 。

不执行这个API，GPU就不会收到任何与这些实例相关的渲染指令。



### 如果不调用？

不调用 DMII ，就**完全没有 draw call 被提交**。

GPU不会执行任何顶点着色器、像素着色器，也不会读取任何实例数据或argsBuffer。

因此这些实例化的mesh在这一帧里**完全不会被绘制**（屏幕上什么都不会出现）。



### 为什么直接使用Unity自带的MeshRenderer就不需要调用这些接口？

Unity 的 MeshRenderer 事实上是一个高层封装组件，它作为引擎自动渲染管线的暴露接口，隐藏了所有底层绘制细节（比如 draw call 的生成与提交）。

开发者无需手动调用 Graphics.DrawMesh 或类似 API，引擎会在每帧渲染循环中自动完成视锥剔除、批处理、实例化收集并发出绘制命令。

换言之，虽然你没有直接调用 Graphics.DrawMesh 或其他图形绘制API，但事实上Unity的渲染管线（引擎代码）已经自动为你完成了这一步。你在Inspector里设置的 mesh、material、shadow casting 等参数，最终都会被引擎翻译成渲染命令，自动提交到GPU。

MeshRenderer的存在，让开发者 “感知不到” draw call 的存在，但引擎在每帧的 Camera.Render 阶段（Culling → Batching → Submit Draw Commands）已经替你完成了所有绘制提交，包括可能的 instanced draw 或 batched draw。

而当我们使用 DrawMeshInstancedIndirect 时，实际上是主动放弃了 Unity 的这套自动管线，转而构建一套自定义的、支持间接绘制的渲染流程：实例数据驻留 GPU、culling 由 ComputeShader 完成、绘制参数从 args buffer 动态读取、单次 draw call 由 GPU 自主执行。

实现间接绘制，意味着你抛弃了Unity自带的这一套自动渲染管线，转为由你自己实现这一套支持间接绘制的渲染管线，你事实上**绕过（或部分取代）**了Unity的自动渲染路径。

* 不再依赖引擎的 CPU culling、Dynamic/Static Batching、SRP Batcher 的自动实例化收集。
* 不再让引擎自动决定“画哪些物体、画多少个”。
* 改为自己管理实例数据（ComputeBuffer）、自己决定绘制参数（argsBuffer）、自己实现 culling（ComputeShader）、自己提交间接绘制命令。 

这就是“自定义渲染管线”的典型表现，尤其在 GPU Driven 场景下。



### 为什么 Graphics.DrawMesh 不支持“间接”机制？

Graphics.DrawMesh（以及它的变体 DrawMeshInstanced ）是 **直接绘制** API。



**它在设计时就要求：**

所有参数（包括 instanceCount、起始索引 等）必须由 CPU 在调用时直接提供，并立即传给GPU。

Unity在实现时，没有给它预留“从 GPU Buffer 读取参数”的路径，所以它天然不支持间接机制。



**Tips：**

“直接绘制”和“间接绘制”是底层图形驱动提供的功能。



### DMII 是怎么实现这个 “间接” 机制的？

DMII 接收一个 args ComputeBuffer（5 个 uint：indexCount、instanceCount、startIndex、baseVertex、startInstance）。



当通过脚本调用DMII时，Unity内部：

* 把 argsBuffer 绑定为“间接参数缓冲区”
* 不从 CPU 读取 instanceCount，而是告诉 GPU：“去这个 buffer 的偏移 4 字节处自己读 instanceCount”
* 让 GPU 在真正执行绘制时动态决定要画多少个实例



这就是“间接”的核心：绘制参数来自 GPU 显存，而不是 CPU 实时传入。



### 在更底层的交互中，DMII是通过什么办法达成的绘制功能？



Unity Graphics 层把 DMII 直接映射成平台原生的间接绘制指令（硬件支持的 Indirect Draw）：

* **DirectX 11/12**：调用 ID3D11DeviceContext::DrawIndexedInstancedIndirect（或 DX12 等效），参数 buffer 作为 Indirect Argument Buffer。

* **OpenGL / OpenGL ES**：调用 glDrawElementsIndirect（或 glMultiDrawElementsIndirect）。

* **Vulkan**：调用 vkCmdDrawIndexedIndirect。

* **Metal**：使用 Indirect Command Buffer 执行 drawIndexedInstancedIndirect。



Unity 只负责绑定 buffer、设置 shader pass 和 subMesh 索引，实际执行绘制的是 GPU 硬件自己从 argsBuffer 读取参数并完成实例化渲染。

这就是 DMII 能让 GPU 自主决定“画多少”的底层实现逻辑。



**Tips：**

换言之，Indirect Draw是GPU厂商（硬件提供商）提供的底层图形功能。如果显卡不支持或驱动不支持，则无法使用Indirect Draw。

Unity的 `Graphics.DrawMeshInstancedIndirect` 只是对这些底层硬件功能的薄层封装。



### 谈谈 Indirect Draw 和 GPU 底层

Indirect Draw 是 GPU 厂商 / 硬件提供的，它是现代图形管线中为了实现 GPU-driven rendering 而引入的硬件级特性。

GPU 硬件本身必须支持从显存中的缓冲区（indirect argument buffer）读取绘制参数（如 instanceCount、indexCount 等），然后自主执行绘制命令，而不需要 CPU 每次都重新提供这些参数。



具体到各大图形 API 的底层实现：

- **DirectX 11/12**：使用 DrawIndexedInstancedIndirect / ExecuteIndirect 等函数。DX12 中有明确的 **ExecuteIndirect tiers**（Tier 1.0、1.1 等），不同硬件支持不同层级，但核心 Indirect Draw 从 DX11 开始就广泛支持（Feature Level 11_0 及以上基本都有）。
- **OpenGL**：glDrawElementsIndirect / glDrawArraysIndirect，从 OpenGL 4.2（或 ARB_draw_indirect 扩展）开始支持。现代硬件（2012 年后）基本全支持。
- **Vulkan**：vkCmdDrawIndexedIndirect / vkCmdDrawIndirect，这是 Vulkan 1.0 核心功能，几乎所有支持 Vulkan 的设备都支持（包括移动端）。Vulkan 1.2 进一步增强了 vkCmdDrawIndexedIndirectCount（GPU 控制 draw count）。
- **Metal**（Apple 平台）：drawIndexedPrimitives 的 indirect 变体，从 Metal 1.0 / iOS 9.0 开始就支持，Apple 硬件（A 系列 / M 系列）原生支持。



**如果显卡不支持或驱动不支持，就不能使用 Indirect Draw 。**

如果硬件 / 驱动不支持 Indirect Draw，Unity 的 `Graphics.DrawMeshInstancedIndirect` 会失败（通常返回空操作，或在某些平台抛出错误 / 黑屏）。

Unity 官方文档中明确提到：`DrawMeshInstancedIndirect` **只在支持 compute shaders 的平台上有效**（因为 Indirect Draw 通常与 Compute Shader 搭配实现 GPU culling / 参数生成，而 compute shader 本身也是硬件特性）。



实际兼容性（2026 年视角）：

- PC：DX11+ 级别的显卡（GTX 600 / HD 7000 系列及以上）基本全支持。
- 移动端：大多数 Android Vulkan 设备、所有 iOS Metal 设备支持。
- 非常老的硬件（DX9 时代、OpenGL 3.x 以下）或某些低端嵌入式 GPU 不支持 → Unity 会 fallback 或直接不支持该 API 调用。
- WebGL：目前（Unity 6+）基本不支持 Indirect Draw（WebGL 2.0 规范中没有 glDrawElementsIndirect 的完整暴露）。



**总结：**

Indirect Draw 是 GPU 硬件 + 图形驱动提供的底层能力。

Unity 只负责把你的调用翻译成对应 API 的 Indirect Draw 指令。

如果硬件不支持，Unity 就发不出这个指令，绘制也就不会发生（或退化成无效操作）。



## GPU Driven / 有没有办法处理巨量角色动画？

结论：传统 Animator 完全无法直接使用，但可以使用替代方案（如 VAT 等）。



### GPU Driven 本身的限制

只能处理 **GPU 能一次性读取到的静态/可采样数据** ，完全绕过了 Unity 的传统 CPU 动画管线。



### 传统 Animator 在 GPU Driven 下完全失效的原因



**SkinnedMeshRenderer 不支持 GPU Instancing**

Unity 官方文档（Unity 6.3 LTS 最新版）明确写死：

* `Graphics.DrawMeshInstancedIndirect` / `RenderMeshIndirect` 只支持普通 Mesh（非 Skinned）。
* SkinnedMeshRenderer 的骨骼变形走 Unity 内置 CPU/GPU 混合 skinning pipeline，和 Indirect Instancing 的 vertex shader 路径冲突。



**Animator 是 CPU 驱动**

* 每帧 Animator 更新 bone matrices（矩阵数组），此为 per-object 的 CPU 操作。
* 100 万个对象 = CPU 直接爆炸（哪怕用 DOTS/Jobs 也扛不住百万级）。
* 就算把 Animator 放在远处对象上，DrawMeshInstancedIndirect 也不会读取这些 bone 数据。



**结论：GPU Driven 的根本限制**

所有动画逻辑必须**完全在 Shader / ComputeShader 里采样 ComputeBuffer**，不能依赖 Unity 的 Animator 组件。



### 技术上是否可行？

存在成熟方案。

但在100万+全骨骼角色规模下，几乎不可能保持可玩帧率。（2026年消费级硬件）



**三个主流方案：**

* 最成熟的 GPU 动画 + Instancing 方案：Vertex Animation Textures (VAT / OpenVAT)
* 灵活但更重：纯 ComputeShader GPU Skinning
* 官方路径：Unity DOTS + Animation + Hybrid Renderer



## GPU Driven / Unity 内置方案对比
Unity 6+ 提供了 **GPU Resident Drawer**（基于 BatchRendererGroup / BRG），自动把支持的 MeshRenderer 转为 GPU 驻留 + Indirect Draw + GPU Occlusion Culling。

- **自定义 ComputeShader culling + RenderMeshIndirect**：灵活（任意数据、自定义 LOD/遮挡），适合 procedural / 海量非 GO 对象。
- **GPU Resident Drawer**：开箱即用（Project Settings + URP/HDRP Asset 启用 Instanced Drawing），自动处理静态/动态 GO，内置 Hi-Z occlusion，CPU 收益高，但自定义少（只支持 MeshRenderer）。
- 启用条件：Forward+ 路径、BatchRendererGroup Variants = Keep All。

实际项目：大量静态物体可用内置，自定义海量实例走自定义。



## 尾注

本文适用于 Unity 2022.3 & Unity 6+ 实践。

如有更新，以官方 Scripting API 为准。
