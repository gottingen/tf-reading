what is llvm ir
===

# 概述

LLVM IR（Intermediate Representation）是一种中间语言表示，作为编译器前端和后端的分水岭。LLVM 编译器的前端——Clang 负责产生 IR，而其后端负责消费 IR。

编译器 IR 的设计体现了权衡的计算思维。低级的 IR（即更接近目标代码的 IR）允许编译器更容易地生成针对特定硬件的优化代码，但不利于支持多目标代码的生成。高级的 IR 允许优化器更容易地提取源代码的意图，但不利于编译器根据不同的硬件特性进行代码优化。

LLVM IR 的设计采用common IR和specific IR相结合的方式。common IR旨在不同的后端共享对源程序的相同理解，以将其转换为不同的目标代码。除此之外，也为多个后端之间共享一组与目标无关的优化提供了可能性。specific IR允许不同的后端在不同的较低级别优化目标代码。这样做，既可以支持多目标代码的生成，也兼顾了目标代码的执行效率。

LLVM IR 有如下 3 种等价形式：

* 内存表示
  * 类`llvm::Function`、`llvm::Instruction`等用于表示`common IR`。
  * 类`llvm::MachineFunction`、`llvm::MachineInstr`等用于表示`specific IR`。

* 位码文件（Bitcode Files，存储在磁盘中）
* 汇编文件（Assembly Files，存储在磁盘中，便于人类可读）

*注*：这里的汇编文件不是通常所说的`汇编语言`文件，而是 LLVM 位码文件的可读表示。

LLVM IR 的特点如下：
* 采用静态单一赋值（Static Single Assignment，SSA），即每个值只有一个定义它的赋值操作
* 代码被组织为三地址指令（Three-address Instructions）
* 有无限多个寄存器

# LLVM IR 的相关命令

LLVM IR 的相关命令如下：
| 命令格式 | 作用 | 示例 |
| :----: | :----: | :----: |
| `clang <source-file or assembly-file> -emit-llvm -c -o <output-file>` | 生成 LLVM IR 的位码文件 | <ul><li>clang test.c -emit-llvm -c -o test.bc</li><li>clang test.ll -emit-llvm -c -o test.bc</li><ul> |
| `clang <source-file or bitcode-file> -emit-llvm -S -c -o <output-file>` | 生成 LLVM IR 的汇编文件 | <ul><li>clang test.c -emit-llvm -S -c -o test.ll</li><li>clang test.bc -emit-llvm -S -c -o test.ll</li></ul> |
| `llvm-as <assembly-file> -o <output-file>` | 生成 LLVM IR 的位码文件 | llvm-as test.ll -o test.bc |
| `llvm-dis <assembly-file> -o <output-file>` | 生成 LLVM IR 的汇编文件 |llvm-dis test.bc -o test.ll |
| `llvm-extract -func=foo <assembly-file or bitcode-file> -o <output-file>` | 提取指定的函数到位码文件 | <ul><li>llvm-extract -func=foo test.ll -o test-fn.bc</li><li>llvm-extract -func=foo test.bc -o test-fn.bc</li></ul> |


# LLVM IR 的语法

## Source Filename

`source_filename`描述了源文件的名称及其所在路径。
语法：

```python 
source_filename = "/path/to/source.c"
```
示例 1：

```pthon
source_filename = "test.c"
```
*注*:上述内容是通过运行下面命令得到。
```shell
clang test.c -emit-llvm -S -c -o test.ll
```
示例 2：
```python
source_filename = "../test.c"
```
*注*:上述内容是通过运行下面命令得到。
```shell
clang ../test.c -emit-llvm -S -c -o test.ll
```
从上面两个示例的结果可以看出，source_filename的值就是包含clang 输入的源文件（原封不动）的字符串。

注：相关的官方文档—— [Source Filename](https://llvm.org/docs/LangRef.html#source-filename)。

## Data Layout

`target datalayout`描述了目标机器中数据的内存布局方式，包括：字节序、类型大小以及对齐方式。

语法：

```pyhton
target datalayout = "layout specification"
```

示例（in Ubuntu 20.04）：

```python
target datalayout = "e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128"
```
*注*：

* 第一个字母表示目标机器的字节序。可选项：小写字母`e`（小端字节序）、大写字母E（大端字节序）。
* `m:<mangling>`，指定输出结果中的名称重编风格。可选项：`e`（ELF 重编风格，即私有符号以 .L 为前缀）、`m`（Mips 重编风格，即私有符号以 $ 为前缀）、`o`（ Mach-O 重编风格，即私有符号以 L 为前缀，其他符号以 _ 为前缀）`x`（ Windows x86 COFF 重编风格）等。
* `p[n]:<size>:<abi>:<pref>:<idx>`，指定在某个地址空间中指针的大小及其对齐方式（单位：比特）。p270:32:32，等价于 p270:32:32:32:32，表示在地址空间 270 中指针的大小为 4 字节并且以 4 字节进行对齐。p272:64:64，等价于 p272:64:64:64:64，表示在地址空间 272 中指针的大小为 8 字节并且以 8 字节进行对齐。
* `i<size>:<abi>:<pref>`，指定整数的对齐方式（单位：比特）。i64:64，等价于 i64:64:64，表示 i64 整数（即占用 8 字节的整数）以 8 字节进行对齐。
* `f<size>:<abi>:<pref>`，指定浮点数的对齐方式（单位：比特）。f80:128，等价于 f80:128:128，表示 f80 浮点数（即占用 80-bit 的浮点数）以 16 字节进行对齐。注：所有的目标机器都支持 float（即占用 32-bit 的浮点数）和 double（即占用 64-bit 的浮点数）。
* `n<size1>:<size2>:<size3>...`，指定目标处理器原生支持的整数宽度（单位：比特）。n8:16:32:64，表示目标处理器原生支持的整数宽度为 1 字节、2 字节、4 字节和 8 字节。
* `S<size>`，指定栈的对齐方式（单位：比特）。S128，表示栈以 16 字节进行对齐。特殊的，S0，表示未指定栈的对齐方式。

*需要注意的是*， `<abi>`对齐方式指定了类型所需的最小对齐方式，而`<pref>`对齐方式指定了一个可能更大的值。`<pref>`可以省略，省略时其值等于`<abi>`的值。

：相关的官方文档——[Data Layout](https://llvm.org/docs/LangRef.html#data-layout) 。

## Target Triple

`target triple`描述了目标机器是什么，从而指示后端生成相应的目标代码。

*注*：可以通过命令选项`-mtriple`覆盖该信息。

语法（典型的）：
```python
target triple = "ARCHITECTURE-VENDOR-OPERATING_SYSTEM"
```

或

```python
target triple = "ARCHITECTURE-VENDOR-OPERATING_SYSTEM-ENVIRONMENT"
```

示例（in Ubuntu 20.04）：

```python
target triple = "x86_64-unknown-linux-gnu"
```
注：上述内容表示目标机器的指令集是`x86_64`，供应商未知，操作系统是`linux`，环境是`GNU`。

*注*：相关的官方文档—— [Target Triple](https://llvm.org/docs/LangRef.html#target-triple)。

## Identifiers

LLVM IR 中的标识符分为：全局标识符和局部标识符。全局标识符以`@`开头，比如：全局函数、全局变量。局部标识符以`%`开头，类似于汇编语言中的寄存器。

标识符有如下 3 种形式：

* 有名称的值（Named Value），表示为带有前缀（`@`或`%`）的字符串。比如：%val、@name。
* 无名称的值（Unnamed Value），表示为带前缀（`@`或`%`）的无符号数值。比如：%0、%1、@2。
* 常量。

注：相关的官方文档—— [Identifiers](https://llvm.org/docs/LangRef.html#identifiers)。

## Functions

`define`用于定义一个函数。

语法：
```python
define [linkage] [PreemptionSpecifier] [visibility] [DLLStorageClass]
       [cconv] [ret attrs]
       <ResultType> @<FunctionName> ([argument list])
       [(unnamed_addr|local_unnamed_addr)] [AddrSpace] [fn Attrs]
       [section "name"] [comdat [($name)]] [align N] [gc] [prefix Constant]
       [prologue Constant] [personality Constant] (!name !N)* { ... }
```

示例（in Ubuntu 20.04）：

```python
define dso_local void @foo(i32 %x) #0 {
  ; 省略 ...
}
```
注：

* `define void @foo(i32 %x) { ... }`，表示定义一个函数。其函数名称为`foo`，返回值的数据类型为`void`，参数（用`%x`表示）的数据类型为 `i32`（占用 4 字节的整型）。
* `#0`，用于修饰函数时表示一组函数属性。这些属性定义在文件末尾。如下：
```python
attributes #0 = { noinline nounwind optnone uwtable "disable-tail-calls"="false" "frame-pointer"="all" "less-precise-fpmad"="false" "min-legal-vector-width"="0" "no-infs-fp-math"="false" "no-jump-tables"="false" "no-nans-fp-math"="false" "no-signed-zeros-fp-math"="false" "no-trapping-math"="true" "stack-protector-buffer-size"="8" "target-cpu"="x86-64" "target-features"="+cx8,+fxsr,+mmx,+sse,+sse2,+x87" "tune-cpu"="generic" "unsafe-fp-math"="false" "use-soft-float"="false" }
```

LLVM IR 中，函数体是由基本块（Basic Blocks）构成的。基本块是由一系列顺序执行的语句构成的，并（可选地）以标签作为起始。不同的标签代表不同的基本块。

基本块的特点如下：

* 仅有一个入口，即基本块中的第一条指令。
* 仅有一个出口，即基本块中的最后一条指令（被称为terminator instruction）。该指令要么跳转到其他基本块（不包括入口基本块），要么从函数返回。
* 函数体中第一个出现的基本块，称为入口基本块（Entry Basic Block）。它是一个特殊的基本块，在进入函数时立即执行该基本块，并且不允许作为其他基本块的跳转目标（即不允许该基本块有前继节点）。

一个完整的函数实现如下：
```
 7 define weak dso_local void @foo(i32 %x) #0 {
  8 entry:
  9   %x.addr = alloca i32, align 4
 10   %y = alloca i32, align 4
 11   %z = alloca i32, align 4
 12   store i32 %x, i32* %x.addr, align 4
 13   %0 = load i32, i32* %x.addr, align 4
 14   %cmp = icmp eq i32 %0, 0
 15   br i1 %cmp, label %if.then, label %if.end
 16 
 17 if.then:                                          ; preds = %entry
 18   store i32 5, i32* %y, align 4
 19   br label %if.end
 20 
 21 if.end:                                           ; preds = %if.then, %entry
 22   %1 = load i32, i32* %x.addr, align 4
 23   %tobool = icmp ne i32 %1, 0
 24   br i1 %tobool, label %if.end2, label %if.then1
 25 
 26 if.then1:                                         ; preds = %if.end
 27   store i32 6, i32* %z, align 4
 28   br label %if.end2
 29 
 30 if.end2:                                          ; preds = %if.then1, %if.end
 31   ret void
 32 }
 ```


注：

* 第 8~16 行，是入口基本块，标签为`entry`。
  * `%x.addr = alloca i32, align 4`，表示在栈上分配一块大小为 4 字节（由于要分配的数据类型为`i32`）的内存，并以 4 字节进行对齐（意味着所分配内存的起始地址是 4 的整数倍），该内存的起始地址保存到局部标识符`%x.addr`中。
  * `store i32 %x, i32* %x.addr, align 4`，表示将参数`x`（由局部标识符%x表示）的值保存到局部标识符`%x.addr`所指向的栈内存中。
  * `%0 = load i32, i32* %x.addr, align 4`，表示将局部标识符`%x.addr`所指向的栈内存中的值保存到局部标识符`%0`中。
  * `%cmp = icmp eq i32 %0, 0`，表示判断局部标识符`%0`的值是否等于 0（对应语句`x == 0`），并将判断结果保存到局部标识符`%cmp`中。如果相等，则返回`true`；否则，返回`false`。
  * `br i1 %cmp, label %if.then, label %if.end`，表示如果局部标识符`%cmp`的值为`true`，则跳转到标签为`%if.then`的基本块；否则，跳转到标签为`%if.end`的基本块。（有条件跳转）
* 第 17~19 行，是一个标签为`if.then`的基本块。
  * `store i32 5, i32* %y, align 4`，表示将立即数 5 保存到局部标识符`%y`所指向的栈内存中（对应语句`y = 5`;）。
  * `br label %if.end`，表示直接跳转到标签为`%if.end`的基本块。（无条件跳转）
* 第 21~24 行，是一个标签为`if.end`的基本块。
  * `%1 = load i32, i32* %x.addr, align 4`，表示将局部标识符`%x.addr`所指向的栈内存中的值保存到局部标识符`%1`中。
  * `%tobool = icmp ne i32 %1, 0`，表示判断局部标识符%1的值是否不等于 0（对应语句`!x`），并将判断结果保存到局部标识符`%tobool`中。如果不相等，则返回`true`；否则，返回`false`。
  * `br i1 %tobool, label %if.end2, label %if.then1`，表示如果局部标识符`%tobool`的值为`true`（即如果参数x的值为`true`），则跳转到标签为`%if.end2`的基本块；否则，跳转到标签为`%if.then1`的基本块。
* 第 26~28 行，是一个标签为`if.then1`的基本块。
  * `store i32 6, i32* %z, align 4`，表示将立即数 6 保存到局部标识符%z所指向的栈内存中（对应语句`z = 6`;）。
  * `br label %if.end2`，表示直接跳转到标签为`%if.end2`的基本块。
* 第 30~31 行，是一个标签为`if.end2`的基本块。
  * `ret void`，表示从函数返回，并且无函数返回值。

对应的源码如下:

```c++
void foo(int x) {
  int y, z;
  if (x == 0)
    y = 5;
  if (!x)
    z = 6;
}
```

注：相关的官方文档——[Functions](https://llvm.org/docs/LangRef.html#functions)、[Attribute Groups](https://llvm.org/docs/LangRef.html#attribute-groups) 、[Runtime Preemption Specifiers](https://llvm.org/docs/LangRef.html#runtime-preemption-specifiers)、[‘alloca’ Instruction](https://llvm.org/docs/LangRef.html#alloca-instruction)、[‘store’ Instruction](https://llvm.org/docs/LangRef.html#store-instruction)、[‘load’ Instruction](https://llvm.org/docs/LangRef.html#load-instruction)、[‘icmp’ Instruction](https://llvm.org/docs/LangRef.html#icmp-instruction)、[‘br’ Instruction](https://llvm.org/docs/LangRef.html#br-instruction)、[‘ret’ Instruction](https://llvm.org/docs/LangRef.html#ret-instruction)。

## Function Attributes

常见的函数属性如下：

| 属性名称 | 作用 |
| :----: | :----: |
|noinline | 表示在任何情况下都不能将函数视为内联函数进行处理|
| nounwind | 表示函数从不引发异常（如果抛出了异常，则为运行期未定义行为）|
| optnone | 表示大多数优化过程将跳过此函数 |
| uwtable | 该选项常见于 ELF x86-64 abi，具体作用？ |
| mustprogress | |

注：相关的官方文档——[Function Attributes](https://llvm.org/docs/LangRef.html#function-attributes) 、[\[IR\] Adds mustprogress as a LLVM IR attribute](https://reviews.llvm.org/D85393#:~:text=Summary%20This%20adds%20the%20LLVM%20IR%20attribute%20mustprogress,this%20attribute%20are%20not%20required%20to%20make%20progress.)。

## Metadata

注：相关的官方文档——[Metadata](https://llvm.org/docs/LangRef.html#metadata) 。

## llvm.loop

`llvm.loop`是元数据（Metadata）之一，用于为循环附加一些属性。这些属性将传递给优化器和代码生成器。

示例：
```asm
for.inc:                                          ; preds = %if.end
  %6 = load i32, i32* %i, align 4
  %inc = add nsw i32 %6, 1
  store i32 %inc, i32* %i, align 4
  br label %for.cond, !llvm.loop !2

; 省略 ...

!2 = distinct !{!2, !3}
!3 = !{!"llvm.loop.mustprogress"}
```

所有的元数据都以英文叹号`!`开头。`!llvm.loop !2`，表示循环元数据节点为!2，它是一系列其他元数据的集合，集合中的每项都表示循环的一个属性。上述示例中，该集合为`!2 = distinct !{!2, !3}`。由于历史遗留原因，该集合的第一项必须是对自身的引用。上述示例中，该循环元数据节点携带的属性只有`llvm.loop.mustprogress`。该属性表示：如果循环不与外部发生交互（可以理解为：移除该循环后也不影响程序的行为），那么循环可能会被移除。

注：相关的官方文档——[‘llvm.loop’](https://llvm.org/docs/LangRef.html#llvm-loop) 、[‘llvm.loop.mustprogress’ Metadata](https://llvm.org/docs/LangRef.html#llvm-loop-mustprogress-metadata)。

## Module Flags Metadata

注：相关的官方文档——[Module Flags Metadata](https://llvm.org/docs/LangRef.html#module-flags-metadata) 。

## llvm.module.flags

`llvm.module.flags`是命名元数据（`Named Metadata`）之一，用于描述模块级别的信息。

`llvm.module.flags`是一个包含一系列元数据节点的集合。集合中的每个元数据节点都是一个形如`{<behavior>, <key>, <value>}`的三元组。其中，`<behavior>`用于指定来自不同模块的`<key>`和`<value>`都相同的元数据合并时如何处理，`<key>`表示元数据在本模块中的唯一标识符（仅在本模块内唯一，来自不同模块的元数据可能有相同的标识符），`<value>`表示元数据所携带信息的值。

常见的`<behavior>`值及行为如下：

| behavior 值 | 行为 |
| :----: | :----: |
| 1 | 如果两个值不一致，则发出错误，否则生成的值就是操作数的值 |

示例：
```asm
!llvm.module.flags = !{!0}

; 省略 ...

!0 = !{i32 1, !"wchar_size", i32 4}
```
上述示例中，`!llvm.module.flags = !{!0}`表示集合中只有一个元数据节点`!0`。`!0 = !{i32 1, !"wchar_size", i32 4}`表示标识符为`"wchar_size"`的元数据所携带信息的值在所有模块中都应该是 4，否则会报错。

## llvm.ident

llvm.ident，用于表示 Clang 的版本信息。

示例：

```asm
!llvm.ident = !{!1}

; 省略 ...

!1 = !{!"clang version 10.0.0-4ubuntu1 "}
```

# References

* [Getting Started with LLVM Core Libraries](http://faculty.sist.shanghaitech.edu.cn/faculty/songfu/course/spring2018/CS131/llvm.pdf)
* [LLVM Language Reference Manual](https://llvm.org/docs/LangRef.html)

