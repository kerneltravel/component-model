# 组件模型高级目标  
（有关比较，请参阅WebAssembly的[原始高级目标]。 

1. 定义一种可移植的、加载和运行时高效的二进制格式 从 WebAssembly 核心模块构建的单独编译的组件 启用可移植的跨语言组合。 
2. 支持可移植、可虚拟化、静态分析的定义， 功能安全、与语言无关的接口，尤其是那些 由 [WASI] 定义。 
3. 维护和增强WebAssembly的独特价值主张： 
   * *语言中立*：避免将组件模型偏向于一个 语言或语言家族。 
   * *可嵌入性*：设计要嵌入到各种集合中的组件 主机执行环境，包括浏览器、服务器、中介、 小型设备和数据密集型系统。 
   * *可优化性*：最大化可用的静态信息 提前编译器可最大限度地降低实例化和 启动。 
   * *形式语义*：在同一语义中定义组件模型 框架作为核心。 
   * *Web平台集成*：确保组件可以原生支持 在浏览器中通过扩展现有的 WebAssembly 集成点： [JS API]、[Web API] 和 [ESM 集成]。在本机支持之前 实现，确保组件可以通过浏览器进行多填充 提前编译到当前支持的浏览器功能。 
5. *增量*定义组件模型：从一组 [初始用例] 并随着时间的推移扩展用例集， 根据反馈和经验优先考虑。 


## 非目标 
1. 不要试图解决 100% 的 WebAssembly 嵌入场景。 
   * 某些方案将需要与上述目标冲突的功能。 
   * 采用分层规范方法，不支持嵌入 可以通过替代分层规范或通过以下方式解决场景 直接嵌入现有的 WebAssembly 核心规范。 
2. 不要试图解决通过某种组合更好地解决的问题 工具链、平台或更高层的规格，包括： 
   * 包管理和版本控制; 
   * 部署和实时升级/动态重新配置; 
   * 持久性和存储性;和 
   * 分布式计算和部分故障。 
4. 不要指定一组“组件服务”。
   * 指定可能由主机实现并公开的服务 组件是 WASI 的域，超出了组件模型的范围。
   * 另请参阅[WASI FAQ条目](FAQ.md#how-does-wasi-relate-to-the-component-model)。

[原始高层目标]: https://github.com/WebAssembly/design/blob/main/HighLevelGoals.md  
[WASI]: https://github.com/WebAssembly/WASI/blob/main/README.md  
[JS API]: https://webassembly.github.io/spec/js-api/index.html  
[Web API]: https://webassembly.github.io/spec/web-api/index.html  
[ESM集成]: https://github.com/WebAssembly/esm-integration/tree/main/proposals/esm-integration  
[初始用例]: UseCases.md#Initial-MVP
