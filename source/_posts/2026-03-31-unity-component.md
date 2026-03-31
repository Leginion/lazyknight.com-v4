---
title: Unity 组件的挂载方式 | 血条案例
date: 2026-3-31 09:00:02 +0800
tags: [note,unity,unity-componenty]
typora-root-url: ..
typora-copy-images-to: ../img/miscs/
# updated: _______________________________________ +0800
---

刚在我的《吸血鬼幸存者》Demo中做了一个简单的血条。

遇到了以下问题：
1. C# 调用 xLua 的函数交互注册方式
2. Unity 组件的挂载方式
3. 动态实例化血条的方式

## C# 调用 xLua 的函数交互注册方式

推荐的方式是全局共用一个 LuaEnv ，并在初始化 LuaEnv 以后通过 Setup 构建一个静态Util。

代码如下：

```csharp
using UnityEngine;
using XLua;

namespace Vampire
{
    public static class LuaCallUtil
    {
        // [CSharpCallLua] 直接标记到 public delegate 上，不要直接放到 static class ，否则 xLua / Generate Code 会无法正确生成。
        [CSharpCallLua] public delegate void LuaAction();
        [CSharpCallLua] public delegate float LuaGetFloatFunction(string s1);
        
        private static LuaGetFloatFunction _ffPlrGetState;
        public static float? GetPlayerState(string name) => _ffPlrGetState?.Invoke(name);

        // LuaEnv 全局初始化完成后，调用此方法完成各类Lua函数引用的注册。
        public static void Setup(LuaEnv luaEnv)
        {
            // ff_plr_get_state 是 Lua 侧的全局 function ，直接绑定到 Util 的内部方法即可，本例中是 _ffPlrGetState 。
            luaEnv.Global.Get("ff_plr_get_state", out _ffPlrGetState);
        }
    }
}
```

## Unity 组件的挂载方式

我刚才编写了一个 HPBar 用于显示特定战斗对象的血量。

某些固定的member可以直接在Inspector中手动关联。

某些动态的member则可以通过两种方式关联：
1. 如果目标GameObject在Scene中固定存在 - 直接拖拽Prefab+手动关联。
2. 需要动态创建GameObject - 实例化后通过代码设置关联。

## 动态实例化血条的方式

在本例中，血条本质上仍然是一个 GameObject 。

我们可以先创建血条所依赖的GameObject（比如敌人对象），然后再创建血条。
随后，我们获取血条的 HPBar 组件，并设置组件上的引用member，指向目标GameObject（敌人对象）。

完成以上操作后，一个血条的动态化绑定操作就完成了，剩下的就由 HPBar 内部的逻辑进行管理即可。
