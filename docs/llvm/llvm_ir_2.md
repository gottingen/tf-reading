
# 摘要
本文基于release/12.x版本的 LLVM 源码，通过示例展示了如何利用 LLVM API 编写一个简单的生成 LLVM IR 的独立程序。从而，初步了解 LLVM IR 的内存表示形式以便更深入地研究相关内容。

# 示例

本节介绍了所研究示例的源码以及通过相关命令生成的 LLVM IR 汇编文件。

*step 1：* 编写示例程序

test.cpp：

```c++
void foo(int x) {
  for (int i = 0; i < 10; i++) {
    if (x % 2 == 0) {
      x += i;
    }
    else {
      x -= i;
    }
  }
}
```

*step 2：* 生成 LLVM IR 汇编文件

```shell
$ clang test.cpp -emit-llvm -c -o test.bc
$ clang test.bc -emit-llvm -S -c -o test.ll
```

**注**：如果通过执行`clang test.cpp -emit-llvm -S -c -o test.ll`生成 `LLVM IR `汇编文件，那么`ModuleID`的值将为`test.cpp`。

生成的汇编文件——test.ll 的内容如下（部分）：

```asm
; ModuleID = 'test.bc'
source_filename = "test.cpp"
target datalayout = "e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128"
target triple = "x86_64-unknown-linux-gnu"

; Function Attrs: noinline nounwind optnone uwtable mustprogress
define dso_local void @_Z3fooi(i32 %x) #0 {
entry:
  %x.addr = alloca i32, align 4
  %i = alloca i32, align 4
  store i32 %x, i32* %x.addr, align 4
  store i32 0, i32* %i, align 4
  br label %for.cond

for.cond:                                         ; preds = %for.inc, %entry
  %0 = load i32, i32* %i, align 4
  %cmp = icmp slt i32 %0, 10
  br i1 %cmp, label %for.body, label %for.end

for.body:                                         ; preds = %for.cond
  %1 = load i32, i32* %x.addr, align 4
  %rem = srem i32 %1, 2
  %cmp1 = icmp eq i32 %rem, 0
  br i1 %cmp1, label %if.then, label %if.else

if.then:                                          ; preds = %for.body
  %2 = load i32, i32* %i, align 4
  %3 = load i32, i32* %x.addr, align 4
  %add = add nsw i32 %3, %2
  store i32 %add, i32* %x.addr, align 4
  br label %if.end

if.else:                                          ; preds = %for.body
  %4 = load i32, i32* %i, align 4
  %5 = load i32, i32* %x.addr, align 4
  %sub = sub nsw i32 %5, %4
  store i32 %sub, i32* %x.addr, align 4
  br label %if.end

if.end:                                           ; preds = %if.else, %if.then
  br label %for.inc

for.inc:                                          ; preds = %if.end
  %6 = load i32, i32* %i, align 4
  %inc = add nsw i32 %6, 1
  store i32 %inc, i32* %i, align 4
  br label %for.cond, !llvm.loop !2

for.end:                                          ; preds = %for.cond
  ret void
}

attributes #0 = { noinline nounwind optnone uwtable mustprogress "disable-tail-calls"="false" "frame-pointer"="all" "less-precise-fpmad"="false" "min-legal-vector-width"="0" "no-infs-fp-math"="false" "no-jump-tables"="false" "no-nans-fp-math"="false" "no-signed-zeros-fp-math"="false" "no-trapping-math"="true" "stack-protector-buffer-size"="8" "target-cpu"="x86-64" "target-features"="+cx8,+fxsr,+mmx,+sse,+sse2,+x87" "tune-cpu"="generic" "unsafe-fp-math"="false" "use-soft-float"="false" }

!llvm.module.flags = !{!0}
!llvm.ident = !{!1}

!0 = !{i32 1, !"wchar_size", i32 4}
!1 = !{!"clang version 12.0.0 (...)"}
!2 = distinct !{!2, !3}
!3 = !{!"llvm.loop.mustprogress"}
```
**注**： `LLVM IR` 汇编文件中的行注释以英文分号;开头，并且仅支持行注释（即无块注释）。

# 编写生成 LLVM IR 的工具

本节介绍了如何利用 LLVM API 编写生成上一节中 LLVM IR 汇编文件的程序，包括：创建模块、创建函数、设置运行时抢占符、设置函数属性、设置函数参数名称、创建基本块、分配栈内存、内存写入、无条件跳转、比较大小、有条件跳转、二元运算（加法）、函数返回 等。

## step 1： 创建模块

模块（Module）是 LLVM IR 中最顶层的实体，包含了所有其他 IR 对象（比如：函数、基本块等）。LLVM IR 程序是由模块组成的，每个模块对应输入程序的一个编译单元。

`创建模块`的代码如下：
```c++
LLVMContext Ctx;
std::unique_ptr<Module> M(new Module("test.cpp", Ctx));
```

上述代码的逻辑为：创建一个模块 ID 为`"test.cpp"`的模块对象`M`。该对象的生命周期通过智能指针`std::unique_ptr`进行管理。

`LLVM IR` 汇编文件首行的注释中包含了模块 ID，示例：

```asm
; ModuleID = 'test.cpp'
```
**注意：**
* 类`llvm::LLVMContext`不是线程安全的。在多线程环境下，不同的线程应该各自持有一份`llvm::LLVMContext`类的实例。
* 一个`llvm::Module`类的对象在进行任何操作之前都必须确保其构造函数中所传入的`llvm::LLVMContext`对象是有效的（引用传递）。

## step 2： 创建函数

`创建函数`的代码如下：
```c++
SmallVector<Type *, 1> FuncTyAgrs;
FuncTyAgrs.push_back(Type::getInt32Ty(Ctx));
auto *FuncTy = FunctionType::get(Type::getVoidTy(Ctx), FuncTyAgrs, false);
auto *FuncFoo =
    Function::Create(FuncTy, Function::ExternalLinkage, "_Z3fooi", M.get());
    ```

上述代码的逻辑为：创建一个函数原型为`void _Z3fooi(int)`的全局函数，由类`llvm::FunctionType`的对象表示（这里是`FuncFoo`）。
```
*注*：
* 构造向量`FuncTyAgrs`时，其初始元素数量应该等于要创建函数的参数个数。从而，避免不必要的性能开销。

* 函数`Type::getXXXTy()`用于获取表示数据类型XXX的对象。

* 函数`FunctionType::get()`的第三个参数表示是否带可变参数。如果值为false，表示该函数没有可变参数...。

* 函数`Function::Createt()`的第二个参数用于指定函数的链接属性。如果值为`Function::ExternalLinkage`，表示该函数是全局函数。

**需要注意的是**， 由于 `C++` 程序会进行名称重编。因此，函数`foo`的真正名称为重编后的名称（这里是`_Z3fooi`）。

## step 3： 设置运行时抢占符

`设置运行时抢占符`的代码如下：

```c++
FuncFoo->setDSOLocal(true);
```
上述代码的逻辑为：将`FuncFoo`所表示函数的运行时抢占符设置为`dso_local`。

## step 4： 设置函数属性

`设置函数属性（数值形式）`的代码如下：
```c++
FuncFoo->addFnAttr(Attribute::NoInline);
FuncFoo->addFnAttr(Attribute::NoUnwind);
```
上述代码的逻辑为：为FuncFoo所表示的函数添加属性：noinline、nounwind。

*注*：这些属性定义在`llvm/IR/Attributes.inc`文件中。该文件是构建 `Clang` 时生成的，位于构建目录的子目录`include`中，比如：`build_ninja/include/llvm/IR/Attributes.inc`。

`设置函数属性（字符串形式）`的代码如下：
```c++
AttrBuilder FuncAttrs;
FuncAttrs.addAttribute("disable-tail-calls", llvm::toStringRef(false));
FuncAttrs.addAttribute("frame-pointer", "all");
FuncFoo->addAttributes(AttributeList::FunctionIndex, FuncAttrs);
```

上述代码的逻辑为：为`FuncFoo`所表示的函数添加属性：`"disable-tail-calls"="false"`、`"frame-pointer"="all"`。

## step 5： 设置函数参数名称

`设置函数参数名称`的代码如下：
```c++
Function::arg_iterator Args = FuncFoo->arg_begin();
Args->setName("x");
```

上述代码的逻辑为：将`FuncFoo`所表示函数的（从左到右）第一个参数的名称设置为`x`。

## step 6： 创建基本块

`创建基本块`的代码如下：
```c++
BasicBlock *EntryBB = BasicBlock::Create(Ctx, "entry", FuncFoo, nullptr);
```
上述代码的逻辑为：在`FuncFoo`所表示的函数中创建一个标签为entry的基本块，由类`llvm::BasicBlock`的对象表示（这里是EntryBB），并且无前继节点。

注：函数`BasicBlock::Create()`的第四个参数用于指定基本块的前继节点。如果值为`nullptr`，表示要创建的基本块无前继节点。

## step 7： 分配栈内存
`分配栈内存`的代码如下：
```c++
auto *AIX = new AllocaInst(Type::getInt32Ty(Ctx), 0, "x.addr", EntryBB);
AIX->setAlignment(Align(4));
```

上述代码的逻辑为：在基本块`EntryBB`的末尾添加一条`alloca`指令，并且以 4 字节进行对齐。

注：类`AllocaInst`构造函数中的第二个参数用于指定地址空间。如果值为 `0`，表示默认的地址空间。

对应的 `LLVM IR` 如下：
```c++
%x.addr = alloca i32, align 4
```

上述内容表示，局部标识符`%x.addr`指向一块大小为 `4` 字节的栈内存，并且以 `4` 字节进行对齐。

# step 8： 内存写入

`内存写入`的代码如下：
```c++
auto *StX = new StoreInst(X, AIX, false, EntryBB);
StX->setAlignment(Align(4));
```

上述代码的逻辑为：在基本块`EntryBB`的末尾添加一条`store`指令，并且以 4 字节进行对齐。

*注*：类`StoreInst`构造函数中的第三个参数表示是否指定`volatile`属性。如果值为`false`，表示不指定。

对应的 LLVM IR 如下：
```c++
store i32 %x, i32* %x.addr, align 4
```

上述内容表示，将局部标识符`%x`的值写入到局部标识符`%x.addr`所指向的内存中，并且以 `4` 字节进行对齐。

## step 9： 无条件跳转

`无条件跳转`的代码如下：

```c++
BranchInst::Create(ForCondBB, EntryBB);
```

上述代码的逻辑为：在基本块`EntryBB`的末尾添加一条`br`指令（无条件跳转）。即直接从基本块`EntryBB`跳转到另一个基本块`ForCondBB`。

对应的 LLVM IR 如下：
```c++
br label %for.cond
```

上述内容表示，直接跳转到标签为`for.cond`的基本块。

## step 10： 比较大小

`比较大小`的代码如下：
```c++
auto *Cmp = new ICmpInst(*ForCondBB, ICmpInst::ICMP_SLT,
    Ld0, ConstantInt::get(Ctx, APInt(32, 10)));
Cmp->setName("cmp");
```

上述代码的逻辑为：在基本块`ForCondBB`的末尾添加一条icmp指令，比较结果保存到局部变量`Cmp`中。

对应的 `LLVM IR` 如下：
```c++
%cmp = icmp slt i32 %0, 10
```

上述内容表示，判断局部标志符`%0`的值是否小于 `10`。如果是，那么局部标识符`%cmp`的值为`true`。

## step 11： 有条件跳转

`有条件跳转`的代码如下：
```c++
BranchInst::Create(ForBodyBB, ForEndBB, Cmp, ForCondBB);
```

上述代码的逻辑为：在基本块`ForCondBB`的末尾添加一条`br`指令（有条件跳转）。即如果局部变量`Cmp`所表示的值为`true`，则跳转到基本块`ForBodyBB`；否则，跳转到基本块`ForEndBB`。

对应的 `LLVM IR` 如下：
```asm
br i1 %cmp, label %for.body, label %for.end
```

上述内容表示，如果局部标识符`%cmp`的值为`true`，则跳转到标签为`for.body`的基本块；否则，跳转到标签为`for.end`的基本块。

## step 12： 二元运算（加法)

`加法运算`的代码如下：
```c++
auto *Add = BinaryOperator::Create(Instruction::Add, Ld3, Ld2, "add", IfThenBB);
Add->setHasNoSignedWrap();
```
上述代码的逻辑为：在基本块`IfThenBB`的末尾添加一条`add`指令，并且加法运算的属性设置为`nsw`。

对应的 LLVM IR 如下：
```c++
%add = add nsw i32 %3, %2
```
上述内容表示，将局部标志符`%3`和`%2`的值相加，结果保存到局部标志符`%add`中。

## step 13： 函数返回
`函数返回`的代码如下：
```c++
ReturnInst::Create(Ctx, nullptr, ForEndBB);
```

上述代码的逻辑为：在基本块`ForEndBB`的末尾添加一条`ret`指令（无返回值）。

注：函数`ReturnInst::Create()`的第二个参数表示函数的返回值类型。如果值为`nullptr`，表示返回值的类型为`void`。

对应的 `LLVM IR` 如下：
```c++
ret void
```
上述内容表示，函数返回值的类型为`void`。

## step 14： 检查 IR 是否合法
```c++
if (verifyModule(*M)) {
  errs() << "Error: module failed verification. This shouldn't happen.\n";
  exit(1);
}
```
注：函数`verifyModule()`用于检查 `IR` 是否合法。比如：一个基本块必须以`teminator instruction`结尾，并且只能有一个。

## step 15： 生成位码文件
```c++
std::error_code EC;
std::unique_ptr<ToolOutputFile> Out(
    new ToolOutputFile("./test.bc", EC, sys::fs::F_None));
if (EC) {
  errs() << EC.message() << '\n';
  exit(2);
}
WriteBitcodeToFile(*M, Out->os());
Out->keep();
```

*注*：

* 语句*Out->keep()*;，用于指示不要删除生成的位码文件。
* 类`ToolOutputFile`构造函数中的第二个参数用于指定要生成的位码文件的名称及路径（这里是`./test.bc`，即生成的位码文件保存到当前目录下的`test.bc`文件中）。

## step 16： 完整的程序
main_ir.cpp：
```c++
#include "llvm/IR/Module.h"
#include "llvm/ADT/SmallVector.h"
#include "llvm/IR/Function.h"
#include "llvm/IR/Mangler.h"
#include "llvm/IR/Instructions.h"
#include "llvm/Support/raw_ostream.h"
#include "llvm/IR/Verifier.h"
#include "llvm/Support/ToolOutputFile.h"
#include "llvm/Support/FileSystem.h"
#include "llvm/Bitcode/BitcodeWriter.h"
#include <system_error>
#include <map>
#include <string>
 
using namespace llvm;
 
static MDNode *getID(LLVMContext &Ctx,
                    Metadata *arg0 = nullptr,
                      Metadata *arg1 = nullptr) {
   MDNode *ID;
   SmallVector<Metadata *, 3> Args;
   // Reserve operand 0 for loop id self reference.
   Args.push_back(nullptr);
 
   if (arg0)
     Args.push_back(arg0);
   if (arg1)
     Args.push_back(arg1);

   ID = MDNode::getDistinct(Ctx, Args);
   ID->replaceOperandWith(0, ID);
   return ID;
 }
 
// define dso_local void @_Z3fooi(i32 %x) #0 {...}
static Function *createFuncFoo(Module *M) {
   assert(M);
 
   SmallVector<Type *, 1> FuncTyAgrs;
   FuncTyAgrs.push_back(Type::getInt32Ty(M->getContext()));
   auto *FuncTy =
       FunctionType::get(Type::getVoidTy(M->getContext()), FuncTyAgrs, false);
 
   auto *FuncFoo =
     Function::Create(FuncTy, Function::ExternalLinkage, "_Z3fooi", M);
 
   Function::arg_iterator Args = FuncFoo->arg_begin();
   Args->setName("x");
 
   return FuncFoo;
 }
 
 static void setFuncAttrs(Function *FuncFoo) {
   assert(FuncFoo);
 
   static constexpr Attribute::AttrKind FuncAttrs[] = {
       Attribute::NoInline, Attribute::NoUnwind, Attribute::OptimizeNone,
       Attribute::UWTable, Attribute::MustProgress,
   };
 
   for (auto Attr : FuncAttrs) {
     FuncFoo->addFnAttr(Attr);
   }
 
   static std::map<std::string, std::string> FuncAttrsStr = {
       {"disable-tail-calls", "false"}, {"frame-pointer", "all"},
       {"less-precise-fpmad", "false"}, {"min-legal-vector-width", "0"},
       {"no-infs-fp-math", "false"}, {"no-jump-tables", "false" },
       {"no-nans-fp-math", "false"}, {"no-signed-zeros-fp-math", "false"},
       {"no-trapping-math", "true"}, {"stack-protector-buffer-size", "8"},
      {"target-cpu", "x86-64"},
      {"target-features", "+cx8,+fxsr,+mmx,+sse,+sse2,+x87"},
       {"tune-cpu", "generic"}, {"unsafe-fp-math", "false"},
      {"use-soft-float", "false"},
  };
 
   AttrBuilder Builder;
   for (auto Attr : FuncAttrsStr) {
     Builder.addAttribute(Attr.first, Attr.second);
   }
   FuncFoo->addAttributes(AttributeList::FunctionIndex, Builder);
 }
 
 int main() {
   LLVMContext Ctx;
   std::unique_ptr<Module> M(new Module("test.cpp", Ctx));
 
   M->setSourceFileName("test.cpp");
   M->setDataLayout(
     "e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128");
  M->setTargetTriple("x86_64-unknown-linux-gnu");
 
   auto *FuncFoo = createFuncFoo(M.get());
   FuncFoo->setDSOLocal(true);
   setFuncAttrs(FuncFoo);
 
   BasicBlock *EntryBB = BasicBlock::Create(Ctx, "entry", FuncFoo, nullptr);
   BasicBlock *ForCondBB = BasicBlock::Create(Ctx, "for.cond", FuncFoo, nullptr);
 
   // entry:
   //   %x.addr = alloca i32, align 4
   //   %i = alloca i32, align 4
   //   store i32 %x, i32* %x.addr, align 4
   //   store i32 0, i32* %i, align 4
   //   br label %for.cond
   auto *AIX = new AllocaInst(Type::getInt32Ty(Ctx), 0, "x.addr", EntryBB);
   AIX->setAlignment(Align(4));
   auto *AII = new AllocaInst(Type::getInt32Ty(Ctx), 0, "i", EntryBB);
   AII->setAlignment(Align(4));
   auto *StX = new StoreInst(FuncFoo->arg_begin(), AIX, false, EntryBB);
   StX->setAlignment(Align(4));
   auto *StI =
       new StoreInst(ConstantInt::get(Ctx, APInt(32, 0)), AII, false, EntryBB);
   StI->setAlignment(Align(4));
   BranchInst::Create(ForCondBB, EntryBB);
 
   BasicBlock *ForBodyBB = BasicBlock::Create(Ctx, "for.body", FuncFoo, nullptr);
   BasicBlock *ForEndBB = BasicBlock::Create(Ctx, "for.end", FuncFoo, nullptr);
 
   // for.cond:                                         ; preds = %for.inc, %entry
   //   %0 = load i32, i32* %i, align 4
   //   %cmp = icmp slt i32 %0, 10
   //   br i1 %cmp, label %for.body, label %for.end
   auto *Ld0 = new LoadInst(Type::getInt32Ty(Ctx), AII, "", false, ForCondBB);
   Ld0->setAlignment(Align(4));
   auto *Cmp = new ICmpInst(*ForCondBB, ICmpInst::ICMP_SLT,
       Ld0, ConstantInt::get(Ctx, APInt(32, 10)));
   Cmp->setName("cmp");
   BranchInst::Create(ForBodyBB, ForEndBB, Cmp, ForCondBB);
 
   BasicBlock *IfThenBB = BasicBlock::Create(Ctx, "if.then", FuncFoo, nullptr);
   BasicBlock *IfElseBB = BasicBlock::Create(Ctx, "if.else", FuncFoo, nullptr);
 
   // for.body:                                         ; preds = %for.cond
   //   %1 = load i32, i32* %x.addr, align 4
   //   %rem = srem i32 %1, 2
   //   %cmp1 = icmp eq i32 %rem, 0
   //   br i1 %cmp1, label %if.then, label %if.else
   auto *Ld1 = new LoadInst(Type::getInt32Ty(Ctx), AIX, "", false, ForBodyBB);
   Ld1->setAlignment(Align(4));
   auto *Rem = BinaryOperator::Create(Instruction::SRem, Ld1,
       ConstantInt::get(Ctx, APInt(32, 2)), "rem", ForBodyBB);
   auto *Cmp1 = new ICmpInst(*ForBodyBB, ICmpInst::ICMP_EQ,
       Rem, ConstantInt::get(Ctx, APInt(32, 0)));
   Cmp1->setName("cmp1");
   BranchInst::Create(IfThenBB, IfElseBB, Cmp1, ForBodyBB);
 
   BasicBlock *IfEndBB = BasicBlock::Create(Ctx, "if.end", FuncFoo, nullptr);
 
   // if.then:                                          ; preds = %for.body
   //   %2 = load i32, i32* %i, align 4
   //   %3 = load i32, i32* %x.addr, align 4
   //   %add = add nsw i32 %3, %2
   //   store i32 %add, i32* %x.addr, align 4
   //   br label %if.end
   auto *Ld2 = new LoadInst(Type::getInt32Ty(Ctx), AII, "", false, IfThenBB);
   Ld2->setAlignment(Align(4));
   auto *Ld3 = new LoadInst(Type::getInt32Ty(Ctx), AIX, "", false, IfThenBB);
   Ld3->setAlignment(Align(4));
   auto *Add = BinaryOperator::Create(Instruction::Add, Ld3, Ld2, "add", IfThenBB);
   Add->setHasNoSignedWrap();
   auto *StAdd = new StoreInst(Add, AIX, false, IfThenBB);
   StAdd->setAlignment(Align(4));
   BranchInst::Create(IfEndBB, IfThenBB);
 
   // if.else:                                          ; preds = %for.body
   //   %4 = load i32, i32* %i, align 4
   //   %5 = load i32, i32* %x.addr, align 4
   //   %sub = sub nsw i32 %5, %4
   //   store i32 %sub, i32* %x.addr, align 4
   //   br label %if.end
   auto *Ld4 = new LoadInst(Type::getInt32Ty(Ctx), AII, "", false, IfElseBB);
   Ld4->setAlignment(Align(4));
   auto *Ld5 = new LoadInst(Type::getInt32Ty(Ctx), AIX, "", false, IfElseBB);
   Ld5->setAlignment(Align(4));
   auto *Sub = BinaryOperator::Create(Instruction::Sub, Ld5, Ld4, "sub", IfElseBB);
   Sub->setHasNoSignedWrap();
   auto *StSub = new StoreInst(Sub, AIX, false, IfElseBB);
   StSub->setAlignment(Align(4));
   BranchInst::Create(IfEndBB, IfElseBB);
 
   BasicBlock *ForIncBB = BasicBlock::Create(Ctx, "for.inc", FuncFoo, nullptr);
 
   // if.end:                                           ; preds = %if.else, %if.then
   //   br label %for.inc
   BranchInst::Create(ForIncBB, IfEndBB);
 
   // for.inc:                                          ; preds = %if.end
   //   %6 = load i32, i32* %i, align 4
   //   %inc = add nsw i32 %6, 1
   //   store i32 %inc, i32* %i, align 4
   //   br label %for.cond, !llvm.loop !2
   auto *Ld6 = new LoadInst(Type::getInt32Ty(Ctx), AII, "", false, ForIncBB);
   Ld6->setAlignment(Align(4));
   auto *Inc = BinaryOperator::Create(Instruction::Add, Ld6,
       ConstantInt::get(Ctx, APInt(32, 1)), "inc", ForIncBB);
   Inc->setHasNoSignedWrap();
   auto *StInc = new StoreInst(Inc, AII, false, ForIncBB);
   StInc->setAlignment(Align(4));
   auto *BI = BranchInst::Create(ForCondBB, ForIncBB);
   {
     SmallVector<Metadata *, 1> Args;
     Args.push_back(MDString::get(Ctx, "llvm.loop.mustprogress"));
     auto *MData =
         MDNode::concatenate(nullptr, getID(Ctx, MDNode::get(Ctx, Args)));
     BI->setMetadata("llvm.loop", MData);
   }
 
   // for.end:                                          ; preds = %for.cond
   //   ret void
   ReturnInst::Create(Ctx, nullptr, ForEndBB);
 
   // !llvm.module.flags = !{!0}
   // !0 = !{i32 1, !"wchar_size", i32 4}
   {
     llvm::NamedMDNode *IdentMetadata = M->getOrInsertModuleFlagsMetadata();
     Type *Int32Ty = Type::getInt32Ty(Ctx);
     Metadata *Ops[3] = {
         ConstantAsMetadata::get(ConstantInt::get(Int32Ty, Module::Error)),
         MDString::get(Ctx, "wchar_size"),
         ConstantAsMetadata::get(ConstantInt::get(Ctx, APInt(32, 4))),
     };
     IdentMetadata->addOperand(MDNode::get(Ctx, Ops));
   }
 
   // !llvm.ident = !{!1}
   // !1 = !{!"clang version 12.0.0 (ubuntu1)"}
   {
     llvm::NamedMDNode *IdentMetadata = M->getOrInsertNamedMetadata("llvm.ident");
     std::string ClangVersion("clang version 12.0.0 (...)");
     llvm::Metadata *IdentNode[] = { llvm::MDString::get(Ctx, ClangVersion) };
     IdentMetadata->addOperand(llvm::MDNode::get(Ctx, IdentNode));
   }
 
   if (verifyModule(*M)) {
     errs() << "Error: module failed verification. This shouldn't happen.\n";
     exit(1);
   }
 
   std::error_code EC;
   std::unique_ptr<ToolOutputFile> Out(
       new ToolOutputFile("./test.bc", EC, sys::fs::F_None));
   if (EC) {
     errs() << EC.message() << '\n';
     exit(2);
   }
   WriteBitcodeToFile(*M, Out->os());
   Out->keep();
 
   return 0;
 }
```

# 构建并运行所编写的工具

## step 1： 编译
方式 1：
```shell
$ g++ -o main_ir main_ir.cpp -g -lLLVMCore -lLLVMSupport -lLLVMBitWriter
```

## step 2： 运行

*  运行
  ```shell
  ./main_ir 
  ```
*  生成 LLVM IR 汇编文件
  ```shell
  $ llvm-dis test.bc -o test.ll
  ```
通过自己编写的工具生成的 test.ll 的内容如下：
```c++
; ModuleID = 'test.bc'
source_filename = "test.cpp"
target datalayout = "e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128"
target triple = "x86_64-unknown-linux-gnu"

; Function Attrs: noinline nounwind optnone uwtable mustprogress
define dso_local void @_Z3fooi(i32 %x) #0 {
entry:
  %x.addr = alloca i32, align 4
  %i = alloca i32, align 4
  store i32 %x, i32* %x.addr, align 4
  store i32 0, i32* %i, align 4
  br label %for.cond

for.cond:                                         ; preds = %for.inc, %entry
  %0 = load i32, i32* %i, align 4
  %cmp = icmp slt i32 %0, 10
  br i1 %cmp, label %for.body, label %for.end

for.body:                                         ; preds = %for.cond
  %1 = load i32, i32* %x.addr, align 4
  %rem = srem i32 %1, 2
  %cmp1 = icmp eq i32 %rem, 0
  br i1 %cmp1, label %if.then, label %if.else

for.end:                                          ; preds = %for.cond
  ret void

if.then:                                          ; preds = %for.body
  %2 = load i32, i32* %i, align 4
  %3 = load i32, i32* %x.addr, align 4
  %add = add nsw i32 %3, %2
  store i32 %add, i32* %x.addr, align 4
  br label %if.end

if.else:                                          ; preds = %for.body
  %4 = load i32, i32* %i, align 4
  %5 = load i32, i32* %x.addr, align 4
  %sub = sub nsw i32 %5, %4
  store i32 %sub, i32* %x.addr, align 4
  br label %if.end

if.end:                                           ; preds = %if.else, %if.then
  br label %for.inc

for.inc:                                          ; preds = %if.end
  %6 = load i32, i32* %i, align 4
  %inc = add nsw i32 %6, 1
  store i32 %inc, i32* %i, align 4
  br label %for.cond, !llvm.loop !2
}

attributes #0 = { noinline nounwind optnone uwtable mustprogress "disable-tail-calls"="false" "frame-pointer"="all" "less-precise-fpmad"="false" "min-legal-vector-width"="0" "no-infs-fp-math"="false" "no-jump-tables"="false" "no-nans-fp-math"="false" "no-signed-zeros-fp-math"="false" "no-trapping-math"="true" "stack-protector-buffer-size"="8" "target-cpu"="x86-64" "target-features"="+cx8,+fxsr,+mmx,+sse,+sse2,+x87" "tune-cpu"="generic" "unsafe-fp-math"="false" "use-soft-float"="false" }

!llvm.module.flags = !{!0}
!llvm.ident = !{!1}

!0 = !{i32 1, !"wchar_size", i32 4}
!1 = !{!"clang version 12.0.0 (...)"}
!2 = distinct !{!2, !3}
!3 = !{!"llvm.loop.mustprogress"}
```
从上面的结果可以看出，我们所编写的工具生成了几乎完全相同的 LLVM IR 汇编文件。

# 资料
[llvm book](../../book/get_start.pdf)



