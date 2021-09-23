> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [zhuanlan.zhihu.com](https://zhuanlan.zhihu.com/p/267341653)

作者：yukkizhang，腾讯 CSIG 专项技术测试工程师

### **一、单元测试**

### **1. 单元测试是什么**

**单元**是应用的最小可测试部件。在过程化编程中，一个单元就是单个程序、函数、过程等；对于面向对象编程，最小单元就是方法，包括基类、超类、抽象类等中的方法。**单元测试**就是软件开发中对最小单位进行正确性检验的测试工作。

不同地方对单元测试有的定义可能会有所不同，但有一些基本共识：

*   单元测试是比较**底层**的，关注代码的**局部**而不是整体。
*   单元测试是**开发人员**在写代码时候写的。
*   单元测试需要比其他测试运行得**快**。

### **2. 单元测试的意义**

*   **提高代码质量**。代码测试都是为了帮助开发人员发现问题从而解决问题，提高代码质量。
*   **尽早发现问题**。问题越早发现，解决的难度和成本就越低。
*   **保证重构正确性**。随着功能的增加，重构（修改老代码）几乎是无法避免的。很多时候我们不敢重构的原因，就是担心其它模块因为依赖它而不工作。有了单元测试，只要在改完代码后运行一下单测就知道改动对整个系统的影响了，从而可以让我们放心的重构代码。
*   **简化调试过程**。单元测试让我们可以轻松地知道是哪一部分代码出了问题。
*   **简化集成过程**。由于各个单元已经被测试，在集成过程中进行的后续测试会更加容易。
*   **优化代码设计**。编写测试用例会迫使开发人员仔细思考代码的设计和必须完成的工作，有利于开发人员加深对代码功能的理解，从而形成更合理的设计和结构。
*   **单元测试是最好的文档**。单元测试覆盖了接口的所有使用方法，是最好的示例代码。而真正的文档包括注释很有可能和代码不同步，并且看不懂。

### **3. 单元测试用例编写的原则**

### **3.1 理论原则**

*   **快**。单元测试是回归测试，可以在开发过程的任何时候运行，因此运行速度必须快
*   **一致性**。代码没有改变的情况下，每次运行得结果应该保持确定且一致
*   **原子性**。结果只有两种情况：Pass / Fail
*   **用例独立**。执行顺序不影响；用例间没有状态共享或者依赖关系；用例没有副作用（执行前后环境状态一致）
*   **单一职责**。一个用例只负责一个场景
*   **隔离**。功能可能依赖于数据库、web 访问、环境变量、系统时间等；一个单元可能依赖于另一部分代码，用例应该解除这些依赖
*   **可读性**。用例的名称、变量名等应该具有可读性，直接表现出该测试的目标
*   **自动化**。单元测试需要全自动执行。测试程序不应该有用户输入；测试结果应该能直接被电脑获取，不应该由人来判断。

### **3.2 规约原则**

在实际编写代码过程中，不同的团队会有不同团队的风格，只要团队内部保持有一定的规约即可，比如：

*   单元测试文件名必须以 xxx_test.go 命名
*   方法必须是 TestXxx 开头，建议风格保持一致（驼峰或者下划线）
*   方法参数必须 t *testing.T
*   测试文件和被测试文件必须在一个包中

### **3.3 衡量原则**

单元测试是要写额外的代码的，这对开发同学的也是一个不小的工作负担，在一些项目中，我们合理的评估单元测试的编写，我认为我们不能走极端，当然理论上来说全写肯定时好的，但是从成本，效率上来说我们必须做出权衡，衡量原则如下：

*   优先编写核心组件和逻辑模块的测试用例
*   逻辑类似的组件如果存在多个，优先编写其中一种逻辑组件的测试用例
*   发现 Bug 时一定先编写测试用例进行 Debug
*   关键 util 工具类要编写测试用例，这些 util 工具适用的很频繁，所以这个原则也叫做热点原则，和第 1 点相呼应。
*   测试用户应该独立，一个文件对应一个，而且不同的测试用例之间不要互相依赖。
*   测试用例的保持更新

### **4. 单元测试用例设计方法**

### **4.1 规范 (规格) 导出法**

**规范 (规格) 导出法**将需求” 翻译 “成测试用例。

例如，一个函数的设计需求如下：

> 函数：一个计算平方根的函数 输入： 实数 输出： 实数 要求： 当输入一个 0 或者比 0 大的实数时，返回其正的平方根；当输入一个小于 0 的实数时，显示错误信息 “平方根非法—输入之小于 0”，并返回 0；库函数`printf()`可以用来输出错误信息。  

在这个规范中有 3 个陈述，可以用两个测试用例来对应:

*   测试用例 1：输入 4，输出 2。
*   测试用例 2：输入 - 1，输出 0。

### **4.2 等价类划分法**

**等价类划分法**假定某一特定的等价类中的所有值对于测试目的来说是等价的，所以在每个等价类中找一个之作为测试用例。

*   按照 [输入条件][有效等价类][无效等价类] 建立等价类表，列出所有划分出的等价类
*   为每一个等价类规定一个唯一的编号
*   设计一个新的测试用例，使其尽可能多地覆盖尚未被覆盖地有效等价类。重复这一步，直到所有的有效等价类都被覆盖为止
*   设计一个新的测试用例，使其仅覆盖一个尚未被覆盖的无效等价类。重复这一步，直到所有的无效等价类都被覆盖为止

例如，注册邮箱时要求用 6~18 个字符，可使用字母、数字、下划线，需以字母开头。

![](https://pic3.zhimg.com/v2-d3f8d00254db7342ae12bac420c32382_r.jpg)

测试用例：

![](https://pic1.zhimg.com/v2-99f0db8c3e72e4bdd882d893209b7c60_r.jpg)

### **4.3 边界值分析法**

**边界值分析法**使用与等价类测试方法相同的等价类划分，只是边界值分析假定 错误更多地存在于两个划分的边界上。

边界值测试在软件变得复杂的时候也会变得不实用。边界值测试对于非向量类型的值 (如枚举类型的值) 也没有意义。

例如，和 **4.1** 相同的需求： 划分 (ii) 的边界为 0 和最大正实数；划分 (i) 的边界为最小负实数和 0。由此得到以下测试用例：

*   输入 {最小负实数}
*   输入 {绝对值很小的负数}
*   输入 0
*   输入 {绝对值很小的正数}
*   输入 {最大正实数}

### **4.4 基本路径测试法**

**基本路径测试法**是在程序控制流图的基础上，通过分析控制构造的环路复杂性，导出基本可执行路径集合，从而设计测试用例的方法。设计出的测试用例要保证在测试中程序的每个可执行语句至少执行一次。

基本路径测试法的基本步骤：

*   程序的控制流图：描述程序控制流的一种图示方法。
*   程序圈复杂度：McCabe 复杂性度量。从程序的环路复杂性可导出程序基本路径集合中的独立路径条数，这是确定程序中每个可执行语句至少执行一次所必须的测试用例数目的上界。
*   导出测试用例：根据圈复杂度和程序结构设计用例数据输入和预期结果。
*   准备测试用例：确保基本路径集中的每一条路径的执行。

### **二、Golang 的测试框架**

Golang 有这几种比较常见的测试框架：

![](https://pic4.zhimg.com/v2-7c39812b824bc15e606123ce986c2adb_r.jpg)

从测试用例编写的简易难度上来说：testify 比 GoConvey 简单；GoConvey 比 Go 自带的 testing 包简单。 　　但在测试框架的选择上，我们更推荐 GoConvey。因为：

*   GoConvey 和其他 Stub/Mock 框架的兼容性相比 Testify 更好。
*   Testify 自带 Mock 框架，但是用这个框架 Mock 类需要自己写。像这样重复有规律的部分在 GoMock 中是一键自动生成的。

### **1. Go 自带的 testing 包**

`testing` 为 Go 语言 package 提供自动化测试的支持。通过 `go test` 命令，能够自动执行如下形式的任何函数：

```
func TestXxx(*testing.T)
```

注意：`Xxx` 可以是任何字母数字字符串，但是第一个字母不能是小写字母。

在这些函数中，使用 `Error`、`Fail` 或相关方法来发出失败信号。

要编写一个新的测试套件，需要创建一个名称以 _test.go 结尾的文件，该文件包含 `TestXxx` 函数，如上所述。 将该文件放在与被测试文件相同的包中。该文件将被排除在正常的程序包之外，但在运行 `go test` 命令时将被包含。 有关详细信息，请运行 `go help test` 和 `go help testflag` 了解。

### **1.1 第一个例子**

被测代码：

```
func Fib(n int) int {
        if n < 2 {
                return n
        }
        return Fib(n-1) + Fib(n-2)
}
```

测试代码：

```
func TestFib(t *testing.T) {
    var (
        in       = 7
        expected = 13
    )
    actual := Fib(in)
    if actual != expected {
        t.Errorf("Fib(%d) = %d; expected %d", in, actual, expected)
    }
}
```

执行 `go test .` ，输出：

```
$ go test .
ok      chapter09/testing    0.007s
```

表示测试通过。 我们将 `Sum` 函数改为：

```
func Fib(n int) int {
        if n < 2 {
                return n
        }
        return Fib(n-1) + Fib(n-1)
}
```

再执行 `go test .` ，输出：

```
$ go test .
--- FAIL: TestSum (0.00s)
    t_test.go:16: Fib(10) = 64; expected 13
FAIL
FAIL    chapter09/testing    0.009s
```

### **1.2 Table-Driven 测试**

Table-Driven 的方式将多个 case 在同一个测试函数中测到：

```
func TestFib(t *testing.T) {
   var fibTests = []struct {
       in       int // input
       expected int // expected result
   }{
       {1, 1},
       {2, 1},
       {3, 2},
       {4, 3},
       {5, 5},
       {6, 8},
       {7, 13},
   }

   for _, tt := range fibTests {
       actual := Fib(tt.in)
       if actual != tt.expected {
           t.Errorf("Fib(%d) = %d; expected %d", tt.in, actual, tt.expected)
       }
   }
}
```

**[Go 自带 testing 包的更多用法](https://link.zhihu.com/?target=https%3A//books.studygolang.com/The-Golang-Standard-Library-by-Example/chapter09/09.1.html)**

### **2. GoConvey：简单断言**

Convey 适用于书写单元测试用例，并且可以兼容到 testing 框架中，`go test`命令或者使用`goconvey`命令访问`localhost:8080`的 Web 测试界面都可以查看测试结果。

```
Convey("Convey return : ", t, func() {
        So(...)
})
```

一般 Convey 用`So`来进行断言，断言的方式可以传入一个函数，或者使用自带的`ShouldBeNil`、`ShouldEqual`、`ShouldNotBeNil`函数等。

### **2.1. 基本用法**

被测代码：

```
func StringSliceEqual(a, b []string) bool {
    if len(a) != len(b) {
        return false
    }

    if (a == nil) != (b == nil) {
        return false
    }

    for i, v := range a {
        if v != b[i] {
            return false
        }
    }
    return true
}
```

测试代码

```
import (
    "testing"
    . "github.com/smartystreets/goconvey/convey"
)

func TestStringSliceEqual(t *testing.T) {
    Convey("TestStringSliceEqual的描述", t, func() {
        a := []string{"hello", "goconvey"}
        b := []string{"hello", "goconvey"}
        So(StringSliceEqual(a, b), ShouldBeTrue)
    })
}
```

### **2.2. 双层嵌套**

```
import (
    "testing"
    . "github.com/smartystreets/goconvey/convey"
)

func TestStringSliceEqual(t *testing.T) {
    Convey("TestStringSliceEqual", t, func() {
        Convey("true when a != nil  && b != nil", func() {
            a := []string{"hello", "goconvey"}
            b := []string{"hello", "goconvey"}
            So(StringSliceEqual(a, b), ShouldBeTrue)
        })

        Convey("true when a ＝= nil  && b ＝= nil", func() {
            So(StringSliceEqual(nil, nil), ShouldBeTrue)
        })
    })
}
```

内层的 Convey 不需要再传入 t *testing.T 参数

**[GoConvey 的更多用法](https://link.zhihu.com/?target=https%3A//www.jianshu.com/p/e3b2b1194830)**

### **3. testify**

testify 提供了 assert 和 require，让你可以简洁地写出`if xxx { t.Fail() }`

### **3.1. assert**

```
func TestSomething(t *testing.T) {

  //断言相等
  assert.Equal(t, 123, 123, "they should be equal")

  //断言不相等
  assert.NotEqual(t, 123, 456, "they should not be equal")

  //对于nil的断言
  assert.Nil(t, object)

  //对于非nil的断言
  if assert.NotNil(t, object) {
 // now we know that object isn't nil, we are safe to make
 // further assertions without causing any errors
 assert.Equal(t, "Something", object.Value)
  }
```

### **3.2. require**

require 和 assert 失败、成功条件完全一致，区别在于 assert 只是返回布尔值（true、false），而 require 不符合断言时，会中断当前运行

### **3.3. 常用的函数**

```
func Equal(t TestingT, expected, actual interface{}, msgAndArgs ...interface{}) bool
func NotEqual(t TestingT, expected, actual interface{}, msgAndArgs ...interface{}) bool

func Nil(t TestingT, object interface{}, msgAndArgs ...interface{}) bool
func NotNil(t TestingT, object interface{}, msgAndArgs ...interface{}) bool

func Empty(t TestingT, object interface{}, msgAndArgs ...interface{}) bool
func NotEmpty(t TestingT, object interface{}, msgAndArgs ...interface{}) bool

func NoError(t TestingT, err error, msgAndArgs ...interface{}) bool
func Error(t TestingT, err error, msgAndArgs ...interface{}) bool

func Zero(t TestingT, i interface{}, msgAndArgs ...interface{}) bool
func NotZero(t TestingT, i interface{}, msgAndArgs ...interface{}) bool

func True(t TestingT, value bool, msgAndArgs ...interface{}) bool
func False(t TestingT, value bool, msgAndArgs ...interface{}) bool

func Len(t TestingT, object interface{}, length int, msgAndArgs ...interface{}) bool

func NotContains(t TestingT, s, contains interface{}, msgAndArgs ...interface{}) bool
func NotContains(t TestingT, s, contains interface{}, msgAndArgs ...interface{}) bool
func Subset(t TestingT, list, subset interface{}, msgAndArgs ...interface{}) (ok bool)
func NotSubset(t TestingT, list, subset interface{}, msgAndArgs ...interface{}) (ok bool)

func FileExists(t TestingT, path string, msgAndArgs ...interface{}) bool
func DirExists(t TestingT, path string, msgAndArgs ...interface{}) bool
```

**[testify 的更多用法](https://link.zhihu.com/?target=https%3A//s0godoc0org.icopy.site/github.com/stretchr/testify)**

### **三、Stub/Mock 框架**

Golang 有以下 Stub/Mock 框架：

*   GoStub
*   GoMock
*   Monkey

一般来说，GoConvey 可以和 GoStub、GoMock、Monkey 中的一个或多个搭配使用。

Testify 本身有自己的 Mock 框架，可以用自己的也可以和这里列出来的 Stub/Mock 框架搭配使用。

### **1. GoStub**

GoStub 框架的使用场景很多，依次为：

*   基本场景：为一个全局变量打桩
*   基本场景：为一个函数打桩
*   基本场景：为一个过程打桩
*   复合场景：由任意相同或不同的基本场景组合而成

### **1.1. 为一个全局变量打桩**

假设 num 为被测函数中使用的一个全局整型变量，当前测试用例中假定 num 的值大于 100，比如为 150，则打桩的代码如下：

```
stubs := Stub(&num, 150)
defer stubs.Reset()
```

stubs 是 GoStub 框架的函数接口 Stub 返回的对象，该对象有 Reset 操作，即将全局变量的值恢复为原值。

### **1.2. 为一个函数打桩**

假设我们产品的既有代码中有下面的函数定义：

```
func Exec(cmd string, args ...string) (string, error) {
    ...
}
```

我们可以对 Exec 函数打桩，代码如下所示：

```
stubs := StubFunc(&Exec,"xxx-vethName100-yyy", nil)
defer stubs.Reset()
```

### **1.3. 为一个过程打桩**

当一个函数没有返回值时，该函数我们一般称为过程。很多时候，我们将资源清理类函数定义为过程。

我们对过程 DestroyResource 的打桩代码为：

```
stubs := StubFunc(&DestroyResource)
defer stubs.Reset()
```

**[GoStub 的更多用法以及 GoStub+GoConvey 的组合使用方法](https://link.zhihu.com/?target=https%3A//www.jianshu.com/p/70a93a9ed186)**

### **2. GoMock**

GoMock 是由 Golang 官方开发维护的测试框架，实现了较为完整的基于 interface 的 Mock 功能，能够与 Golang 内置的 testing 包良好集成，也能用于其它的测试环境中。GoMock 测试框架包含了 GoMock 包和 mockgen 工具两部分，其中 GoMock 包完成对桩对象生命周期的管理，mockgen 工具用来生成 interface 对应的 Mock 类源文件。

### **2.1. 定义一个接口**

我们先定义一个打算 mock 的接口 Repository。

Repository 是领域驱动设计中战术设计的一个元素，用来存储领域对象，一般将对象持久化在数据库中，比如 Aerospike，Redis 或 Etcd 等。对于领域层来说，只知道对象在 Repository 中维护，并不 care 对象到底在哪持久化，这是基础设施层的职责。微服务在启动时，根据部署参数实例化 Repository 接口，比如 AerospikeRepository，RedisRepository 或 EtcdRepository。

```
package db

type Repository interface {
    Create(key string, value []byte) error
    Retrieve(key string) ([]byte, error)
    Update(key string, value []byte) error
    Delete(key string) error
}
```

### **2.2. 生成 mock 类文件**

这下该 mockgen 工具登场了。mockgen 有两种操作模式：源文件和反射。

源文件模式通过一个包含 interface 定义的文件生成 mock 类文件，它通过 -source 标识生效，-imports 和 -aux_files 标识在这种模式下也是有用的。 举例：

```
mockgen -source=foo.go [other options]
```

反射模式通过构建一个程序用反射理解接口生成一个 mock 类文件，它通过两个非标志参数生效：导入路径和用逗号分隔的符号列表（多个 interface）。 举例：

```
mockgen database/sql/driver Conn,Driver
```

生成的 mock_repository.go 文件：

```
// Automatically generated by MockGen. DO NOT EDIT!
// Source: infra/db (interfaces: Repository)

package mock_db

import (
    gomock "github.com/golang/mock/gomock"
)

// MockRepository is a mock of Repository interface
type MockRepository struct {
    ctrl     *gomock.Controller
    recorder *MockRepositoryMockRecorder
}

// MockRepositoryMockRecorder is the mock recorder for MockRepository
type MockRepositoryMockRecorder struct {
    mock *MockRepository
}

// NewMockRepository creates a new mock instance
func NewMockRepository(ctrl *gomock.Controller) *MockRepository {
    mock := &MockRepository{ctrl: ctrl}
    mock.recorder = &MockRepositoryMockRecorder{mock}
    return mock
}

// EXPECT returns an object that allows the caller to indicate expected use
func (_m *MockRepository) EXPECT() *MockRepositoryMockRecorder {
    return _m.recorder
}

// Create mocks base method
func (_m *MockRepository) Create(_param0 string, _param1 []byte) error {
    ret := _m.ctrl.Call(_m, "Create", _param0, _param1)
    ret0, _ := ret[0].(error)
    return ret0
}
...
```

### **2.3. 使用 mock 对象进行打桩测试**

### **2.3.1. 导入 mock 相关的包**

```
import (
    "testing"
    . "github.com/golang/mock/gomock"
    "test/mock/db"
    ...
)
```

### **2.3.2. mock 控制器**

mock 控制器通过 NewController 接口生成，是 mock 生态系统的顶层控制，它定义了 mock 对象的作用域和生命周期，以及它们的期望。多个协程同时调用控制器的方法是安全的。 当用例结束后，控制器会检查所有剩余期望的调用是否满足条件。

控制器的代码如下所示：

```
ctrl := NewController(t)
defer ctrl.Finish()
```

mock 对象创建时需要注入控制器，如果有多个 mock 对象则注入同一个控制器，如下所示：

```
ctrl := NewController(t)
defer ctrl.Finish()
mockRepo := mock_db.NewMockRepository(ctrl)
mockHttp := mock_api.NewHttpMethod(ctrl)
```

### **2.3.3. mock 对象的行为注入**

对于 mock 对象的行为注入，控制器是通过 map 来维护的，一个方法对应 map 的一项。因为一个方法在一个用例中可能调用多次，所以 map 的值类型是数组切片。当 mock 对象进行行为注入时，控制器会将行为 Add。当该方法被调用时，控制器会将该行为 Remove。

假设有这样一个场景：先 Retrieve 领域对象失败，然后 Create 领域对象成功，再次 Retrieve 领域对象就能成功。这个场景对应的 mock 对象的行为注入代码如下所示：

```
mockRepo.EXPECT().Retrieve(Any()).Return(nil, ErrAny)
mockRepo.EXPECT().Create(Any(), Any()).Return(nil)
mockRepo.EXPECT().Retrieve(Any()).Return(objBytes, nil)
```

objBytes 是领域对象的序列化结果，比如：

```
obj := Movie{...}
objBytes, err := json.Marshal(obj)
...
```

当批量 Create 对象时，可以使用 Times 关键字：

```
mockRepo.EXPECT().Create(Any(), Any()).Return(nil).Times(5)
```

当批量 Retrieve 对象时，需要注入多次 mock 行为:

```
mockRepo.EXPECT().Retrieve(Any()).Return(objBytes1, nil)
mockRepo.EXPECT().Retrieve(Any()).Return(objBytes2, nil)
mockRepo.EXPECT().Retrieve(Any()).Return(objBytes3, nil)
mockRepo.EXPECT().Retrieve(Any()).Return(objBytes4, nil)
mockRepo.EXPECT().Retrieve(Any()).Return(objBytes5, nil)
```

**[GoMock 的更多用法以及 GoStub+GoConvey+GoMock 的组合使用方法](https://link.zhihu.com/?target=https%3A//www.jianshu.com/p/f4e773a1b11f)**

### **3. Monkey**

至此，我们已经知道：

*   全局变量可通过 GoStub 框架打桩
*   过程可通过 GoStub 框架打桩
*   函数可通过 GoStub 框架打桩
*   interface 可通过 GoMock 框架打桩

但还有两个问题比较棘手：

1.  方法（成员函数）无法通过 GoStub 框架打桩，当产品代码的 OO 设计比较多时，打桩点可能离被测函数比较远，导致 UT 用例写起来比较痛
2.  过程或函数通过 GoStub 框架打桩时，对产品代码有侵入性

Monkey 是 Golang 的一个猴子补丁（monkeypatching）框架，在运行时通过汇编语句重写可执行文件，将待打桩函数或方法的实现跳转到桩实现，原理和热补丁类似。通过 Monkey，我们可以解决函数或方法的打桩问题，但 Monkey 不是线程安全的，不要将 Monkey 用于并发的测试中。

Monkey 框架的使用场景很多，依次为：

*   基本场景：为一个函数打桩
*   基本场景：为一个过程打桩
*   基本场景：为一个方法打桩
*   复合场景：由任意相同或不同的基本场景组合而成
*   特殊场景：桩中桩的一个案例

> 另有 GoMonkey 框架 [https://github.com/agiledragon/gomonkey](https://link.zhihu.com/?target=https%3A//github.com/agiledragon/gomonkey)，对比 Monkey 来说，写法更简单，有兴趣的读者可以尝试使用。  

### **3.1. 为一个函数打桩**

Exec 是 infra 层的一个操作函数，实现很简单，代码如下所示：

```
func Exec(cmd string, args ...string) (string, error) {
    cmdpath, err := exec.LookPath(cmd)
    if err != nil {
        fmt.Errorf("exec.LookPath err: %v, cmd: %s", err, cmd)
        return "", infra.ErrExecLookPathFailed
    }

    var output []byte
    output, err = exec.Command(cmdpath, args...).CombinedOutput()
    if err != nil {
        fmt.Errorf("exec.Command.CombinedOutput err: %v, cmd: %s", err, cmd)
        return "", infra.ErrExecCombinedOutputFailed
    }
    fmt.Println("CMD[", cmdpath, "]ARGS[", args, "]OUT[", string(output), "]")
    return string(output), nil
}
```

Monkey 的 API 非常简单和直接，我们直接看打桩代码：

```
import (
    "testing"
    . "github.com/smartystreets/goconvey/convey"
    . "github.com/bouk/monkey"
    "infra/osencap"
)

const any = "any"

func TestExec(t *testing.T) {
    Convey("test has digit", t, func() {
        Convey("for succ", func() {
            outputExpect := "xxx-vethName100-yyy"
            guard := Patch(
             osencap.Exec,
             func(_ string, _ ...string) (string, error) {
                 return outputExpect, nil
             })
            defer guard.Unpatch()
            output, err := osencap.Exec(any, any)
            So(output, ShouldEqual, outputExpect)
            So(err, ShouldBeNil)
        })
    })
}
```

Patch 是 Monkey 提供给用户用于函数打桩的 API：

1.  第一个参数是目标函数的函数名
2.  第二个参数是桩函数的函数名，习惯用法是匿名函数或闭包
3.  返回值是一个 PatchGuard 对象指针，主要用于在测试结束时删除当前的补丁

### **3.2. 为一个过程打桩**

当一个函数没有返回值时，该函数我们一般称为过程。很多时候，我们将资源清理类函数定义为过程。 我们对过程 DestroyResource 的打桩代码为：

```
guard := Patch(DestroyResource, func(_ string) {

})
defer guard.Unpatch()
```

### **3.3. 为一个方法打桩**

当微服务有多个实例时，先通过 Etcd 选举一个 Master 实例，然后 Master 实例为所有实例较均匀的分配任务，并将任务分配结果 Set 到 Etcd，最后 Master 和 Node 实例 Watch 到任务列表，并过滤出自身需要处理的任务列表。

我们用类 Etcd 的方法 Get 来模拟获取任务列表的功能，入参为 instanceId：

```
type Etcd struct {

}

func (e *Etcd) Get(instanceId string) []string {
    taskList := make([]string, 0)
    ...
    return taskList
```

我们对 Get 方法的打桩代码如下：

```
var e *Etcd
guard := PatchInstanceMethod(
 reflect.TypeOf(e),
 "Get",
 func(_ *Etcd, _ string) []string {
  return []string{"task1", "task5", "task8"}
 })
defer guard.Unpatch()
```

PatchInstanceMethod API 是 Monkey 提供给用户用于方法打桩的 API：

*   在使用前，先要定义一个目标类的指针变量 x
*   第一个参数是 reflect.TypeOf(x)
*   第二个参数是字符串形式的函数名
*   返回值是一个 PatchGuard 对象指针，主要用于在测试结束时删除当前的补丁

**[Monkey 的更多用法以及 Monkey 和前几种框架的组合使用方法](https://link.zhihu.com/?target=https%3A//www.jianshu.com/p/2f675d5e334e)**

### **四、Mock 场景最佳实践**

### **1. 实例函数 Mock：Monkey**

Monkey 用于对依赖的函数进行 Mock 替换，从而可以完成仅针对当前模块的单元测试。

例子：

`test`包是真实的函数 `mock_test`包是即将用于 mock 的函数

`test.go`:

```
package test

import "fmt"

func PrintAdd(a, b uint32) string {
 return fmt.Sprintf("a:%v+b:%v", a, b)
}


type SumTest struct {
}

func (*SumTest)PrintSum(a, b uint32) string {
 return fmt.Sprintf("a:%v+b:%v", a, b)
}
```

`mock_test.go`:

```
package mock_test

import "fmt"
import "test/24_mock/test"

func PrintAdd(a, b uint32) string {
 return fmt.Sprintf("a:%v+b:%v=%v", a, b, a+b)
}

//对应test文件夹下的PrintSum
func PrintSum(_ *test.SumTest, a, b uint32) string {
 return fmt.Sprintf("a:%v+b:%v=%v", a, b,a+b)
}
```

`main.go`:

```
func test1() {
     monkey.Patch(test.PrintAdd, mock_test.PrintAdd)
     p := test.PrintAdd(1, 2)
     fmt.Println(p)
     monkey.UnpatchAll() //解除所有替换
     p = test.PrintAdd(1, 2)
     fmt.Println(p)
}

func test2() {
     structSum := &test.SumTest{}
     //para1:获取实例的反射类型,para2:被替换的方法名,para3:替换方法
     monkey.PatchInstanceMethod(reflect.TypeOf(structSum), "PrintSum", mock_test.PrintSum)
     p := structSum.PrintSum(1, 2)
     fmt.Println(p)
     monkey.UnpatchAll() //解除所有替换
     p = structSum.PrintSum(1, 2)
     fmt.Println(p)
}
```

### **2. 未实现函数 Mock：GoMock**

假设场景：`Company`公司、`Person`人。

1.  公司可以开会。
2.  公司内部的人继承了`Talker`讨论者接口，拥有`SayHello`说话的方法。

假如现在要测试这个场景，在所有类都实现的情况下，测试应该是这样的：

```
//正常测试
func TestCompany_Meeting(t *testing.T) {
//直接调用Person类的New方法，创建一个Person对象
 talker := NewPerson("小微", "语音服务助手")
 company := NewCompany(talker)
 t.Log(company.Meeting("lyt", "intern"))
}
```

但现在`Person`类并未实现，则可以通过 GoMock 工具来模拟一个`Person`对象。

定义一个`Talker.go`

```
package pojo

type Talker interface {
   SayHello(word, role string) (response string)
}
```

根据该接口，用`mockgen`命令生成一个 Mock 对象

```
mockgen [-source] [-destination] [-package] ... Talker.go
```

接着进行测试用例的编写：

1.  `NewController()`: 新建 Mock 控制器
2.  `NewMockXXX()`: 新建 Mock 对象，这里是调用 NewMockTalker()
3.  `talker.EXPECT().XXX().XXX()..`：撰写一些断言测试
4.  之前 Mock 建立的对象传入到待测方法当中
5.  测试结果通过 testing 框架返回

```
//通过Mock测试
func TestCompany_Meeting2(t *testing.T) {

   //新建Mock控制器
   ctrl := gomock.NewController(t)
   //新建Mock对象-Talker
   talker := mock_pojo.NewMockTalker(ctrl)

   //断言
   talker.EXPECT().SayHello(gomock.Eq("震天嚎"), gomock.Eq("学生")).Return("Hello Faker(身份：学生), welcome to GoLand IDE. My name is 震天嚎")
   //mock对象传入方法
   company := NewCompany(talker)

   //Pass的例子
   t.Log(company.Meeting("震天嚎", "学生"))

   //报错的例子
   //t.Log(company.Meeting("小白", "学生"))

}
```

### **3. 系统内置函数 Mock：Monkey**

`monkey.Patch(json.Unmarshal, mockUnmarshal)`，用 Monkey 的 patch 来 mock 系统内置函数

```
func mockUnmarshal(b []byte, v interface{}) error{
   v = &Common.LoginMessage{
      UserId: 1,
      UserName: "admin",
      UserPwd: "admin",
   }
   return nil
}
```

如果需要取消替换，可以使用

```
monkey.UnPatch(target interface{}) //解除单个Patch
monkey.UnPatchAll()        //解除所有Patch
```

### **4. 数据库行为 Mock**

```
func TestSql(t *testing.T) {
   db, mock, err := sqlmock.New(sqlmock.QueryMatcherOption(sqlmock.QueryMatcherEqual))
   if err != nil {
      fmt.Println("fail to open sqlmock db: ", err)
   }
   defer db.Close()
   rows := sqlmock.NewRows([]string{"id", "pwd"}).
      AddRow(1, "apple").
      AddRow(2, "banana")
   mock.ExpectQuery("SELECT id, pwd FROM users").WillReturnRows(rows)
   res, err := db.Query("SELECT id, pwd FROM users")
   if err != nil {
      fmt.Println("fail to match expected sql.")
      return
   }
   defer res.Close()
   for res.Next() {
      var id int
      var pwd string
      res.Scan(&id, &pwd)
      fmt.Printf("Sql Result:\tid = %d, password = %s.\n",id, pwd)
   }
   if res.Err() != nil {
      fmt.Println("Result Return Error!", res.Err())
   }
}
```

### **5. 服务器行为 Mock：Monkey**

使用 net/http/httptest 模拟服务器行为

```
func TestHttp(t *testing.T) {
   handler := func(w http.ResponseWriter, r *http.Request) {
      io.WriteString(w, "{ \"status\": \"expected service response\"}")
   }

   req := httptest.NewRequest("GET", "https://test.net", nil)
   w := httptest.NewRecorder()
   handler(w, req)//处理该Request

   resp := w.Result()
   body, _ := ioutil.ReadAll(resp.Body)
   fmt.Println(resp.StatusCode)
   fmt.Println(resp.Header.Get("Content-Type"))
   fmt.Println(string(body))
}
```

这里只使用了 Monkey 的 Patch 进行简单测试，但在更一般的情况下，更多的函数还是通过实例函数来编写的，对这部分函数要用`PatchInstanceMethod`才可以进行替换。

`func PatchInstanceMethod(target reflect.Type, methodName string, replacement interface{})`接收三个参数：

*   `reflect.Tpye`通过新建一个待测实例对象，调用 reflect 包的`TypeOf()`方法就可以得到
*   `methodName`是待测实例对象的函数名
*   `replacement`是用于替换的函数

实现如下：

```
var ts *utils.Transfer
monkey.PatchInstanceMethod(reflect.TypeOf(ts), "WritePkg", func(_ *utils.Transfer, _ []byte) error {
      return nil
})
```

假设有如下一个函数`ServerProcessLogin`，用于接收用户名密码，向当前连接的服务器请求登陆，测试如下：

```
func TestServerProcessLogin(t *testing.T) {
   mess := &Common.Message{
      Type: Common.LoginMesType,
      Data: "default",
   }
   user := &UserProcess{
      Conn: nil,
   }

   //对涉及到的单元以外系统函数打Patch
   monkey.Patch(json.Unmarshal, mockUnmarshal)
   monkey.Patch(json.Marshal, mockMarshal)

   //单元测试不涉及实际服务器，故对实例函数Login，WritePkg打Patch
   var udao *model.UserDao
   monkey.PatchInstanceMethod(reflect.TypeOf(udao), "Login", func(_ *model.UserDao, _ int, _ string) (*Common.User,error) {
      return &Common.User{
         UserId: 1,
         UserName: "admin",
         UserPwd: "admin",
      }, nil
   })

   var ts *utils.Transfer
   monkey.PatchInstanceMethod(reflect.TypeOf(ts),"WritePkg", func(_ *utils.Transfer, _ []byte) error {
      return nil
   })

   //执行测试
   convey.Convey("Test Server Login.", t, func() {
      err := user.ServerProcessLogin(mess)
      convey.So(err, convey.ShouldBeNil)
   })

   monkey.UnpatchAll()
   return
}

//用于替换的函数
func mockUnmarshal(b []byte, v interface{}) error{
   v = &Common.LoginMessage{
      UserId: 1,
      UserName: "admin",
      UserPwd: "admin",
   }
   return nil
}

func mockMarshal(v interface{}) ([]byte, error) {
   var rer = []byte{
      'a','d','m','i','n',
   }
   return rer, nil
}
```

### **五、具体案例：聊天室**

### **1. 概览**

该项目是一个具有登录、查看在线用户、私聊、群聊等功能的命令行聊天室 Demo。

项目分为 Client、Server 子项目，都通过 model、Controllor(Processor）、View（Main）来进行功能划分。还有一个 Common 包放置通用性的工具类。

```
├─Client
│  ├─main
│  ├─model
│  ├─processor
│  └─utils
├─Common
└─Server
    ├─main
    ├─model
    ├─processor
    └─utils
```

预期目的：对实现的功能模块补充单元测试代码，度量确保每一个模块的功能的正确性、完整性、健壮性，并在未来修改代码后也能第一时间自测验收。

单元测试应包括模块接口测试、模块局部数据结构测试、模块异常处理测试。

对于接口测试，应对接口的传入参数测试样例设计进行全面的考察，判断每一个参数是否有是有必要的，参数间有没有冗余，进入函数体前引用的指针是否有错等等。

对于局部数据结构测试，应检查局部数据结构是为了保证临时存储在模块内的数据在程序执行过程中完整性、正确性。局部数据结构往往是错误的根源，应仔细设计测试用例。

对于异常处理，主要有如下几种常见错误：

1.  输出的出错信息提示不足
2.  对异常没有进行处理
3.  出错信息与实际不相符
4.  出错信息中未能准确定位出错信息

以上几种错误，都是模块中经常会出现的错误，要针对这些错误来进行边界条件测试检查，只有异常处理机制正确，日后软件的维护和迭代才会更加高效。

在本案例中，Model 层对服务层提供的接口不多，就`WritePkg`，`ReadPkg`两个核心函数，在服务层对其进行封装抽象为具体的业务逻辑。由于涉及网络连接，所以对其进行的测试必须编写桩函数。在服务层，涉及到对多个网络连接调用、数据库调用其它模块依赖，所以也要为其进行 Mock。

由于涉及 Mock 和桩函数编写，可以使用`GoStub`、`Monkey`两个包进行这些工作，它们较简洁地实现了很多实用的测试方式，只需要用户编写依赖的接口文件、用于替换的 Mock 函数，就可以仅在测试过程中替换掉系统函数或者其它依赖的功能模块，使得单元测试起到它应有的作用。

### **2. Model 层、数据库相关测试**

由于是单元测试，所以需要获取一个 Mock 数据库实例，测试增删改查 SQL 语句是否可执行。`userDao_test.go`代码如下：

```
const (
   sql1 = "SELECT id, pwd FROM users"
   sql2 = "DELETE FROM users where id > 600 and id < 700"
   sql3 = "update users set pwd = newPwd where id = 1 and id = 2"
   sql4 = "INSERT INTO users (id, pwd) VALUES (405, 'Lyt')"
)

func TestGetUserById(t *testing.T) {
   db, mock, err := sqlmock.New(sqlmock.QueryMatcherOption(sqlmock.QueryMatcherEqual))
   if err != nil {
      fmt.Println("fail to open sqlmock db: ", err)
   }
   defer db.Close()
   rows1 := sqlmock.NewRows([]string{"id", "pwd"}).
      AddRow(1, "apple").
      AddRow(2, "banana")
   rows2 := sqlmock.NewRows([]string{"id", "pwd"}).
      AddRow(601, "goland").
      AddRow(602, "java")
   rows3 := sqlmock.NewRows([]string{"id", "pwd"}).
      AddRow(1, "newPwd").
      AddRow(2, "newPwd")
   rows4 := sqlmock.NewRows([]string{"id", "pwd"}).
      AddRow(405, "Lyt")

   mock.ExpectQuery(sql1).WillReturnRows(rows1)
   mock.ExpectQuery(sql2).WillReturnRows(rows2)
   mock.ExpectQuery(sql3).WillReturnRows(rows3)
   mock.ExpectQuery(sql4).WillReturnRows(rows4)

   assert.New(t)
   var tests = []struct{
      inputSql string
      expected interface{}
   } {
      {sql1,nil},
      {sql2,nil},
      {sql3,nil},
      {sql4, nil},
   }

   for _, test := range tests {
      res, err := db.Query(test.inputSql)
      assert.Equal(t, err, test.expected)

      for res.Next() {
         var id int
         var pwd string
         res.Scan(&id, &pwd)
         fmt.Printf("Sql Result:\tid = %d, password = %s.\n",id, pwd)
      }
      assert.Equal(t, res.Err(), test.expected)
   }
}
```

### **3. 私聊功能测试**

由于涉及底层数据库交互时需要发送 JSON 转码字符串（`WritePkg`函数），因此将其 Mock 处理，只需关注本函数逻辑是否正确即可。`smsProcess_test.go`如下：

```
func TestSmsProcess_SendOnePerson(t *testing.T) {
   var conn net.Conn
   tf := &utils.Transfer{
      Conn: conn,
   }
   monkey.PatchInstanceMethod(reflect.TypeOf(tf), "WritePkg", func(_ *utils.Transfer,_ []byte) error{
      return nil
   })
   convey.Convey("test send one person:", t, func() {
      err := tf.WritePkg([]byte{})
      convey.So(err, convey.ShouldBeNil)
      fmt.Println("OK.")
   })
}
```

### **4. 登录功能测试**

登录涉及服务器连接操作，服务器的连接逻辑可通过`httptest`包来进行检测，Mock 一个 HTTP 连接，示例代码如下：

```
func TestHttp(t *testing.T) {
   handler := func(w http.ResponseWriter, r *http.Request) {
      // here we write our expected response, in this case, we return a
      // JSON string which is typical when dealing with REST APIs
      io.WriteString(w, "{ \"status\": \"expected service response\"}")
   }

   req := httptest.NewRequest("GET", "https://test.net", nil)
   w := httptest.NewRecorder()
   handler(w, req)

   resp := w.Result()
   body, _ := ioutil.ReadAll(resp.Body)
   fmt.Println(resp.StatusCode)
   fmt.Println(resp.Header.Get("Content-Type"))
   fmt.Println(string(body))
}
```

为登录模块编写用于测试替换的函数以及单元测试主体，`userProcess_test.go`代码如下：

```
func mockUnmarshal(b []byte, v interface{}) error {
   v = &Common.LoginMessage{
      UserId:   1,
      UserName: "admin",
      UserPwd:  "admin",
   }
   return nil
}

func mockMarshal(v interface{}) ([]byte, error) {
   var rer = []byte{
      'a', 'd', 'm', 'i', 'n',
   }
   return rer, nil
}

func TestServerProcessLogin(t *testing.T) {
   mess := &Common.Message{
      Type: Common.LoginMesType,
      Data: "default",
   }
   user := &UserProcess{
      Conn: nil,
   }

   //对涉及到的单元以外系统函数打Patch
   monkey.Patch(json.Unmarshal, mockUnmarshal)
   monkey.Patch(json.Marshal, mockMarshal)

   //对实例函数打Patch
   var udao *model.UserDao
   monkey.PatchInstanceMethod(reflect.TypeOf(udao), "Login", func(_ *model.UserDao, _ int, _ string) (*Common.User, error) {
      return &Common.User{
         UserId:   1,
         UserName: "admin",
         UserPwd:  "admin",
      }, nil
   })

   var ts *utils.Transfer
   monkey.PatchInstanceMethod(reflect.TypeOf(ts), "WritePkg", func(_ *utils.Transfer, _ []byte) error {
      return nil
   })
   //执行测试
   convey.Convey("Test Server Login.", t, func() {
      err := user.ServerProcessLogin(mess)
      convey.So(err, convey.ShouldBeNil)
   })
   monkey.UnpatchAll()
   return
}
```

### **5. 工具类测试**

```
utils_test.go
func mockRead(conn net.Conn, _ []byte) (int, error) {
   return 4, nil
}

func mockMarshal(v interface{}) ([]byte, error) {
   return []byte{'a', 'b', 'c', 'd'}, nil
}

func mockUnmarshal(data []byte, v interface{}) error {
   return nil
}

func TestTransfer_ReadPkg(t *testing.T) {

   monkey.Patch(net.Conn.Read, mockRead)
   monkey.Patch(json.Marshal, mockMarshal)
   monkey.Patch(json.Unmarshal, mockUnmarshal)

   listen, _ := net.Listen("tcp", "localhost:8888")
   defer listen.Close()
   go net.Dial("tcp", "localhost:8888")
   var c net.Conn
   for {

      c, _ = listen.Accept()
      if c != nil {
         break
      }
   }


   transfer := &Transfer{
      Conn: c,
      Buf : [8096]byte{'a', 'b', 'c', 'd'},
   }
   convey.Convey("test ReadPkg", t, func() {
      mes, err := transfer.ReadPkg()
      convey.So(err, convey.ShouldBeNil)
      convey.So(mes, convey.ShouldEqual, "ab")
   })
   monkey.UnpatchAll()

}

func TestTransfer_WritePkg(t *testing.T) {

   monkey.Patch(json.Marshal, mockMarshal)
   monkey.Patch(json.Unmarshal, mockUnmarshal)
   transfer := &Transfer{
      Conn: nil,
      Buf : [8096]byte{},
   }
   convey.Convey("test ReadPkg", t, func() {
      err := transfer.WritePkg([]byte{'a', 'b'})
      convey.So(err, convey.ShouldBeNil)
   })
   monkey.UnpatchAll()
}
```

### **6. 项目总结**

在编写桩模块时会发现，模块之间的调用关系在工程规模并不大的本案例中，也依然比较复杂，需要开发相应桩函数，代码量会增加许多，也会消耗一些开发人员的时间，因此反推到之前流程的开发实践中，可以得出结论就是提高模块的内聚度可简化单元测试，如果每个模块只能完成一个，所需测试用例数目将显著减少，模块中的错误也更容易发现。

Go 单元测试框架是相当易用的，其它的第三方库基本都是建立在 testing 原生框架的基础上进行的增补扩充，在日常开发中，原生包可以满足基本需求，但同样也有缺陷，原生包不提供断言的语法使得代码中的这种片段非常多：

```
if err != nil{
 //...
}
```

所以引入了 convey、assert 包的断言语句，用于简化判断逻辑，使得程序更加易读。

在完成项目单测时，遇到了不少问题，比较重要的比如由于架构分层不够清晰，还是有部分耦合代码，导致单测时需要屏蔽的模块太多，代码写起来不便。因此还是需要倒推到开发模块之前，就要设计更好的结构，在开发的过程中遵循相应的规则，通过测试先行的思想，使开发的工程具有更好的可测试性。

开发过程中遇到的场景肯定不局限于本文所讨论的范围，有关更丰富的最佳实践案例可以参照：

*   **[go-sqlmock](https://link.zhihu.com/?target=https%3A//github.com/DATA-DOG/go-sqlmock)**
*   **[go-mock](https://link.zhihu.com/?target=https%3A//blog.csdn.net/RA681t58CJxsgCkJ31/article/details/86684592%3Futm_medium%3Ddistribute.pc_relevant.none-task-blog-baidujs-1)**

### **六、结语**

### **1. 实践小结**

单元测试大多是由开发人员进行编写，本篇文章旨在指引，不在于面面俱到，具体的单元测试框架的使用语法，开发同学可以自行 Google。

以测试的角度，推行单元测试是不易的，最佳的方式莫过于开发人员，在一定的指引之后，以实际项目出发进行实践，然后自行总结具体的 case，有针对性、有感染力进行内部分享，测试同学及时提供测试用例的指引和规范的约束。

### **2. 特别鸣谢**

*   两位实习生罗宇韬和钟梓轩，在暑假实习期间，协助整理了 Golang 单测的代码示例。

### **3. 推荐阅读**

*   书籍《google 的软件测试之道》
*   书籍《单元测试的艺术》

**更多干货尽在[腾讯技术](https://www.zhihu.com/org/teng-xun-ji-zhu-gong-cheng)，官方微信交流群已建立，交流讨论可加：Journeylife1900（备注腾讯技术） 。**