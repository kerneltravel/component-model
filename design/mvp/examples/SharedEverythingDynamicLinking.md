#共享一切的动态链接

*共享一切动态链接* 是指能够将多个Core WebAssembly模块组合成一个组件，以便可以与其他组件共享公共模块。这提供了一种替代*静态链接*的方法，后者强制将公共代码复制到每个组件中。这种类型的链接应该能够利用现有的本地动态链接支持（`.dll`或`.so`），其中有一个单独的共享线性内存（因此是*共享一切*动态链接）。

共享一切动态链接应该是组件中描述的共享无内容动态链接的*补充*。特别是，动态链接模块不能跨组件实例边界共享线性内存。例如，我们希望左侧的静态依赖图在右侧生成运行时实例图：在右侧创建运行时实例图：


<p align="center"><img src="./images/shared-everything-dynamic-linking.svg" width="600"></p>

在这里，`libc`定义并导出了一个线性内存，该内存由同一组件实例中包含的其他模块实例导入。因此，在运行时，复合应用程序创建了`libc`模块的*三个*实例（创建了*三个*线性内存），但仅包含`libc`代码的*一个*副本。在许多模块系统中实现此用例很棘手，其中共享模块代码意味着共享模块实例状态。


## `libc`

与本地动态链接一样，共享一切的动态链接需要所有生成参与模块的工具链遵循的工具链约定。在这里，与大多数约定一样，libc扮演着特殊的角色，并且假定与编译器捆绑在一起。作为这个特殊角色的一部分，libc定义并导出线性内存，并约定每个其他模块从libc导入内存：
```wasm
;; libc.wat
(module
  (memory (export "memory") 1)
  (func (export "malloc") (param i32) (result i32) ...impl)
  ...
)
```

我们的编译器还将捆绑包含与`libc`兼容的声明的标准库头文件。不过，我们首先需要一些帮助器宏，我们将把它们放在`stddef.h`中：
```c
/* stddef.h */
#define WASM_IMPORT(module,name) __attribute__((import_module(#module), import_name(#name))) name
#define WASM_EXPORT(name) __attribute__((export_name(#name))) name
```

这些宏使用clang特定的属性[`import_module`]，[`import_name`]和[`export_name`]来确保注释的C函数产生正确的wasm导入或导出定义。其他编译器将使用自己的魔法语法来实现相同的效果。使用这些宏，我们可以在`stdlib.h`中声明`malloc`：
```c
/* stdlib.h */
#include <stddef.h>
#define LIBC(name) WASM_IMPORT(libc, name)
void* LIBC(malloc)(size_t n);
```

有了这些注解，包含并调用这个函数的C程序将被编译为包含以下导入：
```wasm
(import "libc" "malloc" (func (param i32) (result i32)))
```


## `libzip`

`libzip`暴露给客户端的接口是一个头文件：
```c
/* libzip.h */
#include <stddef.h>
#define LIBZIP(name) WASM_IMPORT(libzip, name)
void* LIBZIP(zip)(void* in, size_t in_size, size_t* out_size);
```
可以通过以下源文件实现：
```c
/* libzip.c */
#include <stdlib.h>
void* WASM_EXPORT(zip)(void* in, size_t in_size, size_t* out_size) {
  ...
  void *p = malloc(n);
  ...
}
```

请注意，`libzip.h`使用*import*属性注释`zip`声明，以便客户端模块生成正确的wasm*import定义*，而`libzip.c`使用*export*属性注释`zip`定义，以便此函数在编译模块时生成正确的*export定义*。使用`clang -shared libzip.c`编译会产生一个形状如下的模块：
```wasm
;; libzip.wat
(module
  (import "libc" "memory" (memory 1))
  (import "libc" "malloc" (func (param i32) (result i32)))
  (func (export "zip") (param i32 i32 i32) (result i32)
    ...
  )
)
```


## `zipper`

`zipper`组件的主模块是由以下内容实现的源文件：
```c
/* zipper.c */
#include <stdlib.h>
#include "libzip.h"
int main(int argc, char* argv[]) {
  ...
  void *in = malloc(n);
  ...
  void *out = zip(in, n, &out_size);
  ...
}
```

当被一个（未来的）组件识别的 "clang "编译时，产生的组件会看起来像这样：
```wasm
;; zipper.wat
(component
  (import "libc" (core module $Libc
    (export "memory" (memory 1))
    (export "malloc" (func (param i32) (result i32)))
  ))
  (import "libzip" (core module $Libzip
    (import "libc" "memory" (memory 1))
    (import "libc" "malloc" (func (param i32) (result i32)))
    (export "zip" (func (param i32 i32 i32) (result i32)))
  ))

  (core module $Main
    (import "libc" "memory" (memory 1))
    (import "libc" "malloc" (func (param i32) (result i32)))
    (import "libzip" "zip" (func (param i32 i32 i32) (result i32)))
    ...
    (func (export "zip") (param i32 i32) (result i32 i32)
      ...
    )
  )

  (core instance $libc (instantiate (module $Libc)))
  (core instance $libzip (instantiate (module $Libzip))
    (with "libc" (instance $libc))
  ))
  (core instance $main (instantiate (module $Main)
    (with "libc" (instance $libc))
    (with "libzip" (instance $libzip))
  ))
  (func $zip (param (list u8)) (result (list u8)) (canon lift
    (core func $main "zip")
    (memory (core memory $libc "memory")) (realloc (func $libc "realloc"))
  ))
  (export "zip" (func $zip))
)
```
在这里，`zipper`将其自己的私有模块代码(`$Main`)与可共享的`libc`和`libzip`模块代码链接起来，确保每个新的`zipper`实例都获得一个新的、私有的`libc`和`libzip`实例。


## `libimg`

接下来我们创建一个共享模块`libimg`，它依赖于`libzip`：
```c
/* libimg.h */
#include <stddef.h>
#define LIBIMG(name) WASM_IMPORT(libimg, name)
void* LIBIMG(compress)(void* in, size_t in_size, size_t* out_size);
```
```c
/* libimg.c */
#include <stdlib.h>
#include "libzip.h"
void* WASM_EXPORT(compress)(void* in, size_t in_size, size_t* out_size) {
  ...
  void *out = zip(in, in_size, &out_size);
  ...
}
```
用`clang -shared libimg.c`进行编译，产生一个`libimg`模块：
```wat
;; libimg.wat
(module
  (import "libc" "memory" (memory 1))
  (import "libc" "malloc" (func (param i32) (result i32)))
  (import "libzip" "zip" (func (param i32 i32 i32) (result i32)))
  (func (export "compress") (param i32 i32 i32) (result i32)
    ...
  )
)
```


## `imgmgk`

`imgmgk`组件的主要模块是由以下部分实现的`stddef.h`, `libzip.h` 和 `libimg.h`。当被一个（未来的）组件识别的"clang "编译时，得到的组件将看起来像：
```wasm
;; imgmgk.wat
(component $Imgmgk
  (import "libc" (core module $Libc ...))
  (import "libzip" (core module $Libzip ...))
  (import "libimg" (core module $Libimg ...))

  (core module $Main
    (import "libc" "memory" (memory 1))
    (import "libc" "malloc" (func (param i32) (result i32)))
    (import "libimg" "compress" (func (param i32 i32 i32) (result i32)))
    ...
    (func (export "transform") (param i32 i32) (result i32 i32)
      ...
    )
  )

  (core instance $libc (instantiate (module $Libc)))
  (core instance $libzip (instantiate (module $Libzip)
    (with "libc" (instance $libc))
  ))
  (core instance $libimg (instantiate (module $Libimg)
    (with "libc" (instance $libc))
    (with "libzip" (instance $libzip))
  ))
  (core instance $main (instantiate (module $Main)
    (with "libc" (instance $libc))
    (with "libimg" (instance $libimg))
  ))
  (func $transform (param (list u8)) (result (list u8)) (canon lift
    (core func $main "transform")
    (memory (core memory $libc "memory")) (realloc (func $libc "realloc"))
  ))
  (export "transform" (func $transform))
)
```
在这里，我们看到了通过 "实例 "定义表达的动态链接的模块之间的依赖DAG的一般模式出现。通过 "实例 "定义表达的动态链接的模块之间的依赖DAG。


## `app`

最后，我们可以通过组合`zipper`和`imgmgk`来创建`app`组件。由此产生的组件可以看起来像这样：
```wasm
;; app.wat
(component
  (import "libc" (core module $Libc ...))
  (import "libzip" (core module $Libzip ...))
  (import "libimg" (core module $Libimg ...))

  (import "zipper" (component $Zipper ...))
  (import "imgmgk" (component $Imgmgk ...))

  (core module $Main
    (import "libc" "memory" (memory 1))
    (import "libc" "malloc" (func (param i32) (result i32)))
    (import "zipper" "zip" (func (param i32 i32) (result i32 i32)))
    (import "imgmgk" "transform" (func (param i32 i32) (result i32 i32)))
    ...
    (func (export "run") (param i32 i32) (result i32 i32)
      ...
    )
  )

  (instance $zipper (instantiate (component $Zipper)
    (with "libc" (module $Libc))
    (with "libzip" (module $Libzip))
  ))
  (instance $imgmgk (instantiate (component $Imgmgk)
    (with "libc" (module $Libc))
    (with "libzip" (module $Libzip))
    (with "libimg" (module $Libimg))
  ))

  (core instance $libc (instantiate (module $Libc)))
  (core func $zip (canon lower
    (func $zipper "zip")
    (memory (core memory $libc "memory")) (realloc (func $libc "realloc"))
  ))
  (core func $transform (canon lower
    (func $imgmgk "transform")
    (memory (core memory $libc "memory")) (realloc (func $libc "realloc"))
  ))
  (core instance $main (instantiate (module $Main)
    (with "libc" (instance $libc))
    (with "zipper" (instance (export "zip" (func $zipper "zip"))))
    (with "imgmgk" (instance (export "transform" (func $imgmgk "transform"))))
  ))
  (func $run (param string) (result string) (canon lift
    (core func $main "run")
    (memory (core memory $libc "memory")) (realloc (func $libc "realloc"))
  ))
  (export "run" (func $run))
)
```
请注意，`$Libc`被传递给嵌套的`zipper`和`imgmgk`实例作为一个（未实例化的）模块，然后`app`创建了自己的私有实例，该实例与其私有`$Main`模块实例链接。因此，这三个组件共享`libc`*代码*，而不共享`libc`*状态*，实现了开始的实例图。

## 循环依赖

如果需要循环依赖，则可以通过以下方式打破这些循环：
* 在模块依赖图上识别一个[跨度]DAG；
* 将跨度DAG的边缘上的调用保留为正常的函数导入和直接调用（如上所示）；
* 然后将“反向边”上的调用转换为导入的间接调用（`call_indirect`），该导入包含函数表中的索引（`global i32`）。

例如，模块`$A`和`$B`之间的循环可以通过任意地说`$B`可以直接导入`$A`，然后通过共享的可变`funcref`表通过`call_indirect`路由`$A`的导入来打破。

```wat
(module $A
  ;; A通过table+index间接导入B.bar
  (import "linkage" "table" (table funcref))
  (import "linkage" "bar-index" (global $bar-index (mut i32)))

  (type $FooType (func))
  (func $some_use
    (call_indirect $FooType (global.get $bar-index))
  )

  ;; A直接将A.foo导出到B
  (func (export "foo")
    ...
  )
)
```
```wat
(module $B
  ;; B直接导入A.foo
  (import "a" "foo" (func $a_foo)) ;; B可以直接导入A

  ;; B间接地将B.bar导出到A
  (func $bar ...)
  (import "linkage" "table" (table $ftbl funcref))
  (import "linkage" "bar-index" (global $bar-index (mut i32)))
  (elem (table $ftbl) (offset (i32.const 0)) $bar)
  (func $start (global.set $bar-index (i32.const 0)))
  (start $start)
)
```
最后，工具链可以通过发出一个包含共享函数表和“bar-index”可变全局变量的包装适配器模块来将它们链接成一个完整的程序。
```wat
(component
  (import "A" (core module $A ...))
  (import "B" (core module $B ...))
  (core module $Linkage
    (global (export "bar-index") (mut i32))
    (table (export "table") funcref 1)
  )
  (core instance $linkage (instantiate (module $Linkage)))
  (core instance $a (instantiate (module $A)
    (with "linkage" (instance $linkage))
  ))
  (core instance $b (instantiate (module $B)
    (import "a" (instance $a))
    (with "linkage" (instance $linkage))
  ))
)
```


## 函数指针标识

为了确保在共享库中跨C函数指针标识，对于每个导出的函数，共享库都需要导出`func`定义和包含该`func`在全局`funcref`表中的索引的`(global (mut i32))`。

因为共享库无法知道其导出函数在全局`funcref`表中的绝对偏移量，所以表格插槽的偏移量必须是动态的。一种实现这一点的方法是，共享库从共享库的`start`函数中调用`libc`的`ftalloc`导出（类似于`malloc`，但用于从全局`funcref`表中分配），然后可以将元素写入分配的偏移量中，并将它们的索引写入导出的`(global (mut i32))`中。

（理论上，当主程序对其共享库具有更多静态知识时，可能存在更有效的方案。）


## 线性内存堆栈指针

为了实现地址获取的局部变量，变长参数和其他边角情况，wasm编译器在线性内存中维护一个堆栈，该堆栈与本机WebAssembly堆栈保持同步。线性内存堆栈的顶部指针通常在单个`(global (mut i32))`变量中维护，该变量必须由所有链接的实例共享。按照上述链接方案，此全局变量自然将与线性内存一起由`libc`导出。

## 运行时动态链接

在运行时动态链接的一般情况下，类似于`dlopen`的风格，其中一个*先验未知*模块在运行时链接到程序中是不可能纯粹在wasm中实现的，需要额外提供主机API，包括：
*将文件或字节编译为模块；
*读取模块的导入字符串；
*给定导入值列表动态实例化模块；以及
*动态提取实例的导出。

这些API可以作为[WASI]的一部分进行标准化。此外，[JS API]具有上述所有功能，允许在浏览器中原型化和实现WASI API。




[`import_module`]: https://clang.llvm.org/docs/AttributeReference.html#import-module
[`import_name`]: https://clang.llvm.org/docs/AttributeReference.html#import-name
[`export_name`]: https://clang.llvm.org/docs/AttributeReference.html#export-name
[Spanning]: https://en.wikipedia.org/wiki/Spanning_tree
[WASI]: https://github.com/webassembly/wasi
[JS API]: https://webassembly.github.io/spec/js-api/index.html
