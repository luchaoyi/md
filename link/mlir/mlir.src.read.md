**Dialect CMake**

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

**Dialect**

一组MLIR实体的概念组，包含和定义了Dialect实体共有的行为

dialect级别的interfaces, 会注册到dialect对象内的容器，其它实体的的注册对象，直接注册到了dialect关联的context对象中

```

/// Dialects are groups of MLIR operations, types and attributes, as well as
/// behavior associated with the entire group.  For example, hooks into other
/// systems for constant folding, interfaces, default named types for asm
/// printing, etc.
///
/// Instances of the dialect object are loaded in a specific MLIRContext.
class Dialect {
public:

  /// Registered hook to materialize a single constant operation from a given
  /// attribute value with the desired resultant type. This method should use
  /// the provided builder to create the operation without changing the
  /// insertion position.
  /// The generated operation is expected to be constant
  /// like, i.e. single result, zero operands, non side-effecting, etc. On
  /// success, this hook should return the value generated to represent the
  /// constant value. Otherwise, it should return null on failure.
  virtual Operation *materializeConstant(OpBuilder &builder, Attribute value,
                                         Type type, Location loc) {
    return nullptr;
  }

  //===--------------------------------------------------------------------===//
  // Parsing Hooks
  //===--------------------------------------------------------------------===//
  //===--------------------------------------------------------------------===//
  // Verification Hooks
  //===--------------------------------------------------------------------===//
  
  //===--------------------------------------------------------------------===//
  // Interfaces
  //===--------------------------------------------------------------------===//
  /// Lookup an interface for the given ID if one is registered, otherwise
  /// nullptr.
  const DialectInterface *getRegisteredInterface(TypeID interfaceID) {
    auto it = registeredInterfaces.find(interfaceID);
    return it != registeredInterfaces.end() ? it->getSecond().get() : nullptr;
  }
  template <typename InterfaceT> const InterfaceT *getRegisteredInterface() {
    return static_cast<const InterfaceT *>(
        getRegisteredInterface(InterfaceT::getInterfaceID()));
  }
  
protected:

  Dialect(StringRef name, MLIRContext *context, TypeID id);

  /// This method is used by derived classes to add their operations to the set.
  ///
  /*
    template <typenname T,typename... Args> void addOperations() {
      addOperation<T>();
      addOperations<Args>()...; 
    }
    template <typenname T> void addOperations() {
      addOperation<T>();
    }
    llvm的addOperations写法避免了递归展开的方式, 更加简洁
  */
  template <typename... Args> void addOperations() {
    // ","表达式求值是0，此处仅为了展开Args并依次调用addOperation函数
    (void)std::initializer_list<int>{0, (addOperation<Args>(), 0)...};
  }
  template <typename Arg> void addOperation() {
    addOperation(AbstractOperation::get<Arg>(*this));
  }
  void addOperation(AbstractOperation opInfo);

  /// Register a set of type classes with this dialect.
  template <typename... Args> void addTypes() {
    (void)std::initializer_list<int>{0, (addType<Args>(), 0)...};
  }

  /// Register a set of attribute classes with this dialect.
  template <typename... Args> void addAttributes() {
    (void)std::initializer_list<int>{0, (addAttribute<Args>(), 0)...};
  }
  
  /// Register a dialect interface with this dialect instance.
  void addInterface(std::unique_ptr<DialectInterface> interface);

  /// Register a set of dialect interfaces with this dialect instance.
  template <typename... Args> void addInterfaces() {
    (void)std::initializer_list<int>{
        0, (addInterface(std::make_unique<Args>(this)), 0)...};
  }

private:
  Dialect(const Dialect &) = delete;
  void operator=(Dialect &) = delete;
  
  /// The namespace of this dialect.
  StringRef name;

  /// The unique identifier of the derived Op class, this is used in the context
  /// to allow registering multiple times the same dialect.
  TypeID dialectID;

  /// This is the context that owns this Dialect object.
  MLIRContext *context;
  
  /// A collection of registered dialect interfaces.
  DenseMap<TypeID, std::unique_ptr<DialectInterface>> registeredInterfaces;

  friend void registerDialect();
  friend class MLIRContext;
};

```

Types，Attributes,Interfaces,Operations注册等是通过add*添加注册到Dialect关联的context中的。

add* 从具体Attributes,Interfaces,Operations class中提取必要的Info对应的Abstract* Info对象，最终Info对象**注册到了Dialect 对象构造时关联的context impl中**。

Abstract* Info对象也提供了lookup方法从context对象中查询注册的对象

```
/// A builtin dialect to define types/etc that are necessary for the validity of
/// the IR.
struct BuiltinDialect : public Dialect {
  BuiltinDialect(MLIRContext *context)
      : Dialect(/*name=*/"", context, TypeID::get<BuiltinDialect>()) {
    addTypes<ComplexType, ...>();
    addAttributes<AffineMapAttr, ...>();
    addInterfaces<BuiltinOpAsmDialectInterface,...>();
    // TODO: These operations should be moved to a different dialect when they
    // have been fully decoupled from the core.
    addOperations<FuncOp, ModuleOp, ModuleTerminatorOp>();
  }
  static StringRef getDialectNamespace() { return ""; }
};
} // end anonymous namespace.

/*
以addOperations<...>为例，从定义的具体Op的类型(模板参数T)中提取必要信息构造了AbstractOperation对象 OpInfo，
然后最终注册时将{opName,opInfo}注册到了dialect关联的context的impl中
*/
 template <typename T> static AbstractOperation get(Dialect &dialect) {
    return AbstractOperation(
        T::getOperationName(), dialect, T::getOperationProperties(),
        TypeID::get<T>(), T::parseAssembly, T::printAssembly,
        T::verifyInvariants, T::foldHook, T::getCanonicalizationPatterns,
        T::getInterfaceMap(), T::hasTrait);
  }

#MLIRContext.cpp
void Dialect::addOperation(AbstractOperation opInfo) {
  assert((getNamespace().empty() || opInfo.dialect.name == getNamespace()) &&
         "op name doesn't start with dialect namespace");
  assert(&opInfo.dialect == this && "Dialect object mismatch");
  auto &impl = context->getImpl();
  assert(impl.multiThreadedExecutionContext == 0 &&
         "Registering a new operation kind while in a multi-threaded execution "
         "context");
  StringRef opName = opInfo.name;
  if (!impl.registeredOperations.insert({opName, std::move(opInfo)}).second) {
    llvm::errs() << "error: operation named '" << opInfo.name
                 << "' is already registered.\n";
    abort();
  }
}

/// Look up the specified operation in the operation set and return a pointer
/// to it if present.  Otherwise, return a null pointer.
const AbstractOperation *AbstractOperation::lookup(StringRef opName,
                                                   MLIRContext *context) {
  auto &impl = context->getImpl();
  auto it = impl.registeredOperations.find(opName);
  if (it != impl.registeredOperations.end())
    return &it->second;
  return nullptr;
}
```

**DialectRegistry**

DialectRegistry将定义了一个map将方言名称空间映射到与方言匹配的构造函数实现了“可用”的方言与context中加载的方言解耦, **此class由context使用将可用的方言按需延迟加载到context**

```
/*
定义了一个map存储string:{TypeId,DialectAllocatorFunction}，DialectAllocatorFunction函数仅调用了return ctx->getOrLoadDialect<ConcreteDialect>()，allocate一个方言对象所有权在ctx对象
*/
class DialectRegistry {
  using MapTy =
      std::map<std::string, std::pair<TypeID, DialectAllocatorFunction>>;

public:
  template <typename ConcreteDialect>
  void insert() {
    insert(TypeID::get<ConcreteDialect>(),
           ConcreteDialect::getDialectNamespace(),
           static_cast<DialectAllocatorFunction>(([](MLIRContext *ctx) {
             // Just allocate the dialect, the context
             // takes ownership of it.
             // 从ctx中加载dialect对象或直接new一个具体的dialect的对象
             return ctx->getOrLoadDialect<ConcreteDialect>();
           })));
  }
  ...
}
```

**MLIRContext**

 各种实体都被添加到了impl中对应的不同的Dense map中, Op, Type，Attr等实体**一般在Dialect对象构造函数中使用add*方法**添加到关联的context.impl中，构造一个Dialect对象需要构造加载一组实体对象到impl中，因此一般context调用getOrLoadDialect**按需要延迟加载Dialect对象**，而不是一次加载所有。

ctx->getOrLoadDialect<ConcreteDialect>()方法，它会先查找imp的loadedDialects字典，若不存在则new一个新的dialect对象，存储到loadedDialects。

ctx->getOrLoadDialect(StringRef name) 方法是从dialectsRegistry创建加载的，根据name查找dialectsRegistry中注册的构造器，然后构造对象新的dialect对象。

MlIRContext对象的构造函数会预加载builtin dialect，并Initialize several common attributes and types。

```
class MLIRContextImpl {
public:

  /// This is a list of dialects that are created referring to this context.
  /// The MLIRContext owns the objects.
  // 存储了已经load的Dialect对象
  DenseMap<StringRef, std::unique_ptr<Dialect>> loadedDialects;
  DialectRegistry dialectsRegistry;
  
  /// This is a mapping from operation name to AbstractOperation for registered
  /// operations.
  llvm::StringMap<AbstractOperation> registeredOperations;
  
  //===--------------------------------------------------------------------===//
  // Type uniquing
  //===--------------------------------------------------------------------===//

  DenseMap<TypeID, const AbstractType *> registeredTypes;
  StorageUniquer typeUniquer;
  
  //===--------------------------------------------------------------------===//
  // Attribute uniquing
  //===--------------------------------------------------------------------===//

  DenseMap<TypeID, const AbstractAttribute *> registeredAttributes;
  StorageUniquer attributeUniquer;

};
} // end namespace mlir

class MLIRContext {
  template <typename T>
  T *getOrLoadDialect() {
    return static_cast<T *>(
        getOrLoadDialect(T::getDialectNamespace(), TypeID::get<T>(), [this]() {
          std::unique_ptr<T> dialect(new T(this));
          return dialect;
        }));
  }
  
  /// 不存在则创建新的Dialect对象
  Dialect *
MLIRContext::getOrLoadDialect(StringRef dialectNamespace, TypeID dialectID,
                              function_ref<std::unique_ptr<Dialect>()> ctor) {
  auto &impl = getImpl();
  // Get the correct insertion position sorted by namespace.
  std::unique_ptr<Dialect> &dialect = impl.loadedDialects[dialectNamespace];
  if (!dialect) {
    dialect = ctor();
    return dialect.get();
  }
  return dialect.get();
}
}
```

**Value**

此类型保存了value owner的指针（定义它的Operation或Block）以及使用低位保存了指针类型信息

```
 using ImplType = llvm::PointerIntPair<void *, 2, Kind, ImplTypeTraits>;
```

value两个子类，分别代表了块参数和Op的返回结果， OpResult的owner是Operation, BlockArgument的Owner是Block, Block有ParentOp是Operation

```
class OpResult : public Value 
class BlockArgument : public Value
```

 **IROperand**

当前有OpOprand （value 作为操作数）和 BlockOprand (Block作为操作数)

**IROperand 引用了value**, 此class保存了一个引用的value, OpOperand的owner是一个Operation，它是引用它的Op / Block

value 的 use list是一个 双向链表， template <typename OperandType> class IRObjectWithUseList 是链表头保存了firstUse，IROperand是双向链表的节点

```
/// A reference to a value, suitable for use as an operand of an operation.
/// IRValueTy is the root type to use for values this tracks. Derived operand
/// types are expected to provide the following:
///  * static IRObjectWithUseList *getUseList(IRValueTy value);
///    - Provide the use list that is attached to the given value.
template <typename DerivedT, typename IRValueTy> class IROperand {
public:
  using ValueType = IRValueTy;

  IROperand(Operation *owner) : owner(owner) {}
  IROperand(Operation *owner, ValueType value) : value(value), owner(owner) {
    insertIntoCurrent();
  }
  
private:
  /// The value used as this operand. This can be null when in a 'dropAllUses'
  /// state.
  ValueType value = {};

  /// The next operand in the use-chain.
  DerivedT *nextUse = nullptr;

  /// This points to the previous link in the use-chain.
  DerivedT **back = nullptr;

  /// The operation owner of this operand.
  Operation *const owner;

  /// Operands are not copyable or assignable.
  IROperand(const IROperand &use) = delete;
  IROperand &operator=(const IROperand &use) = delete;

  void removeFromCurrent() {
    if (!back)
      return;
    *back = nextUse;
    if (nextUse)
      nextUse->back = back;
  }

  void insertIntoCurrent() {
    auto *useList = DerivedT::getUseList(value);
    back = &useList->firstUse;
    nextUse = useList->firstUse;
    if (nextUse)
      nextUse->back = &nextUse;
    useList->firstUse = static_cast<DerivedT *>(this);
  }
};
```

**Block**

Region 包含一组Block，从属于Operation，它提供了可以直接访问所有Operation的迭代器(Blocks是透明的)

Block拥有一系列Operation和多个BlockArgument, 上级结构为Region

```
/// `Block` represents an ordered list of `Operation`s.
class Block : public IRObjectWithUseList<BlockOperand>,
              public llvm::ilist_node_with_parent<Block, Region> {
private:
  /// Pair of the parent object that owns this block and a bit that signifies if
  /// the operations within this block have a valid ordering.
  llvm::PointerIntPair<Region *, /*IntBits=*/1, bool> parentValidOpOrderPair;

  /// This is the list of operations in the block.
  OpListType operations;

  /// This is the list of arguments to the block.
  std::vector<BlockArgument> arguments;

  Block(Block &) = delete;
  void operator=(Block &) = delete;

  friend struct llvm::ilist_traits<Block>;
};

/// IROperand是双向链表节点，BlockOperand是所有引用Block* value的user list链表节点，
// IRObjectWithUseList<BlockOperand>包含链表的头，因此从Block* value可以得到此链表的头
// 图示:https://mlir.llvm.org/docs/Tutorials/UnderstandingTheIRStructure/

/// Terminator operations can have Block operands to represent successors.
class BlockOperand : public IROperand<BlockOperand, Block *> {
public:
  using IROperand<BlockOperand, Block *>::IROperand;

  /// Provide the use list that is attached to the given block.
  static IRObjectWithUseList<BlockOperand> *getUseList(Block *value);

  /// Return which operand this is in the operand list of the User.
  unsigned getOperandNumber();
}
```

 **前驱块**是跳转到此block的块，即将此block作为BlockOperand，是他的user ; **本块被use**

 **后继块**是从基本块的TerminatorOp得到的, TerminatorOp有几个BlockOperand ; **use其它块**

```
 ^bb3(%c: i64):
  br ^bb4(%c, %a : i64, i64)

^bb4(%d : i64, %e : i64):
  %0 = addi %d, %e : i64
  return %0 : i64   // Return is also a terminator.
 
 pred_iterator pred_begin() {
  return pred_iterator((BlockOperand *)getFirstUse());
 }
 
 Block *Block::getSuccessor(unsigned i) {
  assert(i < getNumSuccessors());
  return getTerminator()->getSuccessor(i);
}
```

**StorageUniquer** 定义了 BaseStorage 和 StorageAllocator ， 主要内容存储在impl的map中，所有的注册和get方法也传递给了impl

```
/// A utility class to get or create instances of "storage classes". These
/// storage classes must derive from 'StorageUniquer::BaseStorage'.
class StorageUniquer {
  class BaseStorage {...};
  /// This is a utility allocator used to allocate memory for instances of
  /// derived types.
  class StorageAllocator {...}

      /// Changes the mutable component of 'storage' by forwarding the trailing
      /// arguments to the 'mutate' function of the derived class.
        	  /// mutationFn主要调用了typename Storage的mutate方法来改变可变组件，可见自定义的mutate方法是回调接口
      template <typename Storage, typename... Args>
      LogicalResult mutate(TypeID id, Storage *storage, Args &&...args) {
        auto mutationFn = [&](StorageAllocator &allocator) -> LogicalResult {
          return static_cast<Storage &>(*storage).mutate(
              allocator, std::forward<Args>(args)...);
        };
        return mutateImpl(id, storage, mutationFn);
      }
    std::unique_ptr<detail::StorageUniquerImpl> impl;、
  }
```

**StorageUniquerImpl 提供了BaseStorge*的Densemap** ，包含map用来唯一存储StorageUniquer， {TypeID ，BaseStorage* }

#! 在conetxt.impl中的数据成员大概率是隐含的DenseMap容器

```
/// This is the implementation of the StorageUniquer class.
struct StorageUniquerImpl {
  using BaseStorage = StorageUniquer::BaseStorage;
  using StorageAllocator = StorageUniquer::StorageAllocator;
  //===--------------------------------------------------------------------===//
  // Instance Storage
  //===--------------------------------------------------------------------===//

  /// Map of type ids to the storage uniquer to use for registered objects.
  DenseMap<TypeID, std::unique_ptr<ParametricStorageUniquer>>
      parametricUniquers;

  /// Map of type ids to a singleton instance when the storage class is a
  /// singleton.
  DenseMap<TypeID, BaseStorage *> singletonInstances;

  /// Allocator used for uniquing singleton instances.
  StorageAllocator singletonAllocator;

  /// Flag specifying if multi-threading is enabled within the uniquer.
  bool threadingIsEnabled = true;
};
} // end namespace detail
} // namespace mlir

```

**StorageUserBase**  在创建具体的Type / Attribute时，将（Type / Attribute），Traits， 打包在一起集成/继承的类 ，具体的ype / Attribute直接继承此类，指定特定的StorageT和Traits

```
template <typename ConcreteT, typename BaseT, typename StorageT,
          typename UniquerT, template <typename T> class... Traits>
class StorageUserBase : public BaseT, public Traits<ConcreteT>... {}
```

Abstract Type / Attribute 与特定的StorageT一起构成了一个自定义Type的所有信息，StorageT存储了特定实体的特定具体信息

**Types**

Type类的实例是唯一的(单例)，具有不变的标识符和可选的可变组件, 类型的可变组件可以在创建类型后进行修改，但不能影响类型的标识；

Type类是一个wrapper对象扮演了引用的概念 , 指向了内部存储对象(internal storage)，该internal storage 在MLIRContext实例中是唯一的；Type 是**按值传递**的 

Type class引用了一个TypeStorage， TypeStorage引用了一个 AbstractType*，  AbstractTypes是注册到context中的Type信息数据结构，通过它可以得到(Dialect, Context, Type Interfaces等信息

而自定义类型会自定义*TypeStorage（TypeStorage的子类）附加额外信息

```
class Type {
public:
  /// Utility class for implementing types.
  template <typename ConcreteType, typename BaseType, typename StorageType,
            template <typename T> class... Traits>
  using TypeBase = detail::StorageUserBase<ConcreteType, BaseType, StorageType,
                                           detail::TypeUniquer, Traits...>;
  TypeStorage *impl
}
Type ->  TypeStorage -> AbstractType 此路径关联查询Dialect, Context, Type Interfaces

// 每一个具体的类型指定一个具体的TypeStorage的子类，用于存储额外的信息 
// 如FunctionTypeStorage存储了函数输入和输出参数的类型和个数，以及继承自TypeStorage的AbstractType* 
指针
class FunctionType
    : public Type::TypeBase<FunctionType, Type, detail::FunctionTypeStorage> 
```

OpResult的Type在owner Operation的resultType上挂着

BlockArgument的Type在 BlockArgumentImpl中自己记录

**Symbol and SymbolTable**

The `Symbol` infrastructure provides mechanism in which to **refer to an operation symbolically with a name**

Symbol是一个命名Operation (名字是sym_name StringAttr属性，其它Operation以SymbolRefAttr属性来使用sym_name以引用Symbol Operation)，它直接驻留在定义SymbolTable的Region（符号表和作用域相关）

Operations defining a `SymbolTable` must use the `OpTrait::SymbolTable` trait.

```
/// 特定 Operation 内部region的特定symbol的所有引用
static Optional<UseRange> getSymbolUses(StringRef symbol, Operation *from);
/// 特定 Operation 内部region的任意symbol的所有引用
static Optional<UseRange> getSymbolUses(Operation *from);
```

**Affinemap**

index space -> access space  ; (dims...)[symbols...] -> (AffineExpr)

AffineExpr是一个简单的仅支持二元整数算数运算的ast 

```
struct AffineExprStorage;
struct AffineBinaryOpExprStorage;  
struct AffineDimExprStorage;       #[i,j]
struct AffineSymbolExprStorage;    #(s0,s1...)
struct AffineConstantExprStorage;   
```

 **Op**

每种特定类型的Op都是**mlir::Op**的派生类，**Op** derived classes引用了一个**Operation**的指针， act as smart pointer wrapper around a **Operation**;

**Op** derived classes提供了特定Op的访问器方法和类型安全的属性，是一个interface for building and interfacing with the **Operation** class，它没有类成员所有的数据结构信息存储在**Operation** class, 因此**Op** derived classes常***passing by value***

Op定义了classof(Operation *op), 可以将Operation cast为对应的Op

```
class Op ... {
/// Return true if this "op class" can match against the specified operation.
  static bool classof(Operation *op) {
    if (auto *abstractOp = op->getAbstractOperation())
      return TypeID::get<ConcreteType>() == abstractOp->typeID;
#ifndef NDEBUG
    if (op->getName().getStringRef() == ConcreteType::getOperationName())
      llvm::report_fatal_error(
          "classof on '" + ConcreteType::getOperationName() +
          "' failed due to the operation not being registered");
#endif
    return false;
  }
}
```

**Operation** 

**Operation**用于一般性地对所有operation建模, the **‘Operation’** class provides a general API into an operation instance

**OperationState** 是 Operation 的 State类，这属于备忘录模式可以保存Operation内存的成员变量，并从OperationState 生成Operation

Operation通过create + destory 创建和释放，构造和析构函数都是private的

```
IROperand {Value(OpResult / BlockArgument)} -> OpOperand
IROperand {Block} -> BlockOperand

 br ^bb4(%c, %a : i64, i64); ^bb4 is BlockOperand , %c and % a is OpOperand
 ^bb4(%d : i64, %e : i64): ; %d and % e is BlockArgument
 %0 = addi %d, %e : i64    ; %0 is OpResult
 
 ;addi is Op, Op wrapper Operation*, addi is % user in it's userlist
 ;Op 和 Operation有对应的State对象
 ;Op - OpState
 ;Operation -OperationState 
 ;Region {Blocks{ Operations { Regions}}}
 
Dialect关联一组Operation, Type , Attribute, 但最终都注册到了Context.impl中

	      Value(OpResult,BlockArgument)			  
				     |			  Block*				
				     |				|				
Value(OpResult) = Op(OpOprand...,BlockOprand) : Attribute...} 
				  |
			    Operation*	
```

OpBuilder

OpBuilder记录了插入点（标识为某个Block中的某个位置）

OpBuilder的create方法调用具体Op的build方法构建一个 OperationState, 然后调用Operation::create（ OperationState& ）生成Operation并插入到插入点，然后dyn_cast构造一个具体Op返回

```
class OpBuilder : public Builder {
 /// Create an operation of specific op type at the current insertion point.
  template <typename OpTy, typename... Args>
  OpTy create(Location location, Args &&... args) {
    OperationState state(location, OpTy::getOperationName());
    if (!state.name.getAbstractOperation())
      llvm::report_fatal_error("Building op `" +
                               state.name.getStringRef().str() +
                               "` but it isn't registered in this MLIRContext");
    OpTy::build(*this, state, std::forward<Args>(args)...);
    auto *op = createOperation(state);
    auto result = dyn_cast<OpTy>(op);
    assert(result && "builder didn't return the right type");
    return result;
  }
  
private:
  /// The current block this builder is inserting into.
  Block *block = nullptr;
  /// The insertion point within the block that this builder is inserting
  /// before.
  Block::iterator insertPoint;
  /// The optional listener for events of this builder.
  Listener *listener;
};

/// Create an operation given the fields represented as an OperationState.
Operation *OpBuilder::createOperation(const OperationState &state) {
  return insert(Operation::create(state));
}

void FuncOp::build(OpBuilder &builder, OperationState &result, StringRef name,
                   FunctionType type, ArrayRef<NamedAttribute> attrs,
                   ArrayRef<MutableDictionaryAttr> argAttrs) {
  result.addAttribute(SymbolTable::getSymbolAttrName(),
                      builder.getStringAttr(name));
  result.addAttribute(getTypeAttrName(), TypeAttr::get(type));
  result.attributes.append(attrs.begin(), attrs.end());
  result.addRegion();

  if (argAttrs.empty())
    return;
  assert(type.getNumInputs() == argAttrs.size());
  SmallString<8> argAttrName;
  for (unsigned i = 0, e = type.getNumInputs(); i != e; ++i)
    if (auto argDict = argAttrs[i].getDictionary(builder.getContext()))
      result.addAttribute(getArgAttrName(i, argAttrName), argDict);
}
```

**Diagnostic** 

基本逻辑是Diagnostic 被Diagnostic handler处理，Diagnostic handler注册在DiagnosticEngine

Diagnostic 是诊断信息体，是一个message package 包含了所有信息（位置、等级、附加参数等）

DiagnosticEngine中注册一系列Diagnostic handler， emit方法可以将 Diagnostic发送给注册的handler处理

Diagnostic和DiagnosticEngine不是直接发生关系的，InFlightDiagnostic将两者关联起来，即为Diagnostic 关联一个DiagnosticEngine （like 为异常关联一系列catch）,  它的report方法代表了一次Diagnostic递送处理，将Diagnostic交给DiagnosticEngine注册的handler处理，然后会reset Diagnostic 和 DiagnosticEngine 为null， 表示一次处理完成后关联解除，InFlightDiagnostic处于未激活的状态。因此，激活InFlightDiagnostic并report表示一次Diagnostic的捕获与处理。

```
class InFlightDiagnostic {
public:
  InFlightDiagnostic() = default;
  InFlightDiagnostic(InFlightDiagnostic &&rhs)
      : owner(rhs.owner), impl(std::move(rhs.impl)) {
    // Reset the rhs diagnostic.
    rhs.impl.reset();
    rhs.abandon();
  }
  ~InFlightDiagnostic() {
    if (isInFlight())
      report();
  }

  /// Stream operator for new diagnostic arguments.
  template <typename Arg> InFlightDiagnostic &operator<<(Arg &&arg) & {
    return append(std::forward<Arg>(arg));
  }
  template <typename Arg> InFlightDiagnostic &&operator<<(Arg &&arg) && {
    return std::move(append(std::forward<Arg>(arg)));
  }

  /// Append arguments to the diagnostic.
  template <typename... Args> InFlightDiagnostic &append(Args &&... args) & {
    assert(isActive() && "diagnostic not active");
    if (isInFlight())
      impl->append(std::forward<Args>(args)...);
    return *this;
  }
  template <typename... Args> InFlightDiagnostic &&append(Args &&... args) && {
    return std::move(append(std::forward<Args>(args)...));
  }

  /// Attaches a note to this diagnostic.
  Diagnostic &attachNote(Optional<Location> noteLoc = llvm::None) {
    assert(isActive() && "diagnostic not active");
    return impl->attachNote(noteLoc);
  }

  /// Reports the diagnostic to the engine.
  void report();

  /// Abandons this diagnostic so that it will no longer be reported.
  void abandon();

  /// Allow an inflight diagnostic to be converted to 'failure', otherwise
  /// 'success' if this is an empty diagnostic.
  operator LogicalResult() const;

private:
  InFlightDiagnostic &operator=(const InFlightDiagnostic &) = delete;
  InFlightDiagnostic &operator=(InFlightDiagnostic &&) = delete;
  InFlightDiagnostic(DiagnosticEngine *owner, Diagnostic &&rhs)
      : owner(owner), impl(std::move(rhs)) {}

  /// Returns true if the diagnostic is still active, i.e. it has a live
  /// diagnostic.
  bool isActive() const { return impl.hasValue(); }

  /// Returns true if the diagnostic is still in flight to be reported.
  bool isInFlight() const { return owner; }

  // Allow access to the constructor.
  friend DiagnosticEngine;

  /// The engine that this diagnostic is to report to.
  DiagnosticEngine *owner = nullptr;

  /// The raw diagnostic that is inflight to be reported.
  Optional<Diagnostic> impl;
};

void InFlightDiagnostic::report() {
  // If this diagnostic is still inflight and it hasn't been abandoned, then
  // report it.
  if (isInFlight()) {
    owner->emit(std::move(*impl));
    owner = nullptr;
  }
  impl.reset();
}
```

**Interface** 

Op定义注册时，附着的Interface保存在AbstractOperation的InterfaceMap中

IntrefaceMap包含容器为std::unique_ptr<llvm::SmallDenseMap**<TypeID, void *>**> interfaces， 它以void * 存储不同类型的 Traits::Concept类，以TypeID标识类型Traits，Traits-> TypeID->Traits::Concept（void\*）得到void*指针的具体指向类型。

```
template <typename ConcreteType, typename Traits>
class OpInterface
    : public detail::Interface<ConcreteType, Operation *, Traits,
                               Op<ConcreteType>, OpTrait::TraitBase> {
                               
/// Returns the impl interface instance for the given operation.
  static typename InterfaceBase::Concept *getInterfaceFor(Operation *op) {
    // Access the raw interface from the abstract operation.
    auto *abstractOp = op->getAbstractOperation();
    return abstractOp ? abstractOp->getInterface<ConcreteType>() : nullptr;
  }
}

template <typename ConcreteType, typename ValueT, typename Traits,
          typename BaseType,
          template <typename, template <typename> class> class BaseTrait>
class Interface : public BaseType {
public:
  using Concept = typename Traits::Concept;
  template <typename T> using Model = typename Traits::template Model<T>;
  using InterfaceBase =
      Interface<ConcreteType, ValueT, Traits, BaseType, BaseTrait>;

  Interface(ValueT t = ValueT())
      : BaseType(t), impl(t ? ConcreteType::getInterfaceFor(t) : nullptr) {
    assert((!t || impl) &&
           "instantiating an interface with an unregistered operation");
  }
  
private:
  /// A pointer to the impl concept object.
  Concept *impl;
};
```

OpInterface是抽象外壳，具体的接口和逻辑在Traits， 因此将定义具体的Traits附加在Op上，Op可以直接通过dyn_cast得到Traits对象的Interface；**容器保存内核，外壳需要时通过内核重新构造**

```
class ExampleOpInterface : public OpInterface<ExampleOpInterface,
                                              ExampleOpInterfaceTraits>                   
class MyOp : public Op<MyOp, ExampleOpInterface::Trait>
ExampleOpInterface example = dyn_cast<ExampleOpInterface>(op)
```

**ODS Interface自动生成**

每一个Interface对应一个具体Traits对象实现接口定义的方法

若ODS中仅声明方法没有定义实现体，则自动生成的方法调用了具体Op的同名方法，附加此Interface的Op会自动生成同名方法签名，一般在c++中实现

若定义了实现体则直接生成实现体，不会在Op生成同名方法

```
// 声明了一个接口方法inferShapes 
def ShapeInferenceOpInterface : OpInterface<"ShapeInference"> {
  let methods = [
    InterfaceMethod<"Infer and set the output shape for the current operation.",
                    "void", "inferShapes">
  ];
}

// .inc
namespace detail {
struct ShapeInferenceInterfaceTraits {
  struct Concept {
    void (*inferShapes)(::mlir::Operation *);
  };
  template<typename ConcreteOp>
  class Model : public Concept {
  public:
    Model() : Concept{inferShapes} {}

    static void inferShapes(::mlir::Operation *tablegen_opaque_val) {
      return (llvm::cast<ConcreteOp>(tablegen_opaque_val)).inferShapes();
    }
  };
};
} // end namespace detail

// 外壳的inferShapes调用了内核的inferShapes
class ShapeInference : public ::mlir::OpInterface<ShapeInference, detail::ShapeInferenceInterfaceTraits> {
  void inferShapes() {
    return getImpl()->inferShapes(getOperation()); 
  }
};

// 内核的interface调用了具体Op的同名方法，Op会生成一个同名函数，一般在c++代码中具体实现
struct ShapeInferenceInterfaceTraits {
  struct Concept {
    void (*inferShapes)(::mlir::Operation *);
  };
  template<typename ConcreteOp>
  class Model : public Concept {
  public:
    Model() : Concept{inferShapes} {}
    
    static void inferShapes(::mlir::Operation *tablegen_opaque_val) {
      return (llvm::cast<ConcreteOp>(tablegen_opaque_val)).inferShapes();
    }
  };
};
```

**Traits**

Op的hasTrait直接调用llvm::is_one_of,因为Traits直接附加到Op，**编译时根据元编程判断, 建议使用**

Operation判断hasTrait使用的是AbstractOperation的hasTraitFn函数，此函数由getHasTraitFn函数返回，运行时根据TypeID判断

```
class Op {
  /// Return if this operation contains the provided trait.
  // Op实际调用的hasTrait函数，使用元编程在编译时判断
  template <template <typename T> class Trait>
  static constexpr bool hasTrait() {
    return llvm::is_one_of<Trait<ConcreteType>, Traits<ConcreteType>...>::value;
  }
	
  /// Implementation of `GetHasTraitFn`
  static AbstractOperation::HasTraitFn getHasTraitFn() {
    return &op_definition_impl::hasTrait<Traits...>;
  }
}

// Operation的hasTrait实际调用的函数，根据TypeID运行时判断
template <template <typename T> class... Traits>
static bool hasTrait(TypeID traitID) {
  TypeID traitIDs[] = {TypeID::get<Traits>()...};
  for (unsigned i = 0, e = sizeof...(Traits); i != e; ++i)
    if (traitIDs[i] == traitID)
      return true;
  return false;
}
```

**Analyses**

concpet约定了isInvalidated接口，model是一个具体<AnalysisT>model:concept泛型类，它的isInvalidated方法调用了analysis_impl::isInvalidated(AnalysisT& ,PreservedAnalyses &)；此方法判断AnalysisT若没有实现isInvalidated方法则默认检查是否被mark preserve，**AnalysisT实现isInvalidated则调用它的isInvalidated** ， 因此AnalysisT必要时才需要实现isInvalidated方法

```
/// Implementation of 'isInvalidated' if the analysis provides a definition.
template <typename AnalysisT>
std::enable_if_t<llvm::is_detected<has_is_invalidated, AnalysisT>::value, bool>
isInvalidated(AnalysisT &analysis, const PreservedAnalyses &pa) {
  return analysis.isInvalidated(pa);
}
/// Default implementation of 'isInvalidated'.
template <typename AnalysisT>
std::enable_if_t<!llvm::is_detected<has_is_invalidated, AnalysisT>::value, bool>
isInvalidated(AnalysisT &analysis, const PreservedAnalyses &pa) {
  return !pa.isPreserved<AnalysisT>();
}
} // end namespace analysis_impl
```

泛型AnalysisT是以TypeId为key标识自己，在各种容器中;AnalysisManager 主要保存了  detail::NestedAnalysisMap 它是 current operation, and child operations 的AnalysisT的容器

```
/// This class represents a cache of analyses for a single operation. All
/// computation, caching, and invalidation of analyses takes place here.
class AnalysisMap {
  /// A mapping between an analysis id and an existing analysis instance.
  using ConceptMap = DenseMap<TypeID, std::unique_ptr<AnalysisConcept>>;
	  Operation *ir;
  	  ConceptMap analyses;
};

/// An analysis map that contains a map for the current operation, and a set of
/// maps for any child operations.
struct NestedAnalysisMap {
  DenseMap<Operation *, std::unique_ptr<NestedAnalysisMap>> childAnalyses;
  /// The analyses for the owning module.
  detail::AnalysisMap analyses
}

class AnalysisManager {
  using ParentPointerT =
      PointerUnion<const ModuleAnalysisManager *, const AnalysisManager *>;
      
public:
  ParentPointerT parent;
  /// A reference to the impl analysis map within the parent analysis manager.
  detail::NestedAnalysisMap *impl;
  /// Allow access to the constructor.
  friend class ModuleAnalysisManager;
};
```

**Pass**

pass 的主要信息保存在PassExecutionState 中，包含作用的Operation

pass可以通过td生成

```
class pass {
  /// Represents a unique identifier for the pass.
  TypeID passID;
  /// The name of the operation that this pass operates on, or None if this is a
  /// generic OperationPass.
  Optional<StringRef> opName;
  /// The current execution state for the pass.
  Optional<detail::PassExecutionState> passState;
}

struct PassExecutionState {
  /// The current operation being transformed and a bool for if the pass
  /// signaled a failure.
  llvm::PointerIntPair<Operation *, 1, bool> irAndPassFailed;

  /// The analysis manager for the operation.
  AnalysisManager analysisManager;

  /// The set of preserved analyses for the current execution.
  detail::PreservedAnalyses preservedAnalyses;

  /// This is a callback in the PassManager that allows to schedule dynamic
  /// pipelines that will be rooted at the provided operation.
  function_ref<LogicalResult(OpPassManager &, Operation *)> pipelineExecutor;
}
```

**PassManager**

 OpToOpPassAdaptor是一个pass, 它保存了多个OpPassManager到 mgrs中，它的runOnOperation对绑定Operation的所有内嵌child算子从mgrs查找对应的OpPassManager执行它里面注册的所有的pass。

```
//===----------------------------------------------------------------------===//
// OpToOpPassAdaptor
//===----------------------------------------------------------------------===//

/// An adaptor pass used to run operation passes over nested operations.
class OpToOpPassAdaptor
    : public PassWrapper<OpToOpPassAdaptor, OperationPass<>> {
public:
  OpToOpPassAdaptor(OpPassManager &&mgr);
  OpToOpPassAdaptor(const OpToOpPassAdaptor &rhs) = default;
  
  /// run*系列方法调用关系自上而下
  /// Run the held pipeline over all operations.
  void runOnOperation() override;
  
private:
  /// Run this pass adaptor synchronously.
  void runOnOperationImpl();

  /// Run this pass adaptor asynchronously.
  void runOnOperationAsyncImpl();
  
  /// Run the given operation and analysis manager on a provided op pass
  /// manager.
  static LogicalResult
  runPipeline(iterator_range<OpPassManager::pass_iterator> passes,
              Operation *op, AnalysisManager am);

  /// Run the given operation and analysis manager on a single pass.
  static LogicalResult run(Pass *pass, Operation *op, AnalysisManager am);
  
  /// A set of adaptors to run.
  SmallVector<OpPassManager, 1> mgrs;

  /// A set of executors, cloned from the main executor, that run asynchronously
  /// on different threads. This is used when threading is enabled.、
  /// OpPassManager阵列
  SmallVector<SmallVector<OpPassManager, 1>, 8> asyncExecutors;

  // For accessing "runPipeline".
  friend class mlir::PassManager;
};
```

detail::OpPassManagerImpl 是pass的容器，里面只能装pass, 因此其它OpPassManager要嵌入(nest)到 parent OpPassManager中是通过OpToOpPassAdaptor(没有Operation name的pass), 它将OpPassManagerAdaptor为一个pass。 

```
struct OpPassManagerImpl {
  /// The name of the operation that passes of this pass manager operate on.
  StringRef name;
  /// The cached identifier (internalized in the context) for the name of the
  /// operation that passes of this pass manager operate on.
  Optional<Identifier> identifier;
  /// Flag that specifies if the IR should be verified after each pass has run.
  bool verifyPasses : 1;
  /// The set of passes to run as part of this pass manager.
  std::vector<std::unique_ptr<Pass>> passes;
};

OpPassManager &OpPassManagerImpl::nest(Identifier nestedName) {
  OpPassManager nested(nestedName, verifyPasses);
  auto *adaptor = new OpToOpPassAdaptor(std::move(nested));
  addPass(std::unique_ptr<Pass>(adaptor));
  return adaptor->getPassManagers().front();
}

void OpPassManagerImpl::addPass(std::unique_ptr<Pass> pass) {
  // If this pass runs on a different operation than this pass manager, then
  // implicitly nest a pass manager for this operation.
  //隐式内嵌
  auto passOpName = pass->getOpName();
  if (passOpName && passOpName != name)
    return nest(*passOpName).addPass(std::move(pass));

  passes.emplace_back(std::move(pass));
  if (verifyPasses)
    passes.emplace_back(std::make_unique<VerifierPass>());
}
```

**ScopedContext**

使用guard来RAII instert point, 使用 enclosingScopedContext 链接parent 

构造的时候 enclosingScopedContext  保存当前ScopedContext ， getCurrentScopedContext() = this 

析构时getCurrentScopedContext() = enclosingScopedContext, 恢复为之前保存的CurrentScopedContext （是一个thread_local static 变量）

```
mlir::edsc::ScopedContext::ScopedContext(OpBuilder &b,
                                         OpBuilder::InsertPoint newInsertPt,
                                         Location location)
    : builder(b), guard(builder), location(location),
      enclosingScopedContext(ScopedContext::getCurrentScopedContext()) {
  getCurrentScopedContext() = this;
  builder.restoreInsertionPoint(newInsertPt);
}

mlir::edsc::ScopedContext::~ScopedContext() {
  getCurrentScopedContext() = enclosingScopedContext;
}
```

***ElementsAttr** 

 ElementsAttr 和DenseElementsAttr（没有storage）是基类提供了用户使用的公共API接口； DenseElementsAttr的**一系列get方法**会根据传入的类型构造具体的DenseStringElementsAttr / DenseIntOrFPElementsAttr；

```
/// A base attribute that represents a reference to a static shaped tensor or
/// vector constant.
class ElementsAttr : public Attribute {}
/// An attribute that represents a reference to a dense vector or tensor object.
class DenseElementsAttr : public ElementsAttr {}
bool ElementsAttr::classof(Attribute attr) {
  return attr.isa<DenseIntOrFPElementsAttr, DenseStringElementsAttr,
                  OpaqueElementsAttr, SparseElementsAttr>();
}
```

e.g. DenseElementsAttr::get的调用栈，最终调用了DenseIntOrFPElementsAttributeStorage::construct 方法

```
 /*
 llvm:SmallVector<double, 6> data;
  for (auto i = 0; i < num; i++) {
    data.push_back(value);
  }
  auto dataAttribute = DenseElementsAttr::get(type, llvm::makeArrayRef(data));
  */
#0  mlir::detail::DenseIntOrFPElementsAttributeStorage::construct (allocator=..., key=...)
    at /mnt/d/Ubuntu/collectcode/LLVM/mlir/lib/IR/AttributeDetail.h:516
#1  0x000000000340931f in mlir::detail::DenseIntOrFPElementsAttributeStorage* mlir::StorageUniquer::get<mlir::detail::DenseIntOrFPElementsAttributeStorage, mlir::ShapedType&, llvm::ArrayRef<char>&, bool&>(llvm::function_ref<void (mlir::detail::DenseIntOrFPElementsAttributeStorage*)>, mlir::TypeID, mlir::ShapedType&, llvm::ArrayRef<char>&, bool&)::{lambda(mlir::StorageUniquer::StorageAllocator&)#1}::operator()(mlir::StorageUniquer::StorageAllocator&) const (this=0x7ffffffec2e0, allocator=...) at /mnt/d/Ubuntu/collectcode/LLVM/mlir/include/mlir/Support/StorageUniquer.h:190
#2  0x00000000034092b0 in llvm::function_ref<mlir::StorageUniquer::BaseStorage* (mlir::StorageUniquer::StorageAllocator&)>::callback_fn<mlir::detail::DenseIntOrFPElementsAttributeStorage* mlir::StorageUniquer::get<mlir::detail::DenseIntOrFPElementsAttributeStorage, mlir::ShapedType&, llvm::ArrayRef<char>&, bool&>(llvm::function_ref<void (mlir::detail::DenseIntOrFPElementsAttributeStorage*)>, mlir::TypeID, mlir::ShapedType&, llvm::ArrayRef<char>&, bool&)::{lambda(mlir::StorageUniquer::StorageAllocator&)#1}>(long, mlir::StorageUniquer::StorageAllocator&) (callable=140737488274144, 
    params=...) at /mnt/d/Ubuntu/collectcode/LLVM/llvm/include/llvm/ADT/STLExtras.h:185
#3  0x000000000446f3bc in llvm::function_ref<mlir::StorageUniquer::BaseStorage* (mlir::StorageUniquer::StorageAllocator&)>::operator()(mlir::StorageUniquer::StorageAllocator&) const (this=0x7ffffffebfe0, params=...) at /mnt/d/Ubuntu/collectcode/LLVM/llvm/include/llvm/ADT/STLExtras.h:209
#4  0x0000000004466d33 in (anonymous namespace)::ParametricStorageUniquer::getOrCreateUnsafe((anonymous namespace)::ParametricStorageUniquer::Shard&, (anonymous namespace)::ParametricStorageUniquer::LookupKey&, llvm::function_ref<mlir::StorageUniquer::BaseStorage* (mlir::StorageUniquer::StorageAllocator&)>) (this=0x5c5a280, shard=..., key=..., ctorFn=...) at /mnt/d/Ubuntu/collectcode/LLVM/mlir/lib/Support/StorageUniquer.cpp:99
#5  0x0000000004465c2e in (anonymous namespace)::ParametricStorageUniquer::getOrCreate(bool, unsigned int, llvm::function_ref<bool (mlir::StorageUniquer::BaseStorage const*)>, llvm::function_ref<mlir::StorageUniquer::BaseStorage* (mlir::StorageUniquer::StorageAllocator&)>) (this=0x5c5a280, 
    threadingIsEnabled=true, hashValue=3455460645, isEqual=..., ctorFn=...)
    at /mnt/d/Ubuntu/collectcode/LLVM/mlir/lib/Support/StorageUniquer.cpp:147
#6  0x000000000446f046 in mlir::detail::StorageUniquerImpl::getOrCreate(mlir::TypeID, unsigned int, llvm::function_ref<bool (mlir::StorageUniquer::BaseStorage const*)>, llvm::function_ref<mlir::StorageUniquer::BaseStorage* (mlir::StorageUniquer::StorageAllocator&)>) (this=0x5c55de0, id=..., 
    hashValue=3455460645, isEqual=..., ctorFn=...) at /mnt/d/Ubuntu/collectcode/LLVM/mlir/lib/Support/StorageUniquer.cpp:257
#7  0x0000000004464ff8 in mlir::StorageUniquer::getParametricStorageTypeImpl(mlir::TypeID, unsigned int, llvm::function_ref<bool (mlir::StorageUniquer::BaseStorage const*)>, llvm::function_ref<mlir::StorageUniquer::BaseStorage* (mlir::StorageUniquer::StorageAllocator&)>) (this=0x5c55b50, 
    id=..., hashValue=3455460645, isEqual=..., ctorFn=...) at /mnt/d/Ubuntu/collectcode/LLVM/mlir/lib/Support/StorageUniquer.cpp:321
#8  0x0000000003407bd7 in mlir::StorageUniquer::get<mlir::detail::DenseIntOrFPElementsAttributeStorage, mlir::ShapedType&, llvm::ArrayRef<char>&, bool&>(llvm::function_ref<void (mlir::detail::DenseIntOrFPElementsAttributeStorage*)>, mlir::TypeID, mlir::ShapedType&, llvm::ArrayRef<char>&, bool&)
    (this=0x5c55b50, initFn=..., id=..., args=@0x7ffffffec4b7: false, args=@0x7ffffffec4b7: false, args=@0x7ffffffec4b7: false)
    at /mnt/d/Ubuntu/collectcode/LLVM/mlir/include/mlir/Support/StorageUniquer.h:198
#9  0x00000000034078da in mlir::detail::AttributeUniquer::get<mlir::DenseIntOrFPElementsAttr, mlir::ShapedType&, llvm::ArrayRef<char>&, bool&> (
    ctx=0x7ffffffee348, args=@0x7ffffffec4b7: false, args=@0x7ffffffec4b7: false, args=@0x7ffffffec4b7: false)
---Type <return> to continue, or q <return> to quit---
    at /mnt/d/Ubuntu/collectcode/LLVM/mlir/include/mlir/IR/AttributeSupport.h:154
#10 0x00000000033f9352 in mlir::detail::StorageUserBase<mlir::DenseIntOrFPElementsAttr, mlir::DenseElementsAttr, mlir::detail::DenseIntOrFPElementsAttributeStorage, mlir::detail::AttributeUniquer>::get<mlir::ShapedType, llvm::ArrayRef<char>, bool> (ctx=0x7ffffffee348, args=false, args=false, 
    args=false) at /mnt/d/Ubuntu/collectcode/LLVM/mlir/include/mlir/IR/StorageUniquerSupport.h:95
#11 0x00000000033f0d8a in mlir::DenseIntOrFPElementsAttr::getRaw (type=..., data=..., isSplat=false)
    at /mnt/d/Ubuntu/collectcode/LLVM/mlir/lib/IR/Attributes.cpp:1086
#12 0x00000000033f1dfb in mlir::DenseIntOrFPElementsAttr::getRawIntOrFloat (type=..., data=..., dataEltSize=8, isInt=false, isSigned=true)
    at /mnt/d/Ubuntu/collectcode/LLVM/mlir/lib/IR/Attributes.cpp:1118
#13 0x00000000033f1cad in mlir::DenseElementsAttr::getRawIntOrFloat (type=..., data=..., dataEltSize=8, isInt=false, isSigned=true)
    at /mnt/d/Ubuntu/collectcode/LLVM/mlir/lib/IR/Attributes.cpp:895
#14 0x00000000022df613 in mlir::DenseElementsAttr::get<double, void> (type=..., values=...)

```

DenseIntOrFPElementsAttributeStorage的  construct 根据传入的raw buffer内容构建新的storage instance, 注意它申**请的是新的空间copy 了 raw buffer的内容**;raw buffer的allocate由*stoage的基类的StorageUniquerImpl 中的allocator成员负责申请，当它析构时会释放所有内存。

即从外部的raw buffer构造*ElementsAttr是安全的，**以copy语义构造由allocator负责Storage内部的内存管理**

```
struct DenseIntOrFPElementsAttributeStorage
    : public DenseElementsAttributeStorage {
      /// Construct a new storage instance.
  static DenseIntOrFPElementsAttributeStorage *
  construct(AttributeStorageAllocator &allocator, KeyTy key) {
    // If the data buffer is non-empty, we copy it into the allocator with a
    // 64-bit alignment.
    ArrayRef<char> copy, data = key.data;
    if (!data.empty()) {
      char *rawData = reinterpret_cast<char *>(
          allocator.allocate(data.size(), alignof(uint64_t)));
      std::memcpy(rawData, data.data(), data.size());

      // If this is a boolean splat, make sure only the first bit is used.
      if (key.isSplat && key.type.getElementType().isInteger(1))
        rawData[0] &= 1;
      copy = ArrayRef<char>(rawData, data.size());
    }
    return new (allocator.allocate<DenseIntOrFPElementsAttributeStorage>())
        DenseIntOrFPElementsAttributeStorage(key.type, copy, key.isSplat);
  }
}
```

**PatternRewriter**

继承自OpBuilder类

允许模式与模式应用程序的驱动程序进行通信，所有IR突变（包括创建）都需要通过PatternRewriter类

提供对Op的删除、替换、事务就地更新等API

class PatternRewriter 可以作为其它自定义PatternRewriter的基类

=======================================**C++!**============================================

 llvm的addOperations写法避免了可变模板参数的递归展开, 更加简洁

```
  /// This method is used by derived classes to add their operations to the set.
  ///
  /*
    template <typenname T,typename... Args> void addOperations() {
      addOperation<T>();
      addOperations<Args>()...; 
    }
    template <typenname T> void addOperations() {
      addOperation<T>();
    }
  */
  
  template <typename... Args> void addOperations() {
    // ","表达式求值是0，此处仅为了展开Args并依次调用addOperation函数
    (void)std::initializer_list<int>{0, (addOperation<Args>(), 0)...};
  }
  template <typename Arg> void addOperation() {
    addOperation(AbstractOperation::get<Arg>(*this));
  }
```

具体的FunctionType, 通过StorageUserBase间接继承了Type以及多个class，可变数量父类继承的一种手法

```
class Type {
public:
  template <typename ConcreteType, typename BaseType, typename StorageType,
            template <typename T> class... Traits>
  using TypeBase = detail::StorageUserBase<ConcreteType, BaseType, StorageType,
                                           detail::TypeUniquer, Traits...>;
                                           
}

class FunctionType
    : public Type::TypeBase<FunctionType, Type, detail::FunctionTypeStorage>
    
template <typename ConcreteT, typename BaseT, typename StorageT,
          typename UniquerT, template <typename T> class... Traits>
class StorageUserBase : public BaseT, public Traits<ConcreteT>... 

```

自定义RTTI机制

llvm 自定义RTTI是对c++ 原生RTTI机制的增强 ，类A::classof (B& / B* object) -> true ，表示B可以安全的转换为A或从B可以安全构造出A，而A和B不一定属于一个继承体系，如从Operation->Op, Operation被compose到了Op中，而不是Op的子类 , 因此llvm RTTI更加的灵活强大

每个c++类型有唯一的TypeId，不同类型get函数实例化为不同的模板函数，相同类型仅生成一个静态局部变量（函数内静态局部变量仅初始化一次）， 以静态局部变量的内存地址标识TypeId。

Type : <TypeID, void *> 也提供了一种将不**同类型对象指针存储在相同容器中的方法**

```
struct LLVM_EXTERNAL_VISIBILITY TypeIDExported {
  template <typename T>
  static TypeID get() {
    static TypeID::Storage instance;
    return TypeID(&instance);
  }
  template <template <typename> class Trait>
  static TypeID get() {
    static TypeID::Storage instance;
    return TypeID(&instance);
  }
};
```

mlir IR数据结构的设计可分为四个部分：外壳wrapper， 内核实体，生命周期容器，创建器

外壳 包含一个内核实体的impl指针，外壳wrapper具有值语义（pass by value）

内核实体 复杂结构会分解为抽象部分(AbstractType, AbstractOperation, AbstractAttribute)和具体部分(*Storage, Operation)

容器 容器对象一般内部以DenseMap存储实体Instance，{TypeID / Identifier, 实体instance }, 需要全局容器的对象具有**长生命周期、单例**等特性

创建器 内核对象一般不直接创建，它会创建到指定的容器中(如MLIRContextImpl)以管理生命周期和保证唯一性，因此提供创建方法在指定位置容器）以指定形式创建对象

| 外壳            | 内核抽象部分                                         | 内核具体部分                            | 容器                                                         | 创建器            |
| --------------- | ---------------------------------------------------- | --------------------------------------- | ------------------------------------------------------------ | ----------------- |
| Type            | AbstractType                                         | TypeStorage以及其子类                   | MLIRContext->StorageUniquer->StorageUniquerImpl              | TypeUniquer       |
| Attribute       | AbstractAttribute                                    | AttributeStorage以及其子类              | MLIRContext->StorageUniquer->StorageUniquerImpl              | AttributeUniquer  |
| Location        | LocationAttr                                         | 不同的LocationAttr子类以及对应的Storage | Location的具体部分是LocationAttr的子类, 因此一种是Attribute对象 | AttributeUniquer  |
| Op              | AbstractOperation                                    | Operation                               | AbstractOperation在MLIRContext，具体Op对象的AbstractOperation是相同的，Operation的容器是Blcok->Region-->Operation->... ，最顶层是Module Operation | Operation::Create |
| Intrface        | Traits:Concept                                       | Traits::Model, Traits:Concept的子类     | InterfaceMap                                                 | -                 |
| AnalysisConcept | <typename AnalysisT> AnalysisModel : AnalysisConcept | AnalysisT                               | AnalysisMap; 以TypeID标识                                    | -                 |

llvm 的最大公约数(gcd)和最小公倍数(lcm)

gcd使用了辗转相除法， lcm*gcd = a * b -> lcm = a*b / gcd 

```
inline int64_t lcm(int64_t a, int64_t b) {
  uint64_t x = std::abs(a);
  uint64_t y = std::abs(b);
  int64_t lcm = (x * y) / llvm::GreatestCommonDivisor64(x, y);
  assert((lcm >= a && lcm >= b) && "LCM overflow");
  return lcm;
}

/*
int gcd(int a,int b)
{
    if(b==0)
        return a;
    else
        return gcd(b,a%b);
}*/
template <typename T>
inline T greatestCommonDivisor(T A, T B) {
  while (B) {
    T Tmp = B;
    B = A % B;
    A = Tmp;
  }
  return A;
}
```

out of line虚方法

out of line 虚方法是在类中声明, 在类外部实现的非纯虚方法，第一个没有定义在类中的非纯虚方法会被选为关键方法，虚表生成一份在包含关键方法定义的.o中

若类所有虚方法都在类里面实现在.h中被多个翻译单元包含，则在每一个翻译单元都会生成一份虚表，而后需要链接器处理，造成空间浪费和链接时间浪费

thread_local

线程创建时申请，线程销毁时释放，线程独立拥有

具有static的初始化特征，线程生命周期的static特征变量