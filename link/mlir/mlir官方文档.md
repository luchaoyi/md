## 1.Start

1.混合ir，支持不同的需求，协同现有的编译器

2.不计划支持底层的代码生成，如寄存器分配等，mlir 的后端为llvm / 其它语言（c/cuda c等）

### 1.1 build

```shell
sudo apt-get install clang lld
git clone https://github.com/llvm/llvm-project.git
mkdir llvm-project/build
cd llvm-project/build
cmake -G Ninja ../llvm \
   -DLLVM_ENABLE_PROJECTS=mlir \
   -DLLVM_BUILD_EXAMPLES=ON \
   -DLLVM_TARGETS_TO_BUILD="X86;NVPTX;AMDGPU" \
   -DCMAKE_BUILD_TYPE=Release \
   -DLLVM_ENABLE_ASSERTIONS=ON \
   -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++ -DLLVM_ENABLE_LLD=ON
   
cmake -G Ninja ../llvm \
   -DLLVM_ENABLE_PROJECTS=mlir \
   -DLLVM_BUILD_EXAMPLES=ON \
   -DLLVM_TARGETS_TO_BUILD="X86;NVPTX;AMDGPU" \
   -DCMAKE_BUILD_TYPE=Debug \
   -DLLVM_ENABLE_ASSERTIONS=ON \
   -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++ -DLLVM_ENABLE_LLD=ON

cmake --build . --target check-mlir
```

### 1.2 术语

转换>方言之间（或内部）的一种转换

翻译> MLIR与外部表示之间的转换，转换为外部表示为Export，转换外部表示为MLIR为import

### 1.3 测试 

check文本形式测试

[FileCheck](https://llvm.org/docs/CommandGuide/FileCheck.html) 

​	reads two files standard input and specified on the command line

​	uses one to verify the other

CHECK-LABEL 划分不同的CHECK块

CHECK 之后

CHECK-NEXT 之后紧挨一行

CHECK_SAME 之后同一行

CHECK-EMPTY 之后一行允许为空

CHECK-NOT之后不允许出现

CHECK-COUNT-n  之后匹配模式n次

Lit测试

https://llvm.org/docs/CommandGuide/lit.html

配置选项参数，搜索测试，启动测试，控制整个测试过程

lit proper is primarily an infrastructure for discovering and running arbitrary tests, and to expose a single convenient interface to these tests. lit itself doesn’t know how to run tests, rather this logic is defined by test suites.

lit将测试套件标识为包含lit.cfg或lit.site.cfg文件的目录，通过递归搜索命令行中所有输入文件的目录层次结构来发现测试套件

gtest单元测试

​	非文本的形式测试，对API接口，缺点是若API用户接口不稳定，每次改动接口会影响一大批测试要重写

​	文本形式的测试，不影响接口改动

## 2 Code Doc

### 2.1 Language Reference

图结构>

Operations是节点，Values是边.

Operation

​	Regions

​		Blocks

​			Operations

Traits 描述IR的验证约束信息

dialects>扩展机制,允许定义new [operations](https://mlir.llvm.org/docs/LangRef/#operations) , as well as [attributes](https://mlir.llvm.org/docs/LangRef/#attributes) and [types](https://mlir.llvm.org/docs/LangRef/#type-system)，不同的dialect可以混合使用在一个module,dialect定义了一个namesapce

**Operation**

```mlir
 op-result (`,` op-result)* `=` string-literal `(` value-use-list? `)`  successor-list?
                      (`(` region-list `)`)? attribute-dict? `:` function-type  trailing-location?
```

0-多个{return results，operands,attributes,successors, enclosed regions},如分支跳转等Op是一个Block的终结Op.

Operation有Oprands,Attributes和Successors,描述了动态参数，静态参数和Dag后继，Terminator Op有Successors

**Module** 

不能隐式捕获模块外部定义的值,Modules is IsolatedFromAbove

**Function**

IsolatedFromAbove，使用参数和属性和外部建立符号连接

可以为结果，输入参数或函数自身指定属性

**Block**

块参数类似函数参数，MLIR利于块参数来表示控制流相关(依赖)的值的传递，而不使用PHI,，与控制流无关的值可以直接引用，不需要通过块参数传递

**Region**

区域所属的Op定义了区域的语义当前定义了两种Region，The kinds of regions within an operation are described using the [RegionKindInterface](https://mlir.llvm.org/docs/Interfaces/#regionkindinterface) .

Region必须被包含到Op中，没有地址和名字，没有类型和属性

SSACFG regions 

​	describe control flow between blocks

Graph regions

​	无控制流计算图，只能包含一个基本块

​	In Graph regions，块中的Op顺序和Region中的block顺序在语义上没有意义，并且非终止符操作可以自由地重新排序 #计算图看的是拓扑序

**标准类型**

defined in a builtin dialect,MLIR内都可以使用

index >

无符号整数，其大小等于目标的自然机器字,由MLIR的affine使用，可用于表示size，dim和下标

int>

支持任意位宽整数类型，i/ui[1-9] [0-9]*

memref>

 The buffer pointed to by a memref can be allocated, aliased and deallocated

rank memref<?x?x?xtype>

unrank memref<*xtype> 目的是允许外部库函数接收任意rank的memref参数

affinemap>

用于dependence analysis, memory access pattern analysis, vectorization, copy elision and in-place updates等，affinemap是memref类型的一部分，默认为identity affine map

tensor>

更高层次的抽象（ aggregate N-dimensional data values），无法控制布局或获取数据指针，低层次的数据访问使用memref

tuple> 类型集合，元素可以具有不同类型

vector> 表示SIMD向量，使用目标特定的Op如AVX等，维度信息是静态已知的

**标准属性**

属性用于在不允许变量的位置，来指定编译时常量数据，属性的含义由上下文确定，标准属性是内置的核心属性集

### 2.2 Dialects

#### 2.2.1 affine 

provides a powerful abstraction for affine operations and analyses

symbol标识符表示未知量，可以将其视为感兴趣区域的常数

affine_map (dim...)[sym...] -> affine_expr 映射关系（仿射函数非正式地是线性函数加常数）

affine_set (dim...)[sym...] -> dim... sym...关系约束

```mlir
affine.yield 
	每次循环将返回值传递到迭代变量，最后次返回for返回值
	if-then/else 返回if返回值
affine.parallel	
	通过affine.yield产生的每个值将通过AtomicRMWKind枚举中定义的规约方法进行accumulated/reduced
```

#### 2.2.2 std & scf

#std的拆解不同部分 (控制流 , 标量 , vector , memref， tensor，dma/io)

**std**

多个不同概念的Op set,可以被划分为多个不同的 more-focused dialects

数值运算 可以在attributes 上附加 for fast math, contraction, rounding mode, and other controls等信息

alloc : allocates a region of memory,返回memref类型

dealloc: frees a region of memory

alloca: allocates memory on the stack， 离开作用域自动释放(AutomaticAllocationScope trait)

constant:定义生成常量SSA Value, 如定义一个函数引用间接调用函数

dim: return dim:index 

rank: return rank:index

select: select %cond ,%true,%false : type

splat:Broadcast the operand to all elements of the result vector or tensor.

tensor_load:Create a tensor from a memref

tensor_store: Stores the contents of a tensor into a memref, 不创建memref

dma_start:非阻塞内存传输，使用%tag[%idx]标识关联一个DMA操作

dma_wait:阻塞%tag[%idx]标识的DMA操作直到完成

 view: 从1维连续i8 memref生成一个N维连续任意类型memref(affine_map is default  identity)

src是满足一维连续内存的i8型memref (affine_map is default  identity，type is i8)

**scf**

std dialect没有高层次控制流，直接使用的是br 和cond br，scf提供了高层次的控制流Op

for 可以传递参数，以及[lb,ub,step)等信息到循环内，使用scf:yield返回循环内的值

if 如果if内有def value 则必须有else,且使用scf:yield传出需要的值

#### **2.2.3 llvm dialect**

包含一个LLVM Context and an LLVM Module( These objects can be obtained from the dialect object using `.getLLVMContext()` and `.getLLVMModule()`)，用于print, parse and manage LLVM IR types与LLVM IR objects交互

**mlir LLVM::LLVMType** 

​	wrapped LLVM IR type ； 包含了llvm::Type* ， 使用.getUnderlyingType()可以获取到， 底层对象 allocated within the LLVM context

​	可以从llvm::Type*构造 LLVM::LLVMType（从内核构造外壳对象）

**Operation** 

llvm.func Attribute 

​		pass-through 使用MLIR的Attribute将LLVM IR函数所需要的信息以字符串传递

​		Linkage 与llvm ir中的func的链接属性对应

llvm.* operation与llvm ir中的同名指令对应

llvm.mlir.*是llvm dialect中和llvm ir不能直接对应的Operation

​	llvm.mlir.constant

​	llvm.mlir.global [linkage属性] [const] 定义全局变量/常量

​	llvm.mlir.addressof 取地址，产生一个SSA Value为指针

### 2.3 Rationale

### 2.4 ODS

automatically generate operand/attribute/result getter methods, operation build methods, operation verify methods, ...

后端 OpDefinitionsGen

基础构造 OpBase.td 包含 dialect ,type, attr, op , 谓词，基于谓词构造的各种约束，匹配模式， Interface的基础定义结构

**2.4.1 Op**

string opName = mnemonic;

​	唯一的标识，用于以文本格式进行解析和打印，也用于模式匹配

dag arguments = (ins);

```
let arguments = (ins
  <type-constraint>:$<operand-name>,
  ...
  <attr-constraint>:$<attr-name>,
  ...
);
```

​	ins 标识了输入参数，两种参数。oprands 运行时产生的值，attributes 编译时知道的值。

​    attributes

​    		自然属性：定义Op时需要指定的属性，这些属性会影响Op的行为（例如，卷积的填充）
​			派生属性：定义Op不需要这些属性，信息是从Op派生的

​	操作数和属性的相对顺序没有要求，操作数的相对顺序本身很重要。 
​    从每个命名的参数中，将生成一个命名的getter，该方法将返回具有返回类型的参数（对于属性，返回类型将从存储类型构造，而对于操作数，为mlir::Value）；Each attribute’s raw value (e.g., as stored) can also be accessed via generated <name>Attr getter

​	可变数量Oprand 

​		wrap the `TypeConstraint` for the operand with `Variadic<...>`

​	Optional operands  

​		wrap the `TypeConstraint` for the operand with `Optional<...>`

​	Optional attribute

​	wrap the `AttrConstraint` for the attribute with `OptionalAttr<...>`.

list<OpTrait> traits = props;

​	Both operation traits, [interfaces](https://mlir.llvm.org/docs/OpDefinitions/#operation-interfaces) , and constraints involving multiple operands/attributes/results are provided as the second template parameter to the `Op` class. They should be deriving from the `OpTrait` class.

   traits, interfaces , and constraints都通过第二个模板参数提供都是OpTrait 的子类

**traits**
    in namesapce mlir::OpTrait, traits提供影响语法或语义的信息

**Interfaces**

`AttrInterface`, `OpInterface`, or `TypeInterface` class will auto-generate the C++ classes for the interface

ConcreteOp is an implicitly defined typename that can be used to refer to the type of the derived operation currently being operated on.

```
def MyInterface : OpInterface<"MyInterface"> {
  let description = ...;
  let methods = [...];
}
// OpInterface represents an interface registered to an operation.
class OpInterface<string name> : Interface<name>, OpInterfaceTrait<name>

// OpInterfaceTrait corresponds to a specific 'OpInterface' class defined in
// C++. The purpose to wrap around C++ symbol string with this class is to make
// interfaces specified for ops in TableGen less alien and more integrated.
class OpInterfaceTrait<string name, code verifyBody = [{}]>
          : NativeOpTrait<""> {
  let trait = name # "::Trait";

  // Specify the body of the verification function. `$_op` will be replaced with
  // the operation being verified.
  code verify = verifyBody;

  // An optional code block containing extra declarations to place in the
  // interface trait declaration.
  code extraTraitClassDeclaration = "";
}

// Interface represents a base interface.
class Interface<string name> {
  // A human-readable description of what this interface does.
  string description = "";

  // The name given to the c++ interface class.
  string cppClassName = name;

  // The C++ namespace that this interface should be placed into.
  //
  // To specify nested namespaces, use "::" as the delimiter, e.g., given
  // "A::B", ops will be placed in `namespace A { namespace B { <def> } }`.
  string cppNamespace = "";

  // The list of methods defined by this interface.
  #InterfaceMethod and StaticInterfaceMethod, 后者派生自InterfaceMethod专门生成静态c++方法
  list<InterfaceMethod> methods = [];

  // An optional code block containing extra declarations to place in the
  // interface declaration.
  code extraClassDeclaration = "";
}

// This class represents a single, optionally static, interface method.
// Note: non-static interface methods have an implicit parameter, either
// $_op/$_attr/$_type corresponding to an instance of the derived value.
class InterfaceMethod<string desc, string retTy, string methodName,
                      dag args = (ins), code methodBody = [{}],
                      code defaultImplementation = [{}]> {
  // A human-readable description of what this method does.
  string description = desc;

  // The name of the interface method.
  string name = methodName;

  // The c++ type-name of the return type.
  string returnType = retTy;

  // A dag of string that correspond to the arguments of the method.
  dag arguments = args;

  #ConcreteOp is an implicitly defined typename that can be used to refer to the type of the derived operation currently being operated on.
  
  // An optional body to the method.
  code body = methodBody;

  // An optional default implementation of the method.
  code defaultBody = defaultImplementation;
}
`
// This class represents a single static interface method.
class StaticInterfaceMethod<string desc, string retTy, string methodName,
                            dag args = (ins), code methodBody = [{}],
                            code defaultImplementation = [{}]>
    : InterfaceMethod<desc, retTy, methodName, args, methodBody,
                      defaultImplementation>;

```

**Builder**

默认根据Op的定义生成多个builder，若不满足可以通过 let builders自定义builder

使用OpBuilderDAG / OpBuilder 来自定义builder，使用`$_builder` and `$_state`引用默认添加的两个参数

OpBuilderDAG的参数是DAG, OpBuilder的参数是字符串， 不推荐使用 OpBuilder , 新的OpBuilderDAG代替它 

```
let builders = [
    OpBuilderDAG<(ins CArg<"float", "0.5f">:$val), [{
      $_state.addAttribute("attr", $_builder.getF32FloatAttr(val));
    }]>

// 将生成
build(::mlir::OpBuilder &builder, ::mlir::OperationState &state,
            float val) {
  state.addAttribute("attr", builder.getF32FloatAttr(val));
}
```

 **verifier code**

验证代码自动从约束相关信息自动生成，可以指定自定义的验证，生成的自定义代码将在自动生成的验证之后调用

生成的c++代码

​	每一个Op都会生成 an operation class and an [operand adaptor](https://mlir.llvm.org/docs/OpDefinitions/#operand-adaptors) class

​	GET_OP_LIST a comma-separated list of all defined ops

​	GET_OP_CLASSES   all the op method definitions

**自定义汇编格式**

1.Declarative Assembly Format

```
  let assemblyFormat = [{
    $callee `(` $args `)` attr-dict `:` functional-type($args, results)
  }];
```

Directives
A type of builtin function, with an optional set of arguments.

Literals

``括起来的关键字或标点符号，以下是有效的标点集：

：，,, =，<，>，（，），{，}，[，]，->

Variables
An entity that has been registered on the operation itself, i.e. an argument(attribute or operand), result, successor, etc. 

2.自定义对应的printer和parser

```
  // td中
  let printer = [{ return ::print(printer, *this); }];
  let parser = [{ return ::parse$cppClass(parser, result); }];
  // 对应在c++中实现
  static void print(mlir::OpAsmPrinter &printer, PrintOp op) {...}
  static mlir::ParseResult parsePrintOp(mlir::OpAsmParser &parser,
                                      mlir::OperationState &result) {...}
  
```

**2.4.2 Constrain**

​	op验证和模式匹配都要满足约束

​	单实体约束  约束单个operand, attribute, or result，直接在ins/outs位置指定

​	多实体约束 约束涉及Op多个operand/attribute/result ，描述实体之间的关系，此约束modeled as `PredOpTrait`，在Op模板参数指定

   Op自身的约束（如：无副作用）

自定义约束 - 约束是带描述的谓词

```
class Constraint<Pred pred, string desc = ""> {
  // The predicates that this constraint requires.
  Pred predicate = pred;
  // User-readable description used in error reporting messages. If empty, a
  // generic message will be used.
  string description = desc;
}
// CPred是原子谓词，是c++的接口
class CPred<code pred> : Pred {
  code predExpr = "(" # pred # ")";
}
```

**debug** 

```sh
# To see op C++ class declaration
mlir-tblgen --gen-op-decls -I /path/to/mlir/include /path/to/input/td/file
# To see op C++ class definition
mlir-tblgen --gen-op-defs -I /path/to/mlir/include /path/to/input/td/file
# To see op documentation
mlir-tblgen --gen-dialect-doc -I /path/to/mlir/include /path/to/input/td/file

# To see op interface C++ class declaration
mlir-tblgen --gen-op-interface-decls -I /path/to/mlir/include /path/to/input/td/file
# To see op interface C++ class definition
mlir-tblgen --gen-op-interface-defs -I /path/to/mlir/include /path/to/input/td/file
# To see op interface documentation
mlir-tblgen --gen-op-interface-doc -I /path/to/mlir/include /path/to/input/td/file
```

### 2.5 DRR

指定重写规则，会自动生成等价的mlir :: RewritePattern子类 

drr擅长描述Op模式到Op模式的匹配转换，但也有不擅长的地方**；注：mlir定义了模式匹配dialect提供了更强的模式匹配描述能力（不使用td描述模式匹配，而是使用mlir描述模式匹配）**

```
class Pattern<
    dag sourcePattern, list<dag> resultPatterns,
    list<dag> additionalConstraints = [],
    dag benefitsAdded = (addBenefit 0)>;
# 1 vs 1, src->dst
class Pat<
    dag sourcePattern, dag resultPattern,
    list<dag> additionalConstraints = [],
    dag benefitsAdded = (addBenefit 0)> :
  Pattern<sourcePattern, [resultPattern], additionalConstraints, benefitAdded>; 
```

source pattern 	

​	dag参数用于捕获参数或进一步约束（捕获什么样的参数，如类型/值约束等）

​	Pat <（AOp $ _，$ b）...  $ _是一个特殊符号，表示忽略捕获参数

​	Pat <（AOp $ a，F32Attr）...  仅指定约束不捕获符号

​	Pat<(AOp (BOp:$b_result) $attr), (COp $b_result $attr)>, 附加到BOp上的$b_result捕获BOp的输出，**此适用于BOp仅有一个输出**

 result pattern

​	dag参数时引用source pattern捕获的参数，传递给result pattern的build

`NativeCodeCall` 用在Pattern中用于包装c++函数实现自定义处理

​	`$_builder` will be replaced by the current `mlir::PatternRewriter`，which is a subclass of `mlir::Builder`

​	`$_self` will be replaced with the entity `NativeCodeCall` is attached to

​	$N substituted by the `dag` object parameters

**auxiliary ops** （多个Op）

多个结果list<dag> resultPatterns，列表包含多个Op。列表中仅最后一个Op最终用于替换src,其它的Op都是中间结果，辅助生成最后一个Op的Op。

**multi-result ops** (一个Op有多个返回result)

多个结果的td声明式，使用了一种定死的命名约定

```
def HasNoUseOf: Constraint<CPred<"$_self.use_empty()">, "has no use">;

def HasSameElementType : Constraint<
    CPred<"$0.cast<ShapedType>().getElementType() == "
          "$1.cast<ShapedType>().getElementType()">,
    "has same element type">;

def : Pattern<(TwoResultOp:$results $input),
              [(...), (...)],
              [(F32Tensor:$results__0), (HasNoUseOf:$results__1),
               (HasSameElementShape $results__0, $input)]>;
这里results_0 和 $results__1 引用TwoResultOp的第0个和第1个参数
捕获的TwoResultOp第1个参数result__1附加到HasNoUseOf:$results__1， HasNoUseOf定义的
$_self即表示了它附加的对象，在这个位置为$results__1
/*
dag的Op或arg上附加的$name不表示此arg/op为这个名字，而需要看使用的位置，它可能标识了一种op、arg等之间的引用关系
比如(HasNoUseOf:$results__1)，$results__1引用了前面的TwoResultOp的第1个参数,它attach到了 HasNoUseOf之上， $name时一种attach tag
*/
```

**benefits**

标识了模式的优先级，higher benefit is applied before one with a lower benefit

基本原则是

- Larger matches are more beneficial than smaller ones.
- If a smaller one is applied first the larger one may not apply anymore

debug

```
mlir-tblgen --gen-rewriters -I /path/to/mlir/include /path/to/input/td/file
# Compilation error: no matching member function for call to ‘build’, 匹配不到Op的build函数
```

### 2.6  DAG-to-DAG Rewriting

三要素 ：what? pattren  when ? benefit how? rewriter

widely used throughout MLIR for **canonicalization, conversion, and general transformation**、

Pattern Definition 

​	RewritePattern时模式的基类

​	benefit  应用给定模式的预期好处，benefit 在构建模式时是静态的，但可以在模式初始化时动态计算

​	match不改变IR

​	rewrite 必须由PatternRewriter类执行

​	**Pattern Rewriter**

​		提供一系列方法updated in-place, replaced, or erased

​		继承自OpBuilder，提供所有的OpBuilder API,创建op，type，attr等

Pattern Application **Driver**

​	一系列定义好的模式，被集合到一个特定的driver提供给 application，driver包含以下几个部分:

​	OwningRewritePatternList 输入的模式被collet集合放在了此List中

​	Driver-specific `PatternRewriter` 

​		driver must provide a PatternRewriter instance with the necessary hooks

​		为了确保驱动程序状态不会因 PatternRewriter中的IR突变而失效，驱动程序必须提供一个有必要 hook 的PatternRewriter实例，以hook某些IR变化； 如果驱动程序不需要钩入某些突变，将直接执行突变

​	定义代价模型

​	**PatternApplicator** patterns和代价模型以及`PatternRewriter` 都给了它，由它执行PatternApplicator :: matchAndRewrite,	driver包含一组patterns, Driver级别的rewriter被提供给PatternApplicator :: matchAndRewrite，它会遍历patterns将rewriter传递给每个pattern的matchAndRewrite函数

### 2.7 Pass Infrastructure

mlir all is op,All passes in MLIR derive from `OperationPass` 

pass manager is designed to work on instances of operations at different levels of nesting

 **Op-Specific pass**

特定于OPp的pass明确地对给定操作类型进行操作

```
struct MyFunctionPass : public OperationPass<MyFunctionPass, FuncOp>
```

**Op-Agnostic pass**

```
struct MyOperationPass : public OperationPass<MyOperationPass> 
```

**Analysis**

在特定Op上计算信息，不修改IR,不同于llvm的**Analysis**是pass，mlir的pass是独立的class

OperationPass类提供API，用于查询和保留当前处理的Op的**Analysis**,在 runOnOperation中可以调用它们来获取和标记保留

```
/// An interesting analysis.
struct MyOperationAnalysis {
  // Compute this analysis with the provided operation.
  MyOperationAnalysis(Operation *op);
};

void MyOperationPass::runOnOperation() {
  
  //查询
  // Query MyOperationAnalysis for the current operation.
  MyOperationAnalysis &myAnalysis = getAnalysis<MyOperationAnalysis>();

  // Query a cached instance of MyOperationAnalysis for the current operation.
  // It will not be computed if it doesn't exist.
  auto optionalAnalysis = getCachedAnalysis<MyOperationAnalysis>();
  if (optionalAnalysis)
    ...

  // Query a cached instance of MyOperationAnalysis for the parent operation of
  // the current operation. It will not be computed if it doesn't exist.
  auto optionalAnalysis = getCachedParentAnalysis<MyOperationAnalysis>();
  if (optionalAnalysis)
    ...
  // 标记保留
  // Mark all analyses as preserved. This is useful if a pass can guarantee
  // that no transformation was performed.
  markAllAnalysesPreserved();

  // Mark specific analyses as preserved. This is used if some transformation
  // was performed, but some analyses were either unaffected or explicitly
  // preserved.
  markAnalysesPreserved<MyAnalysis, MyAnalyses...>();
}
```

**signalPassFailure**

signal a failure to the pass manager， -> no other passes in the pipeline will execute and the `PassManager::run` will return failure

#一种自定义的的异常throw机制

**OpPassManager**

pass的结合，不能直接创建必须由其它**OpPassManager**以nest的方法创建，最顶层是PassManager作用于ModuleOp, **OpPassManager**的嵌入结构对应了MLIR的嵌入结构

Passes are expected to not modify operations at or above the current operation being processed?

一系列pass会首先在一个Op上都执行，然后再另一个Op执行，这样利于cache友好，以及pipeline的异步执行

**‘Pass::Statistic’ class**

​	可定义在pass内，辅助统计信息， constructor arguments: (the parent pass, a name, and a description.)

PassInstrumentation

​	provides **hooks into the PassManager** that **observe various pass events**

​    [PassManager](https://mlir.llvm.org/docs/PassManagement/#pass-manager)::`addInstrumentation` 方法将PassInstrumentation加入到pm的pipenline

​	mlirt提供了一些写好的PassInstrumentation，如时间统计，IR打印等方便分析和调试

### 2.8 Interfaces

接口为方言和操作提供了一种通用机制，可为转换或分析提供信息

转换和分析通过 interfaces这种generic way of interacting with the IR，无需关心方言和Op的特定信息，即信息的获得通过Op和方言关联的 interfaces

interfaces不和转换和分析关联是和Op和方言关联，解耦了Op和转换和分析，使转换和分析的代码规范化

**Dialect Interfaces**

继承DialectInterface::Base定义Dialect 级别Interface，使用 addInterfaces接口，注册到方言中

**Attribute/Operation/Type interfaces**

 These interfaces provide access to derived objects by providing a virtual interface that must be implemented

### 2.9 Traits

抽象共有实现，为对象（Op，Type, Attr等）指定约束和属性，附加hook方法等，某些pass（内置或自定义pass）需要的信息或特质以trait形式附加在对象上，以便pass访问并处理

基类： For attributes, this is `AttributeTrait::TraitBase`. For operations, this is `OpTrait::TraitBase`.

trait在附加在对象上时会直接成为它们的基类，可以直接访问

内置trait

​	IsolatedFromAbove  断言一个Op的regions不捕获或引用defined above the region scope（区域之外定义的）的SSA values

​	Function-Like  provides APIs for operations that behave like functions

​	Commutative 可交换的，满足 x op y == y op x

​	AutomaticAllocationScope  allocations are **automatically freed** when control is transferred back from the regions of such operations

   NoSideEffect 无副作用， 无副作用的无用Op才可以被 canonicalization pass消除

### **2.10 Operation Canonicalization**

mlir有单一的 canonicalization pass（llvm有多个不同的pass, 如instcombine, dag combine等），它以贪婪的方式迭代地应用规范化转换，直到IR收敛为止; 这种局部转换由Op自己定义，有canonicalization pass回调Op注册的pattern或提供的fold函数.

canonicalization pass 执行的转换有

​	消除无副作用的无用Op (死节点消除)

​	常量折叠，  Constant folding hooks are specified by operations.

​	4+x -> x+4 right side

​    constant-like operations are uniqued and hoisted into the entry block of the first parent barrier region.

**getCanonicalizationPatterns**

**！！！pattern描述了一种局部模式与转换，是pass的辅助组件**

getCanonicalizationPatterns  providing canonicalizations as a set of `RewritePattern`s, 此方法插入的patterns**由canonicalization pass调用实现局部转换**

Op自己定义特定的规范化转换步骤:

​	0. let hasCanonicalizer = 1

​	1. 使用ddr/c++定义patterns

​	2. 在算子 getCanonicalizationPatterns 方法中注册patterns

```
# set 1 会生成一个 MyOp::getCanonicalizationPatterns的函数声明
def MyOp : ... {
  let hasCanonicalizer = 1;
}

#cpp自定义实现，将所有自定义的RewritePattern插入到patterns中
Canonicalization pass会使用注册的RewritePattern进行变换
void MyOp::getCanonicalizationPatterns(OwningRewritePatternList &patterns,
                                       MLIRContext *context) {
  patterns.insert<...>(...);
}
```

**Canonicalizing with fold**	

fold不局限于在**canonicalizer pass**使用， 如可以OpBuilder::createOrFold方法在创建Op时直接调用

约束

​	不允许创建新Op，不允许生成新的Value; updating an operation in-place；真正的局部转换，局限于根Op

​     可以returning a set of pre-existing values (or attributes) to replace the operation 

```
ArrayRef<Attribute> operands用于表示operands are constant
SmallVectorImpl<OpFoldResult>用于表示结果为an SSA Value, or an Attribute(for a constant result)， If an SSA Value is provided, it must correspond to an existing value. The fold methods are not permitted to generate new Values

LogicalResult MyOp::fold(ArrayRef<Attribute> operands,
                         SmallVectorImpl<OpFoldResult> &results) {
  ...
}
```

当fold方法返回一个Attribute作为结果时，它表示该结果是“常量”。需要定义一个materializeConstant hook方法，此hook接收由fold返回的Attribute值，产生一个“constant-like” operation**，通过此Operation常量属性转换为SSA Value**

### 2.11 Dialect Conversion

**Mode**

 applyPartialConversion 仅转换mark 为非法的operation，未mark的不转换

 applyFullConversion  对输入的所有operation 合法化转换

 applyAnalysisConversion **没有真的执行转换**，仅分析和记录哪些operation可以成功合法化到给定的 Target

**A Conversion Target**

​	通过以系列mark/add方法来标记非法/合法/动态合法

​	动态合法允许定义更细致的约束，If a specific handler is not provided when setting the action**, the target must override the isDynamicallyLegal hook** provided by ConversionTarget

​	markOpRecursivelyLegal 所有嵌入在此Op的其它Op也被认为是合法的，即使这些Op被视为非法

**A set of Rewrite Patterns**

​	该框架将自动构建一个转换图(传递闭包)，以将非合法操作转换为一组合法操作

​    假如有有patterns: [`bar.add` -> `baz.add`, `baz.add` -> `foo.add`], 即使不存在pattern[`bar.add` -> foo.add]框架也会自动检测到它可以使bar.add-> foo.add合法化

​	**ConversionPattern**

​		在常规的RewritePattern类之外，转换框架提供了一种特殊类型的**ConversionPattern**

​		用于when a pattern relies on interacting with constructs specific to the conversion process(模式依赖于与特定于转换过程的构造交互)

​	例如:  要匹配的Op的操作数类型将与用户期望的操作数不对应，additional  `operands` parameter, containing the remapped operands of the original operation

**A Type Converter** (Optional)

​	提供给转换模式的remapped operands必须是该模式期望的类型 ， TypeConverter object may be defined that details how types should be converted when interfacing with a pattern； a **materialization** can produce IR, whereas a **conversion** cannot；

​	source materialization  将具有“合法”Target类型的value转换回特定的Source类型；转换后的值被未转换的操作使用，则需要将其转换回源类型

​	target mterialization  将具有“非法”源类型的值转换为“合法”类型的值，转换的操作使用了未转换的值，则需要将其转换为目标类型

​	确保在转换过程中保留IR的类型，这意味着在转换过程中Value的类型不会隐式更改。更改Value的类型时，还必须在转换过程中更新该Value的user。If they aren’t, a **type conversion** must be materialized to **ensure** that a value of **the expected type is still present** within the IR.

块参数类型转换

​	ConversionPatternRewriter调用自定义hook -> convertRegionTypes ,它使用提供的type converter的argument materialization hook 对区域内的所有块参数转换; 还有一个可选的TypeConverter :: SignatureConversion参数，将**自定义转换应用于该区域的输入块**(输入块参数可能需要特别处理)， 若**仅转换入口块的签名**, 不转换任何其他块的签名，可以改用applySignatureConversion hook

### 2.12 Conversion => LLVM Dialect

llvm dialect 可以翻译为llvm ir, 其它dialect 可转换为llvm dialect，因此自定义dialect如果可以则选择转换为现存的dialect，然后利于他们的patterns  transitive lowering 为llvm dialect =》llvm ， 仅针对特殊情况自定义lower



