# Go 编码规范 - 完整版

> 本规范在 [Google Golang 代码规范](https://github.com/golang/go/wiki/CodeReviewComments) 的基础上，结合实际工程经验进行了调整和补充。
> 
> 规范落地工具：Gometalinter

---

## 1. 前言

### 规范等级定义

| 等级 | 定义 | 工具行为 |
|------|------|----------|
| **必须（Mandatory）** | 用户必须采用 | 代码扫描工具视为错误 |
| **推荐（Preferable）** | 用户理应采用，特殊情况可例外 | 不视为错误 |
| **可选（Optional）** | 用户自行决定 | 不检查 |

---

## 2. 代码风格

### 2.1 【必须】格式化

代码必须用 `gofmt` 格式化。

### 2.2 【推荐】换行

- 建议一行代码不超过 `120列`
- **例外场景**（可超过列数限制）：
  - 函数签名（长签名可能意味着参数过多，需重新考虑）
  - 长字符串字面量（含换行符时考虑用原始字符串字面量）
  - import 模块语句
  - 工具生成代码
  - struct tag
  - 注释中的文档链接

**错误示例**：
```go
// 不要在函数签名中为了满足推荐列数换行！
func (i *webImpl) GenerateAgentInstallLink(ctx context.Context,
    req *pb.GenerateAgentInstallLinkRequest) (*pb.GenerateAgentInstallLinkResponse, error) {
    ... // gofmt 会使签名与函数内语句对齐，导致可读性降低
}

// 不要为了满足推荐列数换行并拼接字符串！
pubkey := "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAAB..." +
"zi2SqaZVeeXmsF5GAGFJcUylujr78Wf6od8..." // 使搜索和修改更困难
```

### 2.3 【必须】括号和空格

- 遵循 `gofmt` 的逻辑
- 运算符和操作数之间要留空格
- 作为输入参数或数组下标时，运算符和运算数之间不需要空格，紧凑展示

### 2.4 【必须】import 规范

- 使用 `goimports` 自动格式化引入的包名
- `goimports` 会自动按首字母排序，并通过空行分组管理
- 分组顺序：**标准库 → 内部包 → 第三方包**
- **禁止使用相对路径引入包**

```go
// 错误：相对路径
import "../net"

// 正确：完整路径
import "xxxx.com/proj/net"
```

- 包名和 git 路径名不一致时，或多个相同包名冲突时，使用别名：

```go
import (
    "fmt"
    "os"
    "runtime/trace"

    nettrace "golang.net/x/trace"  // 冲突时使用别名
)
```

- 【可选】匿名包引用建议使用新分组并添加注释说明

**完整示例**：
```go
import (
    // standard package & inner package
    "encoding/json"
    "myproject/models"
    "myproject/controller"
    "strings"

    // third-party package
    "github.com/example/utils"
    "github.com/example/beego"
    opentracing "github.com/opentracing/opentracing-go"

    // anonymous import package
    // import filesystem storage driver
    _ "github.com/example/repo/pkg/storage/filesystem"
)
```

### 2.5 【必须】错误处理

#### 2.5.1 error 处理

- `error` 作为函数返回值时，**必须处理**，或赋值给明确忽略的变量
- `defer xx.Close()` 可以不显式处理
- **`error` 必须是最后一个返回参数**

```go
// 错误
func do() (error, int) {}

// 正确
func do() (int, error) {}
```

- 错误描述不需要标点结尾
- 采用独立的错误流进行处理（early return）：

```go
// 错误
if err != nil {
    // error handling
} else {
    // normal code
}

// 正确
if err != nil {
    // error handling
    return
}
// normal code
```

- 错误返回的判断独立处理，不与其他变量组合逻辑判断：

```go
// 错误
x, y, err := f()
if err != nil || y == nil {
    return err   // y 与 err 都为空时会导致错误的调用逻辑
}

// 正确
x, y, err := f()
if err != nil {
    return err
}
if y == nil {
    return errors.New("some error")
}
```

- 【推荐】不需要格式化的错误：`errors.New("xxxx")`
- 【推荐】go1.13+：`fmt.Errorf("module xxx: %w", err)`

#### 2.5.2 panic 处理

- **禁止用 `panic` 进行一般的错误处理**，使用 `error` 和多返回值
- **可以用 `panic` 对不变量（invariant）进行断言**

```go
// 错误：对用户输入进行断言
v, err := strconv.Atoi(userInputFromKeyboard)
if err != nil {
    panic(fmt.Errorf("invalid user input: %v", err))  // 不要这样做
}

// 正确：对不变量进行断言
func readText(n Node) string {
    switch n := n.(type) {
    case *TextNode:
        return n.Text
    case *CommentNode:
        return n.Comment
    default:
        panic(fmt.Errorf("unexpected node type: %T", n))  // 可以
    }
}
```

- `func init()` 中初始化失败影响程序运行时，可以 `panic`
- 全局变量初始化时调用的函数中，可以 `panic`（如 `regexp.MustCompile`）
- **导出的方法一般不允许 `panic`**，必须 `panic` 时使用 `MustXXX` 命名

#### 2.5.3 recover 处理

- **必须在 `defer` 中使用**
- 业务逻辑中一般不需要使用
- 用于捕获具有明确类型的 `panic`，**禁止滥用捕获全部类型的异常**

```go
type FatalError string

func (e FatalError) Error() string { return string(e) }

func main() {
    defer func() {
        e := recover()  // 返回 interface{}，不要命名为 err
        if e != nil {
            err, ok := e.(FatalError)
            if !ok {
                panic(e)  // 继续抛出不认识的异常
            }
            // 响应抛出的错误
        }
    }()
    panic(FatalError("错误信息"))
}
```

### 2.6 【必须】单元测试

- 测试文件名：`example_test.go`
- 测试函数名必须以 `Test` 开头，如 `TestExample`
- `func Foo` 的单测可以为 `func Test_Foo`
- `func (b *Bar) Foo` 的单测可以为 `func TestBar_Foo`
- 单测文件行数限制是普通文件的 2 倍（`1600行`）
- 单测函数行数限制也是普通函数的 2 倍（`160行`）
- **每个重要的可导出函数都要首先编写测试用例**

---

## 3. 注释

- 在编码阶段同步写好变量、函数、包注释，可通过 `godoc` 导出生成文档
- **每个被导出的（大写的）名字都应该有文档注释**
- 非导出类型的方法可以没有文档注释
- **所有注释掉的代码在提交 code review 前都应该删除**，除非添加注释说明原因

### 3.1 【必须】包注释

- 每个包都应该有包注释（main 包除外）
- 格式：`// Package 包名 包信息描述`

```go
// Package math provides basic constants and mathematical functions.
package math
```

### 3.2 【必须】结构体注释

- 每个需要导出的自定义结构体或接口都必须有注释
- 格式：`// 结构体名 结构体信息描述`

```go
// User defines the basic user information.
type User struct {
    Name  string
    Email string
    // Demographic represents the user's ethnic group
    Demographic string
}
```

### 3.3 【必须】方法注释

- 每个需要导出的函数或方法都必须有注释
- 格式：`// 函数名 函数信息描述`

```go
// NewAttrModel is the factory method for attribute data layer operations.
func NewAttrModel(ctx *common.Context) *AttrModel {
    // TODO
}
```

**例外方法**（可无注释）：
- `Write`、`Read`（常见 IO）
- `ServeHTTP`（HTTP 服务）
- `String`（打印）
- `Unwrap`、`Error`（错误处理）
- `Len`、`Less`、`Swap`（排序）

### 3.4 【必须】变量和常量注释

- 每个需要导出的常量和变量都必须有注释
- 格式：`// 变量名 变量信息描述`

```go
// FlagConfigFile is the command line parameter name for config file.
const FlagConfigFile = "--config"

// Command line parameters
const (
    FlagConfigFile1 = "--config" // config file parameter 1
    FlagConfigFile2 = "--config" // config file parameter 2
)
```

### 3.5 【必须】类型注释

- 每个需要导出的类型定义和类型别名都必须有注释
- 格式：`// 类型名 类型信息描述`

```go
// StorageClass represents the storage type.
type StorageClass string

// FakeTime is a type alias for the standard library time.
type FakeTime = time.Time
```

---

## 4. 命名规范

### 4.1 【推荐】包命名

- 保持 `package` 名字和目录一致
- 采用有意义、简短的包名，尽量不和标准库冲突
- **包名应该为小写单词，不要使用下划线或混合大小写**
- 可谨慎使用缩写（如 `strconv`、`syscall`、`fmt`）
- 项目名可以通过中划线连接多个单词
- **禁止使用无意义的包名**：`util`、`common`、`misc`、`global`

### 4.2 【必须】文件命名

- 采用有意义、简短的文件名
- **文件名应该采用小写，并使用下划线分隔各个单词**

### 4.3 【必须】结构体命名

- 采用驼峰命名方式，首字母根据访问控制采用大写或小写
- 结构体名应该是名词或名词短语，如 `Customer`、`WikiPage`
- **避免使用 `Data`、`Info` 这类意义太宽泛的结构体名**
- 声明和初始化采用多行格式

### 4.4 【推荐】接口命名

- 命名规则基本保持和结构体命名规则一致
- 单个函数的接口名以 `er` 作为后缀，如 `Reader`、`Writer`
- 两个函数的接口名综合两个函数名
- 三个以上函数的接口名类似于结构体名

### 4.5 【必须】变量命名

- **必须遵循驼峰式**，首字母根据访问控制决定使用大写或小写
- 特有名词规则：
  - 私有且特有名词为首个单词：使用小写，如 `apiClient`
  - 其他情况：使用原有写法，如 `APIClient`、`repoID`、`UserID`
  - **错误示例**：`UrlArray`，应写成 `urlArray` 或 `URLArray`
- 变量名更倾向于选择短命名（使用位置越远，需要越强的描述性）

### 4.6 【必须】常量命名

- **常量均需遵循驼峰式**
- 枚举类型的常量需要先创建相应类型

```go
// Scheme represents the transfer protocol.
type Scheme string

const (
    // HTTP represents HTTP plain text transfer protocol.
    HTTP Scheme = "http"
    // HTTPS represents HTTPS encrypted transfer protocol.
    HTTPS Scheme = "https"
)
```

### 4.7 【必须】函数命名

- **必须遵循驼峰式**，首字母根据访问控制决定使用大写或小写

---

## 5. 控制结构

### 5.1 【推荐】if

- `if` 接受初始化语句，约定如下方式建立局部变量：

```go
if err := file.Chmod(0664); err != nil {
    return err
}
```

- 对两个值判断时，约定**变量在左，常量在右**：

```go
// 错误
if nil != err {}
if 0 == errorCode {}

// 正确
if err != nil {}
if errorCode == 0 {}
```

- 对于 bool 类型变量，应直接进行真假判断：

```go
// 错误
if allowUserLogin == true {}
if allowUserLogin == false {}

// 正确
if allowUserLogin {}
if !allowUserLogin {}
```

### 5.2 【推荐】for

采用短声明建立局部变量：

```go
sum := 0
for i := 0; i < 10; i++ {
    sum += 1
}
```

### 5.3 【必须】range

- 只需要第一项（key）时，丢弃第二个
- 只需要第二项时，第一项置为下划线

```go
for key := range m {
    if key.expired() {
        delete(m, key)
    }
}

for _, value := range array {
    sum += value
}
```

### 5.4 【必须】switch

**必须有 `default`**：

```go
switch os := runtime.GOOS; os {
    case "darwin":
        fmt.Println("OS X.")
    case "linux":
        fmt.Println("Linux.")
    default:
        fmt.Printf("%s.\n", os)
}
```

### 5.5 【推荐】return

尽早 `return`，一旦有错误发生，马上返回：

```go
f, err := os.Open(name)
if err != nil {
    return err
}
defer f.Close()

d, err := f.Stat()
if err != nil {
    return err
}
codeUsing(f, d)
```

### 5.6 【必须】goto

**业务代码禁止使用 `goto`**，其他框架或底层源码推荐尽量不用。

---

## 6. 函数

### 6.1 【推荐】函数参数

- 返回相同类型的两三个参数，或结果含义不清时，使用命名返回
- 传入变量和返回变量以小写字母开头
- **参数数量不能超过 `5个`**
- 尽量用值传递，非指针传递
- 传入 `map`、`slice`、`chan`、`interface` 时不要传递指针

### 6.2 【必须】defer

- 存在资源管理时，应紧跟 `defer` 函数进行资源释放
- **判断是否有错误发生之后，再 `defer` 释放资源**

```go
resp, err := http.Get(url)
if err != nil {
    return err
}
defer resp.Body.Close()  // 操作成功后再 defer
```

- **禁止在循环中使用 `defer`**（改用匿名函数包裹）

### 6.3 【推荐】方法的接收器

- 【推荐】以类名第一个英文首字母的小写作为接收器命名
- 【推荐】函数超过 `20行` 时不要用单字符
- 【必须】**禁止使用 `me`、`this`、`self` 这类易混淆名称**

### 6.4 【推荐】代码行数

- 【必须】**文件长度不能超过 `800行`**
- 【推荐】**函数长度不能超过 `80行`**

### 6.5 【必须】嵌套

**嵌套深度不能超过 `4层`**。

建议将复杂逻辑抽取为独立函数：

```go
// 错误：嵌套过深
func (s *BookingService) AddArea(areas ...string) error {
    s.Lock()
    defer s.Unlock()
    for _, area := range areas {
        for _, has := range s.areas {
            if area == has {
                return srverr.ErrAreaConflict
            }
        }
        s.areas = append(s.areas, area)
    }
    return nil
}

// 正确：抽取函数
func (s *BookingService) AddArea(areas ...string) error {
    s.Lock()
    defer s.Unlock()
    for _, area := range areas {
        if s.HasArea(area) {
            return srverr.ErrAreaConflict
        }
        s.areas = append(s.areas, area)
    }
    return nil
}

func (s *BookingService) HasArea(area string) bool {
    for _, has := range s.areas {
        if area == has {
            return true
        }
    }
    return false
}
```

### 6.6 【推荐】变量声明

变量声明尽量放在变量第一次使用前面，就近原则。

### 6.7 【必须】魔法数字

**魔数应使用常量或变量做替代。**

```go
// 错误
total := 1.05 * price

// 正确
const TaxRate = 0.05
total := (1.0 + TaxRate) * price
```

**非魔数的例外**（上下文中一眼明白其含义）：
```go
d = b*b - 4*a*c        // 一元二次方程判别式
if x%2 == 0 {}         // 判断奇偶
for i := 0; i < max; i += 1 {}
os.Exit(1)             // 程序错误
scn.Buffer(nil, 10<<20)
strings.IndexOf(s, ":") == -1
```

---

## 7. 依赖管理

### 7.1 【必须】go modules

go1.11 以上**必须使用 `go modules`** 模式：

```bash
go mod init github.com/yourorg/myrepo
```

### 7.2 【推荐】代码提交

- 不对外开源的工程 `module name` 建议使用 `github.com/yourorg/yourrepo`
- 使用 `go modules` 的项目**建议不提交 `vendor` 目录**
- **`go.sum` 文件必须提交**，不要添加到 `.gitignore`

---

## 8. 应用服务

### 8.1 【推荐】README.md

应用服务接口建议有 `README.md`，包括：
- 服务基本描述
- 使用方法
- 部署时的限制与要求
- 基础环境依赖（最低 go 版本、外部通用包版本）

### 8.2 【必须】接口测试

**应用服务必须要有接口测试。**

---

## 附：常用工具

| 工具 | 功能 |
|------|------|
| `gofmt` | 自动格式化代码，保证与官方推荐格式一致 |
| `goimports` | 在 `gofmt` 基础上增加自动删除和引入包 |
| `go vet` | 静态分析源码问题（多余代码、提前 return、struct tag 等） |
| `golint` | 检测代码中不规范的地方 |
