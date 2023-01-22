```json
{
    "data":"2023.01.22 15.30",
    "author":"XinceChan",
    "tags":["Golang","Context"],
    "musicId":"65806"
}
```

Golang标准库。对于程序员而言，标准库与语言本身同样重要，它好比一个百宝箱，能为各种常见的任务提供完美的解决方案。虽然一直都有在使用标准库中的一些包，但是没有认真去了解过里面的内容。这次就借着机会好好了解一下 `context` 包。这里会附带一些我的翻译以及官方文档原文（因为我的翻译太烂）提供相应的参考。

## Context包

官方对Context的定义是这样的：

> Package context defines the Context type, which carries deadlines, cancellation signals, and other request-scoped values across API boundaries and between processes.

> Context包定义了Context类型，它携带截止日期、取消信号和其他跨 API 边界和进程之间的请求范围的值。

Context的源代码对其的描述如下，在 goroutine 的使用过程中十分安全：

> The same Context may be passed to functions running in different goroutines; Contexts are safe for simultaneous use by multiple goroutines.

```go
// A Context carries a deadline, cancellation signal, and request-scoped values
// across API boundaries. Its methods are safe for simultaneous use by multiple
// goroutines.
type Context interface {
    // Done returns a channel that is closed when this Context is canceled
    // or times out.
    Done() <-chan struct{}

    // Err indicates why this context was canceled, after the Done channel
    // is closed.
    Err() error

    // Deadline returns the time when this Context will be canceled, if any.
    Deadline() (deadline time.Time, ok bool)

    // Value returns the value associated with key or nil if none.
    Value(key interface{}) interface{}
}
```

> Incoming requests to a server should create a Context, and outgoing calls to servers should accept a Context. The chain of function calls between them must propagate the Context, optionally replacing it with a derived Context created using WithCancel, WithDeadline, WithTimeout, or WithValue. When a Context is canceled, all Contexts derived from it are also canceled.

> 对服务器的传入请求应该创建一个上下文，对服务器的传出调用应该接受一个上下文。函数调用链必须传递上下文信息。当一个上下文被取消时，从它派生的所有上下文也会被取消。

`Context` 可以通过 `WithCancel() - 携带取消信号` 、 `WithDeadline（）- 携带截止日期`、 `WithTimeout() - 超时设置`、 `WithValue() - 携带参数信息` 来做相应的逻辑控制。

- ####  WithCancel() - 携带取消信号: 

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
```

>WithCancel returns a copy of parent with a new Done channel. The returned context's Done channel is closed when the returned cancel function is called or when the parent context's Done channel is closed, whichever happens first.

通过 `context.WithCancel` 函数可以创建一个接收 cancel 信号的 Context 。通过 Context 的 `Done` 函数可以判断是否发出了cancel信号。父 Context  发出的 cancel 信号，子 Context 也可以接收到。

**注意** 取消此上下文会释放与其关联的资源，因此代码在此上下文中运行的操作完成后立即调用 `cancel()`

**example**

```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel() // cancel when we are finished consuming integers
```

- ####  WithDeadline（）- 携带截止日期:

```go
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc)
```

> WithDeadline returns a copy of the parent context with the deadline adjusted to be no later than d. If the parent's deadline is already earlier than d, WithDeadline(parent, d) is semantically equivalent to parent. The returned context's Done channel is closed when the deadline expires, when the returned cancel function is called, or when the parent context's Done channel is closed, whichever happens first.

**example**

```go
const shortDuration = 1 * time.Millisecond

func main() {
	d := time.Now().Add(shortDuration)
	ctx, cancel := context.WithDeadline(context.Background(), d)

	// Even though ctx will be expired, it is good practice to call its
	// cancellation function in any case. Failure to do so may keep the
	// context and its parent alive longer than necessary.
	defer cancel()

	select {
	case <-time.After(1 * time.Second):
		fmt.Println("overslept")
	case <-ctx.Done():
		fmt.Println(ctx.Err())
	}
}
```

**output**

```go
context deadline exceeded
```

- #### WithTimeout() - 超时设置

```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
```

`context.WithTimeout` 其实本质上是调用了 `context.WithDeadline(parent, time.Now().Add(timeout))`

**注意** 同样，取消此上下文会释放与其关联的资源，因此代码在此上下文中运行的操作完成后立即调用 `cancel()`

**example**

```go
func slowOperationWithTimeout(ctx context.Context) (Result, error) {
	ctx, cancel := context.WithTimeout(ctx, 100*time.Millisecond)
	defer cancel()  // releases resources if slowOperation completes before timeout elapses
	return slowOperation(ctx)
}
```

- #### WithValue() - 携带参数信息

```go
func WithValue(parent Context, key, val any) Context
```

通过 `context.WithValue` 函数可以给 Context 添加参数。其中 key 和 value 都是空接口类型(`interface{}`)。通过 Context 的 `Value` 函数可以获取附加参数信息。

> Use context Values only for request-scoped data that transits processes and APIs, not for passing optional parameters to functions.

只有在请求范围内的数据穿越进程和API时才使用上下文值。

**example**

```go
func main() {
	type favContextKey string

	f := func(ctx context.Context, k favContextKey) {
		if v := ctx.Value(k); v != nil {
			fmt.Println("found value:", v)
			return
		}
		fmt.Println("key not found:", k)
	}

	k := favContextKey("language")
	ctx := context.WithValue(context.Background(), k, "Go")

	f(ctx, k)
	f(ctx, favContextKey("color"))
    
}
```

**output**

```go
found value: Go
key not found: color
```

- #### Background()

```go
func Background() Context
```

来看一下源码

```go
// An emptyCtx is never canceled, has no values, and has no deadline. It is not
// struct{}, since vars of this type must have distinct addresses.
type emptyCtx int
var background = new(emptyCtx)

func Background() Context {
	return background
}
```

> Background returns a non-nil, empty Context. It is never canceled, has no values, and has no deadline. It is typically used by the main function, initialization, and tests, and as the top-level Context for incoming requests.

`context.Background()` 函数返回一个 Context。通常主函数、初始化和测试使用，并作为传入请求的顶级 Context。

- #### TODO()

```go
func TODO() Context
```

来看一下源码

```go
type emptyCtx int
var todo = new(emptyCtx)

func TODO() Context {
	return todo
}
```

> TODO returns a non-nil, empty Context. Code should use context.TODO when it's unclear which Context to use or it is not yet available (because the surrounding function has not yet been extended to accept a Context parameter).

> Do not pass a nil Context, even if a function permits it. Pass context.TODO if you are unsure about which Context to use.

当不清楚要使用哪个 Context 或者它还不可用时（因为周围的函数还没有被扩展到接受 Context 参数），代码应该使用 `context.TODO()`。

### Context使用流程

- Step1: 创建 Context，给 Context 指定超时时间，设置取消信息，或者附加参数信息（链路跟踪里经常使用Context里的附加参数，传递IP等链路跟踪信息）
- Step2: goroutine使用Step 1里的Context作为第一个参数，在该goroutine里就可以做如下事情：
  - 使用Context里的Done函数判断是否达到了Context设置的超时时间或者Context是否被主动取消了。
  - 使用Context里的Value函数获取该Context里的附加参数信息。
  - 使用Context里的Err函数获取错误原因，目前原因就2个，要么是超时，要么是主动取消。

Context 是可以组合使用的。比如，可以通过使用 `context.WithTimeout` 创建一个有超时时间的 Context，再调用 `context.WithValue` 添加一些附加信息。

多个goroutine可以共享同一个Context，可以通过该Context的超时设置、携带取消信号以及附加参数来控制多个goroutine的行为。



### 参考资料

- [Context包官方文档](https://pkg.go.dev/context)   https://pkg.go.dev/context
- [Context包源码](https://cs.opensource.google/go/go/+/refs/tags/go1.19.5:src/context/context.go)       https://cs.opensource.google/go/go/+/refs/tags/go1.19.5:src/context/context.go

- [并发编程中Context使用常见错误](https://blog.csdn.net/perfumekristy/article/details/126687935)  https://blog.csdn.net/perfumekristy/article/details/126687935