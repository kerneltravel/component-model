# 组件模型二进制格式说明

本文档定义了在[说明文档](Explainer.md)中定义的AST的二进制格式。顶级生产是“component”，约定是以“.wasm”为后缀的文件可以包含一个[`core:module`] *或*一个`component`，使用“layer”字段来区分两者的前8个字节（有关更多详细信息，请参见[下面](#组件定义)）。

注意：本文档并不意味着完全定义解码或验证规则，而是合并了两者中最少需要了解的元素，仅提供足够的细节来创建一个原型。二进制格式和验证的完整定义将在[正式规范](../../spec/)中提供。


## 组件定义
（请参见说明文档中的[组件定义](Explainer.md#component-definitions)。）
```
component ::= <preamble> s*:<section>*            => (component flatten(s*))
preamble  ::= <magic> <version> <layer>
magic     ::= 0x00 0x61 0x73 0x6D
version   ::= 0x0a 0x00
layer     ::= 0x01 0x00
section   ::=    section_0(<core:custom>)         => ϵ
            | m:section_1(<core:module>)          => [core-prefix(m)]
            | i*:section_2(vec(<core:instance>))  => core-prefix(i)*
            | t*:section_3(vec(<core:type>))      => core-prefix(t)*
            | c: section_4(<component>)           => [c]
            | i*:section_5(vec(<instance>))       => i*
            | a*:section_6(vec(<alias>))          => a*
            | t*:section_7(vec(<type>))           => t*
            | c*:section_8(vec(<canon>))          => c*
            | s: section_9(<start>)               => [s]
            | i*:section_10(vec(<import>))        => i*
            | e*:section_11(vec(<export>))        => e*
```

注：
- 重用了 Core 二进制规则：[`core:section`]、[`core:custom`]、[`core:module`]
- `core-prefix(t)` 元函数在 `t` 的最左括号后插入 `core` 标记（例如，`core-prefix( (module (func)) )` 是 `(core module (func))`）。
- 上面给出的 `version` 是预标准版本。如果提案在最终标准化之前更改，则 `version` 将从 `0xa` 上调以协调原型。当标准最终确定时，`version` 将最后一次更改为 `0x1`。（这与 Core WebAssembly 1.0 规范采取的路径相同。）
- `layer` 字段旨在在二进制格式早期区分模块和组件。（Core WebAssembly 模块的 4 字节 [`core:version`] 字段中已经隐含了 `layer` 字段为 `0x0`。）

## 实例定义
（请参见说明文档中的[实例定义](Explainer.md#instance-definitions)。）
```
core:instance       ::= ie:<core:instanceexpr>                             => (instance ie)
core:instanceexpr   ::= 0x00 m:<moduleidx> arg*:vec(<core:instantiatearg>) => (instantiate m arg*)
                      | 0x01 e*:vec(<core:inlineexport>)                   => e*
core:instantiatearg ::= n:<core:name> 0x12 i:<instanceidx>                 => (with n (instance i))
core:sortidx        ::= sort:<core:sort> idx:<u32>                         => (sort idx)
core:sort           ::= 0x00                                               => func
                      | 0x01                                               => table
                      | 0x02                                               => memory
                      | 0x03                                               => global
                      | 0x10                                               => type
                      | 0x11                                               => module
                      | 0x12                                               => instance
core:inlineexport   ::= n:<core:name> si:<core:sortidx>                    => (export n si)

instance            ::= ie:<instanceexpr>                                  => (instance ie)
instanceexpr        ::= 0x00 c:<componentidx> arg*:vec(<instantiatearg>)   => (instantiate c arg*)
                      | 0x01 e*:vec(<inlineexport>)                        => e*
instantiatearg      ::= n:<name> si:<sortidx>                              => (with n si)
sortidx             ::= sort:<sort> idx:<u32>                              => (sort idx)
sort                ::= 0x00 cs:<core:sort>                                => core cs
                      | 0x01                                               => func
                      | 0x02                                               => value
                      | 0x03                                               => type
                      | 0x04                                               => component
                      | 0x05                                               => instance
inlineexport        ::= n:<name> si:<sortidx>                              => (export n si)
name                ::= len:<u32> n:<name-chars>                           => n (if len = |n|)
name-chars          ::= l:<label>                                          => l
                      | '[constructor]' r:<label>                          => [constructor]r
                      | '[method]' r:<label> '.' m:<label>                 => [method]r.m
                      | '[static]' r:<label> '.' s:<label>                 => [static]r.s
label               ::= w:<word>                                           => w
                      | l:<label> '-' w:<word>                             => l-w
word                ::= w:[0x61-0x7a] x*:[0x30-0x39,0x61-0x7a]*            => char(w)char(x)*
                      | W:[0x41-0x5a] X*:[0x30-0x39,0x41-0x5a]*            => char(W)char(X)*
```
注：
- 重用了 Core 二进制规则：[`core:name`]、（可变长度编码的）[`core:u32`]
- `core:sort` 的值被选择为与 [`core:importdesc`] 的操作码鉴别标志匹配。
- `type` 被添加到 `core:sort` 中，以期望显示 [type-imports] 提案。在该提案之前，核心模块将无法实际导入或导出类型，但 `type` 排序作为外部别名的一部分是被允许的（如下所述）。
- `module` 和 `instance` 被添加到 `core:sort` 中，以期望显示 [module-linking] 提案，该提案将这些类型添加到 Core WebAssembly。在此之前，它们对于别名是有用的（如下所述）。
- `core:instantiatearg` 的验证初始只允许 `instance` 排序，但会在扩展核心 wasm 时扩展为接受其他排序。
- 对于 `instantiate` 的验证要求 `name` 出现在 `c` 的 `externname` 中（与匹配的类型）。
- 在验证 `instantiate` 之后，每个单独的类型导入使用 `with` 提供后，实际提供的类型立即替换所有导入的使用，以便后续的导入和所有导出现在已针对实际类型进行了特化。
- `sortidx` 中的索引根据它们的 `sort` 的索引空间进行验证，这些索引空间在验证每个定义时逐步构建。
- 验证要求所有带注释的 `name` 仅出现在 `func` `export` 中，并且 `r` 的 `label` 与前面的 `resource` 导出的 `name` 匹配。
- 对于 `[constructor]` 名称的验证要求 `func` 返回 `(result (own $R))`，其中 `$R` 是标记为 `r` 的资源。
- 对于 `[method]` 和 `[static]` 名称的验证要求函数的第一个参数为 `(param "self" (borrow $R))`，其中 `$R` 是标记为 `r` 的资源，并且要求所有字段名都不交。


## 别名定义
（请参见说明文档中的[别名定义](Explainer.md#alias-definitions)。）
```
alias       ::= s:<sort> t:<aliastarget>                => (alias t (s))
aliastarget ::= 0x00 i:<instanceidx> n:<name>           => export i n
              | 0x01 i:<core:instanceidx> n:<core:name> => core export i n
              | 0x02 ct:<u32> idx:<u32>                 => outer ct idx
```

注：
- 重用了 Core 二进制规则：（可变长度编码的）[`core:u32`]
- 对于 `export` 别名，会验证 `i` 是否引用导出具有指定 `sort` 的 `n` 的实例索引空间中的一个实例。
- 对于 `outer` 别名，会验证 `ct` 是否小于等于封闭组件的数量，并且会验证 `i` 是否是封闭组件的第 `i` 个 `sort` 索引空间中的有效索引（从内向外计数，从 `0` 开始引用当前组件）。
- 对于 `outer` 别名，验证将 `sort` 限制为 `type`、`module` 或 `component` 的其中一个，并且还要求外部别名的类型不是生成类型（`resource` 类型）。

## 类型定义
（请参见说明文档中的[类型定义](Explainer.md#type-definitions)。）

```
core:type        ::= dt:<core:deftype>                  => (type dt)        (GC proposal)
core:deftype     ::= ft:<core:functype>                 => ft               (WebAssembly 1.0)
                   | st:<core:structtype>               => st               (GC proposal)
                   | at:<core:arraytype>                => at               (GC proposal)
                   | mt:<core:moduletype>               => mt
core:moduletype  ::= 0x50 md*:vec(<core:moduledecl>)    => (module md*)
core:moduledecl  ::= 0x00 i:<core:import>               => i
                   | 0x01 t:<core:type>                 => t
                   | 0x02 a:<core:alias>                => a
                   | 0x03 e:<core:exportdecl>           => e
core:alias       ::= s:<core:sort> t:<core:aliastarget> => (alias t (s))
core:aliastarget ::= 0x01 ct:<u32> idx:<u32>            => outer ct idx
core:importdecl  ::= i:<core:import>                    => i
core:exportdecl  ::= n:<core:name> d:<core:importdesc>  => (export n d)
```

注：
- 重用了 Core 二进制规则：[`core:import`]、[`core:importdesc`]、[`core:functype`]
- `core:moduledecl` 的验证（当前）拒绝在 `type` 声明符（即嵌套的核心模块类型）中定义 `core:moduletype`。
- 如说明所述，每个模块类型都使用最初为空的类型索引空间进行验证。
- `alias` 声明符目前仅允许 `outer` `type` 别名，但在核心 wasm 添加类型导出时会添加 `export` 别名。
- `outer` 别名的验证无法查看超出封闭核心类型索引空间之外的内容。由于 MVP 中不能嵌套核心模块和核心模块类型，这意味着 MVP 中 `alias` 声明符中的最大 `ct` 为 `1`。

```
type          ::= dt:<deftype>                            => (type dt)
deftype       ::= dvt:<defvaltype>                        => dvt
                | ft:<functype>                           => ft
                | ct:<componenttype>                      => ct
                | it:<instancetype>                       => it
primvaltype   ::= 0x7f                                    => bool
                | 0x7e                                    => s8
                | 0x7d                                    => u8
                | 0x7c                                    => s16
                | 0x7b                                    => u16
                | 0x7a                                    => s32
                | 0x79                                    => u32
                | 0x78                                    => s64
                | 0x77                                    => u64
                | 0x76                                    => float32
                | 0x75                                    => float64
                | 0x74                                    => char
                | 0x73                                    => string
defvaltype    ::= pvt:<primvaltype>                       => pvt
                | 0x72 lt*:vec(<labelvaltype>)            => (record (field lt)*)
                | 0x71 case*:vec(<case>)                  => (variant case*)
                | 0x70 t:<valtype>                        => (list t)
                | 0x6f t*:vec(<valtype>)                  => (tuple t*)
                | 0x6e l*:vec(<label>)                    => (flags l*)
                | 0x6d l*:vec(<label>)                    => (enum l*)
                | 0x6c t*:vec(<valtype>)                  => (union t*)
                | 0x6b t:<valtype>                        => (option t)
                | 0x6a t?:<valtype>? u?:<valtype>?        => (result t? (error u)?)
                | 0x69 i:<typeidx>                        => (own i)
                | 0x68 i:<typeidx>                        => (borrow i)
labelvaltype  ::= l:<label> t:<valtype>                   => l t
case          ::= l:<label> t?:<valtype>? r?:<u32>?       => (case l t? (refines case-label[r])?)
<T>?          ::= 0x00                                    =>
                | 0x01 t:<T>                              => t
valtype       ::= i:<typeidx>                             => i
                | pvt:<primvaltype>                       => pvt
resourcetype  ::= 0x3f 0x7f f?:<funcidx>?                 => (resource (rep i32) (dtor f)?)
functype      ::= 0x40 ps:<paramlist> rs:<resultlist>     => (func ps rs)
paramlist     ::= lt*:vec(<labelvaltype>)                 => (param lt)*
resultlist    ::= 0x00 t:<valtype>                        => (result t)
                | 0x01 lt*:vec(<labelvaltype>)            => (result lt)*
componenttype ::= 0x41 cd*:vec(<componentdecl>)           => (component cd*)
instancetype  ::= 0x42 id*:vec(<instancedecl>)            => (instance id*)
componentdecl ::= 0x03 id:<importdecl>                    => id
                | id:<instancedecl>                       => id
instancedecl  ::= 0x00 t:<core:type>                      => t
                | 0x01 t:<type>                           => t
                | 0x02 a:<alias>                          => a
                | 0x04 ed:<exportdecl>                    => ed
importdecl    ::= en:<externname> ed:<externdesc>         => (import en ed)
exportdecl    ::= en:<externname> ed:<externdesc>         => (export en ed)
externdesc    ::= 0x00 0x11 i:<core:typeidx>              => (core module (type i))
                | 0x01 i:<typeidx>                        => (func (type i))
                | 0x02 t:<valtype>                        => (value t)
                | 0x03 b:<typebound>                      => (type b)
                | 0x04 i:<typeidx>                        => (component (type i))
                | 0x05 i:<typeidx>                        => (instance (type i))
typebound     ::= 0x00 i:<typeidx>                        => (eq i)
                | 0x01                                    => (sub resource)
```

注：
- 类型操作码遵循与 Core WebAssembly 相同的负 SLEB128 方案，其中类型操作码从 SLEB128(-1) (`0x7f`) 开始向下，将非负 SLEB128 预留给类型索引。
- 对 `valtype` 的验证要求 `typeidx` 引用 `defvaltype`。
- 对 `own` 和 `borrow` 的验证要求 `typeidx` 引用资源类型。
- 验证仅允许在 `functype` 的 `param` 中使用 `borrow`。（这很可能在未来的 PR 中发生变化，将 `functype` 转换为类似于 `moduletype` 和 `componenttype` 的复合类型构造函数，并使用作用域来强制执行此约束。）
- 对 `resourcetype` 的验证要求析构函数（如果存在）具有类型 `[i32] -> []`。
- 对 `instancedecl` 的验证（当前）仅允许在 `alias` 声明符中使用 `type` 和 `instance` 类型。
- 如说明所述，每个组件和实例类型都使用最初为空的类型索引空间进行验证。外部别名可用于从包含组件中提取类型定义。
- `exportdecl` 引入一个新的类型索引，可以由后续类型定义使用。在 `(eq i)` 的情况下，新类型索引实际上是类型 `i` 的别名。在 `(sub resource)` 的情况下，新类型索引引用一个与所有现有类型索引空间中的每个现有类型都不相等的*新*抽象类型。（注意：*后续*别名可以引入等效于此新类型的新类型索引。）
- 验证拒绝在 `componenttype` 和 `instancettype` 中定义 `resourcetype` 类型。因此，在 `componenttype` 中的句柄类型只能引用被导入或导出的资源类型。
- 下面描述的 `externname` 的唯一性验证规则也适用于实例和组件类型级别。
- 对 `externdesc` 的验证要求各种 `typeidx` 类型构造函数与前面的 `sort` 匹配。
- 对函数参数和结果名称、记录字段名称、变体情况名称、标志名称和枚举情况名称的验证要求该名称对于函数、记录、变体、标志或枚举类型定义是唯一的。
- 对变体情况的可选 `refines` 子句的验证要求情况索引小于当前情况的索引（因此情况是无环的）。

## 经典定义
（请参见说明文档中的[规范定义](Explainer.md#canonical-definitions)。）
```
canon    ::= 0x00 0x00 f:<core:funcidx> opts:<opts> ft:<typeidx> => (canon lift f opts type-index-space[ft])
           | 0x01 0x00 f:<funcidx> opts:<opts>                   => (canon lower f opts (core func))
           | 0x02 t:<typeidx>                                    => (canon resource.new t (core func))
           | 0x03 t:<valtype>                                    => (canon resource.drop t (core func))
           | 0x04 t:<typeidx>                                    => (canon resource.rep t (core func))
opts     ::= opt*:vec(<canonopt>)                                => opt*
canonopt ::= 0x00                                                => string-encoding=utf8
           | 0x01                                                => string-encoding=utf16
           | 0x02                                                => string-encoding=latin1+utf16
           | 0x03 m:<core:memidx>                                => (memory m)
           | 0x04 f:<core:funcidx>                               => (realloc f)
           | 0x05 f:<core:funcidx>                               => (post-return f)
```
注：
- `canon` 中的第二个 `0x00` 字节代表 `func` 类型，因此 `0x00 <u32>` 对应于 `func` 类型的 `sortidx` 或 `core:sortidx`。
- 验证防止重复或冲突的 `canonopt`。
- 对各个规范定义的验证在 [`CanonicalABI.md`](CanonicalABI.md#canonical-definitions) 中有描述。

## 启动定义
（请参见说明文档中的[启动定义](Explainer.md#start-definitions)。）
```
start ::= f:<funcidx> arg*:vec(<valueidx>) r:<u32> => (start f (value arg)* (result (value))ʳ)
```
注：
- 验证要求 `f` 具有 `functype`，其 `param` 个数和类型与 `arg*` 匹配，并且 `result` 的个数为 `r`。
- 验证将 `f` 的 `result` 类型附加到值索引空间中（使它们可供后续定义引用）。

除了上面提到的类型兼容性检查之外，值定义的验证规则还要求每个值恰好被使用一次。因此，在验证期间，每个值都有一个相关联的“已消耗”布尔标志。当一个值第一次添加到值索引空间中（通过 `import`、`instance`、`alias` 或 `start`）时，标志是未设置的。当一个值被使用时（通过 `export`、`instantiate` 或 `start`），标志被设置。在验证组件的最后一个定义之后，验证要求设置所有值的标志。


## 引入和导出定义
（请参见说明文档中的[导入和导出定义](Explainer.md#import-and-export-definitions)。）
```
import      ::= en:<externname> ed:<externdesc>                => (import en ed)
export      ::= en:<externname> si:<sortidx> ed?:<externdesc>? => (export en si ed?)
externname  ::= n:<name> ea:<externattrs>                      => n ea
externattrs ::= 0x00                                           => ϵ
              | 0x01 url:<URL>                                 => (id url)
URL         ::= b*:vec(byte)                                   => char(b)*, if char(b)* parses as a URL
```
注：
- 所有导出（所有 `sort` 的导出）都引入一个新的索引，该索引别名导出定义，并可以像别名一样由所有后续定义使用。
- 验证要求在导出类型的类型中传递使用的所有资源类型都是通过先前的 `importdecl` 或 `exportdecl` 引入的。
- “解析为 URL” 条件是通过使用 `char(b)*` 作为*输入*，没有可选参数和非致命验证错误（这与 JS 和 `rust-url` 中 `URL` 的定义重合）执行 [基本 URL 解析器] 来定义的。
- 验证要求任何导出的 `sortidx` 都具有有效的 `externdesc`（这不允许除 `core module` 之外的其他核心类型）。当存在可选的 `externdesc` 立即数时，验证要求它是 `sortidx` 推断的 `externdesc` 的超类型。
- `externname` 的 `name` 字段必须在包含组件定义、组件类型或实例类型的所有导入和导出中唯一。 （导入和导出不能使用相同的 `name`。）
- `externname` 的 `id` 字段（如果存在）必须分别在导入和导出中独立唯一。 （导入和导出*可能*具有相同的 `id`。）
- URL 的相等性是通过纯字节标识进行比较的。

## 名称部分
与核心 wasm 的 [名字段](https://webassembly.github.io/spec/core/appendix/custom.html#name-section) 类似，这里为组件指定了一个类似的 `name` 自定义段，以便能够为组件内可能发生的所有声明进行命名。类似于其核心 wasm 对应部分，此自定义段的有效性并非必需，并且引擎不应拒绝具有无效 `name` 段的组件。
```
namesec    ::= section_0(namedata)
namedata   ::= n:<name>                (if n = 'component-name')
               name:<componentnamesubsec>?
               sortnames*:<sortnamesubsec>*
namesubsection_N(B) ::= N:<byte> size:<u32> B     (if size == |B|)

componentnamesubsec ::= namesubsection_0(<name>)
sortnamesubsec ::= namesubsection_1(<sortnames>)
sortnames ::= sort:<sort> names:<namemap>

namemap ::= names:vec(<nameassoc>)
nameassoc ::= idx:<u32> name:<name>
```
这里的 `namemap` 与核心 wasm 中的相同。特定的 `sort` 应该仅在 `name` 段中出现一次，例如组件实例只能命名一次。


[`core:u32`]: https://webassembly.github.io/spec/core/binary/values.html#integers
[`core:section`]: https://webassembly.github.io/spec/core/binary/modules.html#binary-section
[`core:custom`]: https://webassembly.github.io/spec/core/binary/modules.html#custom-section
[`core:module`]: https://webassembly.github.io/spec/core/binary/modules.html#binary-module
[`core:version`]: https://webassembly.github.io/spec/core/binary/modules.html#binary-version
[`core:name`]: https://webassembly.github.io/spec/core/binary/values.html#binary-name
[`core:import`]: https://webassembly.github.io/spec/core/binary/modules.html#binary-import
[`core:importdesc`]: https://webassembly.github.io/spec/core/binary/modules.html#binary-importdesc
[`core:functype`]: https://webassembly.github.io/spec/core/binary/types.html#binary-functype

[type-imports]: https://github.com/WebAssembly/proposal-type-imports/blob/master/proposals/type-imports/Overview.md
[module-linking]: https://github.com/WebAssembly/module-linking/blob/main/proposals/module-linking/Explainer.md

[Basic URL Parser]: https://url.spec.whatwg.org/#concept-basic-url-parser
