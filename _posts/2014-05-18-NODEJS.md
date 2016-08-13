---
layout: post
title:  Node.js开发指南——学习笔记1
categories: [node.js]
tags: [node.js]
description: node.js学习
---

第三章
1.开始使用Node.js编程
supervisor 代码监视器，对于html，在浏览器刷新后就行
2.异步式I/O与事件式编程
单线程，异步式，先执行后面的语句，等IO执行完毕后，执行回调函数
事件式编程EventEmitter
3.模块和包
requir exports 单次加载 覆盖exports module.exports=func
4.调试
类似gdb的调试工具 node debug **.js 可使用eclipse调试（或者远程调试）

第四章
1.全局对象
所有全局变量都是global这个宿主的属性
2.util


