## 1.Pass and Pass Manager

### Pass 

#Pass就是“遍历一遍IR“

自定义需要重写方法

doInitialization

runOn{Passtype} 

​	返回True表示Pass执行了一些修改，否则返回False(如打印Func名字Pass)

doFinalization

**ModulePass** 模块级别Pass,无法以特定顺序引用模块内函数

**FunctionPass** 函数内的Pass,不允许添加或删除函数，仅对当前函数内进行变换，没有对函数处理的顺序定义

**BasicBlockPass** 基本块内的Pass,不允许改变CFG和添加删除基本块

**LoopPass** 处理某个循环的Pass

### Pass Manager 

负责Pass的调度

管理pass依赖关系

​	如指定哪些pass在自己之前执行，自己执行之后导致哪些pass失效

​	pass需要实现getAnalysisUsage方法, 在此方法中filling in the details in the AnalysisUsage object

​	AnalysisUsage object会提供add*来添加信息的接口

​		AnalysisUsage::addPreserved<> 指定的哪些分析Pass在它运行时不会失效，它将保留已经存在的信息，需要分析Pass将不需要再次运行

通过跟踪哪些分析可用，哪些分析无效以及哪些分析是必需的，避免重复计算分析结果。
跟踪分析结果的生命周期，并在不需要时释放存储分析结果的内存，从而实现最佳内存使用。

给定一系列连续的FunctionPass时，它将在第一个函数上执行所有FunctionPass，然后在第二个函数上执行所有FunctionPass，以获得更好的内存和缓存结果，从而改善了编译器的缓存行为。

### Instruction simplification

lib/Analysis/InstructionSimplify.cpp 基于预定义好的模式匹配

### Instruction Combining

生成指令替换老指令，和Instruction Simplify一起用于执行代数化简例如:

%Y	=	add	i32	%X,	1

 %Z	=	add	i32	%Y,	1 

into: 

%Z	=	add	i32	%X,	2

lib/Transforms/InstCombine/InstCombine 文件夹下针对不同指令由不同的合并规则

### Loop Pass

在CFG识别自然循环，主要根据CFG和支配树信息

插入入口块，保证只有一个入口到循环，插入出口块，保证循环所有出口来自循环内,循环变为canonical	form

**循环优化**

循环不变量外提 : 将循环不变量，提到入口和出口块 lib/TransformsScalar/LICM.cpp

**intrinsic	function**

编译器知道如何更好的实现（更粗粒度的Operation，e.g. 手写算子用于CodeGen）

以llvm. 前缀，调用extern函数，llvm利于它直接调用高度优化的外部库以实现特定的功能

**SIMD**

#Loop-Aware	SLP	in	GCC	by	Ira	Rosen,	Dorit	Nuzman, and	Ayal	Zaks.	LLVM	SLP	Vectorization	Pass	implements	the	Bottom	Up	SLP	vectorizer.

步骤:

1.确定模式并确定它是否为有效的矢量化模式

​	检测连续存储使用(发现向量化的机会)，

​	使用use-def链构造可向量化的树 , 首先寻找store指令发现连续store的模式，然后根据store找运算和load，检查关联的运算是否一致连续，load是否连续，经过check可以确定发现了可向量化的模式

2.确定矢量化代码是否有利可图
3.如果第1步和第2步为真，则将代码矢量化

## 2. DAG 指令生成 

#lib/CodeGen/SelectionDAG/

中后端：builder->ir -> target 无关opt ->ir -> dag -> target相关opt->inst

codegen：DAG合并，合法化，指令选择，指令调度等—最终分配寄存器 and emit机器代码，寄存器分配和指令调度是交织在一起的

**DAG生成**

**一个ir inst为一个node把llvm ir变换为dag nodes：**

1.从inst的操作数inst getValue获取操作数的SDValue

2.从inst中提取需要附加的信息

3.附加信息到到SDNodeFlags,使用SDNodeFlags::setXXX

4.SelectionDAGBuilder构建节点,（填入OpType, DataType,操作数的SDValue,Flags等)

5.Node的类型为SDValue类型，将新生成的SDValue setValue到inst上，以便后续指令可以getValue获取	

#根据Inst的def-use信息，通过setValue和getValue将Node串联为DAG

#visit llvm ir and extract info, and then build dag node and link nodes by ir inst def-use info  -> gen dag

#从ir生成dag时**目标无关**的，后续在dag上的操作就和**目标相关**了，需要target的相关实现的支持

**DAG dot**

可视化图形，边由不同的颜色

黑色： 数据流依赖use-def信息

蓝色：强制规定不相关指令直接的执行先后顺序

红色: 相邻节点确保一起执行，直接不能插入其它执行

**DAG合法化**

dag生成不依赖于target，因此生成的dag可能和目标硬件不适应，因此需要合法化，TargetLowering	interface由Target实现， 描述了Target特定的信息

SelectionDAGLegalize::LegalizeOp：

1.根据不同的OpCode类型，调用Target的TargetLowering	interface获取需要执行的Action（需要执行哪种合法化动作）

2.根据得到的Action，调用自身的不同的Node变化方法Legal Node

#SelectionDAGLegalize->{OpCode}->TLI->{Action}->SelectionDAGLegalize,TLI给SelectionDAGLegalize指示信息，由SelectionDAGLegalize执行对应的Legal

Action:

Expand:

promotion 提升type	to	a	larger	type

custom 自定义involves	target-specific	hook

**DAG优化**

DAGCombiner阶段，优化机会可能由于一组特定于体系结构的指令而产生

#Target的相关的需要Target提供分析/分析+转换,和LLVM提供的机制交互

**指令选择**

CodeGenAndEmitDAG()	-> 	DoInstructionSelection()	visits	each	DAG	node calls  -> Select()	

每一个Target都实现了一个XXXDAGToDAGISel是SelectionDGAISel的子类，实现了具体Target的Select函数

例如X86Target:

X86DAGToDAGISel::Select将大部分工作委托给X86DAGToDAGISel :: SelectCode

**X86DAGToDAGISel :: SelectCode由TableGen自动生成**，该函数包含一个match table ，它将调用的SelectionDAGISel::SelectCodeCommon并将match table传递给它。#延迟实现到子类的设计模式

**指令发射与调度**

上一步指令选择是take in place dag, and then 转换dag为指令序列指令序列的容器为MachineBasicBlock

由DAG->inst又是一个conversion,inst由	MachineInstrBuilder 产生

**寄存器分配**

虚拟寄存器映射到物理寄存器，需要target提供硬件寄存器和指令相关信息，llvm内置了多种分配算法

**Code Emission**

传递MachineInstr到MC layer，MC layer中指令被表示为MCInst,更加的轻量携带信息更少,AsmPrinter 使用目标特定的MCInstLowering接口将MachineFunction函数转换为MC label

McInst被传递给MCAsmStreamer的子类去生成汇编文件/直接生成二进制，MC层是LLVM和GCC之间的最大区别之一,GCC总是输出汇编然后使用外部汇编器生成.o, llvm使用自己的汇编器，可以选择直接生成.o

## 3. Target

### 3.1 td

**寄存器集**

**调用约定** 

描述参数传递和值返回的约定方式

在指令选择阶段使用

**指令集**

**Frame**

stack 	Keeping track of return	address , Storage of local variables,Passing arguments	

这些操作由emitPrologue()	and	emitEpilogue()实现

### 3.2 target定义

使用td生成一个Target的描述的enum和class，封装并提供一个必要的支持方法，并向llvm注册这个Target,融入到llvm体系中(llvm的target like mlir中的一个dialect，因此都使用了td机制)