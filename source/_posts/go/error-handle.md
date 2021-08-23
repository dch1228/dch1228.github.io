---
title: Go 错误处理
date: 2021-08-18
tags: go
---

# 常见姿势

## Sentinel error

哨兵模式，在包中预定义一些错误变量，调用者通过导入这些变量进行等值比较。

例如 io 库中的 EOF

```go
var EOF = errors.New("EOF")
```

在外部通过 == 来判断

```go
if err == io.EOF {
	// ...
}
```

这种方式的问题在于把 error 当作 API 暴露给第三方，导致 API 变得脆弱。不利于后期重构，而且这种方式能提供的信息十分有限。

## Error types

实现了 `error` 接口的自定义类型。

例如 os 库中的 PathError

```go
type PathError struct {
	Op   string
	Path string
	Err  error
}

func (e *PathError) Error() string { return e.Op + " " + e.Path + ": " + e.Err.Error() }
```

在外部通过类型断言来判断

```go
switch err := err.(type) {
case nil:
	// 没错误
case *os.PathError:
	// ...
default:
	// 其他错误
}
```

这种方式虽然提供了更多的上下文信息，但同样需要暴露给外部，并且可能创建很多 error values 相同的错误。error 值的判定需要使用类型断言，写起来比较麻烦。

## Opaque errors

比较灵活的错误处理策略，他要求代码和调用者之间的耦合最少。调用者只知道发生了错误，但是看不到错误的内部，调用者只关心有没有成功。

这就是不透明错误处理的全部功能，只需返回错误而不假设其内容。

```go
val, err := bar.Foo()
if err != nil {
	return err
}
// use val
```

在极少数情况，调用方需要调查错误的性质，以确定重试机制是否合理。在这种情况下，我们可以断言错误实现了特定的行为，而不是断言错误是特定的类型或值。

例如 net 库中的 Error

```go
// net 库
type Error interface {
	error
	Timeout() bool   // Is the error a timeout?
	Temporary() bool // Is the error temporary?
}

// 调用方
if nerr, ok := err.(net.Error); ok {
	if nerr.Temporary() {
		// 重试逻辑
	}
}
if err != nil {
	// 处理其他错误
}
```

# 错误处理的代码优化

go 中经常包含大量 `if err` 代码，有一些方法可以减少这些代码。

## AuthenticateRequest

在我们的项目中经常看到类似的代码

```go
func AuthenticateRequest(r *Request) error {
    err := authenticate(r.User)
    if err != nil {
        return err
    }
    return nil
}
```

这里判断了 `err != nil` 但没有进一步的处理，直接返回可以少写很多代码

```go
func AuthenticateRequest(r *Request) error {
    return authenticate(r.User)
}
```

## CounterLines

统计 `io.Reader` 读取内容的行数

```go
func CounterLines(r io.Reader) (int, error) {
	var (
		br    = bufio.NewReader(r)
		lines int
		err   error
	)

	for {
		_, err = br.ReadString('\n')
		lines++
		if err != nil {
			break
		}
	}

	if err != io.EOF {
		return 0, err

	}
	return lines, nil
}
```

改进版本，消除了所有 `if err` 判断，因为 `sc.Scan` 做了很多处理。很多类似的场景都可以通过这种包装处理，这样外部包调用的代码就会很整洁。

```go
func CounterLines(r io.Reader) (int, error) {
	sc := bufio.NewScanner(r)
	lines := 0
	for sc.Scan() {
		lines++
	}
	return lines, sc.Err()
}
```

## WriteResponse

构建 HTTP/1.1 响应

```go
type Header struct {
	Key, Value string
}

type Status struct {
	Code   int
	Reason string
}

func WriteResponse(w io.Writer, st Status, headers []Header, body io.Reader) error {
	_, err := fmt.Fprintf(w, "HTTP/1.1 %d %s\r\n", st.Code, st.Reason)
	if err != nil {
		return err
	}

	for _, header := range headers {
		_, err := fmt.Fprintf(w, "%s: %s\r\n", header.Key, header.Value)
		if err != nil {
			return err
		}
	}

	if _, err := fmt.Fprint(w, "\r\n"); err != nil {
		return err
	}

	_, err = io.Copy(w, body)
	return err
}
```

改进版本，`errWriter` 包装了 `io.Writer` 的 `Write` 方法。他将写操作传递给基础的 `Write` 方法直到发生错误，从那时起，他丢弃所有的写操作并返回上一个错误。

在 `WriteResponse` 中引入 `errWriter` 显著提升了代码的清晰度。

```go
type errWriter struct {
	io.Writer
	err error
}

func (e *errWriter) Write(buf []byte) (int, error) {
	if e.err != nil {
		return 0, e.err
	}

	var n int
	n, e.err = e.Writer.Write(buf)
	return n, nil
}

func WriteResponse(w io.Writer, st Status, headers []Header, body io.Reader) error {
	ew := &errWriter{Writer: w}

	fmt.Fprintf(ew, "HTTP/1.1 %d %s\r\n", st.Code, st.Reason)

	for _, header := range headers {
		fmt.Fprintf(ew, "%s: %s\r\n", header.Key, header.Value)
	}

	fmt.Fprint(w, "\r\n")

	io.Copy(w, body)

	return ew.err
}
```

## 错误只应该处理一次

这个例子中包含两个问题
- 在 JSON 序列化失败后，检查并记录了错误，但是没有 return，这将把损坏的缓冲区传递给 WriteAll，这可能成功，因此配置文件中写入了错误的内容，但是该函数并没有返回错误。
- 在 WriteAll 过程中发生了一个错误，那么一行代码将被写入日志文件中，记录错误发生的文件和行，并且错误也会返回给调用者，调用者可能会记录并返回它，一直返回到程序的顶部。错误被处理了多次。
- 日志记录与错误无关且对调试没有帮助的信息应被视为噪音，应予以质疑。记录的原因是因为某些东西失败了，而日志包含了答案。

```go
func WriteConfig(w io.Writer, conf *Conf) error {
	buf, err := json.Marshal(conf)
	if err != nil {
		log.Printf("could not marshal config: %v", err)
		// oops, forgot to return
	}

	if err := WriteAll(w, buf); err != nil {
		log.Printf("could not write config: %v", err)
		return err
	}
	return nil
}

func main() {
	err := WriteConfig(f, &conf)
	log.Printf("could not write config: %v", err)
}
```

# 错误包装

在之前 `AuthenticateRequest` 的例子中，我们透传了 error 给调用方，调用方可能也会这么做，依此类推。在程序的顶部将错误打印到屏幕或日志文件中，打印出来的只是：没有这样的文件或目录。

没有生成错误的 `file:line` 信息，没有导致错误的调用堆栈跟踪，给错误排查造成了很大的困难。

使用 [github.com/pkg/errors](https://github.com/pkg/errors) 可以包装错误。

例子，在 dao 层遇到错误该如何处理

```go
package main

import (
	"database/sql"
	"fmt"

	"github.com/gin-gonic/gin"
	"github.com/pkg/errors"
)

type Todo struct {
	Id    string
	Title string
}

// dao 层
func daoGetTodoById(id string) (out *Todo, err error) {
	// 根据id模拟一些错误
	switch id {
	case "0":
		err = sql.ErrNoRows
	case "1":
		err = sql.ErrConnDone
	default:
		out = &Todo{
			Id:    id,
			Title: "",
		}
	}

	if err != nil {
		// wrap err
		return nil, errors.Wrapf(err, "daoGetTodoByIdErr, id: %s", id)
	}

	return out, nil
}

// service 层
func serviceGetTodoById(id string) (*Todo, error) {
	todo, err := daoGetTodoById(id)
	if err != nil {
		return nil, err
	}
	return todo, nil
}

// handler 层
func handlerGetTodoById(ctx *gin.Context) {
	id := ctx.Param("id")

	todo, err := serviceGetTodoById(id)
	if err != nil {
		if errors.Is(err, sql.ErrNoRows) {
			ctx.JSON(404, gin.H{
				"code": 404,
				"msg":  "todo not found",
			})
		} else {
			// 其他错误需要记录日志
			fmt.Printf("%+v\n", err)
			ctx.JSON(500, gin.H{
				"code": 500,
				"msg":  "internal error",
			})
		}
		return
	}
	ctx.JSON(200, todo)
}

func main() {
	r := gin.New()

	r.GET("/todo/:id", handlerGetTodoById)

	_ = r.Run(":8000")
}
```

# References

[Eliminate error handling by eliminating errors
](https://dave.cheney.net/2019/01/27/eliminate-error-handling-by-eliminating-errors)