> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [cloud.tencent.com](https://cloud.tencent.com/developer/article/1786332)

> ? 作者：panhuili，腾讯 IEG 后台开发工程师 Go 作为当下最火的开发语言之一，它的优势不必多说。Go 对于高并发的支持，使得它可以很方便的作为独...

![](https://ask.qcloudimg.com/http-save/170434/ncrru1ygag.gif)

作者：panhuili，腾讯 IEG 后台开发工程师

> Go 作为当下最火的开发语言之一，它的优势不必多说。Go 对于高并发的支持，使得它可以很方便的作为独立模块嵌入业务系统。有鉴于我司大量的 C/C++ 存量代码，如何 将 Go 和 C/C++ 进行打通就尤为重要。Golang 自带的 CGO 可以支持与 C 语言接口的互通。本文首先介绍了 cgo 的常见用法，然后根据底层代码分析其实现机制，最后在特定场景下进行 cgo 实践。

### **一、CGO 快速入门**

#### **1.1、启用 CGO 特性**

在 golang 代码中加入 import “C” 语句就可以启动 CGO 特性。这样在进行 go build 命令时，就会在编译和连接阶段启动 gcc 编译器。

```
package main
import "C"      

func main() {
}
```

使用 -x 选项可以查看 go 程序编译过程中执行的所有指令。可以看到 golang 编译器已经为 test1.go 创建了 CGO 编译选项

```
[root@VM-centos ~/cgo_test/golink2]# go build -x test1.go
WORK=/tmp/go-build330287398
mkdir -p $WORK/b001/
cd /root/cgo_test/golink2
CGO_LDFLAGS='"-g" "-O2"' /usr/lib/golang/pkg/tool/linux_amd64/cgo -objdir $WORK/b001/ -importpath command-line-arguments -- -I $WORK/b001/ -g -O2 ./test1.go    # CGO编译选项
cd $WORK
gcc -fno-caret-diagnostics -c -x c - -o /dev/null || true
gcc -Qunused-arguments -c -x c - -o /dev/null || true
gcc -fdebug-prefix-map=a=b -c -x c - -o /dev/null || true
gcc -gno-record-gcc-switches -c -x c - -o /dev/null || true
.......
```

#### **1.2、Hello Cgo**

通过 import “C” 语句启用 CGO 特性后，CGO 会将上一行代码所处注释块的内容视为 C 代码块，被称为**序文（preamble）**。

```
package main


import "C"

func main() {
    C.puts(C.CString("Hello, Cgo\n"))
}
```

在序文中可以使用 C.func 的方式调用 C 代码块中的函数，包括库文件中的函数。对于 C 代码块的变量，类型也可以使用相同方法进行调用。

test2.go 通过 CGO 提供的 C.CString 函数将 Go 语言字符串转化为 C 语言字符串，最后再通过 C.puts 调用 <stdio.h> 中的 puts 函数向标准输出打印字符串。

#### **1.3 cgo 工具**

当你在包中引用 import "C"，go build 就会做很多额外的工作来构建你的代码，构建就不仅仅是向 go tool compile 传递一堆 .go 文件了，而是要先进行以下步骤：

1）cgo 工具就会被调用，在 C 转换 Go、Go 转换 C 的之间生成各种文件。

2）系统的 C 编译器会被调用来处理包中所有的 C 文件。

3）所有独立的编译单元会被组合到一个 .o 文件。

4）生成的 .o 文件会在系统的连接器中对它的引用进行一次检查修复。

cgo 是一个 Go 语言自带的特殊工具，可以使用命令 go tool cgo 来运行。它可以生成能够调用 C 语言代码的 Go 语言源文件，也就是说所有启用了 CGO 特性的 Go 代码，都会首先经过 cgo 的 "预处理"。

对 test2.go，cgo 工具会在同目录生成以下文件

```
_obj--|
      |--_cgo.o             
      |--_cgo_main.c        
      |--_cgo_flags         
      |--_cgo_export.c      
      |--_cgo_export.h      
      |--_cgo_gotypes.go    
      |--test1.cgo1.go      
      |--test1.cgo2.c
```

### **二、CGO 的 N 种用法**

CGO 作为 Go 语言和 C 语言之间的桥梁，其使用场景可以分为两种：Go 调用 C 程序 和 C 调用 Go 程序。

#### **2.1、Go 调用自定义 C 程序**

```
package main


import "C"
import (
    "fmt"
    "unsafe"
)

func main() {
    cblob := C.blob{}                               
    cblob.repeat_time = 0

    cblob.str = C.CString("Hello, World\n")         

    ret := C.SayHello(&cblob)                       

    fmt.Println("ret", ret)
    fmt.Println("repeat_time", cblob.repeat_time)

    C.free(unsafe.Pointer(cblob.str))               
}
```

CGO 会保留序文中的宏定义，但是并不会保留注释，也不支持 #program，**C 代码块中的 #program 语句极可能产生未知错误**。

CGO 中**使用 #cgo 关键字可以设置编译阶段和链接阶段的相关参数**，可以使用 ${SRCDIR} 来表示 Go 包当前目录的绝对路径。

使用 C. 结构名 或 C.struct_结构名 可以在 Go 代码段中定义 C 对象，并通过成员名访问结构体成员。

test3.go 中使用 C.CString 将 Go 字符串对象转化为 C 字符串对象，并将其传入 C 程序空间进行使用，由于 C 的内存空间不受 Go 的 GC 管理，因此需要显示的调用 C 语言的 free 来进行回收。详情见第三章。

#### **2.2、Go 调用 C/C++ 模块**

##### **2.2.1、简单 Go 调 C**

直接将完整的 C 代码放在 Go 源文件中，这种编排方式便于开发人员快速在 C 代码和 Go 代码间进行切换。

```
package main

import "C"
import (
    "fmt"
)

func main() {
    ret := C.SayHello()
    fmt.Println(ret)
}
```

但是当 CGO 中使用了大量的 C 语言代码时，将所有的代码放在同一个 go 文件中即不利于代码复用，也会影响代码的可读性。此时可以将 C 代码抽象成模块，再将 C 模块集成入 Go 程序中。

##### **2.2.2、Go 调用 C 模块**

将 C 代码进行抽象，放到相同目录下的 C 语言源文件 hello.c 中

```
#include <stdio.h>
int SayHello() {
 puts("Hello World");
    return 0;
}
```

在 Go 代码中，声明 SayHello() 函数，再引用 hello.c 源文件，就可以调起外部 C 源文件中的函数了。同理也可以将 **C 源码编译打包为静态库或动态库**进行使用。

```
package main

import "C"
import (
    "fmt"
)

func main() {
    ret := C.SayHello()
    fmt.Println(ret)
}
```

test5.go 中只对 SayHello 函数进行了声明，然后再通过链接 C 程序库的方式加载函数的实现。那么同样的，也可以通过**链接 C++ 程序库**的方式，来实现 Go 调用 C++ 程序。

##### **2.2.3、Go 调用 C++ 模块**

基于 test4。可以抽象出一个 hello 模块，将模块的接口函数在 hello.h 头文件进行定义

```
int SayHello();
```

再使用 C++ 来重新实现这个 C 函数

```
#include <iostream>

extern "C" {
    #include "hello.h"
}

int SayHello() {
 std::cout<<"Hello World";
    return 0;
}
```

最后再在 Go 代码中，引用 hello.h 头文件，就可以调用 C++ 实现的 SayHello 函数了

```
package main

import "C"
import (
    "fmt"
)

func main() {
    ret := C.SayHello()
    fmt.Println(ret)
}
```

CGO 提供的这种面向 C 语言接口的编程方式，使得开发者可以使用是任何编程语言来对接口进行实现，只要最终满足 C 语言接口即可。

#### **2.3、C 调用 Go 模块**

C 调用 Go 相对于 Go 调 C 来说要复杂多，可以分为两种情况。一是原生 Go 进程调用 C，C 中再反调 Go 程序。另一种是原生 C 进程直接调用 Go。

##### **2.3.1、Go 实现的 C 函数**

如前述，开发者可以用任何编程语言来编写程序，只要支持 CGO 的 C 接口标准，就可以被 CGO 接入。那么同样**可以用 Go 实现 C 函数接口**。

在 test6.go 中，已经定义了 C 接口模块 hello.h

```
void SayHello(char* s);
```

可以创建一个 hello.go 文件，来用 Go 语言实现 SayHello 函数

```
package main


import "C"
import "fmt"


func SayHello(str *C.char) {
    fmt.Println(C.GoString(str))
}
```

CGO 的 //export SayHello 指令将 Go 语言实现的 SayHello 函数导出为 C 语言函数。这样再 Go 中调用 C.SayHello 时，最终调用的是 hello.go 中定义的 Go 函数 SayHello

```
package main


import "C"

func main() {
    C.SayHello(C.CString("Hello World"))
}
```

Go 程序先调用 C 的 SayHello 接口，由于 SayHello 接口链接在 Go 的实现上，又调到 Go。

看起来调起方和实现方都是 Go，但实际执行顺序是 Go 的 main 函数，调到 CGO 生成的 C 桥接函数，最后 C 桥接函数再调到 Go 的 SayHello。这部分会在第四章进行分析。

##### **2.3.2、原生 C 调用 Go**

C 调用到 Go 这种情况比较复杂，Go 一般是便以为 c-shared/c-archive 的库给 C 调用。

```
package main

import "C"


func hello(value string)*C.char {   
    return C.CString("hello" + value)
}
func main(){
    
}
```

如果 Go 函数有多个返回值，会生成一个 C 结构体进行返回，结构体定义参考生成的. h 文件

生成 c-shared 文件 命令

```
go build -buildmode=c-shared -o hello.so hello.go
```

在 C 代码中，只需要引用 go build 生成的. h 文件，并在编译时链接对应的. so 程序库，即可从 C 调用 Go 程序

```
#include <stdio.h>
#include <string.h>
#include "hello.h"                       

int main(){
    char c1[] = "did";
    GoString s1 = {c1,strlen(c1)};       
    char *c = hello(s1);
    printf("r:%s",c);
    return 0;
}
```

编译命令

```
gcc -o c_go main.c hello.so
```

C 函数调入进 Go，必须按照 Go 的规则执行，当主程序是 C 调用 Go 时，也同样有一个 Go 的 runtime 与 C 程序并行执行。这个 runtime 的初始化在对应的 c-shared 的库加载时就会执行。因此，在进程启动时就有两个线程执行，一个 C 的，一 (多) 个是 Go 的。

### **三、类型转换**

想要更好的使用 CGO 必须了解 Go 和 C 之间类型转换的规则

#### **3.1、数值类型**

在 Go 语言中访问 C 语言的符号时，一般都通过虚拟的 “C” 包进行。比如 C.int，C.char 就对应与 C 语言中的 int 和 char，对应于 Go 语言中的 int 和 byte。

C 语言和 Go 语言的数值类型对应如下:

![](https://ask.qcloudimg.com/http-save/170434/d2we5pa15r.png)

Go 语言的 int 和 uint 在 32 位和 64 位系统下分别是 4 个字节和 8 个字节大小。它在 C 语言中的导出类型 GoInt 和 GoUint 在不同位数系统下内存大小也不同。

如下是 64 位系统中，Go 数值类型在 C 语言的导出列表

```
typedef signed char GoInt8;
typedef unsigned char GoUint8;
typedef short GoInt16;
typedef unsigned short GoUint16;
typedef int GoInt32;
typedef unsigned int GoUint32;
typedef long long GoInt64;
typedef unsigned long long GoUint64;
typedef GoInt64 GoInt;
typedef GoUint64 GoUint;
typedef __SIZE_TYPE__ GoUintptr;
typedef float GoFloat32;
typedef double GoFloat64;
typedef float _Complex GoComplex64;
typedef double _Complex GoComplex128;
```

需要注意的是**在 C 语言符号名前加上 _Ctype_， 便是其在 Go 中的导出名，因此在启用 CGO 特性后，Go 语言中禁止出现以 _Ctype_ 开头的自定义符号名，类似的还有 _Cfunc_ 等。**

可以在序文中引入_obj/_cgo_export.h 来显式使用 cgo 在 C 中的导出类型

```
package main


import "C"
import "fmt"
func main() {
 
 fmt.Println(C.Add(1, 2))
}
```

如下是 64 位系统中，C 数值类型在 Go 语言的导出列表

```
type _Ctype_char int8
type _Ctype_double float64
type _Ctype_float float32
type _Ctype_int int32
type _Ctype_long int64
type _Ctype_longlong int64
type _Ctype_schar int8
type _Ctype_short int16
type _Ctype_size_t = _Ctype_ulong
type _Ctype_uchar uint8
type _Ctype_uint uint32
type _Ctype_ulong uint64
type _Ctype_ulonglong uint64
type _Ctype_void [0]byte
```

为了提高 C 语言的可移植性，更好的做法是**通过 C 语言的 C99 标准引入的头文件**，不但每个数值类型都提供了明确内存大小，而且和 Go 语言的类型命名更加一致。

#### **3.2、切片**

Go 中切片的使用方法类似 C 中的数组，但是内存结构并不一样。C 中的数组实际上指的是一段连续的内存，而 Go 的切片在存储数据的连续内存基础上，还有一个头结构体，其内存结构如下

![](https://ask.qcloudimg.com/http-save/170434/cscefmn5qa.png)

因此 Go 的切片不能直接传递给 C 使用，而是需要取切片的内部缓冲区的首地址 (即首个元素的地址) 来传递给 C 使用。使用这种方式把 Go 的内存空间暴露给 C 使用，可以大大减少 Go 和 C 之间参数传递时内存拷贝的消耗。

```
package main


import "C"
import (
    "fmt"
    "unsafe"
)

func main() {
    buff := make([]byte, 8)
    C.SayHello((*C.char)(unsafe.Pointer(&buff[0])), C.int(len(buff)))
    a := string(buff)
    fmt.Println(a)
}
```

#### **3.3 字符串**

Go 的字符串与 C 的字符串在底层的内存模型也不一样：

![](https://ask.qcloudimg.com/http-save/170434/x6gf7yb8xq.png)

Go 的字符串并没有以'\0' 结尾，因此使用类似切片的方式，直接将 Go 字符串的首元素地址传递给 C 是不可行的。

##### **3.3.1、Go 与 C 的字符串传递**

cgo 给出的解决方案是标准库函数 C.CString()，它会在 C 内存空间内申请足够的空间，并将 Go 字符串拷贝到 C 空间中。因此 C.CString 申请的内存在 C 空间中，因此需要显式的调用 C.free 来释放空间，如 test3。

如下是 C.CString() 的底层实现

```
func _Cfunc_CString(s string) *_Ctype_char {        
 p := _cgo_cmalloc(uint64(len(s)+1))
 pp := (*[1<<30]byte)(p)
 copy(pp[:], s)
 pp[len(s)] = 0
 return (*_Ctype_char)(p)
}


func _cgo_cmalloc(p0 uint64) (r1 unsafe.Pointer) {
 _cgo_runtime_cgocall(_cgo_bb7421b6328a_Cfunc__Cmalloc, uintptr(unsafe.Pointer(&p0)))
 if r1 == nil {
  runtime_throw("runtime: C malloc failed")
 }
 return
}
```

**_Cfunc_CString**

_Cfunc_CString 是 cgo 定义的从 Go string 到 C char* 的类型转换函数

1）使用_cgo_cmalloc 在 C 空间内申请内存 (即不受 Go GC 控制的内存)

2）使用该段 C 内存初始化一个 []byte 对象

3）将 string 拷贝到 []byte 对象

4）将该段 C 空间内存的地址返回

它的实现方式类似前述，切片的类型转换。不同在于切片的类型转换，是将 Go 空间内存暴露给 C 函数使用。而_Cfunc_CString 是将 C 空间内存暴露给 Go 使用。

**_cgo_cmalloc**

定义了一个暴露给 Go 的 C 函数，用于在 C 空间申请内存

与 C.CString() 对应的是从 C 字符串转 Go 字符串的转换函数 C.GoString()。C.GoString() 函数的实现较为简单，检索 C 字符串长度，然后申请相同长度的 Go-string 对象，最后内存拷贝。

如下是 C.GoString() 的底层实现

```
func _cgo_runtime_gostring(*_Ctype_char) string

func _Cfunc_GoString(p *_Ctype_char) string {           
 return _cgo_runtime_gostring(p)
}


func gostring(p *byte) string {             
 l := findnull(p)
 if l == 0 {
  return ""
 }
 s, b := rawstring(l)
 memmove(unsafe.Pointer(&b[0]), unsafe.Pointer(p), uintptr(l))
 return s
}
```

##### **3.3.2、更高效的字符串传递方法**

C.CString 简单安全，但是它涉及了一次从 Go 到 C 空间的内存拷贝，对于长字符串而言这会是难以忽视的开销。

Go 官方文档中声称 string 类型是” 不可改变的 “，但是在实操中可以发现，除了常量字符串会在编译期被分配到只读段，其他的动态生成的字符串实际上都是在堆上。

因此如果能够获得 string 的内存缓存区地址，那么就可以使用类似切片传递的方式将字符串指针和长度直接传递给 C 使用。

查阅源码，可知 String 实际上是由缓冲区首地址 和 长度构成的。这样就可以通过一些方式拿到缓存区地址。

```
type stringStruct struct {
 str unsafe.Pointer  
 len int             
}
```

test11.go 将 fmt 动态生成的 string 转为自定义类型 MyString 便可以获得缓冲区首地址，将地址传入 C 函数，这样就可以在 C 空间直接操作 Go-String 的内存空间了，这样可以免去内存拷贝的消耗。

```
package main


import "C"
import (
    "fmt"
    "unsafe"
)

type MyString struct {
 Str *C.char
 Len int
}
func main() {
    s := fmt.Sprintf("             ")
    C.SayHello((*MyString)(unsafe.Pointer(&s)).Str, C.int((*MyString)(unsafe.Pointer(&s)).Len))
    fmt.Print(s)
}
```

这种方法背离了 Go 语言的设计理念，如非必要，不要把这种代码带入你的工程，这里只是作为一种 “黑科技” 进行分享。

#### **3.4、结构体，联合，枚举**

cgo 中结构体，联合，枚举的使用方式类似，可以通过 C.struct_XXX 来访问 C 语言中 struct XXX 类型。union,enum 也类似。

##### **3.4.1、结构体**

如果结构体的成员名字中碰巧是 Go 语言的关键字，可以通过在成员名开头添加下划线来访问

如果有 2 个成员：一个是以 Go 语言关键字命名，另一个刚好是以下划线和 Go 语言关键字命名，那么以 Go 语言关键字命名的成员将无法访问（被屏蔽）

C 语言结构体中位字段对应的成员无法在 Go 语言中访问，如果需要操作位字段成员，需要通过在 C 语言中定义辅助函数来完成。对应零长数组的成员 (C 中经典的变长数组)，无法在 Go 语言中直接访问数组的元素，但同样可以通过在 C 中定义辅助函数来访问。

结构体的内存布局按照 C 语言的通用对齐规则，在 32 位 Go 语言环境 C 语言结构体也按照 32 位对齐规则，在 64 位 Go 语言环境按照 64 位的对齐规则。**对于指定了特殊对齐规则的结构体，无法在 CGO 中访问。**

```
package main

import "C"
import "fmt"
func main() {
    test := C.struct_Test{}
    fmt.Println(test.a)
    fmt.Println(test.b)
    fmt.Println(test._type)
    
    fmt.Println(test.arr1[0])
    
    

    test2 := C.struct_Test2{}
    fmt.Println(test2.c)
    
}
```

##### **3.4.2、联合**

Go 语言中并不支持 C 语言联合类型，它们会被转为对应大小的字节数组。

如果需要操作 C 语言的联合类型变量，一般有三种方法：第一种是在 C 语言中定义辅助函数；第二种是通过 Go 语言的 "encoding/binary" 手工解码成员 (需要注意大端小端问题)；第三种是使用`unsafe`包强制转型为对应类型 (这是性能最好的方式)。

test12 给出了 union 的三种访问方式

```
package main

import "C"
import (
    "fmt"
    "unsafe"
    "encoding/binary"
)

func main() {
    SayHello := C.init_sayhello()
    fmt.Println("C-helper ",C.SayHello_Say_helper(&SayHello))           
    buff := C.GoBytes(unsafe.Pointer(&SayHello), 4)
    Say2 := binary.LittleEndian.Uint32(buff)
    fmt.Println("binary ",Say2)                 
    fmt.Println("unsafe modify ", *(*C.int)(unsafe.Pointer(&SayHello)))     
}
```

##### **3.4.3、枚举**

对于枚举类型，可以通过`C.enum_xxx`来访问 C 语言中定义的`enum xxx`结构体类型。

使用方式和 C 相同，这里就不列例子了

#### **3.5、指针**

在 Go 语言中两个指针的类型完全一致则不需要转换可以直接通用。如果一个指针类型是用 type 命令在另一个指针类型基础之上构建的，换言之两个指针底层是相同完全结构的指针，那么也可以通过直接强制转换语法进行指针间的转换。

但是 C 语言中，不同类型的指针是可以显式或隐式转换。cgo 经常要面对的是 2 个完全不同类型的指针间的转换，实现这一转换的关键就是 unsafe.Pointer, 类似于 C 语言中的 Void * 类型指针。

![](https://ask.qcloudimg.com/http-save/170434/v0ivlnu1qb.png)

使用这种方式就可以实现不同类型间的转换，如下是从 Go - int32 到 *C.char 的转换。

### **四、内部机制**

go tool cgo 是分析 CGO 内部运行机制的重要工具，本章根据 cgo 工具生成的中间代码，再辅以 Golang 源码中 runtime 部分，来对 cgo 的内部运行机制进行分析。

cgo 的工作流程为：代码预处理 -> gcc 编译 -> Go Complier 编译。其产生的中间文件如图所示

![](https://ask.qcloudimg.com/http-save/170434/je8tyn2635.png)

#### **4.1、Go 调 C**

Go 调 C 的过程比较简单。test13 中定义了一个 C 函数 sum，并在 Go 中调用了 C.sum。

```
package main


import "C"

func main() {
 println(C.sum(1, 1))
}
```

下面是 cgo 工具产生的中间文件，最重要的是 test13.cgo1.go，test13.cgo1.c，_cgo_gotypes.go

##### **test13.cgo1.go**

test13.cgo1.go 是原本 test13.go 被 cgo 处理之后的文件。

```
package main


import _ "unsafe"

func main() {
 println(( _Cfunc_sum )(1, 1))
}
```

这个文件才是 go complier 真正编译的代码。可以看到原本的`C.sum` 被改写为`_Cfunc_sum`，`_Cfunc_sum`的定义在_cgo_gotypes.go 中。

##### **_cgo_gotypes.go**

```
package main

import "unsafe"

import _ "runtime/cgo"

import "syscall"

var _ syscall.Errno
func _Cgo_ptr(ptr unsafe.Pointer) unsafe.Pointer { return ptr }


var _Cgo_always_false bool              

func _Cgo_use(interface{})              
type _Ctype_int int32                   

type _Ctype_void [0]byte                


func _cgo_runtime_cgocall(unsafe.Pointer, uintptr) int32            


func _cgo_runtime_cgocallback(unsafe.Pointer, unsafe.Pointer, uintptr, uintptr)     


func _cgoCheckPointer(interface{}, interface{})             


func _cgoCheckResult(interface{})               



var __cgofn__cgo_53efb99bd95c_Cfunc_sum byte                
var _cgo_53efb99bd95c_Cfunc_sum = unsafe.Pointer(&__cgofn__cgo_53efb99bd95c_Cfunc_sum)  


func _Cfunc_sum(p0 _Ctype_int, p1 _Ctype_int) (r1 _Ctype_int) {
 _cgo_runtime_cgocall(_cgo_53efb99bd95c_Cfunc_sum, uintptr(unsafe.Pointer(&p0))) 
 if _Cgo_always_false {
  _Cgo_use(p0)            
  _Cgo_use(p1)
 }
 return
}
```

_cgo_gotypes.go 是 Go 调 C 的精髓，这里逐段分析。

##### **_Cgo_always_false & _Cgo_use**

```
var _Cgo_always_false bool              

func _Cgo_use(interface{})              

..........

if _Cgo_always_false {
 _Cgo_use(p0)            
 _Cgo_use(p1)
}
```

_Cgo_always_false 是一个 "常量"，正常情况下永远为 false。

`_Cgo_use`的函数实现如下

```
func cgoUse(interface{}) { throw("cgoUse should not be called") }
```

Go 中变量可以分配在栈或者堆上。栈中变量的地址会随着 go 程调度，发生变化。堆中变量则不会。

而程序进入到 C 空间后，会脱离 Go 程的调度机制，所以必须保证 C 函数的参数分配在堆上。

Go 通过在编译器里做逃逸分析来决定一个对象放栈上还是放堆上，不逃逸的对象放栈上，可能逃逸的放堆上。

由于栈上内存存在不需要 gc，内存碎片少，分配速度快等优点，所以 Go 会将变量更多的放在栈上。

`_Cgo_use`以 interface 类型为入参，编译器很难在编译期知道，变量最后会是什么类型，因此它的参数都会被分配在堆上。

##### **_cgo_runtime_cgocall**

```
func _cgo_runtime_cgocall(unsafe.Pointer, uintptr) int32
```

`_cgo_runtime_cgocall`是从 Go 调 C 的关键函数，这个函数里面做了一些调度相关的安排。

```
func cgocall(fn, arg unsafe.Pointer) int32 {
 if !iscgo && GOOS != "solaris" && GOOS != "illumos" && GOOS != "windows" {
  throw("cgocall unavailable")
 }

 if fn == nil {
  throw("cgocall nil")
 }

 if raceenabled {                
  racereleasemerge(unsafe.Pointer(&racecgosync))
 }

 mp := getg().m
 mp.ncgocall++           
 mp.ncgo++               

 
 mp.cgoCallers[0] = 0    

 
 
 
 
 
 
 
 
 
 
 
 
 
 
 entersyscall()      

 
 
 
 
 
 osPreemptExtEnter(mp)   

 mp.incgo = true
 errno := asmcgocall(fn, arg)            

 
 
 mp.incgo = false
 mp.ncgo--

 osPreemptExtExit(mp)    

 exitsyscall()       

 
 
 if raceenabled {
  raceacquire(unsafe.Pointer(&racecgosync))
 }

 
 
 
 
 
 
 
 
 
 
 KeepAlive(fn)               
 KeepAlive(arg)
 KeepAlive(mp)

 return errno
}
```

Go 调入 C 之后，程序的运行将不受 Go 的 runtime 的管控。一个正常的 Go 函数是需要 runtime 的管控的，即函数的运行时间过长会导致 goroutine 的抢占，以及 GC 的执行会导致所有的 goroutine 被拉齐。

C 程序的执行，限制了 Go 的 runtime 的调度行为。为此，Go 的 runtime 会在进入到 C 程序之后，会标记这个运行 C 的线程 M 将其排除出调度。

此外，由于正常的 Go 程序运行在一个 2K 的栈上，而 C 程序需要一个无穷大的栈。因此在进去 C 函数之前需要把当前线程的栈从 2K 的栈切换到线程本身的系统栈上，即切换到 g0。

cgocall 中几个重要函数功能说明：

1）`entersyscall()` 将当前的 M 与 P 剥离，防止 C 程序独占 M 时，阻塞 P 的调度。

2）`asmcgocall()` 将栈切换到 g0 的系统栈，并执行 C 函数调用

3）`exitsyscall()`寻找合适的 P 来运行从 C 函数返回的 Go 程，优先选择调用 C 之前依附的 P，其次选择其他空闲的 P

下图是 Go 调 C 函数过程中，MPG 的调度过程。

![](https://ask.qcloudimg.com/http-save/170434/1ydfako267.png)

**当 Go 程在调用 C 函数时，会单独占用一个系统线程。因此如果在 Go 程中并发调用 C 函数，而 C 函数中又存在阻塞操作，就很可能会造成 Go 程序不停的创建新的系统线程，而 Go 并不会回收系统线程，过多的线程数会拖垮整个系统。**

##### **_cgoCheckPointer & _cgoCheckResult**

```
func _cgoCheckPointer(interface{}, interface{})             


func _cgoCheckResult(interface{})
```

`_cgoCheckPointer` 检查传入 C 函数的参数，防止其中包含了指向 Go 指针的 Go 指针，防止间接指向的对象在 Go 调度中发生内存位置变化

`_cgoCheckResult` 与`_cgoCheckPointer` 类似 用于检测 C 函数调 Go 函数后，Go 函数的返回值。防止其包含了 Go 指针。

##### **cgofncgo_53efb99bd95c_Cfunc_sum**

```
var __cgofn__cgo_53efb99bd95c_Cfunc_sum byte                
var _cgo_53efb99bd95c_Cfunc_sum = unsafe.Pointer(&__cgofn__cgo_53efb99bd95c_Cfunc_sum)
```

1)`go:cgo_import_static` 将 C 函数`_cgo_53efb99bd95c_Cfunc_sum`加载到 Go 空间中

1.  go:linkname 将 Go 的 byte 对象`__cgofn__cgo_53efb99bd95c_Cfunc_sum`的内存空间链接到 C 函数 `_cgo_53efb99bd95c_Cfunc_sum`的内存空间
2.  创建 Go 对象`_cgo_53efb99bd95c_Cfunc_sum`并赋值 C 函数地址。

前两行的`_cgo_53efb99bd95c_Cfunc_sum`指的是 C 函数的符号

最后一行的`_cgo_53efb99bd95c_Cfunc_sum`指的是 Go 的 unsafe 指针

通过上面三步，cgo 将 C 函数`_cgo_53efb99bd95c_Cfunc_sum`的地址赋值给了 Go 指针`_cgo_53efb99bd95c_Cfunc_sum`

##### **_Cfunc_sum**

`_Cfunc_sum` 是 C 函数 sum 在 Go 空间的入口。它的参数 p0，p1 通过_Cgo_use 逃逸到了堆上。

再将存储 C 函数地址的指针和参数列表传入`_cgo_runtime_cgocall` ，即可完成从 Go 调 C 函数。

```
func _Cfunc_sum(p0 _Ctype_int, p1 _Ctype_int) (r1 _Ctype_int) {
 _cgo_runtime_cgocall(_cgo_53efb99bd95c_Cfunc_sum, uintptr(unsafe.Pointer(&p0))) 
 if _Cgo_always_false {
  _Cgo_use(p0)            
  _Cgo_use(p1)
 }
 return
}
```

其函数调用流程如图示：

![](https://ask.qcloudimg.com/http-save/170434/shzgssxzvx.png)

#### **4.2、C 调 Go**

C 调 Go 的过程相对 Go 调 C 来说更为复杂，又可以分为两种情况。一种是从 Go 调用 C，然后 C 再调 Go。另一种是原生的 C 线程调 Go。

在 test14 中，分别创建了 test14.go 和 hello.go，两者之间通过 C 函数调起。

```
package main


import "C"
import "fmt"


func GSayHello(value *C.char) C.int{   
 fmt.Print(C.GoString(value))
 return C.int(1)
}
```

```
package main


import "C"


func main(){
 buff := C.CString("hello cgo")
    C.CSayHello(buff, C.int(10))
}
```

可以看到 test14 的工作流程是，从 Go 调到 C 的`CSayHello` 函数，再从`CSayHello`调用 Go 的`GSayHello`函数。从 Go 调 C 的流程上节已经分析，这里主要关注从 C 调 Go 的部分。使用 cgo 工具对 hello.go 进行分析，C 调 Go 函数主要在_cgo_gotypes.go(Go 函数导出) 和 _cgo_export.c(C 调 Go 入口)。

##### **_cgo_gotypes.go**

首先对被 C 调用的`GSayHello`函数的分析。`GSayHello`的实现在_cgo_gotypes.go，剔除与 4.1 中重复部分，_cgo_gotypes.go 源码如下

```
func _cgoexp_25bb4eb897ab_GSayHello(a unsafe.Pointer, n int32, ctxt uintptr) {
 fn := _cgoexpwrap_25bb4eb897ab_GSayHello
 _cgo_runtime_cgocallback(**(**unsafe.Pointer)(unsafe.Pointer(&fn)), a, uintptr(n), ctxt);
}

func _cgoexpwrap_25bb4eb897ab_GSayHello(p0 *_Ctype_char) (r0 _Ctype_int) {
 return GSayHello(p0)
}
```

1）`go:cgo_export_dynamic` 在内链模式 (internal linking) 下将 Go 的 hello 函数符号暴露给 C

2）`go:linkname _cgoexp_bb7421b6328a_hello _cgoexp_bb7421b6328a_hello` 将 Go 函数`_cgoexp_bb7421b6328a_hello`链接到符号`_cgoexp_bb7421b6328a_hello`上

3）`go:cgo_export_static _cgoexp_bb7421b6328a_hello`在外链模式 (external linking) 下将`_cgoexp_bb7421b6328a_hello`符号暴露给 C

4）`go:nosplit go:norace` 关闭溢出检测 关闭竞态管理

`_cgoexp_bb7421b6328a_hello` 即为 C 调用 Go 函数的入口函数，之后调用到`_cgoexpwrap_25bb4eb897ab_GSayHello` ，最后调用到用户定义的 Go 函数`GSayHello`。

##### **_cgo_export.c**

_cgo_export.c 包含了 C 调用 Go 函数的入口 和 暴露给 Go 的内存分配函数`_Cfunc__Cmalloc(void *v)`。

C 代码较为简单，不过多分析

```
#include <stdlib.h>
#include "_cgo_export.h"

#pragma GCC diagnostic ignored "-Wunknown-pragmas"
#pragma GCC diagnostic ignored "-Wpragmas"
#pragma GCC diagnostic ignored "-Waddress-of-packed-member"
extern void crosscall2(void (*fn)(void *, int, __SIZE_TYPE__), void *, int, __SIZE_TYPE__);         
extern __SIZE_TYPE__ _cgo_wait_runtime_init_done(void);
extern void _cgo_release_context(__SIZE_TYPE__);

extern char* _cgo_topofstack(void);
#define CGO_NO_SANITIZE_THREAD
#define _cgo_tsan_acquire()
#define _cgo_tsan_release()


#define _cgo_msan_write(addr, sz)

extern void _cgoexp_25bb4eb897ab_GSayHello(void *, int, __SIZE_TYPE__);

CGO_NO_SANITIZE_THREAD
int GSayHello(char* value)          
{
 __SIZE_TYPE__ _cgo_ctxt = _cgo_wait_runtime_init_done();
 struct {
  char* p0;
  int r0;
  char __pad0[4];
 } __attribute__((__packed__, __gcc_struct__)) _cgo_a;
 _cgo_a.p0 = value;
 _cgo_tsan_release();
 crosscall2(_cgoexp_25bb4eb897ab_GSayHello, &_cgo_a, 16, _cgo_ctxt);
 _cgo_tsan_acquire();
 _cgo_release_context(_cgo_ctxt);
 return _cgo_a.r0;
}
```

`crosscall2`对应的底层函数是 runtime.cgocallback，cgocallback 会恢复 Golang 运行时所需的环境包括 Go 函数地址，栈帧和上下文，然后会调用到 cgocallback_gofunc。

`cgocallback_gofunc`，首先判断当前线程是否为 Go 线程，再讲线程栈切到 Go 程栈，再将函数地址，参数地址等信息入 Go 程栈，最后调用到 cgocallbackg。

`cgocallbackg`确认 Go 程准备完毕后，就将线程从系统调用状态退出 (见上节 exitsyscall)，此时程序运行在 G 栈上，进入 cgocallbackg1 函数。

`cgocallbackg1`调用 reflectcall，正式进入到用户定义的 Go 函数。

如下是函数调用关系：

![](https://ask.qcloudimg.com/http-save/170434/8r5vah7v02.png)

从 Go 调入到 C 函数时，系统线程会被切到 G0 运行，之后从 C 再回调到 Go 时，会直接在同一个 M 上从 G0 切回到普通的 Go 程，在这个过程中并不会创建新的系统线程。

从原生 C 线程调用 Go 函数的流程与这个类似，C 程序在一开始就有两个线程，一个是 C 原生线程，一个是 Go 线程，当 C 函数调起 Go 函数时，会切到 Go 线程运行。

如下是 Go 调 C，C 再调 Go 过程中，MPG 的调度流程。

![](https://ask.qcloudimg.com/http-save/170434/ian0kk0cff.png)

### **五、总结**

CGO 是一个非常优秀的工具，大部分使用 CGO 所造成的问题，都是因为使用方法不规范造成的。希望本文可以帮助大家更好的使用 CGO。

参考资料：

1.[Golang 源码](https://github.com/golang/go)

2. [赵志强的博客](https://bbs.huaweicloud.com/community/usersnew/id_1510903197647472)

3.[Go 语言高级编程](https://www.cntofu.com/book/73/index.html)

4. 给出了一种针对 Go 调 C 的优化方法，大大降低了 Go 调 C 的性能开销: https://bbs.huaweicloud.com/blogs/117132

5. 给出了一种会造成线程暴增的 cgo 错误使用方法: http://xiaorui.cc/archives/5408

6. 给出了一种会造成内存溢出的 cgo 错误使用方法: https://blog.csdn.net/wei_gw2012/article/details/86666506

![](https://ask.qcloudimg.com/http-save/170434/105uh6zzub.gif)