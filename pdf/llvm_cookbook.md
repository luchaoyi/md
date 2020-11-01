## 1.Pass

### 1.1 Pass and PassManager

**pass**

1.具有不同层次Level的pass

CGSSCCPass 在 CallGraphSCC上支持的Pass,处于function之上, module之下的层次

Region 有两种

(1)  single entry single exit region, a dominates b, b post dominates a

(2) 如果可以通过添加一个pre-header和post-exit构成definiton(1)中的simple region, 那么称为extended region

2.具有两个Analysis的集合,Require(P)和Preserve(P)

Require(P)表示P这个Pass运行之前需要哪些Analysis （Pass::getAnalysisUsage），在pass内的函数可以使用Pass::getAnalysis(）获取想要的分析信息。

Preserve(P)表示Pass运行过以后哪些Analysis的结果仍然有效

**schedule pass**

每个P都有L(Level）属性，对L这个层级的所有IR run()一遍P，运行P的时候需要已经运行产生Required(P)的analysis的Pass，运行P会使所有Preserve(P)以外的analysis invalidate掉

 llvm里为了考虑对缓存友好，采用的schedule策略是集中运行完某一部分IR的所有层级Pass(包括所有所包含的下级IR的Pass), 再去运行其他IR的Pass

**PassManager**

接受一个Pass列表,负责pass依赖管理以及执行调度

尽可能在多个Pass间共享分析数据，避免重复计算分析结果

一系列Pass一起流水线化运行，let 缓存和内存使用行为友好

**两套PassManager***

#ref https://www.zhihu.com/question/45051197/answer/290078011

legacy PassManager:

pass是基于对象继承体系严格约束旧Pass与PassManager体系

旧版重要的特性就是analysis manager和PassManager高度融合，通过 `getAnalysis<...>` 方法来获取某个分析数据，新版分离了一个独立的AnalysisManager负责管理所有已注册的analysis Pass和它们的分析结果

PassManager：

concept based programming(duck type), 实现run接口的不毕业区分Pass or PassManager，

不同pass如何存放在相同容器:

First, def "PassConcept" pure virtual class

Second, template <typename PassT> class PassModel : PassConcept

So, All PassX extend PassModel<PassX>, then CRTP,**CRTP and Trait/Policy let the class has some common Attributes**

**自定义Pass编写**

源码树外opt方式:

​	->实现pass -> RegisterPass <MyPass> ->编译后opt load

源码树中写一个pass:

ref:https://llvm.comptechs.cn/post/45743.html

   0.实现一个Pass

1. 实现`createXXXPass`函数.
2. 实现`initializeXXXPassPass`函数/ `InitializedPasses.h`文件.
3. `INITIALIZE_PASS_BEGIN`/`END`/`DEPENDENCY` code.
4. 将`initializeXXXPass` 放到正确的位置.
5. 实现`LinkAllPasses.h`文件.
6. 将`createXXXPass`放到正确的位置,即在某处调用此函数，将Pass插入到llvm内部的编译Pipeline中，如PassManagerBuilder::populateFunctionPassManager

**INITIALIZE_PASS宏**

基于llvm/include/PassSupport.h提供的宏**自动代码生成**初始化函数代码定义，然后在构造函数中调用生成的函数实现自定义Pass的初始化。

例子:

INITIALIZE_PASS_BEGIN(FuncBlockCount, " funcblockcount ",                     "Function Block Count", false, false) 

// 在初始化自身之前调用了依赖函数的初始化代码

INITIALIZE_PASS_DEPENDENCY(LoopInfoWrapperPass)
INITIALIZE_PASS_END(FuncBlockCount, "funcblockcount",                   "Function Block Count", false, false) 

此三个宏组合生成一个函数体, 来执行初始化(包含注册PassInfo)

## 2.转换Pass

**入侵式无用代码消除**

首先假定所有的代码都是无用的，然后证明它们是有用的

算法：首先寻找根指令，然后根据use-def链传播活跃性，最后删除不在Alive集合的指令

**内联**

inline.cpp 函数 InlineCost getInlineCost(CallSite CS)实现了内联策略，（决定什么时候内联）

**memcpy优化pass**

寻找根据编译期已知信息，将memcpy转化为memset

**表达式变换重组**

lib/Transforms/Scalar/Reassociate.cpp

利用代数的结合律、交换律、分配律来对表达式重新安 排以实现其他的优化

**#调用图**

表示子程序之间的调用关系,是一种控制流图,点表示一个过程边`(f, g)`表示过程`f`调用过程`g`,一个循环表示递归过程调用，分为运行时调用图与静态调用图

计算静态调用图精确地需要别名分析结果,计算精确的别名需要一个调用图

应用：寻找未被调用的过程，生成作为人类的阅读文档(doxygen)，跟踪过程之间的值流的分析，检测程序执行的异常或代码注入攻击等

## 3.Codegen

**贪心法寄存器分配**

变量的活跃周期信息，活动周期越长的变量先分配寄存器，生存周期短的变量则填补可用寄存器的时间间隙，减少溢出权重

**代码发射**

JIT 直接发射到内存执行

AOT MC框架 生成汇编/二进制

**MachineFunction**

包含的MachineBasicBlock以及了MachineConstantPool、MachineFrameInfo、 MachineFunctionInfo、Machine-RegisterInfo等更底层的细节信息

