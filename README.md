tensorflow reading code
===
[tensorflow README.md](README.md)

base on tensorflow 2.6.2 [source](https://github.com/tensorflow/tensorflow/releases/tag/v2.6.2)

# 架构
TensorFlow 模块化和分层设计十分精良，具体模块和层层如下：

![](images/tf.png)

整个框架以C API为界，分为前端和后端两大部分。
* 前端：提供编程模型，多语言的接口支持，比如Python Java C++等。通过C API建立前后端的连接，后面详细讲解。

* 后端：提供运行环境，完成计算图的执行。进一步分为4层   
  * 运行时：分为分布式运行时和本地运行时，负责计算图的接收，构造，编排等。
  * 计算层：提供各op算子的内核实现，例如conv2d, relu等
  * 通信层：实现组件间数据通信，基于GRPC和RDMA两种通信方式
  * 设备层：提供多种异构设备的支持，如CPU GPU TPU FPGA等

# 源码
[tensorflow](tensorflow/README.md)

[third_party](third_party/README.md)

# 阅读

## 源码目录

| 目录 | 功能 |
| :----: | :----: |
| tensorflow/c | C API代码 |
| tensorflow/cc | C++ API代码 |
| tensorflow/compiler | XLA,JIT等编译优化相关 |
| tensorflow/core | tf核心代码|
| tensorflow/examples | 例子相关代码 |
| tensorflow/go | go API相关代码 |
| tensorflow/java | java API相关代码 |
| tensorflow/python | Python API相关代码 |
| tensorflow/stream_executor | 并行计算框架代码 |
| tensorflow/tools | 各种辅助工具工程代码，例如第二章中生成Python安装包的代码就在这里 |
| tensorflow/user_ops | tf插件代码 |
| third_party/ | 依赖的第三方代码 |
| tools | 工程编译配置相关| 
| tensorflow/docs_src | 文档相关文件 |
| tensorflow/contrib | |

## tensorflow/core


| 目录 | 功能 |
| :----: | :----: |
|tensorflow/core/common_runtime| 公共运行库|
| tensorflow/core/debug | 调试相关 |
| tensorflow/core/distributed_runtime| 分布式运行模块|
| tensorflow/core/example | 例子代码 |
| tensorflow/core/framework | 基础功能模块 |
| tensorflow/core/graph | 计算图相关 |
| tensorflow/core/grappler | 模型优化模块 |
| tensorflow/core/kernels | 操作核心的实现代码，包括CPU和GPU上的实现|
| tensorflow/core/lib | 公共基础库|
| tensorflow/core/ops | 操作代码 |
| tensorflow/core/platform | 平台实现相关代码 |
| tensorflow/core/protobuf | .proto定义文件 |
| tensorflow/core/public | API头文件|

## framwork

* [tensor](docs/framework_tensor.md) 张量
* [allocator](docs/framework_allocator.md) 内存分配器
* [resource](docs/framework_resource.md) 资源管理器
* [op](docs/framework_op.md) 算子定义
* [kernel](docs/framework_kernel.md) 算子实现
* [node](docs/framework_node.md) 图节点
* [graph](docs/framework_graph.md) 图定义
* [device](docs/framework_device.md) 设备
* [function](docs/framework_function.md) 函数
* [shape inference](docs/framework_shape_inference.md) 形状判断


## common runtime 本地运行时

* [device](docs/common_runtime_device.md)
* [executor](docs/common_runtime_executor.md)
* [session](docs/session.md)
* [direct_session](docs/common_runtime_direct_session.md)


# 技术议题

## 版本变化

* [tf 1.x - tf 2.6](docs/tf1x_tf26.md)
  
## 编译优化

* [mlir](tensorflow/compiler/mlir/README.md)
* [what is xla](docs/compiler/what_is_xla.md)

## 模型优化

* [grappler](docs/optimization/grappler.md)
* [train parall](docs/optimization/train_parall.md)
  
## llvm ir

* [llvm code book](book/LLVM%20Cookbook.pdf)
* [LLVM IR 入门](docs/llvm/llvm_ir_1.md)
* [生成 LLVM IR](docs/llvm/llvm_ir_2.md)
* [LLVM IR 优化器](docs/llvm/llvm_ir_3.md)


## mpi

## 机器编译
* [tvm](docs/tvm/README.md)
