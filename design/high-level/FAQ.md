# 常见问题解答

### WASI与组件模型有什么关系？

[WASI]建立在组件模型之上，组件模型提供了定义WASI接口所使用的基础构建块，包括：
* 可以在WASI接口中使用的类型语法；
* WASI可以假定用于组合代码的独立模块、隔离其功能并虚拟化WASI接口的链接功能；
* 核心wasm ABI，核心wasm工具链可以针对WASI进行编译时使用。

与传统操作系统相比，组件模型填充了操作系统进程模型的角色（定义进程如何启动并相互通信），而WASI填充了操作系统的许多I/O接口的角色。

使用WASI不会强制客户端针对组件模型进行目标设置。任何核心wasm生产者都可以简单地针对组件模型为给定WASI接口的签名定义的核心wasm ABI进行目标设置。这种方法重新打开了许多由组件模型回答的问题，特别是当涉及到多个wasm模块时，但对于单模块场景或高度自定义场景，这可能是适当的。

[WASI]: https://github.com/WebAssembly/WASI/blob/main/README.md
