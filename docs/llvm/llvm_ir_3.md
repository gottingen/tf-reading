优化器
===

# LLVM IR 优化概述

根据优化发生的时期，LLVM IR 层面的优化可以分为如下两种：

* 编译期优化。
* 链接期优化（跨编译单元）。

根据不同的优化范围，LLVM IR 层面的优化可以分为如下三种：

* 过程内优化。每次仅对一个函数进行优化。
* 过程间优化。每次对一个编译单元进行优化。
* 链接期优化。通过llvm-link工具将多个编译单元合并成一个，从而每次可以对多个编译单元进行优化。

我们可以通过opt工具启用这些优化。它既可以根据不同的优化等级选项启用不同的优化流水线（即一系列按照特定顺序执行的优化），也可以单独启用指定的优化。

# 如何启用优化

## 启用不同优化等级的优化
命令格式（输出 LLVM IR 位码文件）：
```shell
$ opt -O<level> <bitcode-file or assembly-file> -o <bitcode-file>
```
**注**：opt工具的输入既可以是 LLVM IR 位码文件，也可以是 LLVM IR 汇编文件。

命令格式（输出 LLVM IR 汇编文件）：
```shell
$ opt -O<level> <bitcode-file or assembly-file> -S -o <assembly-file>
```
**需要注意的是**， 无论启用的优化等级是什么，都不会对带optnone属性的函数生效。

在 `LLVM 12.0.0` 版本中，opt工具支持的优化等级选项有：`-O0`、`-O1`、`-O2`、`-O3`、`-Os`、`-Oz`，如下表所示。

| 选项 | 行为 |
| :----:| :----: |
| -O0 | 不优化|
|-O1 | 介于-O0 和 -O2之间|
| -O2 | 启用大多数优化|
| -O3 | 与 -O2 类似，并启用试图使程序运行得更快的优化（代码大小可能会增加）|
|-Os| 与 -O2 类似，并启用减少代码大小的优化|
| -Oz | 与 -O2、-Os 类似，并启用进一步减少代码大小的优化|

示例：

```shell
$ opt -O1 test_without_optnone.ll -o test_without_optnone_O1_opt.bc
```
**注**： 通过手工删除后，test_without_optnone.ll 汇编文件中的函数都不带optnone属性。从而，可以验证-O1选项的优化效果。

## 启用基于传统 Pass 框架的优化

### 启用 opt 工具自带的基于传统 Pass 框架的优化
命令格式：
```shell
$ opt -<pass1-name> -<pass2-name> <bitcode-file or assembly-file> -o <bitcode-file>
```
**需要注意的是**， opt工具自带的基于传统 Pass 框架的优化，都不会对带optnone属性的函数生效。

示例：
```shell
$ opt -mem2reg test_without_optnone.ll -o test_without_optnone_mem2reg_opt.bc
```

### 启用自定义的基于传统 Pass 框架的优化

命令格式（输出优化后的位码文件）：
```shell
$ opt -load <pass-shared-library> -<pass1-name> -<pass2-name> <bitcode-file or assembly-file> -o <bitcode-file>
```

命令格式（不输出优化后的位码文件）：
```shell
$ opt -load <pass-shared-library> -<pass1-name> -<pass2-name> <bitcode-file or assembly-file> > /dev/null
```

或

```shell
$ opt -disable-output -load <pass-shared-library> -<pass1-name> -<pass2-name> <bitcode-file or assembly-file>
```

*注*：对于不输出优化后的位码文件的情况，上述命令中添加> `/dev/null`重定向或者`-disable-output`选项都是为了避免以下内容的输出：

```text
WARNING: You're attempting to print out a bitcode file.
This is inadvisable as it may cause display problems. If
you REALLY want to taste LLVM bitcode first-hand, you
can force output with the `-f' option.
```

需要注意的是， 自定义的基于传统 Pass 框架的优化，对带`optnone`属性的函数是生效的。

示例：

```shell
$ opt -disable-output -load ~/git-projects/llvm-project/build_ninja/lib/LLVMFnArgCntPlugin.so -plugin.fnargcnt test_with_optnone.ll
```
**注**：自定义的基于传统 Pass 框架的优化`plugin.fnargcnt`用于统计函数的参数个数。

输出结果如下：
```shell
FnArgCntPass --- foo: 1
```

启用基于新 Pass 框架的优化

命令格式（不输出优化后的位码文件）：
```shell
$ opt -disable-output -passes=<pass1-name,pass2-name> <bitcode-file or assembly-file>
```
需要注意的是， 基于新 Pass 框架的优化，无论是opt工具自带的还是自定义的，都不会对带optnone属性的函数生效。

示例：

```shell
$ opt -disable-output -passes=fnargcnt test_without_optnone.ll
```

注：自定义的基于新 Pass 框架的优化fnargcnt用于统计函数的参数个数。

输出结果如下：

```shell
FnArgCntPass --- foo: 1
```

## 查看优化的统计信息

使用 clang 驱动器时，查看优化的统计信息

命令格式（示例）：
```shell
$ clang -Xclang -print-stats -emit-llvm -O<level> <source-file> -c -o <bitcode-file>
```

注：上述命令的输出结果中除了优化的统计信息之外，还有其他的统计信息。

示例：
```shell
$ clang -Xclang -print-stats -emit-llvm -O1 test.c -c -o test_O1_clang.bc
```
输出结果如下（部分）：
```text
省略 ...

===-------------------------------------------------------------------------===
                          ... Statistics Collected ...
===-------------------------------------------------------------------------===

15 assume-queries   - Number of Queries into an assume assume bundles
 1 cgscc-passmgr    - Maximum CGSCCPassMgr iterations on one SCC
 6 file-search      - Number of directory cache misses.
 省略 ...
 1 loop-delete      - Number of loops deleted
 1 loop-rotate      - Number of loops rotated
 1 loop-unswitch    - Total number of instructions analyzed
 3 mem2reg          - Number of PHI nodes inserted
 1 reassociate      - Number of insts reassociated
 2 scalar-evolution - Number of loops with predictable loop counts
 7 simplifycfg      - Number of blocks simplified
 1 sroa             - Maximum number of partitions per alloca
 8 sroa             - Maximum number of uses of a partition
14 sroa             - Number of alloca partition uses rewritten
 2 sroa             - Number of alloca partitions formed
 2 sroa             - Number of allocas analyzed for replacement
16 sroa             - Number of instructions deleted
 2 sroa             - Number of allocas promoted to SSA values
 ```

使用 opt 工具时，查看优化的统计信息

命令格式（示例）：

```shell
$ opt -stats -<pass-name> <bitcode-file or assembly-file> -o <bitcode-file>
```

示例：
```shell
$ opt -stats -mem2reg test_without_optnone.ll -o test_without_optnone_mem2reg_opt.bc
```

输出结果如下：
```shell
===-------------------------------------------------------------------------===
                          ... Statistics Collected ...
===-------------------------------------------------------------------------===

3 mem2reg - Number of PHI nodes inserted
2 mem2reg - Number of alloca's promoted
```
## 查看实际启用的优化及依赖关系

命令格式（示例）：

```shell
$ opt -debug-pass=Structure -<pass-name> <bitcode-file or assembly-file> -o <bitcode-file>
```

```shell
$ opt -debug-pass=Structure -mem2reg test_without_optnone.ll -o test_without_optnone_O1_opt.bc
```

输出结果如下：
```shell
Pass Arguments:  -targetlibinfo -tti -targetpassconfig -assumption-cache-tracker -domtree -mem2reg -verify -write-bitcode
Target Library Information
Target Transform Information
Target Pass Configuration
Assumption Cache Tracker
  ModulePass Manager
    FunctionPass Manager
      Dominator Tree Construction
      Promote Memory to Register
      Module Verifier
    Bitcode Writer
```

注：
* 实际启用的优化在Pass Arguments中列出。比如：targetlibinfo、tti、domtree、mem2reg等。
* 在 FunctionPass Manager 中，优化的执行先后顺序依次为：Dominator Tree Construction、Promote Memory to Register。这表明，优化mem2reg依赖于domtree。

## 查看优化的执行时间

使用 clang 驱动器时，查看优化的执行时间

命令格式（示例）：
```text
$ clang -Xclang -ftime-report -emit-llvm -O<level> <source-file> -c -o <bitcode-file>
```
注：上述命令的输出结果中除了优化的执行时间之外，还有其他的执行时间。
示例：

```shell
$ clang -Xclang -ftime-report -emit-llvm -O1 test.c -c -o test_O1_clang.bc
```

输出结果如下（部分）：
```shell
===-------------------------------------------------------------------------===
                         Miscellaneous Ungrouped Timers
===-------------------------------------------------------------------------===

   ---User Time---   --User+System--   ---Wall Time---  --- Name ---
   0.0687 ( 92.3%)   0.0687 ( 92.3%)   0.0691 ( 92.2%)  Code Generation Time
   0.0058 (  7.7%)   0.0058 (  7.7%)   0.0059 (  7.8%)  LLVM IR Generation Time
   0.0745 (100.0%)   0.0745 (100.0%)   0.0750 (100.0%)  Total

===-------------------------------------------------------------------------===
                      ... Pass execution timing report ...
===-------------------------------------------------------------------------===
  Total Execution Time: 0.0453 seconds (0.0455 wall clock)

   ---User Time---   --User+System--   ---Wall Time---  --- Name ---
   0.0049 ( 10.9%)   0.0049 ( 10.9%)   0.0049 ( 10.8%)  Simplify the CFG
   0.0043 (  9.6%)   0.0043 (  9.6%)   0.0043 (  9.5%)  Induction Variable Simplification
   省略 ...
   0.0003 (  0.7%)   0.0003 (  0.7%)   0.0003 (  0.7%)  Memory SSA
   省略 ...
   0.0000 (  0.0%)   0.0000 (  0.0%)   0.0000 (  0.0%)  Unroll loops
   0.0453 (100.0%)   0.0453 (100.0%)   0.0455 (100.0%)  Total

===-------------------------------------------------------------------------===
                          Clang front-end time report
===-------------------------------------------------------------------------===
  Total Execution Time: 0.0844 seconds (0.0848 wall clock)

   ---User Time---   --System Time--   --User+System--   ---Wall Time---  --- Name ---
   0.0803 (100.0%)   0.0041 (100.0%)   0.0844 (100.0%)   0.0848 (100.0%)  Clang front-end timer
   0.0803 (100.0%)   0.0041 (100.0%)   0.0844 (100.0%)   0.0848 (100.0%)  Total
```


使用 opt 工具时，查看优化的执行时间

命令格式（示例）：
```shell
$ opt -time-passes -O<level> <bitcode-file or assembly-file> -o <bitcode-file>
```

示例：
```shell
$ opt -time-passes -O1 test_without_optnone.ll -o test_without_optnone_O1_opt.bc
```

输出结果如下（部分）：

```shell
===-------------------------------------------------------------------------===
                      ... Pass execution timing report ...
===-------------------------------------------------------------------------===
  Total Execution Time: 0.0404 seconds (0.0405 wall clock)

   ---User Time---   --System Time--   --User+System--   ---Wall Time---  --- Name ---
   0.0050 ( 12.4%)   0.0001 ( 73.5%)   0.0051 ( 12.7%)   0.0051 ( 12.7%)  Simplify the CFG
   0.0035 (  8.7%)   0.0000 (  0.0%)   0.0035 (  8.7%)   0.0035 (  8.7%)  Induction Variable Simplification
   省略 ...
   0.0003 (  0.9%)   0.0000 (  0.0%)   0.0003 (  0.8%)   0.0003 (  0.8%)  Memory SSA
   省略 ...
   0.0000 (  0.0%)   0.0000 (  0.0%)   0.0000 (  0.0%)   0.0000 (  0.0%)  Unroll loops
   0.0000 (  0.0%)   0.0000 (  0.0%)   0.0000 (  0.0%)   0.0000 (  0.0%)  Scoped NoAlias Alias Analysis
   0.0000 (  0.0%)   0.0000 (  0.0%)   0.0000 (  0.0%)   0.0000 (  0.0%)  Target Transform Information
   0.0000 (  0.0%)   0.0000 (  0.0%)   0.0000 (  0.0%)   0.0000 (  0.0%)  Profile summary info
   0.0403 (100.0%)   0.0002 (100.0%)   0.0404 (100.0%)   0.0405 (100.0%)  Total

===-------------------------------------------------------------------------===
                                LLVM IR Parsing
===-------------------------------------------------------------------------===
  Total Execution Time: 0.0045 seconds (0.0045 wall clock)

   ---User Time---   --System Time--   --User+System--   ---Wall Time---  --- Name ---
   0.0044 (100.0%)   0.0001 (100.0%)   0.0045 (100.0%)   0.0045 (100.0%)  Parse IR
   0.0044 (100.0%)   0.0001 (100.0%)   0.0045 (100.0%)   0.0045 (100.0%)  Total
```

