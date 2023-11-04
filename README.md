# Go代码风格指南

这是根据多年的经验和会议演讲的灵感/想法对[Effective Go](https://golang.org/doc/effective_go.html)进行补充。

## 目录

- [对错误添加上下文](#对错误添加上下文)
- [一致的错误和日志信息](#一致的错误和日志信息)
- [依赖管理](#依赖管理)
	- [使用模块](#使用模块)
	- [使用语义化版本号](#使用语义化版本号)
- [结构化日志记录](#结构化日志记录)
- [避免全局变量](#避免全局变量)
- [保持主要路径的顺序](#保持主要路径的顺序)
- [测试](#测试)
	- [使用断言库](#使用断言库)
	- [使用子测试来组织功能测试](#使用子测试来组织功能测试)
	- [使用表驱动测试](#使用表驱动测试)
	- [避免模拟](#避免模拟)
	- [避免使用DeepEqual](#避免使用deepequal)
	- [避免测试未导出的函数](#避免测试未导出的函数)
	- [在测试文件中添加示例以演示用法](#在测试文件中添加示例以演示用法)
- [使用代码检查工具](#使用代码检查工具)
- [使用goimports](#使用goimports)
- [使用有意义的变量名](#使用有意义的变量名)
- [避免副作用](#避免副作用)
- [偏爱纯函数](#偏爱纯函数)
- [不要过度使用接口](#不要过度使用接口)
- [不要包名过少](#不要包名过少)
- [处理信号](#处理信号)
- [分割导入语句](#分割导入语句)
- [避免裸返回](#避免裸返回)
- [使用规范的导入路径](#使用规范的导入路径)
- [避免空接口](#避免空接口)
- [先编写main函数](#先编写main函数)
- [使用内部包](#使用内部包)
- [避免使用帮助程序/工具包](#避免使用帮助程序工具包)
- [嵌入二进制数据](#嵌入二进制数据)
- [使用io.WriteString](#使用iowritestring)
- [使用函数选项](#使用函数选项)
- [结构体](#结构体)
	- [使用命名的结构体](#使用命名的结构体)
	- [避免使用new关键字](#避免使用new关键字)
- [一致的头部命名](#一致的头部命名)
- [避免魔法数字](#避免魔法数字)

## 对错误添加上下文

**Don't: **
```go
file, err := os.Open("foo.txt")
if err != nil {
	return err
}
```

使用上述方法可能会导致错误消息不明确，因为缺少上下文。

**Do: **
```go
file, err := os.Open("foo.txt")
if err != nil {
	return fmt.Errorf("打开foo.txt失败：%w", err)
}
```

使用自定义消息包装错误提供上下文，因为它会沿着堆栈传播。这并不总是有意义的。如果你不确定返回的错误的上下文是否始终足够，请进行包装。

## 依赖管理

### 使用模块
使用[模块](https://github.com/golang/go/wiki/Modules)，因为这是内置的Go依赖管理工具，将得到广泛的支持（在Go 1.11+中可用）。

### 使用语义化版本号
使用[Semantic Versioning](http://semver.org)为您的包进行标记，有关发布的最佳实践方面的更多信息，请查看[模块wiki](https://github.com/golang/go/wiki/Modules#how-to-prepare-for-a-release)。您的go包的Git标签应具有`v<major>.<minor>.<patch>`的格式，例如`v1.0.1`。

## 结构化日志记录

**Don't: **
```go
log.Printf("Listening on :%d", port)
http.ListenAndServe(fmt.Sprintf(":%d", port), nil)
// 2017/07/29 13:05:50 Listening on :80
```

**Do: **
```go
import "github.com/sirupsen/logrus"
// ...

logger.WithField("port", port).Info("Server is listening")
http.ListenAndServe(fmt.Sprintf(":%d", port), nil)
// {"level":"info","msg":"Server is listening","port":"7000","time":"2017-12-24T13:25:31+01:00"}
```

这是一个无害的例子，但使用结构化日志记录可以使调试和日志解析变得更容易。

## 避免全局变量

**Don't: **
```go
var db *sql.DB

func main() {
	db = // ...
	http.HandleFunc("/drop", DropHandler)
	// ...
}

func DropHandler(w http.ResponseWriter, r *http.Request) {
	db.Exec("DROP DATABASE prod")
}
```

全局变量使得测试和阅读困难，并且每个方法都可以访问它们（甚至不需要）。

**Do: **
```go
func main() {
	db := // ...
	handlers := Handlers{DB: db}
	http.HandleFunc("/drop", handlers.DropHandler)
	// ...
}

type Handlers struct {
	DB *sql.DB
}

func (h *Handlers) DropHandler(w http.ResponseWriter, r *http.Request) {
	h.DB.Exec("DROP DATABASE prod")
}
```
使用结构体封装变量，并通过为该结构体实现的方法使它们仅对那些实际需要它们的函数可用。

或者，可以使用高阶函数通过闭包注入依赖项。
```go
func main() {
	db := // ...
	http.HandleFunc("/drop", DropHandler(db))
	// ...
}

func DropHandler(db *sql.DB) http.HandleFunc {
	return func (w http.ResponseWriter, r *http.Request) {
		db.Exec("DROP DATABASE prod")
	}
}
```

如果您真的需要全局变量或常量，例如用于定义错误或字符串常量，请将它们放在文件的顶部。

**Don't: **
```go
import "xyz"

func someFunc() {
	//...
}

const route = "/some-route"

func someOtherFunc() {
	// usage of route
}

var NotFoundErr = errors.New("not found")

func yetAnotherFunc() {
	// usage of NotFoundErr
}
```

**Do: **
```go
import "xyz"

const route = "/some-route"

var NotFoundErr = errors.New("not found")

func someFunc() {
	//...
}

func someOtherFunc() {
	// usage of route
}

func yetAnotherFunc() {
	// usage of NotFoundErr
}
```

## 保持主要路径的顺序

**Don't: **
```go
import (
	"encoding/json"
	"github.com/some/external/pkg"
	"fmt"
	"github.com/this-project/pkg/some-lib"
	"os"
)
```

**Do: **
```go
import (
	"encoding/json"
	"fmt"
	"os"

	"github.com/bahlo/this-project/pkg/some-lib"

	"github.com/bahlo/another-project/pkg/some-lib"
	"github.com/bahlo/yet-another-project/pkg/some-lib"

	"github.com/some/external/pkg"
	"github.com/some-other/external/pkg"
)
```

将导入根据从内部到外部的四组分开排序，以提高可读性：
1. 标准库
2. 项目内部包
3. 公司内部包
4. 外部包

## 避免裸返回

**Don't: **
```go
func run() (n int, err error) {
	// ...
	return
}
```

**Do: **
```go
func run() (n int, err error) {
	// ...
	return n, err
}
```

具名返回对于文档很有帮助，裸返回对可读性有害且容易出错。

## 使用规范的导入路径

**Don't: **
```go
package sub
```

**Do: **
```go
package sub // import "github.com/my-package/pkg/sth/else/sub"
```

添加规范的导入路径会为包添加上下文，使导入更加简单。

## 避免空接口

**Don't: **
```go
func run(foo interface{}) {
	// ...
}
```

空接口使代码变得更加复杂和不清晰，在可以的地方尽量避免使用。

## 先编写main函数

**Don't: **
```go
package main // import "github.com/me/my-project"

func someHelper() int {
	// ...
}

func someOtherHelper() string {
	// ...
}

func Handler(w http.ResponseWriter, r *http.Reqeust) {
	// ...
}

func main() {
	// ...
}
```

**Do: **
```go
package main // import "github.com/me/my-project"

func main() {
	// ...
}

func Handler(w http.ResponseWriter, r *http.Reqeust) {
	// ...
}

func someHelper() int {
	// ...
}

func someOtherHelper() string {
	// ...
}
```

将`main()`放在最前面可以更容易地阅读文件。只有`init()`函数应该在它上面。

## 使用内部包

如果您正在创建一个命令行工具，请考虑将库移动到`internal/`中，以防止导入不稳定、易变的包。

## 避免使用帮助程序/工具包

使用清晰的名称，并尽量避免创建`helper.go`、`utils.go`甚至包。

## 嵌入二进制数据

为了能够进行单个二进制部署，使用`//go:embed`指令和[embed](https://pkg.go.dev/embed)包将模板和其他静态资源添加到您的二进制文件中。
对于较早版本的Go（v1.16之前），请使用外部工具（例如[github.com/gobuffalo/packr](https://github.com/gobuffalo/packr)）。

## 使用`io.WriteString`

一些重要的类型满足`io.Writer`，还具有`WriteString`方法，包括`*bytes.Buffer`、`*os.File`和`*bufio.Writer`。`WriteString`是一个行为合同，隐含地同意传递的字符串将以高效的方式写入，而不会进行临时分配。因此，使用`io.WriteString`可能会提高性能，至少可以确保字符串将以任何方式写入。

**Don't: **
```go
var w io.Writer = new(bytes.Buffer)
str := "some string"
w.Write([]byte(str))
```

**Do: **
```go
var w io.Writer = new(bytes.Buffer)
str := "some string"
io.WriteString(w, str)
```

## 使用函数选项

```go

func main() {
	// ...
	startServer(
		WithPort(8080),
		WithTimeout(1 * time.Second),
	)
}

type Config struct {
	port    int
	timeout time.Duration
}

type ServerOpt func(*Config)

func WithPort(port int) ServerOpt {
	return func(cfg *Config) {
		cfg.port = port
	}
}

func WithTimeout(timeout time.Duration) ServerOpt {
	return func(cfg *Config) {
		cfg.timeout = timeout
	}
}

func startServer(opts ...ServerOpt) {
	cfg := new(Config)
	for _, fn := range opts {
		fn(cfg)
	}

	// ...
}


```

## 结构体
### 使用命名的结构体
如果一个结构体有多个字段，请在实例化时包含字段名称。

**Don't: **
```go
params := myStruct{
	1, 
	true,
}
```

**Do: **
```go
params := myStruct{
	Foo: 1,
	Bar: true,
}
```

### 避免使用new关键字
使用正常的语法代替`new`关键字，这样更清楚地说明正在发生什么：创建了一个新的结构体实例`MyStruct{}`，然后使用`&`获取它的指针。

**Don't: **
```go
s := new(MyStruct)
```

**Do: **
```go
s := &MyStruct{}
```

## 一致的头部命名
**Don't: **
```go
r.Header.Get("authorization")
w.Header.Set("Content-type")
w.Header.Set("content-type")
w.Header.Set("content-Type")
```

**Do: **
```go
r.Header.Get("Authorization")
w.Header.Set("Content-Type")
```

## 避免魔法数字
没有名字和上下文的数字只是一个随机值。它没有任何意义，因此避免在代码中使用它们（例外可能是数字0，例如在创建循环时使用）。

**Don't: **
```go
func IsStrongPassword(password string) bool {
	return len(password) >= 8
}
```

**Do: **
```go
const minPasswordLength = 8

func IsStrongPassword(password string) bool {
	return len(password) >= minPasswordLength
}
