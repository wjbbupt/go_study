# [o 进阶训练营 Week02: error 错误处理](https://www.cnblogs.com/zhangpengfei5945/p/14470279.html)

## Error vs Exception

Error:

Go error 就是普通的一个接口，普通的值。Errors are values

```
type error interface {
    Error() string
}
```

经常使用 errors.New() 来返回一个 error 对象，errors.New() 返回的是内部 errorString 对象的指针

errors.go 中

```
package errors

// New returns an error that formats as the given text.
// Each call to New returns a distinct error value even if the text is identical.
func New(text string) error {
	return &errorString{text}
}

// errorString is a trivial implementation of error.
type errorString struct {
	s string
}

func (e *errorString) Error() string {
	return e.s
}
```

### 为什么返回的是 errorString 对象的指针而不是值呢？

我们看一个demo

```
package main

import (
  "errors"
  "fmt"
)

// Create a named type for our new error type.
type errorString string

// Implement the error interface
func (e errorString) Error() string {
   return string(e)
}

// New creates interface values of type error.
func New(text string) error {
   return errorString(text)
}

var ErrNamedType = New("EOF")
var ErrStructType = errors.New("EOF")

func main() {
    if ErrNamedType == New("EOF") {
        fmt.Println("Named Type Error")
     }

    if ErrStructType == errors.New("EOF") {
        fmt.Println("Struct Type Error")
    }
}
```

在对比两个struct 是否相同的时候，会去对比，这两个 struct 里面的各个字段是否是相同的，如果相同就返回true,但是对比指针的时候回去判断两个指针的地址是否一致。

### Go 的异常处理

Go的处理异常逻辑是不引入 exception,支持多参数返回。如果一个函数返回了 value,error, 你不能对 value 做任何的假设，必须先判定 error。唯一可以忽略掉 error 的是，如果你连 value 也不关心。

you only need to check the error value if you care about eh result. --Dave

Go 中有panic的机制。与其他语言不同，当我们抛出异常的时候，相当于把exception 扔给了调用者来处理。

Go panic 意味着 fatal error(就是挂了）。不能假设调用者来解决panic，意味着代码不能继续运行。

使用多个返回值和一个简单的约定，Go解决了让程序员知道什么时候出了问题，并为真正的异常情况保留了panic.

### Go Error 的一些思想

简单

考虑失败，而不是成功（Plan for failure,not success).

没有隐藏的控制流

完全交给你来控制 error

Error are values

## Error Type

### Sentinel Error

预定义的特定错误，我们叫为 sentinel error, 这个名字来源于计算机编程中使用一个特定值来表示不可能进行进一步处理的做法。

```
ErrSomething = errors.New("xxx")

if err == ErrSomething { ... }
```

Sentinel Error 的不足

1. Sentinel errors 成为你API 公共部分，增加了API的表面积。
2. Sentinel errors 在两个包之间创建了依赖

如：检查错误是否等于 io.EOF,代码必须导入 io包。

结论：所以尽可能避免 sentinel errors

### Error types

Error type 是实现了 error 接口的自定义类型。例如 MyError 类型记录了文件和行号以及展示发生了什么。

因为 MyError 是一个type，调用者可以使用断言转换成这个类型，来获取更多的上下文信息。

如下代码所示：

```
package main

import (
  "fmt"
)

type MyError struct {
    Msg string
    File string
    Line int
}

func (e *MyError) Error() string {
    return fmt.Sprintf("%s:%d:%s",e.File,e.Line,e.Msg)
}

func test() error {
    return &MyError{"Something happened", "server.go", 42}
}

func main() {
    err := test()

    switch err := err.(type) {
        case nil:
            // call succeeded,nothing to do
        case *MyError:
                fmt.Println("error occurred on line:", err.Line)
        default:
                // unkonwn error
    }
}
```

调用者要使用类型断言和类型 switch,就要让自定义的 error 变为public。这种模型会导致和调用者产生强耦合，从而导致API变得脆弱。

结论是：

尽量避免使用 error types,虽然错误类型比 sentinel errors 更好，因为它们可以捕获关于出错的更多上下文，但是 error type 共享 error values 需要相同的问题。

所以尽量避免错误类型，或者至少避免将它们成为公共API的一部分。

### Opaque errors

不透明错误处理，虽然您知道了发生了错误，但您没有能力看到错误的内部。作为调用者，关于操作的结果，您所知道的就是它起作用了，或者没有起作用（成功还是失败）。

如

```
import "github.com/quux/bar"

func fn() error {
     x,err := bar.Foo()
    
     if err != nil {
            return err
      } 
      // use x    
}
```

Assert errors for behaviour,not ype

二分错误处理方法是不够的。调用方需要调查错误的性质，以确定重试该操作是否合理。在这种情况下，我们可以断言错误实现了特定的行为，而不是断言错误是特定的类型或值。

```
package net

type Error interface {
     error
     Timeout() bool // Is the error a timeout?
     Temporary() bool // Is the error temporary? 
}

type temporary interface {
    Temporary() bool
}

func IsTemporary(err error) bool {
     te,ok := err.(temporary)
     return ok && te.Temporary()
}

if nerr,ok := err.(net.Error); ok && nerr.Temporary() {
     time.Sleep(1e9)
     continue
}

if err != nil {
  log.Fatal(err)
}
```

关键逻辑是可以在不导入定义错误的包或实际上不了解err的底层类型的情况下实现 --- 我们只对它的行为感兴趣。

## Handling Error

### Indented flow is for errors

无错误的正常错误代码，将成为一条直线，而不是缩进的代码.fail fast

第一种会更好

```
f,err := os.Open(path)
if err != nil {
  // handle error
}
// do stuff
f,errr := os.Open(path)
if err == nil {
    // do stuff
}
// handle error
```

### Eliminate error handling by eliminating errors

#### 如果错误类型相同直接返回

```
func AuthenticateRequest(r *Request) error {
     err := authenticate(r.User)
     if err != nil {
         return err
     }
    return nil
}
```

上面这种可以直接返回

```
func AuthenticateRequest(r *Request) error {
     return authenticate(r.User)
}
```

#### 单独一个方法去判断是否有错误产生

```
func CountLines(r io.Reader) (int, error) {
     var (
            br = bufio.NewReader(r)
            line int
            err error
      )  

      for {
            _, err = br.ReadString('\n')
           lines++
            if err != nil {
                break
           }
       }

     if err != io.EOF {
          return 0,err
      }

      return line,nil
}
```

使用 sc.Scan() 方法，最后判断 sc.Err()

```
func CountLines(r io.Reader) (int,error) {
      sc := bufio.NewScanner(r)
     
      line := 0
     
      for sc.Scan()  {
          lines++
      }

      return lines,sc.Err()
}
```

### 定义结构体，方法中去判断是否有错误

```
type Header struct {
    Key,Value string
}

type Status struct {
    Code int
    Reason string
}

func WriteResponse(w io.Writer,st Status,headers []Header, body io.Reader) error {
      _,err := fmt.Fprint(w, "HTTP/1.1 %d %s\r\n", st.Code,st.Reason)
     if err != nil {
          return err
     }

     for _, h := range headers {
          _, err := fmt.Fprint(w, "%s: %s\r\n",h.Key,h.Value)
          if err != nil {
                  return err
           }
     }

     if _,err := fmt.Fprint(w, "\r\n"); err != nil {
          return err
     }
     
     _,err = io.Copy(w, body)
     return err
}
```

添加结构体定义，在Write 里面处理 err

```
type errWriter struct {
     io.Writer
     err error
}

func (e *errWriter) Write(buf []byte) (int, error) {
    if e.err != nil {
           return 0,e.err
     }
    
     var n int
      n,e.err = e.Writer.Write(buf)
      return n,nil
}

func WriteResponse(w io.Writer, st Status, headers []Header, body io.Reader) error {
        ew := &errWriter{Writer: w}
  
        fmt.Fprintf(ew, "HTTP/1.1 %d %s\r\n", st.Code,st.Reason)
     
        for _,h := range headers {
              fmt.Fprintf(ew, "%s: %s\r\n", h.Key,h.Value)
        }

        fmt.Fprint(ew, "\r\n")
        io.Copy(ew, body)

        return ew.err
}
```

### Wrap errors

```
func AuthenticateRequest(r *Request) error {
    return authenticate(r.User)
}
```

如果 authenticate 返回错误,则 AuthenticateRequest 会将错误返回给调用方，调用者可能也会这样做，依次类推。在程序的顶部，程序的主体将把错误打印到屏幕或日志文件中，打印出来的只是: 没有这样的文件或目录。

修改一下

```
func AuthenticateRequest(r *Request) error {
     err := authenticate(r.User)
     if err != nil {
            return fmt.Errorf("authenticate failed: %v", err)
     }
     return nil
}
```

没有生成错误的 file:line 信息，没有导致错误的调用堆栈的堆栈跟踪。

这种模式与 sentinel errors 或 type assertions 的使用不兼容，因为将错误值转换为字符串，将其与另一个字符串合并，然后将其转换回 fmt.ErrorF 破坏了 原始错误，导致等值判定失败。

#### 只处理一次错误，不要到处抛

you should only handle errors once. Handling an error means inspecting the error value, and making a single decision.

我们经常发现类似的代码，在错误处理中，带了两个任务：记录日志并且再次返回错误。

```
func WriteAll(w io.Writer,buf []byte) error {
     _,err := w.Write(buf)
     
     if err != nil {
          log.Println("unable to write:", err)  // annotated error goes to log file
          return err                                  // unannotated error returned to caller
     }

    return nil
}
```

#### 日志是否记录与错误处理的关系

日志记录与错误无关且对调试没有帮助的信息应被视为噪音，应予以质疑。记录的原因是因为某些东西失败了，而日志包含了答案。

The error has been logged.

错误要被日志记录。

The application is back to 100% integrity.

应用程序处理错误，保证100%完整性。

The current error is not reported any longer.

之后不再报告当前错误。

如果吞掉错误，必须对value 负起责任

1. 返回默认值
2. 返回降级数据信息

#### Wrap errors

通过使用 pkg/errors 包，您可以想错误值添加上下文，这种方式既可以由人也可以由机器检查

Wrap error 原理

Wrap -> err 熟悉->
WithMessage {
cause: 上层 err
}
-> withStack {
cause: 上层 withMessage
}

使用 errors.Cause 获取 root error,再进行和 sentinel error 判定

总结：

Packages that are reusable across many projucts only return root error values

选择 wrap error 是只有 applications 可以选择应用的策略。具有最高可重用性的包只能返回根错误值。此机制与Go标准库中使用的相同（kit 库的sql.ErrNoRows).

If the error is not going to be handled,wrap and return up the call stack.

如果函数/方法不打算处理错误，那么用足够的上下文 wrap errors 并将其返回到调用堆栈中。例如，额外的上下文可以是使用的输入参数或失败的查询语句。记录的上下文足够多还是太多的一个好方法是检查日志并验证它们在开发期间是否为您工作。

Once an error is handled,it is not allowed to be passed up the call stack any longer.

一旦确定函数/方法将处理错误，错误就不再是错误。如果函数/方法仍然需要发出返回，则它不能返回错误值。它应该只返回零（比如降级处理中，你返回了降级数据，然后需要 return nil).

## 参考文章

Errors are values : https://blog.golang.org/errors-are-values

Error handling and Go: https://blog.golang.org/error-handling-and-go

Go错误处理最佳实践: https://lailin.xyz/post/go-training-03.html