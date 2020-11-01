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

Declarative Rewrite Rule (DRR)> 通过如TableGen的形式定义重写规则，自动生成mlir::RewritePattern

Dialect>

扩展mlir系统的一组功能，创建一个namespace定义了op,type,attr

Operation>

An operation can have zero or more regions . Note that this creates a nested IR structure, as regions consist of blocks, which in turn, consist of a list of operations.

Module>顶层结构，包含一个region的 op, region内包含一个block

Function> 包含一个region的op, 不允许捕获函数外部定义的值

Region> A [CFG](https://en.wikipedia.org/wiki/Control-flow_graph) of MLIR [blocks](https://mlir.llvm.org/getting_started/Glossary/#block) . 

### 1.3 测试 

check文本形式测试

[FileCheck](https://llvm.org/docs/CommandGuide/FileCheck.html) 

​	reads two files standard input and specified on the command line

​	uses one to verify the other

Lit

gtest单元测试

​	非文本的形式测试，对API接口，缺点是若API用户接口不稳定，每次改动接口会影响一大批测试要重写

​	文本形式的测试，不影响接口改动

### 1.4 调试

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

类型系统>

开放类型系统,且类型可能具有特定于应用程序的语义(同一个类型可能在不同Target/App中由不同的解释用法)

type aliases

``` 
INT32 = type i32
```

dialect可以自定义类型

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

layout specification 使用 affine map表达，也封装了offset,stride形式的stride-list的语法糖写法

memref dimension定义的是索引空间，

index map 一对一的仿射变换

layout map 逻辑索引空间到物理索引空间的映射，

Strided MemRef 编码了距离

用于dependence analysis, memory access pattern analysis, vectorization, copy elision and in-place updates等，默认为identity affine map

tensor>

更高层次的抽象（ aggregate N-dimensional data values），无法控制布局或获取数据指针，低层次的数据访问使用memref

tuple> 类型集合，元素可以具有不同类型

vector> 表示SIMD向量，使用目标特定的Op如AVX等，维度信息是静态已知的

**标准属性**

属性用于在不允许变量的位置，来指定编译时常量数据，属性的含义由上下文确定，标准属性是内置的核心属性集

### 2.2 Dialects

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

**llvm dialect**

wraps the LLVM IR types and instructions into MLIR types and operations

包含一个LLVM Context and an LLVM Module，用于print, parse and manage LLVM IR types与LLVM IR objects交互

Type 使用!llvm<>包装llvm 的类型

Operation 以llvm.前缀，Operation和 LLVM IR的指令有对应关系

llvm.func

Attribute

​	pass-through 使用MLIR的Attribute将LLVM IR函数所需要的信息以字符串传递

​	Linkage 与llvm ir中的func的链接属性对应

llvm.* operation与llvm ir中的同名指令对应

llvm.mlir.*是llvm dialect中和llvm ir不能直接对应的Operation

​	llvm.mlir.constant

​    llvm.mlir.global [linkage属性] [const] 定义全局变量/常量

​    llvm.mlir.addressof 取地址，产生一个SSA Value为指针

**vector**

VV->HWV->LLVM IR

LLVM IR level vector<4x8x128xf32> lower 为 !llvm<[4 x [ 8 x [ 128 x float ]]]>,对应了llvm 的array type

HWC level 离硬件更近的dialect, 如GPU的NVVM dialect,或 LLVM dialect

VV level 

​	Use Standard and Vector Dialect Ops，	无论是否存在自动矢量化程序，都可以基于VectorOps方言构建一种概念上的矢量语言

​	vector<HW-specific sizes x k x f32>, vector的某些HW-specific sizes在特定硬件特性匹配则能生成更高性能的代码，如GPU的wrap size，Avx 的寄存器位宽

​	lower to LLVM IR 的权衡

​		llvm 仅支持1-D的vector类型，n-D vector(`vector<s_0x...s_{n-1}xf32>`)->llvm的形式：

​		1.`!llvm<"(s_0*...*s_{n-1})xfloat">`  flatten 1-D vector

​		2.`!llvm<"[s_0x[s_1x[...<s_{n-1}xfloat>]]]">`Nested aggregate type of `1-D` vector

​		Conclusion：Nested aggregate type of `1-D` vector更好(#个人观点:从vector到flatten 形式丢失了一些信息，不利于特殊优化)

### 2.3 Rationale

#### 2.3.1 MLIR Rationale

​	MLIR的思想来自用于较低层次的构造LLVM和Swift的IR，同时将它们与多面体抽象的思想相结合，以表示循环嵌套，多维张量数据以及这些实体上的变换， 表达和优化deep loop nests and dense matrices of high dimensionality， introduces notions from the polyhedral loop optimization works as first class concepts

​	**poly**映射，集合以及具有仿射约束的关系是高维循环嵌套和多维数组的多面体表示基础的核心结构。这些structures以接近其数学形式的形式表示为文本表达式。**这些structures用于捕获循环嵌套，张量数据结构**，以及如何**针对目标体系结构对它们进行重新排序和映射**。所有structures或“符合”循环都作为多面体信息的一部分被捕获，张量变量，它们的布局以及对这些张量在内存中的下标访问也是如此。

​	**传统的循环转换**（单模和非单模），例如循环平铺，交换，置换，倾斜，缩放，相对移位，反转，融合和分布/裂变。**数据布局的转换**（如填充和转换为块状布局）也可以通过仿射布局图很好地表示。

​	在较高层次支持硬件相关的低级优化，如映射到**专用向量指令，自动向量化和软件流水线**，由于当前很多神经网络加速器具有专门的单元来处理大数据块,这样的专用单元或指令对多维数据块进行操作, 因此在接近汇编的低级别IR上以提升和重构循环的方法来并执行映射到专用指令非常困难， （经典的指令选择和调度主要只处理最内层的循环体，而多维数据块进行操作进行操作的指令跨越多个循环体，在内存层次结构的较高级别上对数据进行操作）

​	LLVM从类型系统删除整数的符号，而使用指令区分带符号/不带符号整数

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
# 前者Interface通过ODS在td中定义OpInterface
# 后者OpInterfaceTrait包装外部c++代码定义的OpInterface在ODS引用，将自己写的代码和ODS框架自动生成接起来
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

使用OpBuilder来自定义builder，使用`$_builder` and `$_state`引用默认添加的两个参数

```
class OpBuilder<string p, code b = ""> {
  string params = p;
  code body = b;
}

def MyOp : ... {
  ...

  let builders = [
    OpBuilder<"float val = 0.5f", [{
      $_state.addAttribute("attr", $_builder.getF32FloatAttr(val));
    }]>
  ];
}

// 将自动生成
static void build(OpBuilder &builder, OperationState &state, float val = 0.5f) {
  state.addAttribute("attr", builder.getF32FloatAttr(val));
}
```

 **verifier code**

验证代码自动从约束相关信息自动生成，可以指定自定义的验证，生成的自定义代码将在自动生成的验证之后调用

生成的c++代码

​	每一个Op都会生成 an operation class and an [operand adaptor](https://mlir.llvm.org/docs/OpDefinitions/#operand-adaptors) class

​	GET_OP_LIST a comma-separated list of all defined ops

​	GET_OP_CLASSES   all the op method definitions

**2.4.2 Constrain**

​	op验证和模式匹配都要满足约束

​	单实体约束  约束单个operand, attribute, or result，直接在ins/outs位置指定

​	多实体约束 约束涉及Op多个operand/attribute/result ，描述实体之间的关系，此约束modeled as `PredOpTrait`，在Op模板参数指定

​	trait 约束Op自身， modeled as `NativeOpTrait` ，在Op模板参数指定

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

**2.4.3 Attr **

生成代码(结构体和方法)提供对编译时的常量信息的存储、getter、和转换，以及基于约束生成验证。

**2.4.4 Type def **





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

widely used throughout MLIR for **canonicalization, conversion, and general transformation**、

Pattern Definition 

​	RewritePattern时模式的基类

​	benefit  应用给定模式的预期好处，benefit 在构建模式时是静态的，但可以在模式初始化时动态计算

​	match不改变IR

​	rewrite 必须由PatternRewriter类执行

​	**Pattern Rewriter**

​		提供一系列方法updated in-place, replaced, or erased

​		继承自OpBuilder，提供所有的OpBuilder API,创建op，type，attr等

```
class MyPattern : public RewritePattern {
public:
  /// This overload constructs a pattern that only matches operations with the
  /// root name of `MyOp`.
  #MyOp::getOperationName() 指定根Op匹配特定Op
  MyPattern(PatternBenefit benefit, MLIRContext *context)
      : RewritePattern(MyOp::getOperationName(), benefit, context) {}
      
  /// This overload constructs a pattern that matches any operation type.
  # 不指定根Op匹配特定Op时需要使用MatchAnyOpTypeTag()标明匹配anyop
  MyPattern(PatternBenefit benefit)
      : RewritePattern(benefit, MatchAnyOpTypeTag()) {}

  /// In this section, the `match` and `rewrite` implementation is specified
  /// using the separate hooks.
  #match不允许改变IR
  LogicalResult match(Operation *op) const override {
    // The `match` method returns `success()` if the pattern is a match, failure
    // otherwise.
    // ...
  }
  
  #所有IR突变（包括创建）都必须由给定的PatternRewriter执行
  The root operation is required to either be: updated in-place, replaced, or erased.
  
  void rewrite(Operation *op, PatternRewriter &rewriter) {
    // The `rewrite` method performs mutations on the IR rooted at `op` using
    // the provided rewriter. All mutations must go through the provided
    // rewriter.
  }

  /// In this section, the `match` and `rewrite` implementation is specified
  /// using a single hook.
  LogicalResult matchAndRewrite(Operation *op, PatternRewriter &rewriter) {
    // The `matchAndRewrite` method performs both the matching and the mutation.
    // Note that the match must reach a successful point before IR mutation may
    // take place.
  }
};
```

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

转换和分析 通过 interfaces这种generic way of interacting with the IR，无需关心方言和Op的特定信息，即信息的获得通过Op和方言关联的 interfaces

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

### **2.10 Operation Canonicalization**

MLIR has a single canonicalization pass，它以贪婪的方式迭代地应用规范化转换，直到IR收敛为止

所有IR都会执行的转换

​	消除无副作用的无用Op

​	常量折叠

​	4+x -> x+4 right side

特定的规范化转换由Op自己定义

​	getCanonicalizationPatterns  providing canonicalizations as a set of `RewritePattern`s

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

​	fold	

​		updating an operation in-place

​        returning a set of pre-existing values (or attributes) to replace the operation 

​		可以在canonicalizer pass之外直接使用，真正的局部转换，不需要 pattern rewriter

```
ArrayRef<Attribute> operands用于表示operands are constant
SmallVectorImpl<OpFoldResult>用于表示结果为an SSA Value, or an Attribute(for a constant result)， If an SSA Value is provided, it must correspond to an existing value. The fold methods are not permitted to generate new Values

LogicalResult MyOp::fold(ArrayRef<Attribute> operands,
                         SmallVectorImpl<OpFoldResult> &results) {
  ...
}
```

### 2.11 Tutorials

#### **2.11.1 Dialect CMake**

```
add_mlir_dialect_library(MLIRLLVMIR         #Dialect target名字 MLIRXXX
  IR/LLVMDialect.cpp
  IR/LLVMTypes.cpp
  IR/LLVMTypeSyntax.cpp

  ADDITIONAL_HEADER_DIRS
  ${MLIR_MAIN_INCLUDE_DIR}/mlir/Dialect/LLVMIR

  DEPENDS             #tablegen生成并编译的的target
  MLIRLLVMOpsIncGen
  MLIRLLVMConversionsIncGen
  MLIROpenMPOpsIncGen
  intrinsics_gen #dialects that depend on LLVM IR may need to depend on the LLVM ‘intrinsics_gen’ target to ensure that tablegen’d LLVM header files have been generated
 
  LINK_COMPONENTS    #llvm中的依赖target
  AsmParser
  BitReader
  BitWriter
  Core

  LINK_LIBS PUBLIC   #mlir中的依赖target
  MLIRCallInterfaces
  MLIRControlFlowInterfaces
  MLIRIR
  MLIRSideEffectInterfaces
  MLIRSupport
  )
 
# MLIRLLVMOpsIncGen由LLVMOps.td调用mlir_tablegen生成的一系列.inc编译链接生成
set(LLVM_TARGET_DEFINITIONS LLVMOps.td)         
mlir_tablegen(LLVMOps.h.inc -gen-op-decls)
mlir_tablegen(LLVMOps.cpp.inc -gen-op-defs)
mlir_tablegen(LLVMOpsDialect.h.inc -gen-dialect-decls)
mlir_tablegen(LLVMOpsEnums.h.inc -gen-enum-decls)
mlir_tablegen(LLVMOpsEnums.cpp.inc -gen-enum-defs)
add_public_tablegen_target(MLIRLLVMOpsIncGen)

# 转换
add_mlir_conversion_library(MLIRLinalgToLLVM
  LinalgToLLVM.cpp

  ADDITIONAL_HEADER_DIRS
  ${MLIR_MAIN_INCLUDE_DIR}/mlir/Conversion/LinalgToLLVM

  DEPENDS
  MLIRConversionPassIncGen
  intrinsics_gen #直接包含LLVM IR头文件可能需要依赖LLVM的“ intrinsics_gen”目标，以确保已生成tablegen的LLVM头文件

  LINK_COMPONENTS
  Core

  LINK_LIBS PUBLIC
  MLIRAffineToStandard
  MLIREDSC
  MLIRIR
  MLIRLinalg
  MLIRLLVMIR
  MLIRSCFToStandard
  MLIRStandardToLLVM
  MLIRTransforms
  MLIRVectorToLLVM
  MLIRVectorToSCF
  )
```

#### **2.11.2 Toy**

**语言设计**

```
#设计文档 | lcy
#类型
tensor  var a<[3,2],double>
rank <= 2
基本类型 long, double
#Op
elementwise  + - * <3,2> op <3,2> => <3,2>
matrix * <3,2> * <2,3> => <3,3> 
#函数
函数声明是泛化的，仅知道是tensor,不知道rank,shape,datatype
在函数调用点，函数被特化
```

**Op vs Operation**

​	**'Operation'**类用于一般性地对所有operation建模。从某种意义上说，它不描述特定 operations的属性或 operations类型，因此它是“不透明的”, the **‘Operation’** class provides a general API into an operation instance

​	每种特定类型的operation都是**mlir::Op**的派生类，**Op** derived classes引用了一个**Operation**的指针， act as smart pointer wrapper around a **Operation**，**Op** derived classes提供了特定于operation的访问器方法和类型安全的属性，是一个interface for building and interfacing with the **Operation** class，它没有类成员，所有的数据结构信息存储在**Operation** class, 因此**Op** derived classes常***passing by value***

​	#Operation是存储信息的存储对象，Op是Operation的指针包装器

**Types**

​	Types in MLIR (like attributes, locations, and many other things) are value-typed.这意味着Type实例是按值传递的，而不是按指针或按引用传递的。 Type类本身充当内部存储对象的wrapper，**该内部存储对象(internal storage object that holds the actual data for the type)在MLIRContext实例中是唯一的**； ***所以这类wrapper对象本身就扮演了引用的概念***  # impl模式







​	



