# Go 利用 Linux 内核的负载均衡能力

## 在开始之前

在我们使用http服务时，如果已经存在一个端口号为8080的服务，这时我们再新创一个同样地址和端口的服务时，就会发生错误

```
listen tcp :8000: bind: address already in use
```

这是由于默认情况下，操作系统不允许我们打开具有相同源地址和端口的套接字 socket。但如果我们想开启多个服务进程去监听同一个端口。这时就会有人说，我们为什么一定要开同一个端口，我另外开一个不行啊。又或者说，我们在一个端口开启多个服务的意义又何在？

这里我们提到了socket，也许可能会有人不知道socket~~（比如说前几天的我）~~。这里的socket并不是我们熟知的websocket，两者甚至没有多大的关联。websocket是一个应用层的协议。而socket甚至不是一个协议，英文socket的意思是插座，网络中的Socket是一个抽象的接口，可以理解为网络中连接的两端。通常被叫做套接字接口，其意义在对传输层进行封装屏蔽了传输层的复杂性。它并不是一个协议，是为了大家更方便的使用传输层协议产生的一个抽象层。大部分的主流编程语言都提供socket函数。



首先让我们回顾一下网络中如何通信。总所周知，使用TCP/IP协议的应用程序通常采用应用编程接口：UNIX BSD的套接字（socket），来实现网络进程之间的通信。就目前而言，几乎所有的应用程序都是采用socket。

Socket是应用层与TCP/IP协议族通信的中间软件抽象层，它是一组接口。在设计模式中，Socket其实就是一个门面模式，它把复杂的TCP/IP协议族隐藏在Socket接口后面，对用户来说，一组简单的接口就是全部，让Socket去组织数据，以符合指定的协议。

![img](https://img-blog.csdnimg.cn/20190718154523875.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3Bhc2hhbmh1NjQwMg==,size_16,color_FFFFFF,t_70)

到这里我们知道了socket大概是干嘛的，接下来我们再看看socket具体的工作流程

![img](https://img-blog.csdnimg.cn/20190718154556909.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3Bhc2hhbmh1NjQwMg==,size_16,color_FFFFFF,t_70)



首先，服务器会初始化socket，然后与端口进行绑定(bind)和监听(listen)。好，看到这里就差不多了，这就是今天我们要用的东西了。既然socket要与端口绑定，那能不能多个socket绑定一个端口呢?

这就要引出socket五元组了，socket 连接通过五元组唯一标识。任意两条连接，它的五元组不能完全相同。

{<protocol>, <src addr>, <src port>, <dest addr>, <dest port>}

即 协议号、源IP地址、源端口、目的IP地址、目的端口，通过这五个信息可以标识唯一的 socket 连接。

`protocol` 指的是传输层 TCP/UDP 协议，它在 socket 被创建时就已经确定。`src addr` 与 `src port` 分别标识着服务方的地址与端口信息，也是固定好的。但我们发现还有两个信息`dest addr`和`dest port`,只要这两个信息不同，我们仍然可以区别开socket连接。

基于这个理论基础，那实际上，我们可以在同一个网络主机复用相同的 IP 地址和端口号。





## SO_REUSEPORT

SO_REUSEPORT支持多个进程或者线程绑定到同一端口，提高服务器程序的性能，解决的问题：

- 允许多个套接字 bind()/listen() 同一个TCP/UDP端口
  - 每一个线程拥有自己的服务器套接字
  - 在服务器套接字上没有了锁的竞争
- 内核层面实现负载均衡
- 安全层面，监听同一个端口的套接字只能位于同一个用户下面

下面我们来看一下在go中的实现：

```go
package main
 
import (
 "context"
 "fmt"
 "net"
 "net/http"
 "os"
 "syscall"
 
 "golang.org/x/sys/unix"
)
 
var lc = net.ListenConfig{
 Control: func(network, address string, c syscall.RawConn) error {
  var opErr error
  if err := c.Control(func(fd uintptr) {
   opErr = unix.SetsockoptInt(int(fd), unix.SOL_SOCKET, unix.SO_REUSEPORT, 1)
  }); err != nil {
   return err
  }
  return opErr
 },
}
 
func main() {
 pid := os.Getpid()
 l, err := lc.Listen(context.Background(), "tcp", "127.0.0.1:8000")
 if err != nil {
  panic(err)
 }
 server := &http.Server{}
 http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
  w.WriteHeader(http.StatusOK)
  fmt.Fprintf(w, "Client [%s] Received msg from Server PID: [%d] \n", r.RemoteAddr, pid)
 })
 fmt.Printf("Server with PID: [%d] is running \n", pid)
 _ = server.Serve(l)
}
```

## 总结

linux 内核自 3.9 提供的 `SO_REUSEPORT` 选项，可以让多进程监听同一个端口。

这种机制带来了什么：

- **提高服务器程序的吞吐性能**：我们可以运行多个应用程序实例，充分利用多核 CPU 资源，避免出现单核在处理数据包，其他核却闲着的问题。
- **内核级负载均衡**：我们不需要在多个实例前面添加一层服务代理，因为内核已经提供了简单的负载均衡。
- **不停服更新**：当我们需要更新服务时，可以启动新的服务实例来接受请求，再优雅地关闭掉旧服务实例。

socket即是一种特殊的文件，一些socket函数就是对其进行的操作（读/写IO、打开、关闭）。同时，

