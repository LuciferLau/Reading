# Chap 1. 概述
Lua目录结构（v5.4.2）  
```
/* 虚拟机 */
// LUA接口辅助库
lapi.c
lapi.h
// 调试辅助函数
ldebug.c
ldebug.h
// LUA虚拟机
lvm.c
lvm.h
// 函数调用和栈管理
ldo.c
ldo.h
// 函数原型和闭包的辅助库
lfunc.c
lfunc.h
// LUA全局表，记录虚拟机的状态
lstate.c
lstate.h
// 元表tag方法
ltm.c
ltm.h
// 核心API
lua.c
lua.h
lua.hpp

/* 数据结构 */
// LUA数据类型定义 lu_type
lctype.c
lctype.h
// LUA对象定义
lobject.c
lobject.h
// LUA字符串
lstring.c
lstring.h
// LUA表
ltable.c
ltable.h

/* 编译原理 */
// 源码生成器
lcode.c
lcode.h
// 代码预编译
ldump.c
// 加载预编译代码
lundump.c
lundump.h
// LUA编译器
luac.c
// 词法分析器
llex.c
llex.h
// 语法语义分析器
lparser.c
lparser.h
// make
Makefile 

/* 内存 */
// 内存管理
lmem.c
lmem.h
// 垃圾回收
lgc.c
lgc.h

// 定义了虚拟机的操作跳转
ljumptab.h 
// LUA虚拟机操作码
lopcodes.c
lopcodes.h
// LUA操作码数组定义，有序
lopnames.h 
// 跨平台兼容
lprefix.h
// 一些限制，基础类型，安装依赖的定义
llimits.h

/* LIBRARY */
linit.c     // 负责库的初始化
lbaselib.c  // 基础库
lcorolib.c  // 协程库
ldblib.c    // 调式库
lutf8lib.c  // 标准UTF-8库
liolib.c    // I/O库
lmathlib.c  // 数学库
loadlib.c   // 动态扩展库加载器
loslib.c    // 标准系统调用库
lstrlib.c   // 字符串库
ltablib.c   // 表库
lualib.h    // LUA标准库
//辅助函数库：主要提供了错误处理接口和虚拟机状态维护
lauxlib.c
lauxlib.h

// 配置文件
luaconf.h

// 字节流
lzio.c
lzio.h
```
