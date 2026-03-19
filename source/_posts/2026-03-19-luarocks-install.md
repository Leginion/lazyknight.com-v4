---
title: Windows安装LuaRocks+GCC
date: 2026-3-19 07:42:33 +0800
tags: [note,luarocks]
---

* 官方文档 - [Installation instructions for Windows](https://github.com/luarocks/luarocks/blob/main/docs/installation_instructions_for_windows.md)

* LuaRocks - Binary Latest Windows-32.zip - https://luarocks.github.io/luarocks/releases/

* [luarocks-3.13.0-windows-64.zip](https://luarocks.github.io/luarocks/releases/luarocks-3.13.0-windows-64.zip)



1. 下载二进制程序
2. 解压
3. 配置PATH
4. 安装 [Mingw](https://mingw-w64.org/) - 包含gcc，用于编译源码
    * [Windows / MSYS2 (GCC and mingw-w64)](https://www.mingw-w64.org/getting-started/msys2/)
5. 安装 [Lua for windows](https://github.com/rjpcomputing/luaforwindows/releases)



其中 MSYS2 (GCC) 的安装：

1. 确保先安装 [MSYS2](https://www.msys2.org/)
2. 打开 MSYS2 UCRT64
3. 安装 C / C++ Compiler
    1. `pacman -S mingw-w64-ucrt-x86_64-gcc`
    2. 检查版本 - GCC / mingw-w64: 
```
$ pacman -Qi mingw-w64-ucrt-x86_64-gcc | grep Version
Version         : 15.2.0-8  # GCC Version
$ pacman -Qi mingw-w64-ucrt-x86_64-headers | grep Version
Version         : 13.0.0.r354.g40ab95d18-1  # mingw-w64 Version
```



构建代码：

```c
// hello.c
#include <stdio.h>

int main(void) {
    printf("Hello, Windows!\n");
    return 0;
}
```

`gcc hello.c -o hello.exe`



测试代码：

```
./hello.exe
```



Proxy Settings：

https://github.com/luarocks/luarocks/blob/main/docs/luarocks_through_a_proxy.md

```
set http_proxy=http://server:1234
set https_proxy=http://server:1234
```



Github Protocol Fix：

`git config --global url."https://github.com/".insteadOf git://github.com/`



Using LuaRocks:

https://github.com/luarocks/luarocks/blob/main/docs/using_luarocks.md



Use 5.1:

`luarocks config lua_version 5.1`

*see: C:\Users\Administrator\AppData\Roaming\luarocks*



.vscode/settings.json:

```json
{
  "Lua.runtime.version": "Lua 5.1",
  "Lua.runtime.path": [
    "${workspaceFolder}/lua_modules/share/lua/5.1/?.lua",
    "${workspaceFolder}/lua_modules/share/lua/5.1/?/init.lua",
    "${workspaceFolder}/lua_modules/share/lua/5.1/?/?.lua"
  ],
  "Lua.workspace.library": [
    "${workspaceFolder}/lua_modules/share/lua/5.1"
  ],
  "Lua.workspace.checkThirdParty": false,
  "Lua.telemetry.enable": false,
  "Lua.runtime.pathStrict": false,
  "Lua.diagnostics.globals": [],
  "stylua.targetReleaseVersion": "latest"
}
```



luarocks:

```
luarocks init
luarocks install ...
```



.luarocks/config-5.1.lua

```lua
variables = {
   LUA = "D:\\Program Files (x86)\\Lua\\5.1\\lua.exe",
   LUA_BINDIR = "D:\\Program Files (x86)\\Lua\\5.1",
   LUA_DIR = "D:\\Program Files (x86)\\Lua\\5.1",
   LUA_INCDIR = "D:\\Program Files (x86)\\Lua\\5.1/include"
}
```



.luarocks/default-lua-version.lua

```lua
return "5.1"
```



package路径调试

```lua
for p in package.path:gmatch("[^;]+") do print("  " .. p) end
```



注意，部分老包体按照luarocks2规则编写，不符合luarocks3规范，引入后可能会有问题。

（通常和require路径相关）
