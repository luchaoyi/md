[1]http://llvm.org/docs/ProgrammersManual.html

[2]https://my.oschina.net/u/232554?q=llvm

**自定义RTTI**

 llvm/Support/Casting.h

llvm使用了RTTI的自定义形式，因为c++ dynamic_cast<>只适用于具有v-table的类

isa<>: like Java “instanceof” 

cast<>:已检查的强制转换

dyn_cast<>:like dynamic_cast<>,同样注意不要滥用，考虑使用 InstVisitor 类直接分派指令类型

cast_or_null<> ,`dyn_cast_or_null<>`:它允许空指针作为参数(然后将其传播),这有时很有用, 将多个null检查合并到一个检查中。

````
**llvm 自定义的RTTI | lcy**

每一个以XXX为基类的class族都要实现一个静态方法static bool classof(XXX*/ XXX&)返回true/false来判断基类的引用/指针指向的对象是否classof当前class, 实现了classof则可以融入llvm RTTI体系

一种实现方法是class可能需要自定义一个字段来标识自己在族系中的kind ，通过kind判断引用的对象的类别

llvm自定义RTTI给了用户对于RTTI机制的可操作性（如自定义classof规则，支持&等更加的灵活等）把权利从编译器手中收回完全自己控制，避免了编译器隐藏动作对程序员的欺骗，具有跨编译器的特性

展示了c++语言核心可以更简单的潜力以及元编程的强大
````

**字符串传递**

能够接受可能包含空字符的字符串, 不能简单地接受`const char *`,而const std::string&，string来源需要执行堆分配，大量小字符串在堆上，性能不好。

StringRef:

表示一个固定不变的字符串的引用,不拥有内存所有权，因此外部内存生命周期要长于stringref

足够小和普遍，因此它应该总是通过值传递

ADT/StringRef.h

```
class StringRef {
  const char *Data;  
  size_t Length;
  //...
}
```

Twine：

高效表达字符串的连接，一个二叉树结构表示字符串的拼接关系，在需要得到字符串时，通过前序遍历生成真的字符串,避免在多个字符串拼接过程中产生临时的 string 对象。

节点保持的时字符对象的指针，因此指针指向的数据生命周期要保持

ADT/Twine.h

```
class Twine {
  union Child LHS, RHS;  // 二叉树的左、右节点。
  char/*enum NodeKind*/ LHSKind, RHSKind;  // 左右节点的类型。
  Twine(...)  // 非常多种形态的构造 *注1
  concat()    // 拼接(Twine = Twine + Twine)
  str() // 进行实际拼接，并返回为 std::string
  toVector() // 拼接并转换为 vector<char>
  toStringRef(),toNullTerminatedStringRef() // 灵巧转换为 StringRef
  print(),dump() // 输出、调试 Twine。
}
```

formatv:

形式为{N[[,align]:style]}*，生成格式化字符串，会在编译时借助类型推导格式化字符串

通过扩展机制**`llvm::format_provider<T>`** and **llvm::FormatAdapter**可以自定义类型的format形式。#formatv定义了机制，类型提供的策略

function_ref:

llvm 的*ref都是存储了一个指针，只引用不存储，因此要注意被引用对象的声明周期

std::function是真正包装了一个callable对象，而function_ref仅引用了一个callable对象，*ref都足够小建议值传递

**ADT**

map/set/seq/string/bit

ArrayRef:

引用一段连续内存，本身不拥有元素，可生成一个vector

```
template <typename T> class ArrayRef {
  const T *Data; // 指向外部数组缓冲区，作为数组的第0个元素。
  size_type Length; // 元素个数
  
  ArrayRef(...) // 多种构造函数
  begin(), end(), empty(), size(), [], at() 等 STL 标准接口函数。
  slice(), equals() 等辅助函数
  vec() 生成std::vector<T>
}
```

#小内存分配优化的容器

SmallVector:

初始在类中保存内容纳N个元素的空间，这个空间不在堆上分配,大于N个元素在堆分配

在堆栈上使用小向量是最有用的,**SmallVector结合了静态数组和动态vector的特性**，直接静态持有部分空间，不够然后动态扩张

```
template <typename T, unsigned N>  // 注1
class SmallVector : public SmallVectorImpl<T> {
   U InlineElts[NUM];
   // ...
}
```

std::list

一个非常低效的类，很少有用,插入其中的每个元素执行堆分配 

ilist:

存储多态对象，在插入或从列表中删除元素时通知traits类，并且ilists保证支持常量时间拼接操作

侵入式(intrusive)双向链接列表,模板参数类 T 里面提供有 next/prev 指针，并且实现 getNext(), setNext(), getPrev(), setPrev() 的函数

SmallString :

SmallVector的子类，并封装了一些字符串接口，实际拥有char buffer

适用于在堆栈上的小字符串对象，但不适合值传递/返回

```
template<unsigned InternalLen> class SmallString 
  : public SmallVector<char, InternalLen> {  // 使用 SmallVector 做底层存储
  // 从 SmallVector, SmallVectorBase 等基类获得数据成员。参见 SmallVector
  char *BeginX, *EndX, *CapacityX;
  char in_place_buffer[InternalLen];   // 在类中的字符串缓冲，用于说明，实际是 union U 类型的，大小也不是。

  // SmallString 自己的。
  this()  // 多种形式的构造
  assign(), append(), compare(), startswith(), endswith(), find()
  substr(), slice(), +=() 等众多的字符串操作函数。
}
```

SmallSet

小于N个元素使用SmallVector保存，额外的使用std::set

SmallVector部分是采用线性搜索，因此N比较小搜索性能好

```
template <T, N, C> class SmallSet {  // 注1:T, N, C
  SmallVector<T,N> Vector;  // 内部使用 SmallVector 保存集合数据。
  std::set<T,C> Set;   // 当元素多了使用 std::set 保存集合数据。

  this() // 构造
  empty(), size(), count(), insert(), erase(), clear() 等容器方法。
}
```

SetVector:

可以按照插入顺序迭代的集合

适配器底层使用vector+SmallSet存储两份,vec存储插入顺序，set保存集合数据，在vector中查找是线性的，SmallSetVector 底层是SmallVector和SmallSet

StringSet:

StringMap的包装器

DenseMap:

DenseSet内部使用了Densemap

**Error**

逻辑错误 assert ，llvm_unaccessible ，直接报错

可恢复错误(检测，传递，处理)是异常机制，llvm定义了错误模型

Error:

Error类是用户定义错误类型的轻量级包装器，允许附加任意信息来描述错误，可以隐式地转换为`bool`。

Expected<T>：

可是接收类型T的对象和Error类型构造，可隐式转换为bool值。

if true可以通过*访问T值，if false 则可以使用takeError()方法提取Error

handleErrors:

 一个函数实现类似c+ + catch的机制,若错误没有match则函数会将错误返回以便用户继续处理或传播

consumeError：

某些类型的错误被认为是良性的，使用consumeError可以吃掉它

**Debug**

 LLVM_DEBUG:

将代码包装在此macro内，仅在-debug模式下起作用

statistics：

跟踪pass执行次数的机制

DEBUG_COUNTER：

使代码的某些部分只执行一定次数

图的可视化:

llvm的图的数据结构如DAG,CFG等提供了::viewGraph,::viewCFG等方法来弹出图形可视化窗口，在gdb中也可使用

可调用graph的setGraphAttrs附加属性（DAG.clearGraphAttrs()清除之前的设置），控制可视化效果,如颜色

可视化需要graphviz的支持(src2src生成dot语言?)

**Core IR**

Type:

所有type类的一个超类,每个`Value`都有一个`Type`,Type不能直接实例化.

一个Type仅存在一个实例(单例)，so 两个Type*值，如果指针相同，则类型相同

Module:

拥有函数、全局变量、符号表、

使用迭代器来访问函数、全局变量

Value:

一个类型化的值，常量，参数，指令，函数等都是Value

Value类保存了使用它的所有Users的一个列表,表示`def-use`信息,可以通过`use_*`方法访问.

User:

Value的子类，所有可以引用Values的节点的基类

User由一个Operand列表，保持了它引用的所有Value

Constant:

常量的基类,有不同类型的子类ConstantInt`、`ConstantArray等，[GlobalValue](http://llvm.org/docs/ProgrammersManual.html#globalvalue)也是一个子类

GlobalValue：

全局范围内可见，需要linkage 规则约束（LinkageTypes枚举定义），

Function：

Function类跟踪基本块列表、形式参数列表和符号表

Argument：

`Value`的子类，函数的形式参数定义接口

BasicBlock:

维护一个Instruction列表， 列表的end是一个Terminator指令

它是一个Value的子类，由branches之类的指令引用

BasicBlock(const std::string &Name = "", Function *Parent = 0),指定了Parent参数，则在指定Function的末尾自动插入新的BasicBlock；否则必须手动将BasicBlock插入Function

Instruction:

是User和Value的子类

`Instruction`类本身跟踪的主要数据是操作码(指令类型)和嵌入Instruction的父BB

LLVMContext：

内存IR中的每个LLVM实体(模块、值、类型、常量等)都属于一个LLVMContext，不同上下文中的实体不能相互交互，因此不同实体由不同线程并行编译是安全的
