---
title: Unity 音频音量低于预期问题临时处理
date: 2026-4-4 16:37:29 +0800
tags: [note,unity,unity-audio]
typora-root-url: ..
typora-copy-images-to: ../img/miscs/
# updated: _______________________________________ +0800
---

选中AudioClip资源，勾选 `Force To Mono+Normalize` 。

设置AudioSource对应GO，设置 `Spatial Blend` 为 0 （2D）。

这疑似和3D音效机制有关系，以后再研究。
