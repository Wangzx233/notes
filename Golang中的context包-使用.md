# Golang中的context包-使用

![10614156b3849a7872f31d6da754f51f.png](https://img-blog.csdnimg.cn/img_convert/10614156b3849a7872f31d6da754f51f.png)



## 创建

### 根节点创建：

根节点不具备任何功能，具体功能需要搭配with系列函数。

context的根节点创建分为两种：

```go
context.Backgroud()
```

```
context.TODO()
```

官方解释：

```
// Background returns a non-nil, empty Context. It is never canceled, has no
// values, and has no deadline. It is typically used by the main function,
// initialization, and tests, and as the top-level Context for incoming
// requests.

// TODO returns a non-nil, empty Context. Code should use context.TODO when
// it's unclear which Context to use or it is not yet available (because the
// surrounding function has not yet been extended to accept a Context
// parameter).
```

`Background`通常被用于主函数、初始化以及测试中，作为一个顶层的`context`，也就是说一般我们创建的`context`都是基于`Background`；而`TODO`是在不确定使用什么`context`的时候才会使用。

所以一般只在不知道使用什么`context`时使用`TODO`。正如其名“todo”。其他一般情况均用`Background`。

### with函数

```go
func WithValue(parent Context, key, val interface{}) Context
```

创建一个可携带值的`context`	，值的存储与获取采用键值关系储存，类比于map，即一个`key`对应一个`value`。

value存入和取出都为`interface`类型，注意断言正确。

简单示例：

```go
ctx := context.WithValue(context.Background(), KEY, "hello")

fmt.Println(ctx.Value(KEY).(string))
```

------

![img](https://pic3.zhimg.com/80/v2-dcdc25c62efcb902015058e1bebb8cde_1440w.jpg)

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
```

创建一个可取消的context

直接看一个示例：

```go
func speak(ctx context.Context) {
	for range time.Tick(time.Second*1) {
		select {
		case <-ctx.Done():
			fmt.Println("stop")
			return
		default:
			fmt.Println("1")
		}
	}
}

func main() {
	ct, cancel := context.WithCancel(context.Background())

	go speak(ct)

	time.Sleep(time.Second*5)

	cancel()

	time.Sleep(time.Second*1)
}
```

输出结果：

```
1
1
1
1
1
stop
```



```go
func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
```

创建两种有时间期限的可取消的`context`与之前的`WithCancel`函数功能相同，不过多了`deadline`和`timeout`	，即到达指定时间自己取消或者经过指定时间自己取消。也可调用返回多`CancelFunc`手动取消。

我们将上方的示例改一下就能看出区别：

```go
func speak(ctx context.Context) {
	for range time.Tick(time.Second*1) {
		select {
		case <-ctx.Done():
			fmt.Println("stop")
			return
		default:
			fmt.Println("1")
		}
	}
}

func main() {
	ct, cancel := context.WithTimeout(context.Background(),time.Second*3)

	go speak(ct)

	time.Sleep(time.Second*5)

	cancel()

	time.Sleep(time.Second*1)
}
```

输出结果：

```
1
1
1
stop
```

建议即便设置了超时后也要手动取消一下，以免计时空转。

同时我们看一下context接口

```go
type Context interface {

    Deadline() (deadline time.Time, ok bool)

    Done() <-chan struct{}

    Err() error

    Value(key interface{}) interface{}
}
```

我们可以发现`emptyCtx`之所以要实现四个无用的空方法即是为了使它自己也成为`context`。这么解释可能有点点怪，但便于理解。

依次解释一下四个方法，`Deadline()`返回过期时间以及是否设置了deadline。

`Done()`判断当前context是否被取消。

`Err()`返回取消context的原因

`Value`返对应key值存储的值

使用案例：

[Kevin Yan：学会使用context取消goroutine执行的方法](https://zhuanlan.zhihu.com/p/136664236)

[Kevin Yan：并发问题的解决思路以及Go语言调度器工作原理](https://zhuanlan.zhihu.com/p/145882778)

参考：

https://blog.csdn.net/qq_39397165/article/details/121092507?spm=1001.2014.3001.5501

https://zhuanlan.zhihu.com/p/110085652
