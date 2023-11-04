# Go 编码规范

本文是[Effective Go](https://golang.org/doc/effective_go.html)的补充，根据多年的经验和灵感/想法来自会议演讲。

## 目录

- [给错误添加上下文](#给错误添加上下文)
- [错误和日志消息一致](#错误和日志消息一致)
- [依赖管理](#依赖管理)
	- [使用模块](#使用模块)
	- [使用语义化版本](#使用语义化版本)
- [结构化日志](#结构化日志)
- [避免全局变量](#避免全局变量)
- [留下良好的路径](#留下良好的路径)
- [测试](#测试)
	- [使用断言库](#使用断言库)
	- [使用子测试来组织功能测试](#使用子测试来组织功能测试)
	- [使用表驱动测试](#使用表驱动测试)
	- [避免模拟](#避免模拟)
	- [避免深度匹配](#避免深度匹配)
	- [避免测试未导出的函数](#避免测试未导出的函数)
	- [在测试文件中添加示例以演示用法](#在测试文件中添加示例以演示用法)
- [使用 Linters](#使用-linters)
- [使用 goimports](#使用-goimports)
- [使用有意义的变量名](#使用有意义的变量名)
- [避免副作用](#避免副作用)
- [青睐纯函数](#青睐纯函数)
- [不要过度使用接口](#不要过度使用接口)
- [不要增加包](#不要增加包)
- [处理信号](#处理信号)
- [分离导入](#分离导入)
- [避免未修饰的返回](#避免未修饰的返回)
- [使用规范的导入路径](#使用规范的导入路径)
- [避免空接口](#避免空接口)
- [主函数先写](#主函数先写)
- [使用内部包](#使用内部包)
- [避免 helper/util 包](#避免-helper/util-包)
- [嵌入二进制数据](#嵌入二进制数据)
- [使用 io.WriteString](#使用-io.writestring)
- [使用函数选项](#使用函数选项)
- [结构体](#结构体)
	- [使用命名结构体](#使用命名结构体)
	- [避免 new 关键字](#避免-new-关键字)
- [一致的头部命名](#一致的头部命名)
- [避免魔法数字](#避免魔法数字)

## 给错误添加上下文

**Don't:**
```go
file, err := os.Open("foo.txt")
if err != nil {
	return err
}
```

使用上述方法可能导致错误消息不明确，因为缺少上下文。

**Do:**
```go
file, err := os.Open("foo.txt")
if err != nil {
	return fmt.Errorf("无法打开 foo.txt 文件：%w", err)
}
```

使用自定义消息包装错误提供了上下文，因为它会向上传播到堆栈上。这并不总是有意义的。如果你不确定返回的错误的上下文是否一直足够，请将其包装。

## 依赖管理

### 使用模块
使用[模块](https://github.com/golang/go/wiki/Modules)，因为它是 Go 的内置依赖管理工具，并且将被广泛支持（在 Go 1.11+ 版本中可用）。

### 使用语义化版本化
使用[语义化版本](http://semver.org)为您的包打标签，有关发布的最佳实践，请参阅[模块 wiki](https://github.com/golang/go/wiki/Modules#how-to-prepare-for-a-release)。您的 Go 包的 git 标签应该具有格式 `v<major>.<minor>.<patch>`，例如 `v1.0.1`。

## 结构化日志

**Don't:**
```go
log.Printf("Listening on :%d", port)
http.ListenAndServe(fmt.Sprintf(":%d", port), nil)
// 2017/07/29 13:05:50 Listening on :80
```

**Do:**
```go
import "github.com/sirupsen/logrus"
// ...

logger.WithField("port", port).Info("服务器正在监听")
http.ListenAndServe(fmt.Sprintf(":%d", port), nil)
// {"level":"info","msg":"Server is listening","port":"7000","time":"2017-12-24T13:25:31+01:00"}
```

这只是一个无害的示例，但使用结构化日志可以使调试和日志解析更容易。

## 避免全局变量

**Don't:**
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

全局变量使测试和可读性变得困难，并且每个方法都可以访问它们（即使那些不需要它们的方法）。

**Do:**
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

使用结构体封装变量，并且通过为该结构体实现的方法使它们仅对那些实际需要它们的函数可用。

或者可以使用高阶函数，通过闭包注入依赖项。

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

如果确实需要全局变量或常量（例如定义错误或字符串常量），请将它们放在文件顶部。

**Don't:**
```go
import "xyz"

func someFunc() {
	//...
}

const route = "/some-route"

func someOtherFunc() {
	// 使用 route
}

var NotFoundErr = errors.New("not found")

func yetAnotherFunc() {
	// 使用 NotFoundErr
}
```

**Do:**
```go
import "xyz"

const route = "/some-route"

var NotFoundErr = errors.New("not found")

func someFunc() {
	//...
}

func someOtherFunc() {
	// 使用 route
}

func yetAnotherFunc() {
	// 使用 NotFoundErr
}
```

## 留下良好的路径

**Don't:**
```go
import (
	"encoding/json"
	"github.com/example/external/pkg"
	"fmt"
	"github.com/example/project/pkg/some-lib"
	"os"
)
```

**Do:**
```go
import (
	"encoding/json"
	"fmt"
	"os"

	"github.com/example/project/pkg/some-lib"

	"github.com/company/example/pkg/some-lib"
	"github.com/company/another-example/pkg/some-lib"

	"github.com/example/external/pkg"
	"github.com/another/example/external/pkg"
)
```

按照从内部到外部的顺序对导入进行分组以提高可读性：
1.标准库
2.项目内部包
3.公司内部包
4.外部包

## 知道上下文
**Don't:**
```go
r.Header.Get("authorization")
w.Header.Set("Content-type")
w.Header.Set("content-type")
w.Header.Set("content-Type")
```

**Do:**
```go
r.Header.Get("Authorization")
w.Header.Set("Content-Type")
```

添加上下文路径可以方便导包，并且容易导入。

## Testing

### 使用断言库

**Don't:**
```go
func TestAdd(t *testing.T) {
	actual := 2 + 2
	expected := 4
	if (actual != expected) {
		t.Errorf("预期结果是 %d，但得到的结果是 %d", expected, actual)
	}
}
```

**Do:**
```go
import "github.com/stretchr/testify/assert"

func TestAdd(t *testing.T) {
	actual := 2 + 2
	expected := 4
	assert.Equal(t, expected, actual)
}
```

使用断言库使测试更易读，需要更少的代码，并提供一致的错误输出。

### 使用子测试来组织功能测试
**Don't:**
```go
func TestSomeFunctionSuccess(t *testing.T) {
	// ...
}

func TestSomeFunctionWrongInput(t *testing.T) {
	// ...
}
```

**Do:**
```go
func TestSomeFunction(t *testing.T) {
	t.Run("成功", func(t *testing.T){
		//...
	})

	t.Run("错误的输入", func(t *testing.T){
		//...
	})
}
```

### 使用表驱动测试

**Don't:**
```go
func TestAdd(t *testing.T) {
	assert.Equal(t, 1+1, 2)
	assert.Equal(t, 1+-1, 0)
	assert.Equal(t, 1, 0, 1)
	assert.Equal(t, 0, 0, 0)
}
```

上面的方法看起来更简单，但是很难找到失败的案例，尤其是当有数百个案例时。

**Do:**
```go
func TestAdd(t *testing.T) {
	cases := []struct {
		A, B, Expected int
	}{
		{1, 1, 2},
		{1, -1, 0},
		{1, 0, 1},
		{0, 0, 0},
	}

	for _, tc := range cases {
		tc := tc
		t.Run(fmt.Sprintf("%d + %d", tc.A, tc.B), func(t *testing.T) {
			t.Parallel()
			assert.Equal(t, tc.Expected, tc.A+tc.B)
		})
	}
}
```

将表驱动测试与子测试结合使用可以直接了解哪个案例失败，以及测试了哪些案例。 - [Mitchell Hashimoto at GopherCon 2017](https://youtu.be/8hQG7QlcLBk?t=7m34s)

在并行运行子测试可以在有很多测试案例的情况下获得快速的构建时间。- [Go Blog](https://blog.golang.org/subtests)

需要`tc := tc`。因为如果没有它，只有一个案例会被检查到。- [Be Careful with Table Driven Tests and t.Parallel()](https://gist.github.com/posener/92a55c4cd441fc5e5e85f27bca008721)

### 避免使用模拟

**Don't:**
```go
func TestRun(t *testing.T) {
	mockConn := new(MockConn)
	run(mockConn)
}
```

**Do:**
```go
import "github.com/stretchr/testify/assert"

func TestRun(t *testing.T) {
	ln, err := net.Listen("tcp", "127.0.0.1:0")
	assert.Nil(t, err)

	go func() {
		defer ln.Close()
		_, err := ln.Accept()
		assert.Nil(t, err)
	}()

	client, err := net.Dial("tcp", ln.Addr().String())
	assert.Nil(t, err)

	run(client)
}
```

除非没有其他方法，否则只能使用模拟，而且更倾向于使用真实的实现。- [Mitchell Hashimoto at GopherCon 2017](https://youtu.be/8hQG7QlcLBk?t=26m51s)

### 避免深度匹配

**Don't:**
```go
type myType struct {
	id         int
	name       string
	irrelevant []byte
}

func TestSomething(t *testing.T) {
	actual := &myType{/* ... */}
	expected := &myType{/* ... */}
	assert.True(t, reflect.DeepEqual(expected, actual))
}
```

**Do:**
```go
type myType struct {
	id         int
	name       string
	irrelevant []byte
}

func (m *myType) testString() string {
	return fmt.Sprintf("%d.%s", m.id, m.name)
}

func TestSomething(t *testing.T) {
	actual := &myType{/* ... */}
	expected := &myType{/* ... */}
	if actual.testString() != expected.testString() {
		t.Errorf("预期结果为 '%s'，得到的结果为 '%s'", expected.testString(), actual.testString())
	}
	// or assert.Equal(t, actual.testString(), expected.testString())
}
```

使用`testString()`来比较结构体对于具有许多字段对等性检查不相关的复杂结构体是有帮助的。这种方法只对非常大或树状结构的结构体有意义。- [Mitchell Hashimoto at GopherCon 2017](https://youtu.be/8hQG7QlcLBk?t=30m45s)

Google 开源了他们的[go-cmp](http://github.com/google/go-cmp)包，作为较强大且更安全的 `reflect.DeepEqual` 的替代方案。- [Joe Tsai](https://twitter.com/francesc/status/885630175668346880)。

### 避免测试未导出的函数

只有当您无法通过导出的函数访问路径时，才测试未导出的函数。因为它们是未公开的，所以它们很容易改变。

### 在测试文件中添加示例以演示用法
```go
func ExampleSomeInterface_SomeMethod(){
	instance := New()
	result, err := instance.SomeMethod()
	fmt.Println(result, err)
	// Output: someResult, <nil>
}
```

## 使用 Linters

在提交之前对项目中的所有文件使用[golangci-lint](https://github.com/golangciolangci-lint)中包含的所有 linters 进行 lint。

```bash
# 安装 - 用你想要使用的版本替换 vX.X.X
GO111MODULE=on go get github.com/golangci/golangci-lint/cmd/golangci-lint@vX.X.X
# 传统方法，不使用 go module
go get -u github.com/golangci/golangci-lint/cmd/golangci-lint


# 在项目中使用
golangci-lint run
```
有关详细用法和 CI 进程安装指南，请访问[golangci-lint](https://github.com/golangci/golangci-lint)。

## 使用 goimports

只提交经过 gofmt 格式化的文件。使用 `goimports` 格式化/更新导入语句。

## 使用有意义的变量名
避免单个字母的变量名。它们可能在您编写代码时对您来说更易读，但它们会使代码难以理解并造成困惑。

**Don't:**
```go
func findMax(l []int) int {
	m := l[0]
	for _, n := range l {
		if n > m {
			m = n
		}
	}
	return m
}
```

**Do:**
```go
func findMax(inputs []int) int {
	max := inputs[0]
	for _, value := range inputs {
		if value > max {
			max = value
		}
	}
	return max
}
```
在以下情况下，单个字母的变量名是可以的。
当它们是绝对标准的时候，比如，
	测试中的 `t`
	http 请求处理程序中的 `r` 和 `w`
	在循环中的 `i`
它们命名了方法的接收器，例如 `func (s *someStruct) myFunction(){}`

当然，也要避免太长的变量名，比如 `createInstanceOfMyStructFromString`。

## 避免副作用

**Don't:**
```go
func init() {
	someStruct.Load()
}
```

只有在特殊情况下允许副作用（例如，在 CMD 中解析标记）。如果您找不到其他办法，请重新思考和重构。

## 青睐纯函数

> 在计算机编程中，如果关于函数的两个陈述都成立，那么函数可以被认为是纯函数：
> 1. 如果给定相同的参数值，函数始终计算相同的结果值。函数结果值不能取决于在程序执行过程中可能更改的任何隐藏信息或状态，也不能依赖于来自 I/O 设备的任何外部输入。
> 2. 评估结果不会引起任何语义可观察的副作用或输出，例如可变对象的修改或输出到 I/O 设备。

–[维基百科](https://en.wikipedia.org/wiki/Pure_function)

**Don't:**
```go
func MarshalAndWrite(some *Thing) error {
	b, err := json.Marshal(some)
	if err != nil {
		return err
	}

	return ioutil.WriteFile("some.thing", b, 0644)
}
```

**Do:**
```go
// Marshal 是一个纯函数（尽管无用）
func Marshal(some *Thing) ([]byte, error) {
	return json.Marshal(some)
}

// ...
```

这显然不是在所有情况下都可能的，但是尝试使每个可能的函数都是纯函数可以使代码更易于理解，并改进调试。

## 不要过度使用接口

**Don't:**
```go
type Server interface {
	Serve() error
	Some() int
	Fields() float64
	That() string
	Are([]byte) error
	Not() []string
	Necessary() error
}

func debug(srv Server) {
	fmt.Println(srv.String())
}

func run(srv Server) {
	srv.Serve()
}
```

首选小接口，并且只期望您的功能中需要的接口。

## 不要增加包

删除或合并包比拆分大包更容易。如果不确定是否可以拆分包，请拆分。

## 处理信号

**Don't:**
```go
func main() {
	for {
		time.Sleep(1 * time.Second)
		ioutil.WriteFile("foo", []byte("bar"), 0644)
	}
}
```

**Do:**
```go
func main() {
	logger := // ...
	sc := make(chan os.Signal, 1)
	done := make(chan bool)

	go func() {
		for {
			select {
			case s := <-sc:
				logger.Info("Received signal, stopping application",
					zap.String("signal", s.String()))
				done <- true
				return
			default:
				time.Sleep(1 * time.Second)
				ioutil.WriteFile("foo", []byte("bar"), 0644)
			}
		}
	}()

	signal.Notify(sc, os.Interrupt, os.Kill)
	<-done // Wait for go-routine
```

处理信号允许我们优雅地停止服务器，关闭打开的文件和连接，从而防止文件损坏等问题。

## 分离导入

**Don't:**
```go
import (
	"encoding/json"
	"github.com/example/external/pkg"
	"fmt"
	"github.com/example/project/pkg/some-lib"
	"os"
)
```

**Do:**
```go
import (
	"encoding/json"
	"fmt"
	"os"

	"github.com/example/project/pkg/some-lib"

	"github.com/company/example/pkg/some-lib"
	"github.com/company/another-example/pkg/some-lib"

	"github.com/example/external/pkg"
	"github.com/another/example/external/pkg"
)
```

按照从内部到外部的顺序对导入进行分组以提高可读性：
1. 标准库
2. 项目内部包
3. 公司内部包
4. 外部包

## 避免未修饰的返回

**Don't:**
```go
func run() (n int, err error) {
	// ...
	return
}
```

**Do:**
```go
func run() (n int, err error) {
	// ...
	return n, err
}
```

命名返回对于文档很有用，而未修饰的返回对于可读性和容易出错。

## 使用规范的导入路径

**Don't:**
```go
package sub
```

**Do:**
```go
package sub // import "github.com/me/my-project/sub"
```

添加规范导入路径为该包添加上下文，并且使导入变得容易。

## 避免空接口

**Don't:**
```go
func run(foo interface{}) {
	// ...
}
```

空接口会使代码变得更加复杂和不清晰，请在可以的地方避免使用它们。

## 主函数先写

**Don't:**
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

**Do:**
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

将 `main()` 放在前面可以使文件阅读更加容易。只有 `init()` 函数应该在它上面。

## 使用内部包

如果您创建的是命令行，可以考虑将库移到 `internal/` 中，以防止导入不稳定、变化的包。

## 避免 helper/util 包

使用清晰的名称，并尽量避免创建 `helper.go`、`utils.go` 或甚至 `package`。

## 嵌入二进制数据

使用 `//go:embed` 指令和 [embed](https://pkg.go.dev/embed) 包将模板和其他静态资源添加到二进制文件中，实现单一二进制部署。 对于 Go v1.16 之前的版本，请使用外部工具（例如[github.com/gobuffalo/packr](https://github.com/gobuffalo/packr)）。

## 使用 `io.WriteString`

满足 `io.Writer` 的一些重要类型也有一个 `WriteString` 方法，包括 `*bytes.Buffer`、`*os.File` 和 `*bufio.Writer`。`WriteString` 是与期望
