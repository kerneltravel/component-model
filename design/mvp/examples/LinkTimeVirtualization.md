# 链接时虚拟化

**链接时虚拟化** 的用例是将左侧的静态依赖图（其中所有3个组件都导入了 `wasi:filesystem` 接口）转换为右侧的运行时实例图，其中 `parent` 实例已创建了一个 `virtualized` 实例并将其提供给新的 `child` 实例作为 `wasi:filesystem` 实现。

<p align="center"><img src="./images/link-time-virtualization.svg" width="500"></p>

重要的是，`child` 实例无法访问由 `parent` 实例导入的 `wasi:filesystem` 实例。

我们从已经单独编写和编译的 `child.wat` 开始，而不考虑 `parent.wasm`：

```wasm
;; child.wat
(component
  (import "wasi:filesystem" (instance
    (export "read" (func ...))
    (export "write" (func ...))
  ))
  ...
)
```

我们想编写一个父组件，重用子组件，为子组件提供虚拟文件系统。这个虚拟文件系统可以被分解并作为一个单独的组件重用：

```wasm
;; virtualize.wat
(component
  (import "wasi:filesystem" (instance $fs
    (export "read" (func ...))
    (export "write" (func ...))
  ))
  (func (export "read")
    ... transitively calls (func $fs "read)
  )
  (func (export "write")
    ... transitively calls (func $fs "write")
  )
)
```


我们现在通过将`child.wasm`与`virtualize.wasm`组合来编写父组件：
```wasm
;; parent.wat
(component
  (import "wasi:filesystem" (instance $real-fs ...))
  (import "./virtualize.wasm" (component $Virtualize ...))
  (import "./child.wasm" (component $Child ...))
  (instance $virtual-fs (instantiate (component $Virtualize)
    (with "wasi:filesystem" (instance $real-fs))
  ))
  (instance $child (instantiate (component $Child)
    (with "wasi:filesystem" (instance $virtual-fs))
  ))
)
```
在这里，我们导入了`child`和`virtualize`组件，但它们也可以通过在父组件中使用嵌套组件定义来轻松地复制到`parent`组件中：
```wasm
;; parent.wat
(component
  (import "wasi:filesystem" (instance $real-fs ...))
  (component $Virtualize ... copied inline ...)
  (component $Child ... copied inline ...)
  (instance $virtual-fs (instantiate (component $Virtualize)
    (with "wasi:filesystem" (instance $real-fs))
  ))
  (instance $child (instantiate (component $Child)
    (with "wasi:filesystem" (instance $virtual-fs))
  ))
)
```