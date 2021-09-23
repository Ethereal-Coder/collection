> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [zhuanlan.zhihu.com](https://zhuanlan.zhihu.com/p/391478681)

**前言**

单元测试，通常是单独测试一个方法、类或函数，让开发者确信自己的代码在按预期运行，为确保代码可以测试且测试易于维护。 另一方面，DevOps 里提倡自动化测试，并且主张越早发现代价越小。关于单元测试的更多思考，可以看看本文最后一节。

![](https://pic3.zhimg.com/v2-a161cd21d0d91573b59209b3df5f1ba6_r.jpg)

本文结合了公司级漏洞扫描系统洞犀在 DevOps 上探索的经验，以 Golang 为例，列举了编写单元测试需要的工具和方法，然后针对写单测遇到的各种依赖问题，提出相应的解决办法，并展示了自动化单元测试的结果。最后再和大家探讨一下关于单元测试上的一些思考。也感谢鹏哥等同事们对文章提出了一些建议。

### **测试工具和方法**

### **测试框架**

相信大家都熟悉 go 内置了`go test`测试框架来执行和管理测试用例。通过文件名`_test.go`结尾来表示测试文件，通过函数以`Test`开头并只有一个参数`*testing.T`来表示一个测试函数。例如：

```
// sample_test.go

package sample_test

import ( "testing" )

func TestDownload(t *testing.T) {}
func TestUpload(t *testing.T) {}
```

而其中测试框架`testing`的类型`*T`提供了一系列方法，例如主要会用到的下面三个方法：

*   `t.Fatal`：会让测试函数立刻返回错误
*   `t.Error`：会输出错误并记录失败，但任然会继续运行
*   `t.Log`： 输出 debug 信息，`go test -v`参数下有效

除此之外，还有其他用的比较多的测试包。例如断言包`"github.com/stretchr/testify/assert"`，比如如果想判断返回的错误是否是空，如果用原生方法会是:

```
if err != nil
 t.Errorf("got error %v", err)
}
```

但用`assert`包只需要一行代码就可以实现上述功能，而且可以输出具体错误代码行： `assert.Nil(t, err)` 另外还有封装了 testing 的测试框架 [https://github.com/smartystreets/goconvey](https://link.zhihu.com/?target=https%3A//github.com/smartystreets/goconvey) ，里面包含了子测试断言等功能。

### **表格驱动测试**

表格驱动测试通过定义一组不同的输入，可以让代码得到充分的测试，同时也能有效地测试负路径。 例如下面函数会判断参数类型，如果是 int 就乘以二，如果是 string 就先转成 int 然后乘以二，如果是其他类型就返回错误：

```
func twice(i interface{}) (int, error) {
 switch v := i.(type) {
 case int:
  return v * 2, nil
 case string:
  value, err := strconv.Atoi(v)
  if err != nil {
   return 0, errors.Wrapf(err, "invalid string num %s", v)
  }
  return value * 2, nil
 default:
  return 0, errors.New("unknown type")
 }
}
```

可以看到该函数有多个分支，如果要覆盖到不同分支，就需要不同类型输入，那么这就很适合表格驱动测试：

```
func Test_twice(t *testing.T) {
 type args struct {
  i interface{}
 }
 tests := []struct {
  name    string
  args    args
  want    int
  wantErr bool
 }{
  {
   name: "int",
   args: args{i: 10},
   want: 20,
  },
  {
   name: "string success",
   args: args{i: "11"},
   want: 22,
  },
  {
   name:    "string failed",
   args:    args{i: "aaa"},
   wantErr: true,
  },
  {
   name:    "unknown type",
   args:    args{i: []byte("1")},
   wantErr: true,
  },
 }
 for _, tt := range tests {
  t.Run(tt.name, func(t *testing.T) {
   got, err := twice(tt.args.i)
   if (err != nil) != tt.wantErr {
    t.Errorf("twice() error = %v, wantErr %v", err, tt.wantErr)
    return
   }
   if got != tt.want {
    t.Errorf("twice() got = %v, want %v", got, tt.want)
   }
  })
 }
}
```

上面还用到了`go test`的子测试功能`t.Run(name string, subTest func(t *T))`。如果想在一个测试函数里面执行多个测试用例，例如要同时测试一个函数的返回成功和失败等各种情况，那么可以使用子测试来区分不同情况。 另外，上面表格测试代码框架是用 goland 自动生成的，自己只需要填写`tests`数组就行了。点击函数名然后右键，选择 generate，然后选择 test for function 就会自动生成测试函数了。不过上面生成的函数没有校验返回的错误内容，如有需要可以自己稍微修改一下。

### **解决常见的依赖等问题**

解决常见的依赖等问题目前有两种思路：

*   通过 mock 方式替换实际依赖，并通过打桩操作其返回内容
*   通过本地启动一个模拟依赖环境，比如模拟 redis 服务等，然后直接访问模拟服务。

下面几小节详细介绍了上述两种办法在不通场景下的应用，其中替换函数或方法、依赖接口类型和 mysql 数据库依赖对应了第一种思路；访问访问 http 接口、mysql 数据库依赖和 redis 依赖对应了上面第二条思路。

### **访问 http 接口**

代码里经常会遇到要访问 http 接口的情况，这时如果在测试代码里不做处理直接访问，可能遇到环境不同访问不通等问题。为此 go 标准库内置了专门用于测试 http 服务的包`net/http/httptest`，不过我们这里并不用它来测试 http 服务，而是用来模拟要请求的 http 服务。 基本流程是先创建一个路由器，然后注册一个响应函数用来模拟要请求的服务：

```
mux := http.NewServeMux()
mux.HandleFunc("/", func(writer http.ResponseWriter, request *http.Request) {
 writer.Write([]byte(`{"code":0,"msg":"ok"}`))
})
```

接着启动这个服务，httptest 会真的在 localhost 启动它，然后这个 URL 就是要访问的服务地址了。

```
server := httptest.NewServer(mux)
defer server.Close()
url := server.URL
```

### **替换函数或方法**

大家用的最多的应该就是`monkey`补丁库了，可以用它来替换各种函数和方法，使用起来非常方便，这类库原理大致相同，通过运行时用 unsafe 包替换函数地址来实现。比如 [https://github.com/agiledragon/gomonkey](https://link.zhihu.com/?target=https%3A//github.com/agiledragon/gomonkey) ，不过这次我们用公司内部同源测试团队封装的 monkey 库来演示`ngmock`。 首先是替换函数，新建一个函数 mock 对象，参数有`*testing.T`和要 mock 的函数。比如被测函数需要调用`db.New`新建一个 DB，那么下面就 mock 了`db.New`函数。

```
dbNewMock := ngmock.MockFunc(t, db.New)
defer dbNewMock.Finish()
```

然后在执行被测函数之前，设置 mock 函数接收什么参数，并且要返回什么，比如下面指定接收一个任意参数并且让`db.New`返回指定错误。该设置默认只会生效一次，如果要生效多次或者一直生效可以配置次数。

```
dbNewMock.Mock(ngmock.Any()).Return(nil, errors.New("fake err")).AnyTimes()
```

接下来就是执行被测函数函数来验证是否生效了，这里用到了上面提到的另一个测试框架`convey`，`convey.Convey`同`*T.Run()`，`convey.So`是 assert。

```
func TestNewDBs(t *testing.T) {
 convey.Convey("TestNewDBs", t, func() {
  dbNewMock := ngmock.MockFunc(t, db.New)
  defer dbNewMock.Finish()
  convey.Convey("TestNewDBs failed", func() {
   dbNewMock.Mock(ngmock.Any()).Return(nil, errors.New("fake err")).AnyTimes()
   dbs, err := NewDBs(DbUrl{}) // 执行被测函数
   convey.So(dbs, convey.ShouldResemble, &DBs{})
   convey.So(err, convey.ShouldNotBeNil)
   convey.So(err.Error(), convey.ShouldEqual, "fake err") // 验证是否生效
  })
 })
}
```

可以看到，mock 依赖函数之后执行被测函数，会返回我们设置的错误`fake error`，在调用完成获得返回错误之后可以判断一下是否是我们设置的错误。 还可以 mock 结构体方法，使用方式和上面类似，第二个参数传结构体或者指针，第三个是 mock 模式：

```
execCmdMock := ngmock.MockStruct(t, exec.Cmd{}, ngmock.Silent)
defer execCmdMock.Finish()
```

mock 模式主要有两种：

*   Silent 结构体内没有 mock 的方法返回类型默认值，调用没有 mock 的方法不会报错，最后对调用方法的统计不会报错
*   KeepOrigin 结构体内没有 mock 的方法按照原方法逻辑返回数据，调用没有 mock 的方法不会报错，最后对调用方法的统计不会报错。

在 mock 方法时，需要指定方法名，比如下面就 mock 了该结构体的`Output`方法，方法如果有参数的话，可以在后面加上参数。其他的就和前面一样了。

```
execCmdMock.Mock("Output").Return([]byte("1"), nil)
```

如果在 MacOS 上执行测试遇到了`permission denied`的错误，这是 MacOS 保护机制导致的，具体解决办法见 [https://github.com/eisenxp/macos-golink-wrapper](https://link.zhihu.com/?target=https%3A//github.com/eisenxp/macos-golink-wrapper) 。

### **依赖接口类型**

如果依赖的数据是接口类型，那么可以很方便的通过依赖注入的方式传入测试用的接口实现来替换原始依赖。go 官方出品的`gomock` 可以根据接口定义自动生成相应实现的 mock 桩代码：[https://github.com/golang/mock](https://link.zhihu.com/?target=https%3A//github.com/golang/mock) 。 gomock 库会有个二进制文件`mockgen`用来生成代码， 比如文件里有一些接口定义：

```
// interfaces.go

// Encoder 编码器
type Encoder interface {
 Encode(obj interface{}, w io.Writer) error
}
//go:generate mockgen -destination=./mockdata/interfaces_mock.go -package=mockdata -self_package=./mockdata -source=interfaces.go
```

可以执行 mockgen 来生成上述接口，具体命令如上，`-destination`指定生成文件名，`-package`是生成文件包名，`-self_package`指定生成的包路径，`-source`就是源接口文件路径名。如果最后不指定接口名的话，会生成所有接口或者可以指定要生成的接口，多个用逗号连接。 当然也可以读取标准库的接口：`mockgen database/sql/driver Conn,Driver` 桩代码生成好了之后，就可以调用代码里类似 NewMockXXXX(ctrl) 方法来创建 mock 对象，如下所示，这样创建的`encoderMock`实现了上面的`Encoder`接口，接下来就用这个`encoderMock`来初始化被测函数依赖的接口即可。

```
ctrl := gomock.NewController(t) // *testing.T
defer ctrl.Finish()
// mockdata 是上面生成的桩代码目录
encoderMock := mockdata.NewMockEncoder(ctrl)
```

在调用被测函数之前，需要先打桩：我们希望如果`encoderMock`在执行`Encode`方法时传入会两个指定参数，那么就执行指定的函数并返回：

```
codecMock.EXPECT().Encode(gomock.Any(), gomock.Any()).DoAndReturn(func(obj interface{}, w io.Writer) error {
 w.Write([]byte("test_data"))
 return nil
})
```

接下来执行被测函数，当实际调用到`Encode`方法时，就会执行我们设置的函数。看起来和上面一节的替换函数和方法类似是吧？这种希望当调用函数`Encode()`并且参数一致，那么就执行指定逻辑的方式，就是打桩 (stub)。打桩过程还可以配置执行次数和执行顺序等，如果不知道打桩函数具体会被传入什么参数可以用`gomock.Any()`来代替。通过打桩可以控制依赖接口的行为，解决测试时接口依赖的问题。

### **mysql 数据库依赖**

数据库依赖也是经常要遇到的一个问题，如何解决测试过程中的依赖呢？我这里总结了两种办法： 首先是`sqlmock`：[https://github.com/DATA-DOG/go-sqlmock](https://link.zhihu.com/?target=https%3A//github.com/DATA-DOG/go-sqlmock) 。看到 mock 字眼大家大概也知道它是怎么使用的了，也是通过对执行 sql 语句打桩来完成测试。 首先初始化 mock 对象，返回第一个是`*sql.DB`，用来传给被测代码依赖的 db，第二个就是 mock 对象，用来设置打桩代码。控制 sqlDB 的行为。

```
sqlDB, dbMock, err := sqlmock.New()
```

具体使用项目文档里有，我这里简单说一下： 比如下面一个函数执行一些 sql 语句，先调用`Begin`创建事务，然后分别`Query`和`Exec`执行 sql，最后如果返回错误则`Rollback`否则`Commit`。

```
func testFunc(db *sql.DB) error {
 tx, err := db.Begin()
 if err != nil {
  return err
 }
 defer func() {
  if err != nil {
   tx.Rollback()
  } else {
   tx.Commit()
  }
 }()
 rows, err := tx.Query("select * from test where id > 10")
 if err != nil {
  return err
 }
 defer rows.Close()
 for rows.Next() {
  // 省略
 }
 if _, err := tx.Exec("update test set num = num +1"); err != nil {
  return err
 }
 return nil
}
```

那么针对上面函数，编写测试用例如下。其中打桩代码按照上面顺序，希望先执行`Begin`；然后执行`Query`，并且希望 sql 语句满足正则`select .* from test`并返回两行结果；然后执行`Exec`，希望 sql 满足正则`update test`并返回错误；最后执行`Rollback`。接下来执行被测函数，如果被测函数按照打桩代码的顺序执行相应 sql 的话就会返回指定内容，否则就会报错。

```
func Test_testFunc(t *testing.T) {
 convey.Convey("Test_testFunc exec failed", t, func() {
  sqlDB, dbMock, err := sqlmock.New()
  convey.So(err, convey.ShouldBeNil)

  dbMock.ExpectBegin()
  // sql支持正则匹配
  dbMock.ExpectQuery("select .* from test").
   WillReturnRows(sqlmock.NewRows([]string{"id", "num"}).
    AddRow(1, 10).AddRow(2, 20))
  dbMock.ExpectExec("update test").WillReturnError(errors.New("fake error"))
  dbMock.ExpectRollback()

  err = testFunc(sqlDB) // 执行被测函数

  convey.So(err, convey.ShouldNotBeNil)
  convey.So(err.Error(), convey.ShouldEqual, "fake error") // 验证打桩是否生效

  // 确认所有打桩都被调用
  convey.So(dbMock.ExpectationsWereMet(), convey.ShouldBeNil)
 })
}
```

有时候我们的代码不会直接使用`*sql.DB`，而是用到一些第三方 ORM 框架，那么需要想办法让这些框架使用我们的 mock db，比如对于 gorm 框架，可以这么配置：

```
sqlDB, dbMock, err := sqlmock.New()
// "gorm.io/driver/mysql"
// "gorm.io/gorm"
db, err = gorm.Open(mysql.New(mysql.Config{Conn: sqlDB}), &gorm.Config{})
```

谈到 gorm 框架，那么问题来了，如果我不直接操作`*sql.DB`而是用的框架，但我不知道最后生成的 sql 是什么那该怎么办？或者说被测函数有一堆 sql 语句，一个一个打桩起来实在是太麻烦。那么对于这种情况如果能有一个本地数据库环境就好了，省去了打桩的麻烦，但是如果是 mysql 这种 DB 的话，本地建一个最快也是用容器跑才行。那么有没有更轻量化的办法呢？ 可以本地临时创建一个 sqlite 数据库来代替当前依赖的数据库比如 mysql 等，sqlite 是可以在本地直接跑的轻量级数据库，常见 sql 语句增删改查什么的和 mysql 区别不大。不过需要注意的是目前所有的 go sqlite 驱动都是基于 CGO 的，因为 sqlite 使用 C 写的。所以引用这些驱动会导致测试前程序编译速度变慢和跨平台支持问题，不过目前测试在 MacOS 和 linux 上是没有问题的。 如下所示首先创建一个临时的 sqlite gorm 框架 DB，其中连接地址置空，这样在关闭 db 之后数据库也会自动删除。之后就可以正常使用了。它底层使用的是这个驱动`github.com/mattn/go-sqlite3`。

```
import(
 "gorm.io/driver/sqlite"
 "gorm.io/gorm"
)
db, err := gorm.Open(sqlite.Open(""), &gorm.Config{})
```

如果使用场景只是增删改查什么的，问题不会很大，我目前遇到的和 mysql 不兼容的就是`create table a like b`这种 sql。而且如果不直接执行 sql 而用框架取调用相关函数的话，兼容性会好很多。

### **redis 依赖**

很多项目还会依赖 redis 数据库，那么这种该怎么解决依赖问题呢？可以使用 miniredis 库解决问题：[https://github.com/alicebob/miniredis](https://link.zhihu.com/?target=https%3A//github.com/alicebob/miniredis) 。 `miniredis`是一个纯 GO 写的测试用的 redis 服务，它支持绝大多数 redis 命令，具体可以看项目介绍。 使用起来很简单，直接调用 Run 函数启动一个测试服务，服务对象的`Addr()`方法返回服务连接地址。接下来可以就可以拿着这个地址替换当前依赖了。

```
import "github.com/alicebob/miniredis/v2"
import "github.com/go-redis/redis/v8"

mr, err := miniredis.Run()
addr := mr.Addr() // redis服务的tcp连接地址

// 比如创建一个客户端
opt, err := redis.ParseURL("redis://:@" + mr.Addr())
cli := redis.NewClient(opt)
```

### **执行测试用例前后设置**

有时需要在测试之前或之后进行额外的设置（setup）或拆卸（teardown），为此 testing 包提供了 TestMain 函数作为测试文件的入口。如下所示，该文件的测试用例都会在`m.Run`里运行，如果成功返回 0 否则非零，因此可以判断执行是否成功。值得注意的是最后应该使用 code 作为`os.Exit`参数退出。

```
func TestMain(m *testing.M) {
 code := m.Run()
 if code == 0 {
  TearDone(true)
 } else {
  TearDone(false)
 }
 os.Exit(code)
}

func TearDone(isSuccess bool) {
 fmt.Println("Global test environment tear-down")
 if isSuccess {
  fmt.Println("[  PASSED  ]")
 } else {
  fmt.Println("[  FAILED  ]")
 }
}
```

### **忽略指定目录**

有时需要忽略指定目录，例如自动生成的桩代码，proto 文件等，以提高覆盖率，那么对于下面的测试命令： `go test -v -covermode=count -coverprofile=coverage_unit.out '-gcflags=all=-N -l' ./...` 如果要忽略掉`mockdata`目录的话，后面加上`grep -v mockdata`即可：

```
go test -v -covermode=count -coverprofile=coverage_unit.out  '-gcflags=all=-N -l' `go list ./... | grep -v /mockdata`
```

然后可以运行`go tool cover -html=coverage_unit.out -o cover.html`，生成网页版报告，查看覆盖率情况。 当然还有一个比较 tricky 的方法，如果生成的桩代码仅限于某个包内使用，那么直接把桩代码文件名改成`_test.go`后缀的就行了。

### **总结**

本篇介绍了 go 语言写单元测试的工具、方法，详细介绍了通过 Mock 的方式解决各种常用依赖，方便读者在写 go 语言 UT 的时候，遇到依赖问题，能够快速找到解决方案。

### **思考：关于单元测试**

1.  **单测的意义**

首先必须承认有了单元测试之后，增加了代码质量的保障。而且在做修改和重构的时候，也能降低心智负担，相信大家都体验过对一堆没有单测的代码做修改时心里都会有点打怵，生怕改出什么问题。

但是对于没有单元测试的人来说，刚开始写单测无疑是让人非常头大，简直寸步难行。因为已经维护的代码可能在设计上就很难测试，各种耦合各种依赖没有抽象混在一起，一行代码成百上千行，这些都加深了接入单元测试的难度和工作量。而由于没有质量保证又不敢动这些祖传代码，从而导致陷入死循环。

但总得想办法改变现状，最近看了公司内部技术论坛的测试专题，之后也是跟各路大神学习到了一些东西。首先可以先让重要逻辑代码有测试。其次就是关注代码设计问题，对新增代码坚持写单侧，我在码客上看到有前辈说，**UT 不是用来找 BUG 的，而是通过 UT 来改良设计，从而提升代码质量，降低 BUG 数量。** 反之如果 UT 不好写，说明代码结构混乱，出现 BUG 的概率也变高。

1.  **不能为了单测而单测**

单元测试覆盖率高真的可以确保质量吗？是否能消除 BUG？这个按我个人经验其实是不能完全保证的。首先得考虑单测覆盖代码分支是否完备？有时候为了偷懒只测了主路径，对于其他负路径等没有测试，那么肯定会有问题的。其次测试环境和线上实际环境的潜在差异可能也会导致代码 BUG 没测试出来。我遇到过在写打桩代码的时候，懒得校验参数，直接用 mock.Any 代替，导致做集成测试的时候发现参数传错了，写这种单测除了浪费时间之外基本上也发现不了什么问题。

1.  **有没有更好折中方案**

有时候函数逻辑比较复杂导致插桩过程繁琐，或者有些依赖不方便 mock，那么是否能在执行测试用例的时候创建一个本地测试环境，里面包含了各种依赖，这样或许会方便很多。比如上一节介绍解决依赖的办法里有提到为了解决 DB 依赖，可以临时创建一个 sqlite 数据库，或者启动一个容器来模拟执行环境。

腾讯技术交流群已建立，交流讨论可加 **QQ 群：**160315980**（备注腾讯技术） ，微信交流群加：**teg_helper**。**