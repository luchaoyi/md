## **1. build**

configure 构建，生成makefile 然后make makefile

cmake 构建 cmake可以生成makefile/ninjia等文件，选择不同的构建方式，ninjia可以替代makefile，ninjia具有编译缓存

## 2. llvm

### **2.1前端**

前端的ir时ast,ast节点由decl、stmt、type三大类，整个ast的根节点为TranslationUnitDecl

lex+parser后生成ast节点，同时进行语义检查

遍历ast可以进行分析，lower，dump信息等  #ast pass

### **2.2 llvm ir**

**基本库**

libLLVM*

->Codegen 目标无关的通用codegen逻辑部分，完整的代码生成调用此lib的接口，但它需要目标相关的codegen 的支撑

->Arch*Codegen 目标相关codegen

->Target 目标无关掉目标相关不是直接调用的，因为目标无关对应多个目标相关，因此它提供了一个

Codegen到Arch*Codegen的通信网关(调用转发)

**def-use / use-def**

extend form **Value** 的class可以**def**了被其它指令使用**use**的结果，Value可以访问所有使用它的Class #def-use link  

extend form **User**的class使用(**use**)一个或多个Value,User可以访问它使用的Value 

#use-def link

Function和Instruction是Value和User的子类，BasicBlock是Value的子类(仅能被use)

**优化**

优化主要依赖分析pass和转换pass，pass管理器负责管理依赖关系，调度执行pass

由作用与不同粒度的pass，选择合适的粒度有利于pass执行的性能

### **2.3后端** 

LLVM IR -> SelectionDAG  ->  MachineInstr ->  MCInst

​							指令选择 ->{寄存器前分配指令调度->寄存器分配->寄存器分配后指令调度}->Code Emission

**tablegen** 

用于target相关的描述inst,reg,dag模式匹配图等信息

td定义的指令的Pattern字段中包含了匹配的dag模式信息，用于指令选择时dag模式匹配

**dag**

一个BB对应一个dag,因此dag没有控制流信息，EntryToken标识基本块入口，GraphRoot是根节点

黑色： 数据流依赖use-def信息

蓝色：强制规定不相关指令直接的执行先后顺序

红色: 相邻节点确保一起执行，直接不能插入其它执行

**指令选择**

target相关的select方法调用selectcode方法，tablegen为target生成selectcode方法

可以在select函数中调用selectcode之前提供自定义的匹配代码，以应付tablegen不能处理的情况

**寄存器**

分配 虚拟->物理，插入复制消除PHI解构SSA

合并 分析寄存器的活跃区间等信息，消除多余的复制指令

前后调度

**MC框架**

用于创建llvm的汇编/反汇编器，可以不依赖外部汇编器直接生成二进制(gcc 只能生成汇编依赖外部汇编器)

**MachineInstr Pass**

针对MachineInstr的pass写法非常类似 IR 的pass，在Target目录下直接修改后端增加相关pass

## 3. JIT

内存编译: 运行时传入IR Module, 函数为最小粒度按需编译，在内存中生成二进制大对象并返回函数指针

外部符号解析: Module外的符号解析

内存管理: 编译的二进制大对象要写入内存(w权限)，并拿到函数指针后跳转执行(x权限)，内存管理要负责内存的分配，回收以及设置内存权限等

#jit可以将module编译并返回函数指针，将函数指针转为对应类型就可以执行















