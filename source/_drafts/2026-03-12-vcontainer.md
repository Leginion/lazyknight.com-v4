---
title: VContainer笔记
date: 2026-3-12 17:48:14 +0800
tags: [note]
typora-root-url: ..
---

VContainer为Unity提供DI功能。

## 核心概念

* 容器 - VContainer 本身作为一个容器，能够接纳诸多模块，动态分配到不同目标
* 注入目标 - 目标可以声明自己需要的模块，由容器自动为该目标提供相关模块，不再需要用户手动索引
* 注册 - 只有被注册到容器内的模块才能够被容器管理

## 一些模块类型

3种主要的模块类型构成了VContainer的主要成员：Scope，Presenter，Service。

你需要在Scope中注册需要被容器管理的Presenter、Service，并定义它们的生命周期。
你可以在Service中实现特定的业务逻辑功能，能够被视作Service的模块包括但不限于Manager、Controller等等。
你可以在Presenter中初始化游戏、建立游戏核心逻辑，管理Initialize、Start、Update等逻辑。

GameLifetimeScope.cs:

```csharp
public class GameLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        builder.Register<HelloWorldService>(Lifetime.Singleton);
        builder.RegisterEntryPoint<GamePresenter>();
    }
}
```

GamePresenter.cs:

```csharp
public class GamePresenter : ITickable
{
    readonly HelloWorldService helloWorldService;

    public GamePresenter(HelloWorldService helloWorldService)
    {
        this.helloWorldService = helloWorldService;
    }

    void ITickable.Tick()
    {
        // helloWorldService.Hello();
    }
}
```

HelloWorldService.cs:

```csharp
public class HelloWorldService
{
    public void Hello()
    {
        UnityEngine.Debug.Log("Hello world!");
    }
}
```

## 实践

### MonoBehaviour

你可以注册需要交给VContainer管理的MonoBehaviour（Component）。

TODO// 作用……？

### ObjectPool

理解ObjectPool与VContainer的结合方式至关重要。



**相关仓库**

* [VContainer-ViewPool](https://github.com/Valerio-Corso/VContainer-ViewPool)
