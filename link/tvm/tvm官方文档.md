## 1. deploy and integration

compiler负责编译和优化

runtime在目标设备运行

### 1.1 deploy rpc

推荐交叉编译时使用RPC机制在运行时设备调试，最终调试完成部署到目标设备不依赖RPC：

​	device 设备仅需要编译tvm runtime, host机器要完全编译

​	配置Triple, 本地交叉编译可以在目标机器运行的lib.tar

​	配置目标机器RPC服务器的IP和PORT

​	upload 本地的lib.tar到device,使用load_module加载 remote module object

```
rpc机制解析：
基于python开源web框架构建通信client,server
基于json构建rpc消息序列化
load module代码由c++实现，python加载动态链接库并将外部函数以setattr动态注册到module中，提供支持
```

### 1.2 deploy without rpc

**c++ api** 

编译器编译生成.so, 使用c++ runtime api加载.so并执行，需要链接libtvm_runtime.so # like dlopen机制

**android**

构建安卓目标runtime,使用tvm 提供的java api加载执行

## 2. tutorials

### 2.1 tvmc

complie 编译，指定target等信息

run 运行 指定输入，输出

tune 运行不同的Op实现， 并记录相关元数据到record，寻找性能更优的方案，tune的输出可作为以后tune和compile的输入

### 2.2 模型编译

```
mod.astext(show_meta_data=False) #生成ir文本形式
```

relay.build 返回json 计算图，编译的lib  and  the parameter blobs of the model

relay计算图级别优化，tvm 提供tensor级别优化

### 2.3 TE

In TVM, we can always inspect lower level IR to **debug** or optimize our schedule. Here is the generated IR using our baseline schedule.

```
print(tvm.lower(s, [A, B, C], simple_mode=True))
```

compute 与 schedule, 由完全手动写算子向程序员提供schedule进化，是一种模式归纳

**Hook Python Function as Extern python** 函数可以注册到tvm运行时，被tvm运行时调用

每一个内部的te.*  math函数对应1-多个target相关的外部函数，对应规则为一个python函数, 此函数定义后需要注册

**Scan**  describe symbolic loop

​	# use it can support rnn

​	`s_state` is a placeholder that describes the transition state

​    `s_init` describes how we can initialize

​	`s_update` describes how to update

### 2.4 调度原语

split 指定axis，以factor为步长切割循环

tile 在两个轴split, 平面上切块

fuse 两个axis的融合

reorder 循环内外轴顺序交互

bind bind axis 到 thread axis  常用于gpu编程

tensorize replace a unit of computation with the corresponding intrinsics，  intrinsics 可能时一个extern 函数的形式

### 2.5 使用外部库

layer fusion发生在tvm ir, 如 conv + bn + relu, 若使用cudnn 的 lib 替换 conv 此时 lib 库外部函数被视为黑盒无法融合 

### 2.6 Micro Tvm

建立一个session，来模拟设备运行tvm 编译好的模型

#TVM和框架的不同点在于导入模型后，有一个显式的编译过程，模型编译后deploy到目标设备执行，可以JIT/AOT, 可以在Micro Tvm 开启Session绑定虚拟设备运行

### 2.7 TOPI

provides numpy-style generic operations and schedules with higher abstractions // 猜测topi 应该时对te的进一步封装接口

topi.的计算接口，一般有对应的内置的调度接口，在较高层次上使用topi提供的接口，而减少用户的的微操

如 topi.nn.softmax ： topi.cuda.schedule_softmax 

### 2.8 Auto Tvm

**Template-based**

1.defining a search space

​	很多schdule都有超参数，直接设置超参数可能不能适应不同的硬件环境，因此定义一组候选值根据在硬件环境上的测度来寻找超参数

​	@autotvm.template mark 函数为模板函数，因为cfg生成的不同参数导致编译后生成不同的代码，因此可视为模板函数的概念

   cfg.define_xxx提供了高级api定义, 为对应的xxx schdule提供knob

2.running a search algorithm to explore through this space,  结果保存在log中

3.最终将最好的结果从日志中提取处理，应用的最后的build中

```
with autotvm.apply_history_best("matmul.log"):
    with tvm.target.Target("llvm"):
        s, arg_bufs = matmul(N, L, M, "float32")
        func = tvm.build(s, arg_bufs)
```

**Template-free**

 auto-scheduler does not require any templates

without any schedule commands or templates, 用户仅需要指定计算（最少干预）

**基于rpc的多设备tune**

多设备分布式tune可加快速度

host 主机启动tracker服务，RPC Tracker is master node to manage distributed devices

每个device 启动server 服务, 并register to the tracker， 使用string key to distinguish the types of devices

## 3. Language Reference

### 3.0 额外补充

```
scala模式匹配
	scala class不能有static成员和方法，有同名object 称为伴生对象，伴生对象里面的成员和方法实现了static的作用
	apply 注入方法生成对象的方法
	unapply 提取器 结构对象的方法，模式匹配的方法必须有unapply方法
	Option[T] 是个容器，有东西是为Some，没东西是为None,Some和None都是Option的子类

性能估算
	band wdith 是io的速度（GB/s）
	io_time = (input_bytes + output_bytes)(总数据量) / band_wdith
	f 频率GHZ，T=1/f, 一个cycle时间为1/f, N/cycle为计算速度，一个cycle算N字节
	compute_time = inst_num(指令条数) * (input_bytes / N/cycle) * (1/f) 

性能度量
	T为实际运行的总时间 
	io 效率  = 实际总数据量 / T * band_wdith 
	计算效率 = 实际总数据量 / T * 计算速度（N/cycle) * (1/f)
	含义解析: T实际处理的实际的总数量/ T实际理想情况（打满）处理的数据量，越接近1越好，要求在T实际片内IO和计算尽量不被block -> 理想下数据和计算尽量重叠(软流水)且速度匹配
```

 ### 3.1 Relay

algebraic data types, closures, control flow, and recursion

design for 机器学习

**Expressions in Relay**

​	 Dataflow fragment  仅包含影响数据流的relay 表达式程序（无数据流）可视为传统计算图

​	 Control flow expressions  会改变图的图拓扑结构

​     从计算图角度看，函数时一个子图，一个函数调用内联该子图；因此被调用函数是否不包含控制流影响了被调用函数是否为 Dataflow fragment

​	Functions 

​		使用关键字fn定义，默认返回最后一句的求值，没有return关键字

​		支持模板多态，类型参数有kind,可以约束

​		type relations  函数输入参数和输出参数类型关系的约束，type checking is thus treated as a constraint-solving 

​	**Operators**  

​		an operator is a external function

​		 declared in the global, registry in C++

​	ADT Constructors

​		函数类型， constructor produces an ADT instance ，stores the values of the arguments to the constructor （like scale的apply,将参数打包到ADT中）

​		在模式匹配时会解构ADT，根据其构造函数标签来检索存储在ADT实例中的值(like scale unapply)

​	**Module**

​		global data structure , keep track of the definitions of global functions, mapping of global variables to the function expressions they denote

​	let 

​		let绑定序列被视为数据流图，没有use-def依赖的let bind 可以安全的reorder

​		let绑定将创建一个命名变量，该变量绑定到给定值并作用于后续表达式

​	图绑定

​		与let绑定不同，图绑定在Relay中不表示为AST节点，而是表示引用其AST节点值的元变量

​	   每个对该变量的引用都对应于数据流图中的一条边。这具有在表达式出现的位置处替换表达式的语义

​		#不创建节点，在使用位置会被替换为被引用的对象和关系，绑定为被引用的let 节点增加一点边

**Type System**

​	Static types，信息更丰富，利于优化(such as runtime shape, data layout, and storage, without needing to run the program)

​	将shape作为一种类型，将形状推导视为类型推导问题，在编译时试图推导所有张量的形状,包括使用分支和函数调用的程序

​	tensor

​		（shape,...）+  type

​		   scalars are tensors with a shape of `()`,a rank-zero shape

​    tuple

​		是元组，使用 `.index` 提取元素

​	funtion fn<type_params>(arg_types) -> ret_type where type_constraints

​	Type Parameter(	 **type polymorphic**  )

​		Type top  level type  tensor types, tuple types, and function types

​		BaseType base type of a tensor (e.g., `float32`, `bool`)

​		Shape tensor shape

​		`ShapeVar`,  variables within a tensor shape, shape内的变量值

  incomplete type

​		“type variables” or “type holes”,will be replaced by another type at a later point

​		only used during type inference

​     Adts	

​		The module keeps a mapping of ADT names (global type variables) to the list of constructors for that ADT

​		maybe can viewed as  enum + union struct

​		Adt由一组构造器函数定义，每个构造函数得到一组参数并返回一个 Adt类型,ADT的实例是一个容器，该容器存储参数的值和构造函数的名称;

​		ADT可以接受类型参数

​		#函数式编程语言常用函数 map ,reduce,filter, zip, flod

​		#函数式编程常用核心结构 list,tuple,dict

### 3.2 Pattern Matching in Relay

描述模式匹配的语言，用于在pass 转换和 codegen中匹配特定模式

provide a regular-expression like capability for matching data-flow graphs and doing rewriting

默认重写 模式匹配 +  替换新模式

模式划分 

​	将匹配的子图划分为单独的Relay函数，然后对该函数执行其他处理使用

​	pattern.partition为每个匹配的子图创建一个新的Relay函数

### **3.3 Design and Architecture**

autotvm

​	成本模型和特征提取。

​	一种记录格式，用于存储计划基准结果以进行成本模型构建。

​	一组有关程序转换的搜索策略

​	