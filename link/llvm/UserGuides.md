## [6. Additional Topics](https://llvm.org/docs/UserGuides.html#id8)

### 6.1 CommandLine 2.0 Library Manual

#include "llvm/Support/CommandLine.h"

命令行解析由 cl::ParseCommandLineOptions(argc, argv)开始, 用户以声明式定义提供**捕获器**(cl:opt， cl::list, cl::bits等)捕获感兴趣的参数

捕获器需要声明捕获的类型和storage存储位置和parser；默认storage存储在捕获器内部，可以以cl::location指定存储到外部storage对象；大多数常用类型由默认的parser class，必要时可自定义parser；典型的捕获器如下:

```
namespace cl {
  template <class DataType, bool ExternalStorage = false,
            class ParserClass = parser<DataType> >
  class opt;
  
  template <class DataType, class Storage = bool,
            class ParserClass = parser<DataType> >
  class list;
  
  template <class DataType, class Storage = bool,
            class ParserClass = parser<DataType> >
  class bits;
}
```

声明定义捕获器时，使用不同的参数和选项可以配置捕获参数的类型，个数，方式，初始值，约束，help打印等内容



