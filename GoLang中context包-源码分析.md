# GoLang中context包-源码分析

## 前言

在已知道context包如何使用的前提下解释



## 创建 context

创建根节点的context分为两种方法

```go
ctx, cancel := context.Background()
```

```go
ctx, cancel := context.TODO()
```

看一下源码

From https://golang.org/src/context/context.go:

```go
var (
	background = new(emptyCtx)
	todo       = new(emptyCtx)
)


func Background() Context {
	return background
}

func TODO() Context {
	return todo
}
```

发现`Background()`和`TODO()`都是用来生成同样的`emptyCtx`。那接下来就看`emptyCtx`

```go
type emptyCtx int

func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
	return
}

func (*emptyCtx) Done() <-chan struct{} {
	return nil
}

func (*emptyCtx) Err() error {
	return nil
}

func (*emptyCtx) Value(key interface{}) interface{} {
	return nil
}

func (e *emptyCtx) String() string {
	switch e {
	case background:
		return "context.Background"
	case todo:
		return "context.TODO"
	}
	return "unknown empty Context"
}
```

​	`emptyCtx`其实就是一个int类型，但没有实际用处。实现了四种方法。

前四种方法均直接返回空，即像之前使用说的那样，根`context`没有具体作用，仅作为树的根节点。

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



### WithCancel

```go
type CancelFunc func()


func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	c := newCancelCtx(parent)
	propagateCancel(parent, &c)
	return &c, func() { c.cancel(true, Canceled) }
}

func newCancelCtx(parent Context) cancelCtx {
	return cancelCtx{Context: parent}
}
```

首先判断是否有父节点，没有就无法创建，即cancelCtx无法作为根节点。

然后新建一个cancelCtx结构体

#### newCancelCtx

```go
type cancelCtx struct {
	Context

	mu       sync.Mutex            // protects following fields
	done     atomic.Value          // of chan struct{}, created lazily, closed by first cancel call
	children map[canceler]struct{} // set to nil by the first cancel call
	err      error                 // set to non-nil by the first cancel call
}
```

结构体中嵌套了一个context用来指向父节点。mu加了个锁来确保线程安全。

`atomic.Value `的结构为

```go
type Value struct {
	v interface{}
}
```

`atomic.Value`类型对外暴露的方法就两个：

- `v.Store(c)` - 写操作，将原始的变量`c`存放到一个`atomic.Value`类型的`v`里。
- `c = v.Load()` - 读操作，从线程安全的`v`中读取上一步存放的内容。

简洁的接口使得它的使用也很简单，只需将需要作并发保护的变量读取和赋值操作用`Load()`和`Store()`代替就行了。

具体解释可看https://studygolang.com/articles/23242?fr=sidebar，在这因为篇幅原因不过多的解释。这里只需要知道它是取消后用来写入，然后`Done()`方法获取是否取消。

`children map[canceler]struct{}`用于存储当前cancelCtx的子节点。当前节点取消时，通知子节点。

`err`存储取消时的取消原因。



#### propagateCancel

调用propagateCancel构建父子节点关系，使得当父节点取消时，子节点也取消。

```go
func propagateCancel(parent Context, child canceler) {
  // 如果返回nil，说明当前父`context`从来不会被取消，是一个空节点，直接返回即可。
 done := parent.Done()
 if done == nil {
  return // parent is never canceled
 }
 
  // 提前判断一个父context是否被取消，如果取消了也不需要构建关联了，
  // 把当前子节点取消掉并返回
 select {
 case <-done:
  // parent is already canceled
  child.cancel(false, parent.Err())
  return
 default:
 }
 
  // 这里目的就是找到可以“挂”、“取消”的context
 if p, ok := parentCancelCtx(parent); ok {
  p.mu.Lock()
    // 找到了可以“挂”、“取消”的context，但是已经被取消了，那么这个子节点也不需要
    // 继续挂靠了，取消即可
  if p.err != nil {
   child.cancel(false, p.err)
  } else {
      // 将当前节点挂到父节点的childrn map中，外面调用cancel时可以层层取消
   if p.children == nil {
        // 这里因为childer节点也会变成父节点，所以需要初始化map结构
    p.children = make(map[canceler]struct{})
   }
   p.children[child] = struct{}{}
  }
  p.mu.Unlock()
 } else {
    // 没有找到可“挂”，“取消”的父节点挂载，那么就开一个goroutine，监听parent.Done()和child.Done()，一旦parent.Done()返回的channel关闭，即context链中某个祖先节点context被取消，则将当前context也取消。
  atomic.AddInt32(&goroutines, +1)
  go func() {
   select {
   case <-parent.Done():
    child.cancel(false, parent.Err())
   case <-child.Done():
   }
  }()
 }
}


```

