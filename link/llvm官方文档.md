## 1.LLVM IR

[1]https://llvm.org/docs/LangRef.html

### 1.1 高层结构

**Identifiers**

@,%前缀标识避免和关键字冲突，以;为注释

parser后会自动执行验证pass检验，IR是否符合规范

!前缀的为metadata

**Target Triple**

```
ARCHITECTURE-VENDOR-OPERATING_SYSTEM
ARCHITECTURE-VENDOR-OPERATING_SYSTEM-ENVIRONMENT
target triple = "x86_64-apple-macosx10.7.0"
```

**MetaData**

```
attached to instructions ,onvey extra information about the code to the optimizers and code generator
Metadata 可以看成“结构体变量,可以包含整形数据、字符串数据或者其他 Metada 数据
```

**Data Layout**

大小端，对齐，位宽等信息

**Attributes**

属性附加在具体的IR上，传递额外的信息给后端，引导代码生成器的动作(约束/优化信息)

#IR描述了行为(计算)，属性添加了调度，tvm计算与调度分离，ir仅描述计算

**Module**

由全局变值组成(函数、变量、符号表条目)，全局值的类型是指针，!标识元数据，linker可以合并module

**Func**

define定义，decalre声明 。函数体是以basic block组成的CFG，返回类型和函数类型可以关联参数属性，传达有关函数结果或参数的其他信息

返回类型和参数类型可以附加属性

**Structure** 

```
%mytype = type { %mytype*, i32 } 自定义了类型（结构体）

```

**Intrinsic Functions**

extension mechanism for the LLVM language ，does not require changing all of the transformations

`llvm.` prefix

```
@llvm.ctpop.i8(i8 %val) and i29 @llvm.ctpop.i29(i29 %val)是一个函数的不同重载形式
内置函数常对应外部的库函数，使用手工优化的库函数的接口
```

## 2. TableGen  Reference

[1] https://bcain-llvm.readthedocs.io/projects/llvm/en/latest/TableGen/LangIntro/

[2] https://llvm.org/docs/TableGen/ProgRef.html

[3] https://llvm.org/docs/TableGen/BackEnds.html

[4] https://llvm.org/docs/TableGen/BackGuide.html

### 2.1 简介

td文件可以include其他td文件，一般`<Target>.td`应包括所有其他文件。这保证了所需的所有信息都可以访问，并且在TableGen文件中不需要重复。

参数指定TableGen打印不同的内容，这个被称为一个backends(后端)，td时一种信息翻译，可以将td描述的域相关信息翻译成需要的信息格式，可以自己实现BackEnds。

包含两个概念：*abstract records*(classes) and *concrete records*(records).抽象记录不映射到c++ classes,Tablegen语言生成以及描述的信息安全取决于不同后端，**tablegen处理一组记录的集合，产生输出**

**tablegen的前端，定义的一组词法语法规则，经过前端parse处理后会产生一组record(前端是从.td->.td的变换规则)，这组record的具体语义由后端赋予，后端定义规则将一组record转化为需要的输出文件.**

```
/*
td的写法类似c++模板，会生成一组classes和record,如何将classes和record映射到特定领域，取决于某种后端
template <string n>
class Register;

template <bits<16> Enc, string n>
class TOYReg : Register<n>;
*/

class TOYReg<bits<16> Enc, string n> : Register<n> {
        let HWEncoding = Enc;
        let Namespace = "TOY";
}

// def R0-3
foreach i = 0-3 in {
        def R#i: TOYReg<i, "r"#i>;
}

def SP: TOYReg<13, "sp">;
def LR: TOYReg<14, "lr">;
def CPSR : TOYReg<16, "cspr">;
```

### **2.2 语法**

**预处理**

```
PreprocessorDirective ::=  "#define" | "#ifdef" | "#ifndef"
IncludeDirective ::=  "include" TokString
```

**类型**

bits<n>：位串，需要时，它将隐式转换为整数,{ a, b, 0b10 }

int:64 bit int 

string：字符串, ""

code: 代码片段，本质上还是字符串， [{}]

list<ty>: 类似list模板类，[ X, Y, Z ]<type>，一般type可推断出来

dag:有向无环图,(DEF DagArg ，*),DEF是一个有def定义的记录，除第一个参数外，其他参数为 Value[“：” TokVarName] |TokVarName

自定义record，直接使用标识符引用

**内置运算符**

对基本类型进行操作的内置函数

```
BangOperator ::=  one of
                  !add    !and         !cast         !con         !dag
                  !empty  !eq          !foldl        !foreach     !ge
                  !getop  !gt          !head         !if          !isa
                  !le     !listconcat  !listsplat    !lt          !mul
                  !ne     !or          !setop        !shl         !size
                  !sra    !srl         !strconcat    !subst       !tail
CondOperator ::=  !cond
/*
str1#str2：两个字符串合并成一个字符串，是!strconcat(a, b)的简化用法，隐式调用!cast<string>操作转换非字符串类型的操作数
!cast<type>(a)：
如果“ a”类型与类型不匹配，则TableGen会中止并显示错误。
否则，执行普通类型的转换，例如 在int和bit之间，或在记录类型之间。 这允许将记录强制转换为子类！cast <string>是一种特殊情况，因为参数可以是int或记录。 在后一种情况下，将返回记录的名称。
*/
```

**值、表达式和语句**

所有值都有将其从一种类型转换为另一种类型的规则

```
TableGenFile ::=  Statement*
Statement    ::=  Class | Def | Defm | Defset | Defvar | Foreach
                 | If | Let | MultiClass
```

class

```
定义了一个抽象记录类，其他类和记录可以从中继承
通过“模板参数”列表来对类进行参数化,未使用=为模板参数分配默认值，则该参数未初始化，在继承该类时必须在模板参数列表中指定该默认值
类C从继承D，D的字段合并到的C中
class或record的主题内容有三种形式:
BodyItem ::=  Type TokIdentifier ["=" Value] ";" 字段（filed）定义
             | "let" TokIdentifier ["{" RangeList "}"] "=" Value ";" 字段重赋值
             | "defvar" TokIdentifier "=" Value ";" 变量定义
变量不是字段，提供变量保存body临时值,变量的值可以在其他值表达式中使用
def 继承多个class时如果两个或多个父类提供相同的字段，则记录取值为最后一个父类的字段值
```

let

```
let语句绑定一组字段值（有时称为 ），并将它们应用于范围内的所定义的所有类和记录
需要在多个记录中覆盖几个字段时, let in
```

multiclasses

**multiclasses 像一个宏，defm在引用位置传入参数展开宏，生成一批def**

```
multiclasses提供了一种方便的方法来一次定义多个记录
The body of the multiclass contains a series of statements that define records, using Def and Defm.
defm “invoke” multiclasses 批量生成def record
multiclasses 内部可以使用defm调用其它multiclasses来生成一批def语句
```

defset

```
Defset ::=  "defset" Type TokIdentifier "=" "{" Statement* "}"
type must be list<class>, where class is some record class
```

defvar

```
Defvar ::=  "defvar" TokIdentifier "=" Value ";"
定义变量，类型可自动推导，一旦定义变量，值不可重新设置
```

dags

```
( operator operand1, operand2, … )
operator must be present and must be a record
The operator and operands can have three formats:
value	     |operand value
value:name	 |operand value and associated name
name	     | operand name with unset (uninitialized) value
a name is to tag an operator or operand in a DAG
!setop((foo 1, 2), bar) results in (bar 1, 2).
!getop((foo 1, 2)) results in foo
```

### 2.3 后端开发

前端将td文件处理为一组具体记录，进一步由后端处理。开发后端有两种方式，直接基于工具的c++源代码扩展编写新后端，或者先选择**通用json后端**输出json IR, 然后处理json

**通用json后端**

使用-dump-json选项将所有记录输出为JSON数据结构，可通过使用C ++扩展TableGen本身，也可以生成通用json(josn此时被视为前端和后端连接的通用IR)，然后处理json生成

**Extend c++**

信息编码到包含class和record的td文件中，后端提取record的子集生成输出文件

#### 2.3.1 数据结构

所有的外在表现都有一组内部数据结构对应，这组结构是前后端交互的IR

 **RecordKeeper**

#maps key is name

classes and records的容器，two maps one for classes and one for records

defvar定义的global variables maps

**Record**

#表示class or record ，使用boolean flag区分

record name

the vector of **field** names and their values,

and the vector of **superclasses** of the record 

a vector of the **class’s template arguments**

unique record ID

** `RecTy`**

 #表示the types of field values，或应该称为FieldTy

enumerated type specifying the specific type 

子类有: `BitRecTy`, `BitsRecTy`, `CodeRecTy`, `DagRecTy`, `IntRecTy`, `ListRecTy`, `RecordRecTy`, and `StringRecTy`, 自定义classes

**`Init` **

#表示值

 These classes include `BitInit`, `BitsInit`, `CodeInit`, `DagInit`, `DefInit`, `IntInit`, `ListInit`, and `StringInit`.

#当前llvm/lib/TableGen目录下提供了tablegen的前端和后端框架，基于这个lib可以开发自己的后端实现自定义tablegen工具，前端parser后会将信息存储到以上数据结构中，然后后端调用get接口获取want信息，进行翻译

