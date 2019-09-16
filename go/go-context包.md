# go context使用

### Context简单使用:

使用规则：

1，不要将Context放入结构体，Context应该作为第一个参数传入，命名为ctx。

2，即使函数允许，也不要传入nil得Context，如果不知道使用哪种Context，可以使用context.TODO()。

3，使用Context得Value相关方法，只应该用于程序和接口中传递请求相关数据，不能用它来传递一个可选参数

4，相同得Context可以传递给不同得goroutine；Context时并发安全得。

我们测试一个Context对与并发情况下得关闭情况

```go
package main

import (
   "context"
   "fmt"
   "net/http"
   _ "net/http/pprof"
   "time"
)

func main() {
   go http.ListenAndServe(":8080", nil)
   ctx, _ := context.WithTimeout(context.Background(), (10 * time.Second))
   go testA(ctx)
   select {}
}
// 测试，如果TestB 没有没有在超时时关闭，那么A就会接受到B通过管道传递回来得数据，
// 测试证明，B关闭后，A也关闭，没有发生数据传递，说明Context正常关闭了两个协程
func testA(ctx context.Context) {
   ctxA, _ := context.WithTimeout(ctx, (5 * time.Second))
   ch := make(chan int)
   go testB(ctxA, ch)

   select {
   case <-ctx.Done():
      fmt.Println("testA Done")
      return
   case i := <-ch:
      fmt.Println(i)
   }

}

func testB(ctx context.Context, ch chan int) {
   //模拟读取数据
   sumCh := make(chan int)
   go func(sumCh chan int) {
      sum := 10
      time.Sleep(10 * time.Second)
      sumCh <- sum
   }(sumCh)

   select {
   case <-ctx.Done():
      fmt.Println("testB Done")
      <-sumCh
      return
      //case ch  <- <-sumCh: 注意这样会导致资源泄露
   case i := <-sumCh:
      fmt.Println("send", i)
      ch <- i
   }

}
```

### Context结构

```go
type Context interface{
    Done() <- chan struct{}
    Err() error
    Deadline()(deadline time.Time,ok bool)
    Value(key interface{}) interface{}
}
```

**Done**方法在Context被取消活超时时返回一个close得channel，close得channel可以作为挂广播通知告诉给context相关得所有函数要停止当前工作然后返回。

**Err**方法返回context为什么被取消。

**Deadline**返回context何时会超时。

**Value**返回context相关的数据。

### 继承的Context

**BackGround**

 是所有Context的root，不能被cancel

**WithCancel**

返回一个继承的Cintext，这个Context在父Context的Done被关闭时关闭自己的Done通道，或者在自己被cancel的时候关闭自己的Done。

WithCancel同时还返回一个取消函数cancel，这个cancel用于取消当前的Context。

同时关闭自身不会影响父Context

### withDeadline withTimeout

WithTimeout 等价于 WithDeadline(parent, time.Now().Add(timeout)).

### WithValue

func WithValue(parent Context, key interface{}, val interface{}) Context

调用CancelFunc会取消child以及child生成的context，取出父context对这个child的引用，停止相关的计数器。