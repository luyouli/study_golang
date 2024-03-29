在go的http包的server中,每一个请求都有一个对应的goroutine取处理请求,请求处理函数通常会启动额外的goroutine用来访问后端服务,比如数据库和RPC服务,同来处理一个请求的goroutine通常需要访问一些与请求特定的数据,不如终端用户的身份认证信息、验证相同的token、请求的截止时间等,当一个请求被取消或者超时的时候,所用来处理这个请求的goroutine都应该迅速退出,然后系统才能释放这些goroutine占用的资源.

### 基本的例子
```go
package main

import (
	"fmt"
	"sync"
	"time"
)

var wg sync.WaitGroup

func f1() {
	defer wg.Done()
	for {
		fmt.Println("worker")
		time.Sleep(time.Second)
	}
}

func main() {
	wg.Add(1)
	go f1()
	wg.Wait()
	fmt.Println("over")
}
```

上面的程序是会一直执行的,因为子 goroutine 是循环执行的,主 goroutine 是处于阻塞的状态,那么怎么优雅的退出子 goroutine 呢?

#### 方式 1: 使用全局变量的方式
```go
package main

import (
	"fmt"
	"sync"
	"time"
)

var wg sync.WaitGroup
var Isexit bool

func f1() {
	defer wg.Done()
	for {
		fmt.Println("worker")
		time.Sleep(time.Second)
		if Isexit {
			break
		}
	}
}

func main() {
	wg.Add(1)
	go f1()
	time.Sleep(3 * time.Second)
	Isexit = true
	wg.Wait()
	fmt.Println("over")
}
```

#### 方式 2: 通道的方式
```go
package main

import (
	"fmt"
	"runtime"
	"sync"
	"time"
)

var wg sync.WaitGroup
var Isexit = make(chan bool)

func f1() {
	defer wg.Done()

	for {
		fmt.Println("worker")
		time.Sleep(time.Second)
		select {
		case <-Isexit:
			runtime.Goexit()
		default:
		}
	}
}

func main() {
	wg.Add(1)
	go f1()
	time.Sleep(3 * time.Second)
	Isexit <- true
	wg.Wait()
	fmt.Println("over")
}
```

#### 方式 3:使用context
```go
package main

import (
	"context"
	"fmt"
	"runtime"
	"sync"
	"time"
)

var wg sync.WaitGroup

func f1(ctx context.Context) {
	defer wg.Done()

	for {
		fmt.Println("worker")
		time.Sleep(time.Second)
		select {
		case <-ctx.Done():
			runtime.Goexit()
		default:
		}
	}
}

func main() {
	ctx, cancel := context.WithCancel(context.Background())
	wg.Add(1)
	go f1(ctx)
	time.Sleep(3 * time.Second)
	cancel() // 通知子goroutine结束
	wg.Wait()
	fmt.Println("over")
}
```

如果有多个子 goroutine 的话,同样只需要将 ctx 传入即可
```go
package main

import (
	"context"
	"fmt"
	"sync"
	"time"
)

var wg sync.WaitGroup

func f1(ctx context.Context) {
	defer wg.Done()
	go f2(ctx)
LOOP:
	for {
		fmt.Println("worker")
		time.Sleep(time.Second)
		select {
		case <-ctx.Done():
			break LOOP
		default:
		}
	}
}
func f2(ctx context.Context) {
	defer wg.Done()
LOOP:
	for {
		fmt.Println("worker2")
		time.Sleep(time.Second)
		select {
		case <-ctx.Done():
			break LOOP
		default:
		}
	}
}

func main() {
	ctx, cancel := context.WithCancel(context.Background())
	wg.Add(1)
	go f1(ctx)
	time.Sleep(3 * time.Second)
	cancel() // 通知子goroutine结束
	wg.Wait()
	fmt.Println("over")
}
```

## Context
go 1.7 之后加入了一个新的标准库`context`,它定义了`context`类型,专门用来简化对于处理单个请求的多个 goroutine 之间与请求域的数据、取消信号、截止时间等相关操作，这些操作可能涉及多个 API 调用。

对服务器传入的请求应该创建上下文，而对服务器的传出调用应该接受上下文。它们之间的函数调用链必须传递上下文，或者可以使用`WithCancel`、`WithDeadline`、`WithTimeout`或`WithValue`创建的派生上下文。当一个上下文被取消时，它派生的所有上下文也被取消。

## Context接口

`context.Context`是一个接口，该接口定义了四个需要实现的方法。具体签名如下：
```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key interface{}) interface{}
}
```
其中：

- Deadline():方法需要返回的是当前 Context 被取消的时间,也就是完成工作的截止时间(deadline;)
- Done 方法需要返回一个 channel,这个 channel 会在当前工作完成或者上下文被取消之后关闭,多次调用 Done 方法会返回同一个 channel
- Err 方法会返回当前 Context 技术的原因,它只会在 Done 返回的 channel 被关闭时才会返回非空的值
  - 如果当前 Context 被取消就会返回 Canceled 错误
  - 如果当前 Context 查实就会返回 DeadlineExceeded 错误

- Value 方法会从 Context 中返回键对应的值,对于同一个上下文来说,多次调用 Value 并传入相同的 Key 会返回相同的记过,该方法仅用于传递 API 和进程间的请求的数据

### Background()和 TODO()

go 内置了两个函数: `Background()`和`TODO()`,这两个函数分别返回一个实现了`Context`接口的`Background()`和`TODO()`,我们代码中最开始都是以这两个内置的上下文对象作为最顶层的`partent context`，衍生出更多的子上下文对象。

`Background()`主要用于 main 函数、初始化以及测试代码中,作为Context这个树戒恶狗最顶层的Context,也就是根Context

`TOCO()`它目前还不知道具体的使用场景，如果我们不知道该使用什么Context的时候，可以使用这个。

`background`和`todo`本质上都是`emptyCtx`结构体类型，是一个不可取消，没有设置截止时间，没有携带任何值的Context。

## With系列函数

`context`包中还定义了四个带with系列的函数


1.  WithCancel
```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	c := newCancelCtx(parent)
	propagateCancel(parent, &c)
	return &c, func() { c.cancel(true, Canceled) }
}
```

WithCancel 返回带有新 Done 通道的父节点的副本,当调用返回的 cancel 函数或当关闭父上下文的 Done 通道的时候,将关闭返回上下文的 Done 通道,无论先发生什么情况

取消此上下文将释放与其管理的资源,因此代码应该在此上下文运行的操作完成后立即调用 cancel
```go
func gen(ctx context.Context) <-chan int {
		dst := make(chan int)
		n := 1
		go func() {
			for {
				select {
				case <-ctx.Done():
					return // return结束该goroutine，防止泄露
				case dst <- n:
					n++
				}
			}
		}()
		return dst
	}
func main() {
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel() // 当我们取完需要的整数后调用cancel

	for n := range gen(ctx) {
		fmt.Println(n)
		if n == 5 {
			break
		}
	}
}
```

上面的代码中,`gen`函数在单独的 goroutine 中生成整数,并将他们发送到返回的通道中,gen 的调用者使用生成的整数之后调用 cancel()函数取消上线文,以免 gen 函数的内部 goroutine 发生泄露

2. WithDeadline()
```go
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc)
```
返回的是父上下文的副本,也就是子节点,并将 deadline 调整为设置的时间,有两种情况,就是当父上下文的 deadline 时间早于设置的时间的时候,也就是在设置的时间之前执行的 cancle 函数,或者晚于设置的时间,无论是哪一种情况,都以县发生的为准,都会关闭父上下文的Done 通道,以及子上下文的 Done 通道

> 取消此上下文将释放与其向关联的资源,因此代码应该在此上下文中运行的操作完成之后立即调用 cancel

```go
func main() {
	d := time.Now().Add(50 * time.Millisecond)
	ctx, cancel := context.WithDeadline(context.Background(), d)

	// 尽管ctx会过期，但在任何情况下调用它的cancel函数都是很好的实践。
	// 如果不这样做，可能会使上下文及其父类存活的时间超过必要的时间。
    // 也就是说如果不调用 cancel,那么上下文关闭后将会等待回收机制去释放资源,但是调用了之后,会立即释放资源
	defer cancel()

	select {
	case <-time.After(1 * time.Second):
		fmt.Println("overslept")
	case <-ctx.Done():
		fmt.Println(ctx.Err())
	}
}
```

上面的代码中，定义了一个50毫秒之后过期的deadline，然后我们调用`context.WithDeadline(context.Background(), d)`得到一个上下文（ctx）和一个取消函数（cancel），然后使用一个select让主程序陷入等待：等待1秒后打印`overslept`退出或者等待ctx过期后退出。 因为ctx50秒后就过期，所以`ctx.Done()`会先接收到值，上面的代码会打印ctx.Err()取消原因。

### WithTimeout

`WithTimeout`的函数签名如下：

```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
```

`WithTimeout`返回`WithDeadline(parent, time.Now().Add(timeout))`。

取消此上下文将释放与其相关的资源，因此代码应该在此上下文中运行的操作完成后立即调用cancel，通常用于数据库或者网络连接的超时控制。具体示例如下：

```go
package main

import (
	"context"
	"fmt"
	"sync"

	"time"
)

// context.WithTimeout

var wg sync.WaitGroup

func worker(ctx context.Context) {
LOOP:
	for {
		fmt.Println("db connecting ...")
		time.Sleep(time.Millisecond * 10) // 假设正常连接数据库耗时10毫秒
		select {
		case <-ctx.Done(): // 50毫秒后自动调用
			break LOOP
		default:
		}
	}
	fmt.Println("worker done!")
	wg.Done()
}

func main() {
	// 设置一个50毫秒的超时
	ctx, cancel := context.WithTimeout(context.Background(), time.Millisecond*50)
	wg.Add(1)
	go worker(ctx)
	time.Sleep(time.Second * 5)
	cancel() // 通知子goroutine结束
	wg.Wait()
	fmt.Println("over")
}
```

### WithValue

`WithValue`函数能够将请求作用域的数据与 Context 对象建立关系。声明如下：

```go
func WithValue(parent Context, key, val interface{}) Context
```

`WithValue`返回父节点的副本，其中与key关联的值为val。

仅对API和进程间传递请求域的数据使用上下文值，而不是使用它来传递可选参数给函数。

所提供的键必须是可比较的，并且不应该是`string`类型或任何其他内置类型，以避免使用上下文在包之间发生冲突。`WithValue`的用户应该为键定义自己的类型。为了避免在分配给interface{}时进行分配，上下文键通常具有具体类型`struct{}`。或者，导出的上下文关键变量的静态类型应该是指针或接口。

```go
package main

import (
	"context"
	"fmt"
	"sync"

	"time"
)

// context.WithValue

type TraceCode string

var wg sync.WaitGroup

func worker(ctx context.Context) {
	key := TraceCode("TRACE_CODE")
	traceCode, ok := ctx.Value(key).(string) // 在子goroutine中获取trace code
	if !ok {
		fmt.Println("invalid trace code")
	}
LOOP:
	for {
		fmt.Printf("worker, trace code:%s\n", traceCode)
		time.Sleep(time.Millisecond * 10) // 假设正常连接数据库耗时10毫秒
		select {
		case <-ctx.Done(): // 50毫秒后自动调用
			break LOOP
		default:
		}
	}
	fmt.Println("worker done!")
	wg.Done()
}

func main() {
	// 设置一个50毫秒的超时
	ctx, cancel := context.WithTimeout(context.Background(), time.Millisecond*50)
	// 在系统的入口中设置trace code传递给后续启动的goroutine实现日志数据聚合
	ctx = context.WithValue(ctx, TraceCode("TRACE_CODE"), "12512312234")
	wg.Add(1)
	go worker(ctx)
	time.Sleep(time.Second * 5)
	cancel() // 通知子goroutine结束
	wg.Wait()
	fmt.Println("over")
}
```

## 使用Context的注意事项

*   推荐以参数的方式显示传递Context
*   以Context作为参数的函数方法，应该把Context作为第一个参数。
*   给一个函数方法传递Context的时候，不要传递nil，如果不知道传递什么，就使用context.TODO()
*   Context的Value相关方法应该传递请求域的必要数据，不应该用于传递可选参数
*   Context是线程安全的，可以放心的在多个goroutine中传递

## 客户端超时取消示例

调用服务端API时如何在客户端实现超时控制？

### server端

```go
// context_timeout/server/main.go
package main

import (
	"fmt"
	"math/rand"
	"net/http"

	"time"
)

// server端，随机出现慢响应

func indexHandler(w http.ResponseWriter, r *http.Request) {
	number := rand.Intn(2)
	if number == 0 {
		time.Sleep(time.Second * 10) // 耗时10秒的慢响应
		fmt.Fprintf(w, "slow response")
		return
	}
	fmt.Fprint(w, "quick response")
}

func main() {
	http.HandleFunc("/", indexHandler)
	err := http.ListenAndServe(":8000", nil)
	if err != nil {
		panic(err)
	}
}
```

### client端

```go
// context_timeout/client/main.go
package main

import (
	"context"
	"fmt"
	"io/ioutil"
	"net/http"
	"sync"
	"time"
)

// 客户端

type respData struct {
	resp *http.Response
	err  error
}

func doCall(ctx context.Context) {
	transport := http.Transport{
	   // 请求频繁可定义全局的client对象并启用长链接
	   // 请求不频繁使用短链接
	   DisableKeepAlives: true, 	}
	client := http.Client{
		Transport: &transport,
	}

	respChan := make(chan *respData, 1)
	req, err := http.NewRequest("GET", "http://127.0.0.1:8000/", nil)
	if err != nil {
		fmt.Printf("new requestg failed, err:%v\n", err)
		return
	}
	req = req.WithContext(ctx) // 使用带超时的ctx创建一个新的client request
	var wg sync.WaitGroup
	wg.Add(1)
	defer wg.Wait()
	go func() {
		resp, err := client.Do(req)
		fmt.Printf("client.do resp:%v, err:%v\n", resp, err)
		rd := &respData{
			resp: resp,
			err:  err,
		}
		respChan <- rd
		wg.Done()
	}()

	select {
	case <-ctx.Done():
		//transport.CancelRequest(req)
		fmt.Println("call api timeout")
	case result := <-respChan:
		fmt.Println("call server api success")
		if result.err != nil {
			fmt.Printf("call server api failed, err:%v\n", result.err)
			return
		}
		defer result.resp.Body.Close()
		data, _ := ioutil.ReadAll(result.resp.Body)
		fmt.Printf("resp:%v\n", string(data))
	}
}

func main() {
	// 定义一个100毫秒的超时
	ctx, cancel := context.WithTimeout(context.Background(), time.Millisecond*100)
	defer cancel() // 调用cancel释放子goroutine资源
	doCall(ctx)
}
```