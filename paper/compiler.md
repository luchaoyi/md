**MLIR:A Compiler Infrastructure forthe End of Moore’s Law**

```
Design Principles
	最小化核心概念，types, operations and attributes, 自定义扩展性， 风险是碎片化
	nested regions as a first-class concept
	混合不同曾经IR,逐步下降
	Declarative rewrite patterns Defining、
	
IR Design Details
	Operations
		all is Op（from instruction” to “function” to “module”）
		rich support for describing the semantics（traits, privileged operation hooks and optimization interfaces）
		Opcode result Oprand , result and Oprand is value
		Ops may also have Attributes, Regions, Block Arguments, and Location
Information
	Attributes
		structured compile-time static information
	Regions and blocks
		Op - Region - Block - Op
		the mechanism for nested structure in MLIR
		Instead of using phi nodes， a functional form of SSA，terminators pass values
into block arguments。不使用phi, 而是terminators op 向跳转的基本传参，以函数调用的形式避免使用phi
	Symbols and symbol tables
		symbols, associating names to IR objects
		Ops can have a symbol table attached, MLIR provides a mechanism to reference symbols from an Op
	Dialects
		Ops, attributes and types under a unique namespace
		允许方言混合使用，逐步下降
		混合表达，多级抽象，四方借力
	Type system
		Types provide compile-time semantics
		强制执行严格的类型相等检查，并且不提供类型转换规则（无隐藏类型转换规则，强制类型匹配检查）
		enforces strict type equality checking and does not provide type conversion rules
		mlir提供了一组标准类型，这是一种便利，但这不是必要的their use is not required

IR Infrastructure
	for defining IR elements such as dialects, Ops, pattern rewrites, verification and reusable passes
	ODS 
		 for MLIR Op definition
	     arguments that specify Op’s operands and attributes 输入
	     results 输出
	     更细致的定义可以通过C++ code injected via builder, printer, parser, verifier clauses
	 Declarative rewrites
	 	某些转换 can be expressed as simple rewrites on the DAG
	 	capturing full-fledged transformations as a composition of small local patterns and controlling which pattern rewrites are applied at the granularity of an individual operation. 
	 Pass manager
	 	由于mlir all is op, 因此不像llvm pass工作在不同的level
	 	支持持并发遍历和修改ir(SSA use-def chains cannot cross the region boundaries of these ops，define a tree of regions that may be processed in parallel)
	Pass
		# all is Op, 所有pass的操作粒度就是Op
    	Fundamental operation traits
    		Some pass(Dead Code Elimination and Common Subexpression Elimination) rely on very simple properties(like “has no sideeffect” or “is commutative”),define as Op traits
    		# 某些pass是内部支持好的，仅需要为Op附加属性就可以被优化
    	Privileged operation hooks
    		hooks,implemented on a **per-operation** | **Dialect** object 
    		getCanonicalizationPatterns allows one to specify folding patterns that apply to an operation
    	Optimization interfaces
    		接口是通用pass的回调机制，通用pass的流程代码是Object无关的，将Object相关的依赖抽象为接口，由Op/Dialect等对象实现
    		提供了模块化优势，因为方言特定的逻辑是在方言本身内部实现的，而不影响通用的common代码
    		例子: 内联接口，各个操作和方言可以向MLIR注册其对该接口的实现，由通用内联pass调用
		Dialect specific passes
		
Interoperability
	外部系统，定义一种对应的dialect IR,将外部系统变换到mlir
    例子：functional-style control flow operators—“functional while” and “functional if” are common in machine learning graphs
```

**Relay:A New IR for Machine Learning Frameworks**

```
Abstract
	Relay
		purely-functional
		statically-typed language
		balancing efficient compilation, expressiveness, and portability
Introduction
	models, hardware, and systems be co-designed 
	NNVM 是计算图，一种流行的可微分计算IR，计算图是ast的变形，for 某些优化pass很困难
    现有框架限制了程序的计算表达能力
    	TF使用静态图，用户一般使用dsl接口构造，而不是如函数的高层抽象
    	命令式框架(like Pytorch),允许构建具有动态拓扑的图，PyTorch的模型需要一个Python解释器，部署比较有挑战性
   		 静态图易于优化，但缺乏高级语言中的表现力；动态图提供了这种缺失的表现力，但引入了新的编译和执行时挑战
    Relay
    	从用于可微分计算的编程语言的角度设计Relay,可更多的借力经典的编译技术，相比计算Graph
        Higher-order automatic differentiation 、shape-dependent tensor type system
        a statically typed, purely functional, differentiable IR
Background
	TF
		restricted control edges to represent differentiable programs
		反转模型，实现差分
		在指定设备执行子图实现异构计算
		计算图拓扑固定
	Pytorch
		计算图是define then run,动态框架是defien by run model
        控制流是python处理，后端处理数据流，丢失控制流信息，优化有限
        交互式需要来回host<->device搬动数据
    TVM
    	高层级优化，如算子融合
    	算子内存重用
    	张量计算
Design
	python接口
		libray 包含算子和relay特定函数
		装饰器 转换被包装的函数到 relay ast 并生成一个可以在relay evaluation mechanisms执行的函数		tvm node system 可以 expose c++ classes到python
	relay decorator declares a function that can be run without any hidden state
	relay_model decorator declares a function that cannot be run by default and instead must first be instantiated by a call to relay.create_model.
	自动微分
		可以通过对函数式编程语言使用局部程序转换来实现	
	类型系统
		small core language, 保持类型系统small
		所有都是张量，标量也是张量则可以将优化统一
Future
	Runtime 
		使用TVM JIT编译后运行
		内存管理和值的生命周期
	优化Pass
	测试调试
		编译relay到python调试/测试，如使用python autograd lib 对比测试自动差分的结果
	低精度数值支持
	类型系统
		支持无固定形状的张量 // like mlir *, ?功能
		隔离处理不同资源（例如随机数生成器，状态，I / O等）的代码
		track individual tensors’ data layouts
```
