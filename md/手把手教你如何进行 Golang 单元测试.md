> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [zhuanlan.zhihu.com](https://zhuanlan.zhihu.com/p/377834750)

作者：stevennzhou，腾讯 PCG 前端开发工程师

> 本篇是对单元测试的一个总结，通过完整的单元测试手把手教学，能够让刚接触单元测试的开发者从整体上了解一个单元测试编写的全过程。最终通过两个问题，也能让写过单元测试的开发者收获单测执行时的一些底层细节知识。  

### **引入**

随着工程化开发在司内大力的推广，单元测试越来越受到广大开发者的重视。在学习的过程中，发现网上针对 Golang 单元测试大多从理论角度出发介绍，缺乏完整的实例说明，晦涩难懂的 API 让初学接触者难以下手。

本篇不准备大而全的谈论单元测试、笼统的介绍 Golang 的单测工具，而将从 Golang 单测的使用场景出发，以最简单且实际的例子讲解如何进行单测，最终由浅入深探讨 go 单元测试的两个比较细节的问题。

在阅读本文时，请务必对 Golang 的单元测试有最基本的了解。

### **一段需要单测的 Golang 代码**

```
package unit

import (
 "encoding/json"
 "errors"
 "github.com/gomodule/redigo/redis"
 "regexp"
)

type PersonDetail struct {
 Username string `json:"username"`
 Email    string `json:"email"`
}

// 检查用户名是否非法
func checkUsername(username string) bool {
 const pattern = `^[a-z0-9_-]{3,16}$`

 reg := regexp.MustCompile(pattern)
 return reg.MatchString(username)
}

// 检查用户邮箱是否非法
func checkEmail(email string) bool {
 const pattern = `^[a-zA-Z0-9_-]+@[a-zA-Z0-9_-]+(\.[a-zA-Z0-9_-]+)+$`

 reg := regexp.MustCompile(pattern)
 return reg.MatchString(email)
}

// 通过 redis 拉取对应用户的资料信息
func getPersonDetailRedis(username string) (*PersonDetail, error) {
 result := &PersonDetail{}

 client, err := redis.Dial("tcp", ":6379")
 defer client.Close()
 data, err := redis.Bytes(client.Do("GET", username))

 if err != nil {
  return nil, err
 }

 err = json.Unmarshal(data, result)
 if err != nil {
  return nil, err
 }

 return result, nil
}

// 拉取用户资料信息并校验
func GetPersonDetail(username string) (*PersonDetail, error) {
 // 检查用户名是否有效
 if ok := checkUsername(username); !ok {
  return nil, errors.New("invalid username")
 }

 // 从 redis 接口获取信息
 detail, err := getPersonDetailRedis(username)
 if err != nil {
  return nil, err
 }

 // 校验
 if ok := checkEmail(detail.Email); !ok {
  return nil, errors.New("invalid email")
 }

 return detail, nil
}
```

这是一段典型的有 I/O 的功能代码，主体功能是传入用户名，校验合法性之后通过 redis 获取信息，之后校验获取值内容的合法性后并返回。

### **后台服务单测场景**

对于一个传统的后端服务，它主要有以下几点的职责和功能：

*   接收外部请求，controller 层分发请求、校验请求参数  
    
*   请求有效分发后，在 service 层与 dao 层进行交互后做逻辑处理  
    
*   dao 层负责数据操作，主要是数据库或持久化存储相关的操作  
    

因此，从职责出发来看，在做后台单测中，核心主要是验证 service 层和 dao 层的相关逻辑，此外 controller 层的参数校验也在单测之中。

细分来看，对于相关逻辑的单元测试，笔者倾向于把单测分为两种：

*   无第三方依赖，纯逻辑代码  
    
*   有第三方依赖，如文件、网络 I/O、第三方依赖库、数据库操作相关的代码  
    

**注：单元测试中只是针对单个函数的测试，关注其内部的逻辑，对于网络 / 数据库访问等，需要通过相应的手段进行 mock。**

### **Golang 单测工具选型**

由于我们把单测简单的分为了两种：

*   对于无第三方依赖的纯逻辑代码，我们只需要验证**相关逻辑**即可，这里只需要使用 `assert` **（断言）**，通过控制输入输出比对结果即可。  
    
*   对于有第三方依赖的代码，在验证相关代码逻辑之前，我们需要将相关的依赖 `mock` **（模拟）**，之后才能通过断言验证逻辑。这里需要借助第三方工具库来处理。  
    

因此，对于 `assert` **（断言）** 工具，可以选择 testify 或 convery，笔者这里选择了 testify。对于 `mock` **（模拟）** 工具，笔者这里选择了 gomock 和 gomonkey。关于 mock 工具同时使用 gomock 和 gomonkey，这里跟 Golang 的语言特性有关，下面会详细的说明。

### **完善测试用例**

这里我们开始对示例代码中的函数做单元测试。

### **生成单测模板代码**

首先在 Goland 中打开项目，加载对应文件后右键找到 Generate 项，点击后选择 Tests for package，之后生成以 `_test.go` 结尾的单测文件。（如果想针对某一特定函数做单测，请选择对应的函数后右键选定 Generate 项执行 Tests for selection。）

这里展示通过 IDE 生成的 `TestGetPersonDetail` 测试函数：

```
package unit

import (
  "reflect"
  "testing"
)

func TestGetPersonDetail(t *testing.T) {
 type args struct {
  username string
 }
 tests := []struct {
  name    string
  args    args
  want    *PersonDetail
  wantErr bool
 }{
  // TODO: Add test cases.
 }
 for _, tt := range tests {
  t.Run(tt.name, func(t *testing.T) {
   got, err := GetPersonDetail(tt.args.username)
   if (err != nil) != tt.wantErr {
    t.Errorf("GetPersonDetail() error = %v, wantErr %v", err, tt.wantErr)
    return
   }
   if !reflect.DeepEqual(got, tt.want) {
    t.Errorf("GetPersonDetail() got = %v, want %v", got, tt.want)
   }
  })
 }
}
```

由 Goland 生成的单测模板代码使用的是官方的 testing 框架，为了更方便的断言，我们把 testing 改造成 testify 的断言方式。

这里其实只需要引入 testify 后修改 test 函数最后的断言代码即可，这里我们以 `TestGetPersonDetail` 为例子，其他函数不赘述。

```
package unit
import (
  "github.com/stretchr/testify/assert" // 这里引入了 testify
  "reflect"
  "testing"
)

func TestGetPersonDetail(t *testing.T) {
 type args struct {
  username string
 }
 tests := []struct {
  name    string
  args    args
  want    *PersonDetail
  wantErr bool
 }{
  // TODO: Add test cases.
 }
 for _, tt := range tests {
  got, err := GetPersonDetail(tt.args.username)
  // 改写这里断言的方式即可
  assert.Equal(t, tt.want, got)
  assert.Equal(t, tt.wantErr, err != nil)
 }
}
```

### **分析代码生成测试用例**

对 `checkUsername` 、 `checkEmail` 纯逻辑函数编写测试用例，这里以 `checkEmail` 为例。

```
func Test_checkEmail(t *testing.T) {
 type args struct {
  email string
 }
 tests := []struct {
  name string
  args args
  want bool
 }{
  {
   name: "email valid",
   args: args{
    email: "1234567@qq.com",
   },
   want: true,
  },
  {
   name: "email invalid",
   args: args{
    email: "test.com",
   },
   want: false,
  },
 }
 for _, tt := range tests {
  got := checkEmail(tt.args.email)
  assert.Equal(t, tt.want, got)
 }
}
```

### **使用 gomonkey 打桩**

对于 `GetPersonDetail` 函数而言，该函数调用了 `getPersonDetailRedis` 函数获取具体的 `PersonDetail` 信息。为此，我们需要为它打一个 “桩”。

**所谓的 “桩”，也叫做 “桩代码”，是指用来代替关联代码或者未实现代码的代码。**

对于函数、成员方法或者是变量的打桩，我们通常使用 gomonkey 来进行打桩。具体 API 请参考：[https://pkg.go.dev/github.com/agiledragon/gomonkey](https://link.zhihu.com/?target=https%3A//pkg.go.dev/github.com/agiledragon/gomonkey)

```
// 拉取用户资料信息并校验
func GetPersonDetail(username string) (*PersonDetail, error) {
 // 检查用户名是否有效
 if ok := checkUsername(username); !ok {
  return nil, errors.New("invalid username")
 }

 // 从 redis 接口获取信息
 detail, err := getPersonDetailRedis(username)
 if err != nil {
  return nil, err
 }

 // 校验
 if ok := checkEmail(detail.Email); !ok {
  return nil, errors.New("invalid email")
 }

 return detail, nil
}
```

从 `GetPersonDetail` 函数可见，为了能够完全覆盖该函数，我们需要控制 `getPersonDetailRedis` 函数不同的输出来保证后续代码都能够被覆盖运行到。因此，这里需要使用 gomonkey 来给 `getPersonDetailRedis` 函数打一个 “桩序列”。

所谓的函数 “桩序列” 指的是提前指定好调用函数的返回值序列，当该函数多次调用时候，能够按照原先指定的返回值序列依次返回。

```
func TestGetPersonDetail(t *testing.T) {
 type args struct {
  username string
 }
 tests := []struct {
  name    string
  args    args
  want    *PersonDetail
  wantErr bool
 }{
  {name: "invalid username", args: args{username: "steven xxx"}, want: nil, wantErr: true},
  {name: "invalid email", args: args{username: "invalid_email"}, want: nil, wantErr: true},
  {name: "throw err", args: args{username: "throw_err"}, want: nil, wantErr: true},
  {name: "valid return", args: args{username: "steven"}, want: &PersonDetail{Username: "steven", Email: "12345678@qq.com"}, wantErr: false},
 }

 // 为函数打桩序列
 // 使用 gomonkey 打函数桩序列
 // 第一个用例不会调用 getPersonDetailRedis，所以只需要 3 个值
 outputs := []gomonkey.OutputCell{
  {
   Values: gomonkey.Params{&PersonDetail{Username: "invalid_email", Email: "test.com"}, nil},
  },
  {
   Values: gomonkey.Params{nil, errors.New("request err")},
  },
  {
   Values: gomonkey.Params{&PersonDetail{Username: "steven", Email: "12345678@qq.com"}, nil},
  },
 }
 patches := gomonkey.ApplyFuncSeq(getPersonDetailRedis, outputs)
 // 执行完毕后释放桩序列
 defer patches.Reset()

 for _, tt := range tests {
  got, err := GetPersonDetail(tt.args.username)
  assert.Equal(t, tt.want, got)
  assert.Equal(t, tt.wantErr, err != nil)
 }
}
```

当使用桩序列时，要分析好单元测试用例和序列值的对应关系，保证最终被测试的代码块都能被完整覆盖。

### **使用 gomock 打桩**

最后剩下 `getPersonDetailRedis` 函数，我们先来看一下这个函数的逻辑。

```
// 通过 redis 拉取对应用户的资料信息
func getPersonDetailRedis(username string) (*PersonDetail, error) {
 result := &PersonDetail{}

 client, err := redis.Dial("tcp", ":6379")
 defer client.Close()
 data, err := redis.Bytes(client.Do("GET", username))

 if err != nil {
  return nil, err
 }

 err = json.Unmarshal(data, result)
 if err != nil {
  return nil, err
 }

 return result, nil
}
```

`getPersonDetailRedis` 函数的核心在于生成了 `client` 调用了它的 `Do` 方法，通过分析得知 `client` 实际上是一个符合 `Conn` 接口的结构体。如果我们使用 gomonkey 来进行打桩，需要先声明一个结构体并实现 `Client` 接口拥有的方法，之后才能使用 gomonkey 给函数打桩。

```
// redis 包中关于 Conn 的定义
// Conn represents a connection to a Redis server.
type Conn interface {
 // Close closes the connection.
 Close() error

 // Err returns a non-nil value when the connection is not usable.
 Err() error

 // Do sends a command to the server and returns the received reply.
 Do(commandName string, args ...interface{}) (reply interface{}, err error)

 // Send writes the command to the client's output buffer.
 Send(commandName string, args ...interface{}) error

 // Flush flushes the output buffer to the Redis server.
 Flush() error

 // Receive receives a single reply from the Redis server
 Receive() (reply interface{}, err error)
}

// 实现接口
type Client struct {}
func (c *Client) Close() error {
  return nil
}
func (c *Client) Err() error {
  return nil
}
func (c *Client) Do(commandName string, args ...interface{}) (interface{}, error) {
  return nil, nil
}
func (c *Client) Send(commandName string, args ...interface{}) error {
  return nil
}
func (c *Client) Flush() error {
  return nil
}
func (c *Client) Receive() (interface{}, error) {
  return nil, nil
}

// 实现接口
type Client struct {}
func (c *Client) Close() error {
 return nil
}
func (c *Client) Err() error {
 return nil
}
func (c *Client) Do(commandName string, args ...interface{}) (interface{}, error) {
 return nil, nil
}
func (c *Client) Send(commandName string, args ...interface{}) error {
 return nil
}
func (c *Client) Flush() error {
 return nil
}
func (c *Client) Receive() (interface{}, error) {
 return nil, nil
}
// 进行测试
func test() {
 c := &Client{}
 gomonkey.ApplyFunc(redis.Dial, func(_ string, _ string, _ ...redis.DialOption) (redis.Conn, error) {
  return c, nil
 })
 gomonkey.ApplyMethod(reflect.TypeOf(c), "Do", func(commandName string, args ...interface{}) (interface{}, error) {
  var result interface{}
  return result, nil
 })
}
```

可见，如果接口实现的方法更多，那么打桩需要手写的代码会更多。因此这里需要一种能自动根据原接口的定义生成接口的 mock 代码以及更方便的接口 mock 方式。于是这里我们使用 gomock 来解决这个问题。

### **本地安装 gomock**

```
# 打开终端后依次执行
go get -u github.com/golang/mock/gomock
go install github.com/golang/mock/mockgen
# 备注说明，很重要！！！
# 安装完成之后，执行 mockgen 看命令是否生效 # 如果显示命令无效，请找到本机的 GOPATH 安装目录下的 bin 文件夹是否有 mockgen 二进制文件
# GOPATH 可以执行 go env 命令找到
# 如果命令无效但是 GOPATH 路径下的 bin 文件夹中存在 mockgen，请将 GOPATH 下 bin 文件夹的绝对路径添加到全局 PATH 中
```

### **生成 gomock 桩代码**

安装完毕后，找到要进行打桩的接口，这里是 [http://github.com/gomodule/redigo/redis](https://link.zhihu.com/?target=http%3A//github.com/gomodule/redigo/redis) 包里面的 `Conn` 接口。

在当前代码目录下执行以下指令，这里我们只对某个特定的接口生成 mock 代码。

```
mockgen -destination=mock_redis.go -package=unit github.com/gomodule/redigo/redis Conn
# 更多指令参考：https://github.com/golang/mock#flags
```

生成的代码参考 **[mock_redis.go](https://link.zhihu.com/?target=https%3A//github.com/xunan007/go_unit_test/blob/master/mock_redis.go)**

### **完善 gomock 相关逻辑**

```
func Test_getPersonDetailRedis(t *testing.T) {
 tests := []struct {
  name    string
  want    *PersonDetail
  wantErr bool
 }{
  {name: "redis.Do err", want: nil, wantErr: true},
  {name: "json.Unmarshal err", want: nil, wantErr: true},
  {name: "success", want: &PersonDetail{
   Username: "steven",
   Email:    "1234567@qq.com",
  }, wantErr: false},
 }
 ctrl := gomock.NewController(t)
 defer ctrl.Finish()

 // 1. 生成符合 redis.Conn 接口的 mockConn
 mockConn := NewMockConn(ctrl)

 // 2. 给接口打桩序列
 gomock.InOrder(
  mockConn.EXPECT().Do("GET", gomock.Any()).Return("", errors.New("redis.Do err")),
  mockConn.EXPECT().Close().Return(nil),
  mockConn.EXPECT().Do("GET", gomock.Any()).Return("123", nil),
  mockConn.EXPECT().Close().Return(nil),
  mockConn.EXPECT().Do("GET", gomock.Any()).Return([]byte(`{"username": "steven", "email": "1234567@qq.com"}`), nil),
  mockConn.EXPECT().Close().Return(nil),
 )

 // 3. 给 redis.Dail 函数打桩
 outputs := []gomonkey.OutputCell{
  {
   Values: gomonkey.Params{mockConn, nil},
   Times:  3, // 3 个用例
  },
 }
 patches := gomonkey.ApplyFuncSeq(redis.Dial, outputs)
 // 执行完毕之后释放桩序列
 defer patches.Reset()

 // 4. 断言
 for _, tt := range tests {
  actual, err := getPersonDetailRedis(tt.name)
  // 注意，equal 函数能够对结构体进行 deap diff
  assert.Equal(t, tt.want, actual)
  assert.Equal(t, tt.wantErr, err != nil)
 }
}
```

从上面可以看到，给 `getPersonDetailRedis` 函数做单元测试主要做了四件事情：

*   生成符合 `redis.Conn` 接口的 `mockConn`  
    
*   给接口打桩序列  
    
*   给函数 `redis.Dial` 打桩  
    
*   断言  
    

这里面同时使用了 gomock、gomonkey 和 testify 三个包作为压测工具，日常使用中，由于复杂的调用逻辑带来繁杂的单测，也无外乎使用这三个包协同完成。

### **查看单测报告**

单元测试编写完毕之后，我们可以调用相关的指令来查看覆盖范围，帮助我们查看单元测试是否已经完全覆盖逻辑代码，以便我们及时调整单测逻辑和用例。本文中完整的单测代码参考：**[get_person_detail_test.go](https://link.zhihu.com/?target=https%3A//github.com/xunan007/go_unit_test/blob/master/get_person_detail_test.go)**

### **使用 go test 指令**

默认情况下，我们在当前代码目录下执行 `go test` 指令，会自动的执行当前目录下面带 `_test.go` 后缀的文件进行测试。如若想展示具体的测试函数以及覆盖率，可以添加 `-v` 和 `-cover` 参数，如下所示：

```
☁️  go_unit_test [master]    go test -v -cover
=== RUN   TestGetPersonDetail
--- PASS: TestGetPersonDetail (0.00s)
=== RUN   Test_checkEmail
--- PASS: Test_checkEmail (0.00s)
=== RUN   Test_checkUsername
--- PASS: Test_checkUsername (0.00s)
=== RUN   Test_getPersonDetailRedis
--- PASS: Test_getPersonDetailRedis (0.00s)
PASS
coverage: 60.8% of statements
ok      unit    0.131s
```

如果想指定测试某一个函数，可以在指令后面添加 `-run ${test文件内函数名}` 来指定执行。

```
☁️  go_unit_test [master]    go test -cover -v  -run Test_getPersonDetailRedis
=== RUN   Test_getPersonDetailRedis
--- PASS: Test_getPersonDetailRedis (0.00s)
PASS
coverage: 41.9% of statements
ok      unit    0.369s
```

在执行 `go test` 命令时，需要加上 `-gcflags=all=-l` 防止编译器内联优化导致单测出现问题，这跟打桩代码存在密切的关系，后面我们会详细的介绍这一点。

因此，一个完整的单测指令可以是 `go test -v -cover -gcflags=all=-l -coverprofile=coverage.out`

### **生成覆盖报告**

最后，我们可以执行 `go tool cover -html=coverage.out` ，查看代码的覆盖情况，使用前请先安装好 go tool 工具。

![](https://pic4.zhimg.com/v2-27501a08040de36917a8dc0c7de8555b_r.jpg)

可以看到待测的代码覆盖率达到 100% 了，完整的代码仓库可以参考：[https://github.com/xunan007/go_unit_test](https://link.zhihu.com/?target=https%3A//github.com/xunan007/go_unit_test)

关于 `go test` 更多的使用方法，可以参考：[https://golang.org/pkg/cmd/go/internal/test/](https://link.zhihu.com/?target=https%3A//golang.org/pkg/cmd/go/internal/test/)

### **思考**

上面我们已经详细的介绍了如何对 go 代码进行单元测试。下面探讨两个问题，帮助我们深入理解 go 单元测试的过程。

### **Q1：桩代码在单测中是如何执行的**

在上面的案例中，针对 interface 我们通过 gomock 来帮我们自动生成符合接口的类后，只需要通过 gomock 约定的 API 就能够对 interface 中的函数按期望和需要来模拟，这个很好理解。

对于函数以及方法的 mock，由于本身代码逻辑已经声明好（go 是静态强类型语言），我们很难通过编码的方式将其 mock 掉，这对我们做单元测试提供了很大的挑战。实际上 gomonkey 提供了让我们在运行时替换原函数 / 方法的能力。虽然说我们在语言层面很难去替换运行中的函数体，但是本身代码最终都会转换成机器可以理解的汇编指令，我们可以通过创建指令来改写函数。

在 gomonkey 打桩的过程中，其核心函数其实是 `ApplyCore`。

```
func (this *Patches) ApplyCore(target, double reflect.Value) *Patches {
 this.check(target, double)
 if _, ok := this.originals[target]; ok {
  panic("patch has been existed")
 }

 this.valueHolders[double] = double
 original := replace(*(*uintptr)(getPointer(target)), uintptr(getPointer(double)))
 this.originals[target] = original
 return this
}
```

不管是对函数打桩还是对方法打桩，实际上最后都会调用这个 `ApplyCore` 函数。

在第 8 行的位置，获取到传入的原始函数和替换函数做了一个 `replace` 的操作，这里就是替换的逻辑所在了。

```
func replace(target, double uintptr) []byte {
 code := buildJmpDirective(double)
 bytes := entryAddress(target, len(code))
 original := make([]byte, len(bytes))
 copy(original, bytes)
 modifyBinary(target, code)
 return original
}

// 关键函数：构建跳转指令
func buildJmpDirective(double uintptr) []byte {
    d0 := byte(double)
    d1 := byte(double >> 8)
    d2 := byte(double >> 16)
    d3 := byte(double >> 24)
    d4 := byte(double >> 32)
    d5 := byte(double >> 40)
    d6 := byte(double >> 48)
    d7 := byte(double >> 56)

    return []byte{
        0x48, 0xBA, d0, d1, d2, d3, d4, d5, d6, d7, // MOV rdx, double
        0xFF, 0x22,     // JMP [rdx]
    }
}

// 关键函数：重写目标函数
func modifyBinary(target uintptr, bytes []byte) {
    function := entryAddress(target, len(bytes))

    page := entryAddress(pageStart(target), syscall.Getpagesize())
    err := syscall.Mprotect(page, syscall.PROT_READ|syscall.PROT_WRITE|syscall.PROT_EXEC)
    if err != nil {
        panic(err)
    }
    copy(function, bytes)

    err = syscall.Mprotect(page, syscall.PROT_READ|syscall.PROT_EXEC)
    if err != nil {
        panic(err)
    }
}
```

从上面的代码可以看出，`buildJmpDirective` 构建了一个函数跳转的指令，把目标函数指针移动到寄存器 rdx 中，然后跳转到寄存器 rdx 中函数指针指向的地址。之后通过 `modifyBinary` 函数，先通过 `entryAddress` 方法获取到原函数所在的内存地址，之后通过 `syscall.Mprotect` 方法打开内存保护，将函数跳转指令以 bytes 数组的形式调用 `copy` 方法写入到原函数所在内存之中，最终达到替换的目的。此外，这里 `replace` 方法还保留了原函数的副本，方便后续函数 mock 的恢复。

为什么 `buildJmpDirective` 要构建这样的跳转指令呢？这里只说结论，具体的推导过程可以参考：[https://bou.ke/blog/monkey-patching-in-go](https://link.zhihu.com/?target=https%3A//bou.ke/blog/monkey-patching-in-go)

```
package main
func a() int { return 1 }
func main() {
  f := a
  f()
}
```

上面这段代码，`a` 是一个指向函数实体的指针，`f` 是指向函数 `a` 指针的指针。把上面函数的调用反汇编，能够看到操作寄存器的具体细节。（ 如果对汇编不是很了解，可以先阅读 [http://www.ruanyifeng.com/blog/2018/01/assembly-language-primer.html](https://link.zhihu.com/?target=http%3A//www.ruanyifeng.com/blog/2018/01/assembly-language-primer.html) ）

![](https://pic2.zhimg.com/v2-91db06bcf911cba6819e45c61c98ad75_r.jpg)

第一行，`lea` 为 load effective address，这里是将 `f` 变量这个值直接赋给 rdx 寄存器， `f` 变量的值是指向 `a` 函数的地址。

第二行，`mov` 表示移动，这里是取到内存地址为 rdx 的数据赋值给 rbx，此时内存地址 rbx 指向的刚好就是 `a` 函数。

最后，调用 rbx 里面的内容，其实也就是执行函数体。

因此，我们想改写函数，只要想办法把需要跳转的函数的地址加载到 rdx 寄存器中，之后使用指令跳转执行。

```
MOV rdx, double
JMP [rdx]
```

最终，把汇编指令翻译成 go 能够识别的版本。

这其实也是汇编里面很常见的热补丁，多用于进程中函数的替换。

### **Q2：执行 -gcflags=all=-l 具体有什么作用**

`-gcflags` 用于在 go 编译构建时进行参数的传递，`all` 表示覆盖所有在 `GOPATH` 中的包，`-l` 表示禁止编译的内联优化。该指令可以防止编译时代码内联优化使得 mock 失败，**最终导致执行单元测试不通过**。下面我们具体来探讨一下 “内联” 以及给单元测试带来的影响。

通俗来讲，内联指的是把简短的函数在调用它的地方展开。由于函数调用有固定的开销（栈和抢占检查），在编译过程中，编译器可以针对代码进行内联，减少函数调用开销。内联优化是高性能编程的一种重要手段。

在 go 中，编译器不会对所有简单函数进行内联优化。go 在决策是否要对函数进行内联时有一个标准：函**数体内包含：闭包调用，select ，for ，defer，go 关键字的的函数不会进行内联。并且除了这些，还有其它的限制。当解析 AST 时，Go 申请了 80 个节点作为内联的预算。每个节点都会消耗一个预算。当一个函数的开销超过了这个预算，就无法内联。**（ 参考自：[https://juejin.cn/post/6924888439577903117](https://link.zhihu.com/?target=https%3A//juejin.cn/post/6924888439577903117) ）

下面我们通过一段简短的代码来理解 go 编译过程的内联优化过程。我们从 gomonkey 关于内联的 issue 摘取了一段代码：

```
package main
import "fmt"
func G2() string {  return "G2" }
func G() string {  return G2() }
func main() {
  g := G()
  fmt.Println(g)
}
```

上面这段代码很简单，`main` 函数中调用了 `G` 函数拿到返回值赋值变量给 `g` 后打印结果。其中 `G` 函数调用了 `G2` 函数，`G2` 函数返回了字符串 `"G2"`。

然而，经过编译器内联优化后的代码，`G` 函数实际被展开了，最终 `main` 函数被内联优化成：

```
func main() {
  // 展开 g := G()
  // => g := "G2"

  // 展开 fmt.Println(g)
  // => 相关
}
```

可见，`G` 函数和 `G2` 函数原本执行时候带来函数栈申请回收，优化过后将不再有。

这里我们执行 `go run -gcflags="-m -m" main.go` 来查看编译在进行以上代码的内联优化。

```
☁️  test  go run -gcflags="-m -m" main.go
# command-line-arguments
./main.go:5:6: can inline G2 as: func() string { return "G2" } ./main.go:9:6: can inline G as: func() string { return G2() } ./main.go:10:11: inlining call to G2 func() string { return "G2" } ./main.go:13:6: cannot inline main: function too complex: cost 87 exceeds budget 80
./main.go:14:8: inlining call to G func() string { return G2() } ./main.go:14:8: inlining call to G2 func() string { return "G2" } ./main.go:15:13: inlining call to fmt.Println func(...interface {}) (int, error) { var fmt..autotmp_3 int; fmt..autotmp_3 = <N>; var fmt..autotmp_4 error; fmt..autotmp_4 = <N>; fmt..autotmp_3, fmt..autotmp_4 = fmt.Fprintln(io.Writer(os.Stdout), fmt.a...); return fmt..autotmp_3, fmt..autotmp_4 }
./main.go:15:13: g escapes to heap ./main.go:15:13: main []interface {} literal does not escape
./main.go:15:13: io.Writer(os.Stdout) escapes to heap <autogenerated>:1: (*File).close .this does not escape G2
```

从打印出的内容可以看，`G2\G\fmt.Println` 都被内联了。

上面提到了 gomokey 打桩的逻辑，它是在函数调用的时候通过机器指令将函数的指向替换了。由于函数编译后被内联，实际上不存在函数的调用，导致单测执行不通过，这也是内联导致 gomonkey 打桩无效的问题所在。

### **参考**

**[内联函数和编译器对 Go 代码的优化](https://link.zhihu.com/?target=https%3A//juejin.cn/post/6924888439577903117)**

**[monkey patching in go](https://link.zhihu.com/?target=https%3A//bou.ke/blog/monkey-patching-in-go/)**

**[阮一峰 -- 汇编入门](https://link.zhihu.com/?target=http%3A//www.ruanyifeng.com/blog/2018/01/assembly-language-primer.html)**