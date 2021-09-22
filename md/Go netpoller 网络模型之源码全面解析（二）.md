> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [zhuanlan.zhihu.com](https://zhuanlan.zhihu.com/p/299047984)

作者：allanpan，腾讯 IEG 后台开发工程师

本文接着上一篇：

### **实现原理**

使用 Go 编写一个典型的 TCP echo server:

```
package main

import (
 "log"
 "net"
)

func main() {
 listen, err := net.Listen("tcp", ":8888")
 if err != nil {
  log.Println("listen error: ", err)
  return
 }

 for {
  conn, err := listen.Accept()
  if err != nil {
   log.Println("accept error: ", err)
   break
  }

  // start a new goroutine to handle the new connection.
  go HandleConn(conn)
 }
}

func HandleConn(conn net.Conn) {
 defer conn.Close()
 packet := make([]byte, 1024)
 for {
  // block here if socket is not available for reading data.
  n, err := conn.Read(packet)
  if err != nil {
   log.Println("read socket error: ", err)
   return
  }

  // same as above, block here if socket is not available for writing.
  _, _ = conn.Write(packet[:n])
 }
}
```

上面是一个基于 Go 原生网络模型（基于 netpoller）编写的一个 TCP server，模式是 `goroutine-per-connection` ，在这种模式下，开发者使用的是同步的模式去编写异步的逻辑而且对于开发者来说 I/O 是否阻塞是无感知的，也就是说开发者无需考虑 goroutines 甚至更底层的线程、进程的调度和上下文切换。而 Go netpoller 最底层的事件驱动技术肯定是基于 epoll/kqueue/iocp 这一类的 I/O 事件驱动技术，只不过是把这些调度和上下文切换的工作转移到了 runtime 的 Go scheduler，让它来负责调度 goroutines，从而极大地降低了程序员的心智负担！

Go 的这种同步模式的网络服务器的基本架构通常如下：

![](https://pic4.zhimg.com/v2-69a8343815dee0174a41ba1b05de2757_r.jpg)

上面的示例代码中相关的在源码里的几个数据结构和方法：

```
// TCPListener is a TCP network listener. Clients should typically
// use variables of type Listener instead of assuming TCP.
type TCPListener struct {
 fd *netFD
 lc ListenConfig
}

// Accept implements the Accept method in the Listener interface; it
// waits for the next call and returns a generic Conn.
func (l *TCPListener) Accept() (Conn, error) {
 if !l.ok() {
  return nil, syscall.EINVAL
 }
 c, err := l.accept()
 if err != nil {
  return nil, &OpError{Op: "accept", Net: l.fd.net, Source: nil, Addr: l.fd.laddr, Err: err}
 }
 return c, nil
}

func (ln *TCPListener) accept() (*TCPConn, error) {
 fd, err := ln.fd.accept()
 if err != nil {
  return nil, err
 }
 tc := newTCPConn(fd)
 if ln.lc.KeepAlive >= 0 {
  setKeepAlive(fd, true)
  ka := ln.lc.KeepAlive
  if ln.lc.KeepAlive == 0 {
   ka = defaultTCPKeepAlive
  }
  setKeepAlivePeriod(fd, ka)
 }
 return tc, nil
}

// TCPConn is an implementation of the Conn interface for TCP network
// connections.
type TCPConn struct {
 conn
}

// Conn
type conn struct {
 fd *netFD
}

type conn struct {
 fd *netFD
}

func (c *conn) ok() bool { return c != nil && c.fd != nil }

// Implementation of the Conn interface.

// Read implements the Conn Read method.
func (c *conn) Read(b []byte) (int, error) {
 if !c.ok() {
  return 0, syscall.EINVAL
 }
 n, err := c.fd.Read(b)
 if err != nil && err != io.EOF {
  err = &OpError{Op: "read", Net: c.fd.net, Source: c.fd.laddr, Addr: c.fd.raddr, Err: err}
 }
 return n, err
}

// Write implements the Conn Write method.
func (c *conn) Write(b []byte) (int, error) {
 if !c.ok() {
  return 0, syscall.EINVAL
 }
 n, err := c.fd.Write(b)
 if err != nil {
  err = &OpError{Op: "write", Net: c.fd.net, Source: c.fd.laddr, Addr: c.fd.raddr, Err: err}
 }
 return n, err
}
```

### **net.Listen**

调用 `net.Listen` 之后，底层会通过 Linux 的系统调用 `socket` 方法创建一个 fd 分配给 listener，并用以来初始化 listener 的 `netFD` ，接着调用 netFD 的 `listenStream` 方法完成对 socket 的 bind&listen 操作以及对 `netFD` 的初始化（主要是对 netFD 里的 pollDesc 的初始化），调用链是 `runtime.runtime_pollServerInit` --> `runtime.poll_runtime_pollServerInit` --> `runtime.netpollGenericInit`，主要做的事情是：

1.  调用 `epollcreate1` 创建一个 epoll 实例 `epfd`，作为整个 runtime 的唯一 event-loop 使用；
2.  调用 `runtime.nonblockingPipe` 创建一个用于和 epoll 实例通信的管道，这里为什么不用更新且更轻量的 eventfd 呢？我个人猜测是为了兼容更多以及更老的系统版本；
3.  将 `netpollBreakRd` 通知信号量封装成 `epollevent` 事件结构体注册进 epoll 实例。

相关源码如下：

```
// 调用 linux 系统调用 socket 创建 listener fd 并设置为为阻塞 I/O
s, err := socketFunc(family, sotype|syscall.SOCK_NONBLOCK|syscall.SOCK_CLOEXEC, proto)
// On Linux the SOCK_NONBLOCK and SOCK_CLOEXEC flags were
// introduced in 2.6.27 kernel and on FreeBSD both flags were
// introduced in 10 kernel. If we get an EINVAL error on Linux
// or EPROTONOSUPPORT error on FreeBSD, fall back to using
// socket without them.

socketFunc        func(int, int, int) (int, error)  = syscall.Socket

// 用上面创建的 listener fd 初始化 listener netFD
if fd, err = newFD(s, family, sotype, net); err != nil {
 poll.CloseFunc(s)
 return nil, err
}

// 对 listener fd 进行 bind&listen 操作，并且调用 init 方法完成初始化
func (fd *netFD) listenStream(laddr sockaddr, backlog int, ctrlFn func(string, string, syscall.RawConn) error) error {
 ...
  
 // 完成绑定操作
 if err = syscall.Bind(fd.pfd.Sysfd, lsa); err != nil {
  return os.NewSyscallError("bind", err)
 }
  
 // 完成监听操作
 if err = listenFunc(fd.pfd.Sysfd, backlog); err != nil {
  return os.NewSyscallError("listen", err)
 }
  
 // 调用 init，内部会调用 poll.FD.Init，最后调用 pollDesc.init
 if err = fd.init(); err != nil {
  return err
 }
 lsa, _ = syscall.Getsockname(fd.pfd.Sysfd)
 fd.setAddr(fd.addrFunc()(lsa), nil)
 return nil
}

// 使用 sync.Once 来确保一个 listener 只持有一个 epoll 实例
var serverInit sync.Once

// netFD.init 会调用 poll.FD.Init 并最终调用到 pollDesc.init，
// 它会创建 epoll 实例并把 listener fd 加入监听队列
func (pd *pollDesc) init(fd *FD) error {
 // runtime_pollServerInit 通过 `go:linkname` 链接到具体的实现函数 poll_runtime_pollServerInit，
 // 接着再调用 netpollGenericInit，然后会根据不同的系统平台去调用特定的 netpollinit 来创建 epoll 实例
 serverInit.Do(runtime_pollServerInit)
  
 // runtime_pollOpen 内部调用了 netpollopen 来将 listener fd 注册到 
 // epoll 实例中，另外，它会初始化一个 pollDesc 并返回
 ctx, errno := runtime_pollOpen(uintptr(fd.Sysfd))
 if errno != 0 {
  if ctx != 0 {
   runtime_pollUnblock(ctx)
   runtime_pollClose(ctx)
  }
  return syscall.Errno(errno)
 }
 // 把真正初始化完成的 pollDesc 实例赋值给当前的 pollDesc 代表自身的指针，
 // 后续使用直接通过该指针操作
 pd.runtimeCtx = ctx
 return nil
}

var (
 // 全局唯一的 epoll fd，只在 listener fd 初始化之时被指定一次
 epfd int32 = -1 // epoll descriptor
)

// netpollinit 会创建一个 epoll 实例，然后把 epoll fd 赋值给 epfd，
// 后续 listener 以及它 accept 的所有 sockets 有关 epoll 的操作都是基于这个全局的 epfd
func netpollinit() {
 epfd = epollcreate1(_EPOLL_CLOEXEC)
 if epfd < 0 {
  epfd = epollcreate(1024)
  if epfd < 0 {
   println("runtime: epollcreate failed with", -epfd)
   throw("runtime: netpollinit failed")
  }
  closeonexec(epfd)
 }
 r, w, errno := nonblockingPipe()
 if errno != 0 {
  println("runtime: pipe failed with", -errno)
  throw("runtime: pipe failed")
 }
 ev := epollevent{
  events: _EPOLLIN,
 }
 *(**uintptr)(unsafe.Pointer(&ev.data)) = &netpollBreakRd
 errno = epollctl(epfd, _EPOLL_CTL_ADD, r, &ev)
 if errno != 0 {
  println("runtime: epollctl failed with", -errno)
  throw("runtime: epollctl failed")
 }
 netpollBreakRd = uintptr(r)
 netpollBreakWr = uintptr(w)
}

// netpollopen 会被 runtime_pollOpen 调用，注册 fd 到 epoll 实例，
// 注意这里使用的是 epoll 的 ET 模式，同时会利用万能指针把 pollDesc 保存到 epollevent 的一个 8 位的字节数组 data 里
func netpollopen(fd uintptr, pd *pollDesc) int32 {
 var ev epollevent
 ev.events = _EPOLLIN | _EPOLLOUT | _EPOLLRDHUP | _EPOLLET
 *(**pollDesc)(unsafe.Pointer(&ev.data)) = pd
 return -epollctl(epfd, _EPOLL_CTL_ADD, int32(fd), &ev)
}
```

我们前面提到的 epoll 的三个基本调用，Go 在源码里实现了对那三个调用的封装：

```
#include <sys/epoll.h>  
int epoll_create(int size);  
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);  
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);

// Go 对上面三个调用的封装
func netpollinit()
func netpollopen(fd uintptr, pd *pollDesc) int32
func netpoll(block bool) gList
```

netFD 就是通过这三个封装来对 epoll 进行创建实例、注册 fd 和等待事件操作的。

### **Listener.Accept()**

`netpoll` accept socket 的工作流程如下：

1.  服务端的 netFD 在 `listen` 时会创建 epoll 的实例，并将 listenerFD 加入 epoll 的事件队列
2.  netFD 在 `accept` 时将返回的 connFD 也加入 epoll 的事件队列
3.  netFD 在读写时出现 `syscall.EAGAIN` 错误，通过 pollDesc 的 `waitRead` 方法将当前的 goroutine park 住，直到 ready，从 pollDesc 的 `waitRead` 中返回

`Listener.Accept()` 接收来自客户端的新连接，具体还是调用 `netFD.accept` 方法来完成这个功能：

```
// Accept implements the Accept method in the Listener interface; it
// waits for the next call and returns a generic Conn.
func (l *TCPListener) Accept() (Conn, error) {
 if !l.ok() {
  return nil, syscall.EINVAL
 }
 c, err := l.accept()
 if err != nil {
  return nil, &OpError{Op: "accept", Net: l.fd.net, Source: nil, Addr: l.fd.laddr, Err: err}
 }
 return c, nil
}

func (ln *TCPListener) accept() (*TCPConn, error) {
 fd, err := ln.fd.accept()
 if err != nil {
  return nil, err
 }
 tc := newTCPConn(fd)
 if ln.lc.KeepAlive >= 0 {
  setKeepAlive(fd, true)
  ka := ln.lc.KeepAlive
  if ln.lc.KeepAlive == 0 {
   ka = defaultTCPKeepAlive
  }
  setKeepAlivePeriod(fd, ka)
 }
 return tc, nil
}

func (fd *netFD) accept() (netfd *netFD, err error) {
 // 调用 poll.FD 的 Accept 方法接受新的 socket 连接，返回 socket 的 fd
 d, rsa, errcall, err := fd.pfd.Accept()
 if err != nil {
  if errcall != "" {
   err = wrapSyscallError(errcall, err)
  }
  return nil, err
 }
 // 以 socket fd 构造一个新的 netFD，代表这个新的 socket
 if netfd, err = newFD(d, fd.family, fd.sotype, fd.net); err != nil {
  poll.CloseFunc(d)
  return nil, err
 }
 // 调用 netFD 的 init 方法完成初始化
 if err = netfd.init(); err != nil {
  fd.Close()
  return nil, err
 }
 lsa, _ := syscall.Getsockname(netfd.pfd.Sysfd)
 netfd.setAddr(netfd.addrFunc()(lsa), netfd.addrFunc()(rsa))
 return netfd, nil
}
```

`netFD.accept` 方法里会再调用 `poll.FD.Accept` ，最后会使用 Linux 的系统调用 `accept` 来完成新连接的接收，并且会把 accept 的 socket 设置成非阻塞 I/O 模式：

```
// Accept wraps the accept network call.
func (fd *FD) Accept() (int, syscall.Sockaddr, string, error) {
 if err := fd.readLock(); err != nil {
  return -1, nil, "", err
 }
 defer fd.readUnlock()

 if err := fd.pd.prepareRead(fd.isFile); err != nil {
  return -1, nil, "", err
 }
 for {
  // 使用 linux 系统调用 accept 接收新连接，创建对应的 socket
  s, rsa, errcall, err := accept(fd.Sysfd)
  // 因为 listener fd 在创建的时候已经设置成非阻塞的了，
  // 所以 accept 方法会直接返回，不管有没有新连接到来；如果 err == nil 则表示正常建立新连接，直接返回
  if err == nil {
   return s, rsa, "", err
  }
  // 如果 err != nil，则判断 err == syscall.EAGAIN，符合条件则进入 pollDesc.waitRead 方法
  switch err {
  case syscall.EAGAIN:
   if fd.pd.pollable() {
    // 如果当前没有发生期待的 I/O 事件，那么 waitRead 会通过 park goroutine 让逻辑 block 在这里
    if err = fd.pd.waitRead(fd.isFile); err == nil {
     continue
    }
   }
  case syscall.ECONNABORTED:
   // This means that a socket on the listen
   // queue was closed before we Accept()ed it;
   // it's a silly error, so try again.
   continue
  }
  return -1, nil, errcall, err
 }
}

// 使用 linux 的 accept 系统调用接收新连接并把这个 socket fd 设置成非阻塞 I/O
ns, sa, err := Accept4Func(s, syscall.SOCK_NONBLOCK|syscall.SOCK_CLOEXEC)
// On Linux the accept4 system call was introduced in 2.6.28
// kernel and on FreeBSD it was introduced in 10 kernel. If we
// get an ENOSYS error on both Linux and FreeBSD, or EINVAL
// error on Linux, fall back to using accept.

// Accept4Func is used to hook the accept4 call.
var Accept4Func func(int, int) (int, syscall.Sockaddr, error) = syscall.Accept4
```

`pollDesc.waitRead` 方法主要负责检测当前这个 pollDesc 的上层 netFD 对应的 fd 是否有『期待的』I/O 事件发生，如果有就直接返回，否则就 park 住当前的 goroutine 并持续等待直至对应的 fd 上发生可读 / 可写或者其他『期待的』I/O 事件为止，然后它就会返回到外层的 for 循环，让 goroutine 继续执行逻辑。

poll.FD.Accept() 返回之后，会构造一个对应这个新 socket 的 netFD，然后调用 init() 方法完成初始化，这个 init 过程和前面 net.Listen() 是一样的，调用链：netFD.init() --> poll.FD.Init() --> poll.pollDesc.init()，最终又会走到这里：

```
var serverInit sync.Once

func (pd *pollDesc) init(fd *FD) error {
 serverInit.Do(runtime_pollServerInit)
 ctx, errno := runtime_pollOpen(uintptr(fd.Sysfd))
 if errno != 0 {
  if ctx != 0 {
   runtime_pollUnblock(ctx)
   runtime_pollClose(ctx)
  }
  return syscall.Errno(errno)
 }
 pd.runtimeCtx = ctx
 return nil
}
```

然后把这个 socket fd 注册到 listener 的 epoll 实例的事件队列中去，等待 I/O 事件。

### **Conn.Read/Conn.Write**

我们先来看看 `Conn.Read` 方法是如何实现的，原理其实和 `Listener.Accept` 是一样的，具体调用链还是首先调用 conn 的 `netFD.Read` ，然后内部再调用 `poll.FD.Read` ，最后使用 Linux 的系统调用 read: `syscall.Read` 完成数据读取：

```
// Implementation of the Conn interface.

// Read implements the Conn Read method.
func (c *conn) Read(b []byte) (int, error) {
 if !c.ok() {
  return 0, syscall.EINVAL
 }
 n, err := c.fd.Read(b)
 if err != nil && err != io.EOF {
  err = &OpError{Op: "read", Net: c.fd.net, Source: c.fd.laddr, Addr: c.fd.raddr, Err: err}
 }
 return n, err
}

func (fd *netFD) Read(p []byte) (n int, err error) {
 n, err = fd.pfd.Read(p)
 runtime.KeepAlive(fd)
 return n, wrapSyscallError("read", err)
}

// Read implements io.Reader.
func (fd *FD) Read(p []byte) (int, error) {
 if err := fd.readLock(); err != nil {
  return 0, err
 }
 defer fd.readUnlock()
 if len(p) == 0 {
  // If the caller wanted a zero byte read, return immediately
  // without trying (but after acquiring the readLock).
  // Otherwise syscall.Read returns 0, nil which looks like
  // io.EOF.
  // TODO(bradfitz): make it wait for readability? (Issue 15735)
  return 0, nil
 }
 if err := fd.pd.prepareRead(fd.isFile); err != nil {
  return 0, err
 }
 if fd.IsStream && len(p) > maxRW {
  p = p[:maxRW]
 }
 for {
  // 尝试从该 socket 读取数据，因为 socket 在被 listener accept 的时候设置成
  // 了非阻塞 I/O，所以这里同样也是直接返回，不管有没有可读的数据
  n, err := syscall.Read(fd.Sysfd, p)
  if err != nil {
   n = 0
   // err == syscall.EAGAIN 表示当前没有期待的 I/O 事件发生，也就是 socket 不可读
   if err == syscall.EAGAIN && fd.pd.pollable() {
    // 如果当前没有发生期待的 I/O 事件，那么 waitRead 
    // 会通过 park goroutine 让逻辑 block 在这里
    if err = fd.pd.waitRead(fd.isFile); err == nil {
     continue
    }
   }

   // On MacOS we can see EINTR here if the user
   // pressed ^Z.  See issue #22838.
   if runtime.GOOS == "darwin" && err == syscall.EINTR {
    continue
   }
  }
  err = fd.eofError(n, err)
  return n, err
 }
}
```

`conn.Write` 和 `conn.Read` 的原理是一致的，它也是通过类似 `pollDesc.waitRead` 的 `pollDesc.waitWrite` 来 park 住 goroutine 直至期待的 I/O 事件发生才返回恢复执行。

### **pollDesc.waitRead/pollDesc.waitWrite**

`pollDesc.waitRead` 内部调用了 `poll.runtime_pollWait` --> `runtime.poll_runtime_pollWait` 来达成无 I/O 事件时 park 住 goroutine 的目的：

```
//go:linkname poll_runtime_pollWait internal/poll.runtime_pollWait
func poll_runtime_pollWait(pd *pollDesc, mode int) int {
 err := netpollcheckerr(pd, int32(mode))
 if err != pollNoError {
  return err
 }
 // As for now only Solaris, illumos, and AIX use level-triggered IO.
 if GOOS == "solaris" || GOOS == "illumos" || GOOS == "aix" {
  netpollarm(pd, mode)
 }
 // 进入 netpollblock 并且判断是否有期待的 I/O 事件发生，
 // 这里的 for 循环是为了一直等到 io ready
 for !netpollblock(pd, int32(mode), false) {
  err = netpollcheckerr(pd, int32(mode))
  if err != 0 {
   return err
  }
  // Can happen if timeout has fired and unblocked us,
  // but before we had a chance to run, timeout has been reset.
  // Pretend it has not happened and retry.
 }
 return 0
}

// returns true if IO is ready, or false if timedout or closed
// waitio - wait only for completed IO, ignore errors
func netpollblock(pd *pollDesc, mode int32, waitio bool) bool {
 // gpp 保存的是 goroutine 的数据结构 g，这里会根据 mode 的值决定是 rg 还是 wg，
  // 前面提到过，rg 和 wg 是用来保存等待 I/O 就绪的 gorouine 的，后面调用 gopark 之后，
  // 会把当前的 goroutine 的抽象数据结构 g 存入 gpp 这个指针，也就是 rg 或者 wg
 gpp := &pd.rg
 if mode == 'w' {
  gpp = &pd.wg
 }

 // set the gpp semaphore to WAIT
 // 这个 for 循环是为了等待 io ready 或者 io wait
 for {
  old := *gpp
  // gpp == pdReady 表示此时已有期待的 I/O 事件发生，
  // 可以直接返回 unblock 当前 goroutine 并执行响应的 I/O 操作
  if old == pdReady {
   *gpp = 0
   return true
  }
  if old != 0 {
   throw("runtime: double wait")
  }
  // 如果没有期待的 I/O 事件发生，则通过原子操作把 gpp 的值置为 pdWait 并退出 for 循环
  if atomic.Casuintptr(gpp, 0, pdWait) {
   break
  }
 }

 // need to recheck error states after setting gpp to WAIT
 // this is necessary because runtime_pollUnblock/runtime_pollSetDeadline/deadlineimpl
 // do the opposite: store to closing/rd/wd, membarrier, load of rg/wg
  
 // waitio 此时是 false，netpollcheckerr 方法会检查当前 pollDesc 对应的 fd 是否是正常的，
 // 通常来说  netpollcheckerr(pd, mode) == 0 是成立的，所以这里会执行 gopark 
 // 把当前 goroutine 给 park 住，直至对应的 fd 上发生可读/可写或者其他『期待的』I/O 事件为止，
 // 然后 unpark 返回，在 gopark 内部会把当前 goroutine 的抽象数据结构 g 存入
 // gpp(pollDesc.rg/pollDesc.wg) 指针里，以便在后面的 netpoll 函数取出 pollDesc 之后，
 // 把 g 添加到链表里返回，接着重新调度 goroutine
 if waitio || netpollcheckerr(pd, mode) == 0 {
  // 注册 netpollblockcommit 回调给 gopark，在 gopark 内部会执行它，保存当前 goroutine 到 gpp
  gopark(netpollblockcommit, unsafe.Pointer(gpp), waitReasonIOWait, traceEvGoBlockNet, 5)
 }
 // be careful to not lose concurrent READY notification
 old := atomic.Xchguintptr(gpp, 0)
 if old > pdWait {
  throw("runtime: corrupted polldesc")
 }
 return old == pdReady
}

// gopark 会停住当前的 goroutine 并且调用传递进来的回调函数 unlockf，从上面的源码我们可以知道这个函数是
// netpollblockcommit
func gopark(unlockf func(*g, unsafe.Pointer) bool, lock unsafe.Pointer, reason waitReason, traceEv byte, traceskip int) {
 if reason != waitReasonSleep {
  checkTimeouts() // timeouts may expire while two goroutines keep the scheduler busy
 }
 mp := acquirem()
 gp := mp.curg
 status := readgstatus(gp)
 if status != _Grunning && status != _Gscanrunning {
  throw("gopark: bad g status")
 }
 mp.waitlock = lock
 mp.waitunlockf = unlockf
 gp.waitreason = reason
 mp.waittraceev = traceEv
 mp.waittraceskip = traceskip
 releasem(mp)
 // can't do anything that might move the G between Ms here.
  // gopark 最终会调用 park_m，在这个函数内部会调用 unlockf，也就是 netpollblockcommit，
 // 然后会把当前的 goroutine，也就是 g 数据结构保存到 pollDesc 的 rg 或者 wg 指针里
 mcall(park_m)
}

// park continuation on g0.
func park_m(gp *g) {
 _g_ := getg()

 if trace.enabled {
  traceGoPark(_g_.m.waittraceev, _g_.m.waittraceskip)
 }

 casgstatus(gp, _Grunning, _Gwaiting)
 dropg()

 if fn := _g_.m.waitunlockf; fn != nil {
  // 调用 netpollblockcommit，把当前的 goroutine，
  // 也就是 g 数据结构保存到 pollDesc 的 rg 或者 wg 指针里
  ok := fn(gp, _g_.m.waitlock)
  _g_.m.waitunlockf = nil
  _g_.m.waitlock = nil
  if !ok {
   if trace.enabled {
    traceGoUnpark(gp, 2)
   }
   casgstatus(gp, _Gwaiting, _Grunnable)
   execute(gp, true) // Schedule it back, never returns.
  }
 }
 schedule()
}

// netpollblockcommit 在 gopark 函数里被调用
func netpollblockcommit(gp *g, gpp unsafe.Pointer) bool {
 // 通过原子操作把当前 goroutine 抽象的数据结构 g，也就是这里的参数 gp 存入 gpp 指针，
 // 此时 gpp 的值是 pollDesc 的 rg 或者 wg 指针
 r := atomic.Casuintptr((*uintptr)(gpp), pdWait, uintptr(unsafe.Pointer(gp)))
 if r {
  // Bump the count of goroutines waiting for the poller.
  // The scheduler uses this to decide whether to block
  // waiting for the poller if there is nothing else to do.
  atomic.Xadd(&netpollWaiters, 1)
 }
 return r
}
```

`pollDesc.waitWrite` 的内部实现原理和 `pollDesc.waitRead` 是一样的，都是基于 `poll.runtime_pollWait` --> `runtime.poll_runtime_pollWait`，这里就不再赘述。

### **netpoll**

前面已经从源码的层面分析完了 netpoll 是如何通过 park goroutine 从而达到阻塞 Accept/Read/Write 的效果，而通过调用 gopark，goroutine 会被放置在某个等待队列中，这里是放到了 epoll 的 "interest list" 里，底层数据结构是由红黑树实现的 `eventpoll.rbr`，此时 G 的状态由 `_Grunning`为`_Gwaitting` ，因此 G 必须被手动唤醒 (通过 goready)，否则会丢失任务，应用层阻塞通常使用这种方式。

所以我们现在可以来从整体的层面来概括 Go 的网络业务 goroutine 是如何被规划调度的了：

![](https://pic1.zhimg.com/v2-bae882be10b54c89d50afcc8405b14d4_r.jpg)

> 首先，client 连接 server 的时候，listener 通过 accept 调用接收新 connection，每一个新 connection 都启动一个 goroutine 处理，accept 调用会把该 connection 的 fd 连带所在的 goroutine 上下文信息封装注册到 epoll 的监听列表里去，当 goroutine 调用 `conn.Read` 或者 `conn.Write` 等需要阻塞等待的函数时，会被 `gopark` 给封存起来并使之休眠，让 P 去执行本地调度队列里的下一个可执行的 goroutine，往后 Go scheduler 会在循环调度的 `runtime.schedule()` 函数以及 sysmon 监控线程中调用 `runtime.nepoll` 以获取可运行的 goroutine 列表并通过调用 injectglist 把剩下的 g 放入全局调度队列或者当前 P 本地调度队列去重新执行。  
> 那么当 I/O 事件发生之后，netpoller 是通过什么方式唤醒那些在 I/O wait 的 goroutine 的？答案是通过 `runtime.netpoll`。  

`runtime.netpoll` 的核心逻辑是：

1.  根据调用方的入参 delay，设置对应的调用 `epollwait` 的 timeout 值；
2.  调用 `epollwait` 等待发生了可读 / 可写事件的 fd；
3.  循环 `epollwait` 返回的事件列表，处理对应的事件类型， 组装可运行的 goroutine 链表并返回。

```
// netpoll checks for ready network connections.
// Returns list of goroutines that become runnable.
// delay < 0: blocks indefinitely
// delay == 0: does not block, just polls
// delay > 0: block for up to that many nanoseconds
func netpoll(delay int64) gList {
 if epfd == -1 {
  return gList{}
 }

 // 根据特定的规则把 delay 值转换为 epollwait 的 timeout 值
 var waitms int32
 if delay < 0 {
  waitms = -1
 } else if delay == 0 {
  waitms = 0
 } else if delay < 1e6 {
  waitms = 1
 } else if delay < 1e15 {
  waitms = int32(delay / 1e6)
 } else {
  // An arbitrary cap on how long to wait for a timer.
  // 1e9 ms == ~11.5 days.
  waitms = 1e9
 }
 var events [128]epollevent
retry:
 // 超时等待就绪的 fd 读写事件
 n := epollwait(epfd, &events[0], int32(len(events)), waitms)
 if n < 0 {
  if n != -_EINTR {
   println("runtime: epollwait on fd", epfd, "failed with", -n)
   throw("runtime: netpoll failed")
  }
  // If a timed sleep was interrupted, just return to
  // recalculate how long we should sleep now.
  if waitms > 0 {
   return gList{}
  }
  goto retry
 }

 // toRun 是一个 g 的链表，存储要恢复的 goroutines，最后返回给调用方
 var toRun gList
 for i := int32(0); i < n; i++ {
  ev := &events[i]
  if ev.events == 0 {
   continue
  }

  // Go scheduler 在调用 findrunnable() 寻找 goroutine 去执行的时候，
  // 在调用 netpoll 之时会检查当前是否有其他线程同步阻塞在 netpoll，
  // 若是，则调用 netpollBreak 来唤醒那个线程，避免它长时间阻塞
  if *(**uintptr)(unsafe.Pointer(&ev.data)) == &netpollBreakRd {
   if ev.events != _EPOLLIN {
    println("runtime: netpoll: break fd ready for", ev.events)
    throw("runtime: netpoll: break fd ready for something unexpected")
   }
   if delay != 0 {
    // netpollBreak could be picked up by a
    // nonblocking poll. Only read the byte
    // if blocking.
    var tmp [16]byte
    read(int32(netpollBreakRd), noescape(unsafe.Pointer(&tmp[0])), int32(len(tmp)))
    atomic.Store(&netpollWakeSig, 0)
   }
   continue
  }

  // 判断发生的事件类型，读类型或者写类型等，然后给 mode 复制相应的值，
    // mode 用来决定从 pollDesc 里的 rg 还是 wg 里取出 goroutine
  var mode int32
  if ev.events&(_EPOLLIN|_EPOLLRDHUP|_EPOLLHUP|_EPOLLERR) != 0 {
   mode += 'r'
  }
  if ev.events&(_EPOLLOUT|_EPOLLHUP|_EPOLLERR) != 0 {
   mode += 'w'
  }
  if mode != 0 {
   // 取出保存在 epollevent 里的 pollDesc
   pd := *(**pollDesc)(unsafe.Pointer(&ev.data))
   pd.everr = false
   if ev.events == _EPOLLERR {
    pd.everr = true
   }
   // 调用 netpollready，传入就绪 fd 的 pollDesc，
   // 把 fd 对应的 goroutine 添加到链表 toRun 中
   netpollready(&toRun, pd, mode)
  }
 }
 return toRun
}

// netpollready 调用 netpollunblock 返回就绪 fd 对应的 goroutine 的抽象数据结构 g
func netpollready(toRun *gList, pd *pollDesc, mode int32) {
 var rg, wg *g
 if mode == 'r' || mode == 'r'+'w' {
  rg = netpollunblock(pd, 'r', true)
 }
 if mode == 'w' || mode == 'r'+'w' {
  wg = netpollunblock(pd, 'w', true)
 }
 if rg != nil {
  toRun.push(rg)
 }
 if wg != nil {
  toRun.push(wg)
 }
}

// netpollunblock 会依据传入的 mode 决定从 pollDesc 的 rg 或者 wg 取出当时 gopark 之时存入的
// goroutine 抽象数据结构 g 并返回
func netpollunblock(pd *pollDesc, mode int32, ioready bool) *g {
 // mode == 'r' 代表当时 gopark 是为了等待读事件，而 mode == 'w' 则代表是等待写事件
 gpp := &pd.rg
 if mode == 'w' {
  gpp = &pd.wg
 }

 for {
  // 取出 gpp 存储的 g
  old := *gpp
  if old == pdReady {
   return nil
  }
  if old == 0 && !ioready {
   // Only set READY for ioready. runtime_pollWait
   // will check for timeout/cancel before waiting.
   return nil
  }
  var new uintptr
  if ioready {
   new = pdReady
  }
  // 重置 pollDesc 的 rg 或者 wg
  if atomic.Casuintptr(gpp, old, new) {
      // 如果该 goroutine 还是必须等待，则返回 nil
   if old == pdWait {
    old = 0
   }
   // 通过万能指针还原成 g 并返回
   return (*g)(unsafe.Pointer(old))
  }
 }
}

// netpollBreak 往通信管道里写入信号去唤醒 epollwait
func netpollBreak() {
 // 通过 CAS 避免重复的唤醒信号被写入管道，
 // 从而减少系统调用并节省一些系统资源
 if atomic.Cas(&netpollWakeSig, 0, 1) {
  for {
   var b byte
   n := write(netpollBreakWr, unsafe.Pointer(&b), 1)
   if n == 1 {
    break
   }
   if n == -_EINTR {
    continue
   }
   if n == -_EAGAIN {
    return
   }
   println("runtime: netpollBreak write failed with", -n)
   throw("runtime: netpollBreak write failed")
  }
 }
}
```

Go 在多种场景下都可能会调用 `netpoll` 检查文件描述符状态，`netpoll` 里会调用 `epoll_wait` 从 epoll 的 `eventpoll.rdllist` 就绪双向链表返回，从而得到 I/O 就绪的 socket fd 列表，并根据取出最初调用 `epoll_ctl` 时保存的上下文信息，恢复 `g`。所以执行完`netpoll` 之后，会返回一个就绪 fd 列表对应的 goroutine 链表，接下来将就绪的 goroutine 通过调用 `injectglist` 加入到全局调度队列或者 P 的本地调度队列中，启动 M 绑定 P 去执行。

具体调用 `netpoll` 的地方，首先在 Go runtime scheduler 循环调度 goroutines 之时就有可能会调用 `netpoll` 获取到已就绪的 fd 对应的 goroutine 来调度执行。

首先 Go scheduler 的核心方法 `runtime.schedule()` 里会调用一个叫 `runtime.findrunable()` 的方法获取可运行的 goroutine 来执行，而在 `runtime.findrunable()` 方法里就调用了 `runtime.netpoll` 获取已就绪的 fd 列表对应的 goroutine 列表：

```
// One round of scheduler: find a runnable goroutine and execute it.
// Never returns.
func schedule() {
 ...
  
  if gp == nil {
  gp, inheritTime = findrunnable() // blocks until work is available
 }
  
 ...
}

// Finds a runnable goroutine to execute.
// Tries to steal from other P's, get g from global queue, poll network.
func findrunnable() (gp *g, inheritTime bool) {
  ...
  
  // Poll network.
 if netpollinited() && (atomic.Load(&netpollWaiters) > 0 || pollUntil != 0) && atomic.Xchg64(&sched.lastpoll, 0) != 0 {
  atomic.Store64(&sched.pollUntil, uint64(pollUntil))
  if _g_.m.p != 0 {
   throw("findrunnable: netpoll with p")
  }
  if _g_.m.spinning {
   throw("findrunnable: netpoll with spinning")
  }
  if faketime != 0 {
   // When using fake time, just poll.
   delta = 0
  }
  list := netpoll(delta) // 同步阻塞调用 netpoll，直至有可用的 goroutine
  atomic.Store64(&sched.pollUntil, 0)
  atomic.Store64(&sched.lastpoll, uint64(nanotime()))
  if faketime != 0 && list.empty() {
   // Using fake time and nothing is ready; stop M.
   // When all M's stop, checkdead will call timejump.
   stopm()
   goto top
  }
  lock(&sched.lock)
  _p_ = pidleget() // 查找是否有空闲的 P 可以来就绪的 goroutine
  unlock(&sched.lock)
  if _p_ == nil {
   injectglist(&list) // 如果当前没有空闲的 P，则把就绪的 goroutine 放入全局调度队列等待被执行
  } else {
   // 如果当前有空闲的 P，则 pop 出一个 g，返回给调度器去执行，
   // 并通过调用 injectglist 把剩下的 g 放入全局调度队列或者当前 P 本地调度队列
   acquirep(_p_)
   if !list.empty() {
    gp := list.pop()
    injectglist(&list)
    casgstatus(gp, _Gwaiting, _Grunnable)
    if trace.enabled {
     traceGoUnpark(gp, 0)
    }
    return gp, false
   }
   if wasSpinning {
    _g_.m.spinning = true
    atomic.Xadd(&sched.nmspinning, 1)
   }
   goto top
  }
 } else if pollUntil != 0 && netpollinited() {
  pollerPollUntil := int64(atomic.Load64(&sched.pollUntil))
  if pollerPollUntil == 0 || pollerPollUntil > pollUntil {
   netpollBreak()
  }
 }
 stopm()
 goto top
}
```

另外， `sysmon` 监控线程会在循环过程中检查距离上一次 `runtime.netpoll` 被调用是否超过了 10ms，若是则会去调用它拿到可运行的 goroutine 列表并通过调用 injectglist 把 g 列表放入全局调度队列或者当前 P 本地调度队列等待被执行：

```
// Always runs without a P, so write barriers are not allowed.
//
//go:nowritebarrierrec
func sysmon() {
  ...
  
  // poll network if not polled for more than 10ms
  lastpoll := int64(atomic.Load64(&sched.lastpoll))
  if netpollinited() && lastpoll != 0 && lastpoll+10*1000*1000 < now {
   atomic.Cas64(&sched.lastpoll, uint64(lastpoll), uint64(now))
   list := netpoll(0) // non-blocking - returns list of goroutines
   if !list.empty() {
    // Need to decrement number of idle locked M's
    // (pretending that one more is running) before injectglist.
    // Otherwise it can lead to the following situation:
    // injectglist grabs all P's but before it starts M's to run the P's,
    // another M returns from syscall, finishes running its G,
    // observes that there is no work to do and no other running M's
    // and reports deadlock.
    incidlelocked(-1)
    injectglist(&list)
    incidlelocked(1)
   }
  }
  
  ...
}
```

Go runtime 在程序启动的时候会创建一个独立的 M 作为监控线程，叫 `sysmon` ，这个线程为系统级的 daemon 线程，无需 P 即可运行， `sysmon` 每 20us~10ms 运行一次。 `sysmon` 中以轮询的方式执行以下操作（如上面的代码所示）：

1.  以非阻塞的方式调用 `runtime.netpoll` ，从中找出能从网络 I/O 中唤醒的 g 列表，并通过调用 injectglist 把 g 列表放入全局调度队列或者当前 P 本地调度队列等待被执行，调度触发时，有可能从这个全局 runnable 调度队列获取 g。然后再循环调用 `startm` ，直到所有 P 都不处于 `_Pidle` 状态。
2.  调用 `retake` ，抢占长时间处于 `_Psyscall` 状态的 P。

综上，Go 借助于 epoll/kqueue/iocp 和 runtime scheduler 等的帮助，设计出了自己的 I/O 多路复用 netpoller，成功地让 `Listener.Accept` / `conn.Read` / `conn.Write` 等方法从开发者的角度看来是同步模式。

### **Go netpoller 的价值**

通过前面对源码的分析，我们现在知道 Go netpoller 依托于 runtime scheduler，为开发者提供了一种强大的同步网络编程模式；然而，Go netpoller 存在的意义却远不止于此，Go netpoller I/O 多路复用搭配 Non-blocking I/O 而打造出来的这个原生网络模型，它最大的价值是把网络 I/O 的控制权牢牢掌握在 Go 自己的 runtime 里，关于这一点我们需要从 Go 的 runtime scheduler 说起，Go 的 G-P-M 调度模型如下：

![](https://pic1.zhimg.com/v2-c706ce72622f48e8d96952b9cd49464c_r.jpg)

G 在运行过程中如果被阻塞在某个 system call 操作上，那么不光 G 会阻塞，执行该 G 的 M 也会解绑 P(实质是被 sysmon 抢走了)，与 G 一起进入 sleep 状态。如果此时有 idle 的 M，则 P 与其绑定继续执行其他 G；如果没有 idle M，但仍然有其他 G 要去执行，那么就会创建一个新的 M。当阻塞在 system call 上的 G 完成 syscall 调用后，G 会去尝试获取一个可用的 P，如果没有可用的 P，那么 G 会被标记为 `_Grunnable` 并把它放入全局的 runqueue 中等待调度，之前的那个 sleep 的 M 将再次进入 sleep。

现在清楚为什么 netpoll 为什么一定要使用非阻塞 I/O 了吧？就是为了避免让操作网络 I/O 的 goroutine 陷入到系统调用从而进入内核态，因为一旦进入内核态，整个程序的控制权就会发生转移 (到内核)，不再属于用户进程了，那么也就无法借助于 Go 强大的 runtime scheduler 来调度业务程序的并发了；而有了 netpoll 之后，借助于非阻塞 I/O ，G 就再也不会因为系统调用的读写而 (长时间) 陷入内核态，当 G 被阻塞在某个 network I/O 操作上时，实际上它不是因为陷入内核态被阻塞住了，而是被 Go runtime 调用 gopark 给 park 住了，此时 G 会被放置到某个 wait queue 中，而 M 会尝试运行下一个 `_Grunnable` 的 G，如果此时没有 `_Grunnable` 的 G 供 M 运行，那么 M 将解绑 P，并进入 sleep 状态。当 I/O available，在 epoll 的 `eventpoll.rdr` 中等待的 G 会被放到 `eventpoll.rdllist` 链表里并通过 `netpoll` 中的 `epoll_wait` 系统调用返回放置到全局调度队列或者 P 的本地调度队列，标记为 `_Grunnable` ，等待 P 绑定 M 恢复执行。

### **Goroutine 的调度**

这一小节主要是讲处理网络 I/O 的 goroutines 阻塞之后，Go scheduler 具体是如何像前面几个章节所说的那样，避免让操作网络 I/O 的 goroutine 陷入到系统调用从而进入内核态的，而是封存 goroutine 然后让出 CPU 的使用权从而令 P 可以去调度本地调度队列里的下一个 goroutine 的。

**温馨提示**：这一小节属于延伸阅读，涉及到的知识点更偏系统底层，需要有一定的汇编语言基础才能通读，另外，这一节对 Go scheduler 的讲解仅仅涉及核心的一部分，不会把整个调度器都讲一遍（事实上如果真要解析 Go scheduler 的话恐怕重开一篇几万字的文章才能基本讲清楚。。。），所以也要求读者对 Go 的并发调度器有足够的了解，因此这一节可能会稍显深奥。当然这一节也可选择不读，因为通过前面的整个解析，我相信读者应该已经能够基本掌握 Go netpoller 处理网络 I/O 的核心细节了，以及能从宏观层面了解 netpoller 对业务 goroutines 的基本调度了。而这一节主要是通过对 goroutines 调度细节的剖析，能够加深读者对整个 Go netpoller 的彻底理解，接上前面几个章节，形成一个完整的闭环。如果对调度的底层细节没兴趣的话这也可以直接跳过这一节，对理解 Go netpoller 的基本原理影响不大，不过还是建议有条件的读者可以看看。

从源码可知，Go scheduler 的调度 goroutine 过程中所调用的核心函数链如下：

```
runtime.schedule --> runtime.execute --> runtime.gogo --> goroutine code --> runtime.goexit --> runtime.goexit1 --> runtime.mcall --> runtime.goexit0 --> runtime.schedule
```

> Go scheduler 会不断循环调用 `runtime.schedule()` 去调度 goroutines，而每个 goroutine 执行完成并退出之后，会再次调用 `runtime.schedule()`，使得调度器回到调度循环去执行其他的 goroutine，不断循环，永不停歇。  
> 当我们使用 `go` 关键字启动一个新 goroutine 时，最终会调用 `runtime.newproc` --> `runtime.newproc1`，来得到 g，`runtime.newproc1` 会先从 P 的 `gfree` 缓存链表中查找可用的 g，若缓存未生效，则会新创建 g 给当前的业务函数，最后这个 g 会被传给 `runtime.gogo` 去真正执行。  

这里首先需要了解一个 gobuf 的结构体，它用来保存 goroutine 的调度信息，是 `runtime.gogo` 的入参：

```
// gobuf 存储 goroutine 调度上下文信息的结构体
type gobuf struct {
 // The offsets of sp, pc, and g are known to (hard-coded in) libmach.
 //
 // ctxt is unusual with respect to GC: it may be a
 // heap-allocated funcval, so GC needs to track it, but it
 // needs to be set and cleared from assembly, where it's
 // difficult to have write barriers. However, ctxt is really a
 // saved, live register, and we only ever exchange it between
 // the real register and the gobuf. Hence, we treat it as a
 // root during stack scanning, which means assembly that saves
 // and restores it doesn't need write barriers. It's still
 // typed as a pointer so that any other writes from Go get
 // write barriers.
 sp   uintptr // Stack Pointer 栈指针
 pc   uintptr // Program Counter 程序计数器
 g    guintptr // 持有当前 gobuf 的 goroutine
 ctxt unsafe.Pointer
 ret  sys.Uintreg
 lr   uintptr
 bp   uintptr // for GOEXPERIMENT=framepointer
}
```

执行 `runtime.execute()`，进而调用 `runtime.gogo`：

```
func execute(gp *g, inheritTime bool) {
 _g_ := getg()

 // Assign gp.m before entering _Grunning so running Gs have an
 // M.
 _g_.m.curg = gp
 gp.m = _g_.m
 casgstatus(gp, _Grunnable, _Grunning)
 gp.waitsince = 0
 gp.preempt = false
 gp.stackguard0 = gp.stack.lo + _StackGuard
 if !inheritTime {
  _g_.m.p.ptr().schedtick++
 }

 // Check whether the profiler needs to be turned on or off.
 hz := sched.profilehz
 if _g_.m.profilehz != hz {
  setThreadCPUProfiler(hz)
 }

 if trace.enabled {
  // GoSysExit has to happen when we have a P, but before GoStart.
  // So we emit it here.
  if gp.syscallsp != 0 && gp.sysblocktraced {
   traceGoSysExit(gp.sysexitticks)
  }
  traceGoStart()
 }
 // gp.sched 就是 gobuf
 gogo(&gp.sched)
}
```

这里还需要了解一个概念：g0，Go G-P-M 调度模型中，g 代表 goroutine，而实际上一共有三种 g：

1.  执行用户代码的 g；
2.  执行调度器代码的 g，也即是 g0；
3.  执行 `runtime.main` 初始化工作的 main goroutine；

第一种 g 就是使用 `go` 关键字启动的 goroutine，也是我们接触最多的一类 g；第三种 g 是调度器启动之后用来执行的一系列初始化工作的，包括但不限于启动 `sysmon` 监控线程、内存初始化和启动 GC 等等工作；第二种 g 叫 g0，用来执行调度器代码，g0 在底层和其他 g 是一样的数据结构，但是性质上有很大的区别，首先 g0 的栈大小是固定的，比如在 Linux 或者其他 Unix-like 的系统上一般是固定 8MB，不能动态伸缩，而普通的 g 初始栈大小是 2KB，可按需扩展，g0 其实就是线程栈，我们知道每个线程被创建出来之时都需要操作系统为之分配一个初始固定的线程栈，就是前面说的 8MB 大小的栈，g0 栈就代表了这个线程栈，因此每一个 m 都需要绑定一个 g0 来执行调度器代码，然后跳转到执行用户代码的地方。

`runtime.gogo` 是真正去执行 goroutine 代码的函数，这个函数由汇编实现，为什么需要用汇编？因为 `gogo` 的工作是完成线程 M 上的堆栈切换：从系统堆栈 g0 切换成 goroutine `gp`，也就是 CPU 使用权和堆栈的切换，这种切换本质上是对 CPU 的 PC、SP 等寄存器和堆栈指针的更新，而这一类精度的底层操作别说是 Go，就算是最贴近底层的 C 也无法做到，这种程度的操作已超出所有高级语言的范畴，因此只能借助于汇编来实现。

`runtime.gogo` 在不同的 CPU 架构平台上的实现各不相同，但是核心原理殊途同归，我们这里选用 amd64 架构的汇编实现来分析，我会在关键的地方加上解释：

```
// func gogo(buf *gobuf)
// restore state from Gobuf; longjmp
TEXT runtime·gogo(SB), NOSPLIT, $16-8
	// 将第一个 FP 伪寄存器所指向的 gobuf 的第一个参数存入 BX 寄存器, 
	// gobuf 的一个参数即是 SP 指针
	MOVQ	buf+0(FP), BX
	MOVQ	gobuf_g(BX), DX  // 将 gp.sched.g 保存到 DX 寄存器
	MOVQ	0(DX), CX		// make sure g != nil

	// 将 tls (thread local storage) 保存到 CX 寄存器，然后把 gp.sched.g 放到 tls[0]，
	// 这样以后调用 getg() 之时就可以通过 TLS 直接获取到当前 goroutine 的 g 结构体实例，
	// 进而可以得到 g 所在的 m 和 p，TLS 里一开始存储的是系统堆栈 g0 的地址
	get_tls(CX)
	MOVQ	DX, g(CX)

	// 下面的指令则是对函数栈的 BP/SP 寄存器(指针)的存取，
	// 最后进入到指定的代码区域，执行函数栈帧
	MOVQ	gobuf_sp(BX), SP	// restore SP
	MOVQ	gobuf_ret(BX), AX
	MOVQ	gobuf_ctxt(BX), DX
	MOVQ	gobuf_bp(BX), BP

	// 这里是在清空 gp.sched，因为前面已经把 gobuf 里的字段值都存入了寄存器，
	// 所以 gp.sched 就可以提前清空了，不需要等到后面 GC 来回收，减轻 GC 的负担
	MOVQ	$0, gobuf_sp(BX)	// clear to help garbage collector
	MOVQ	$0, gobuf_ret(BX)
	MOVQ	$0, gobuf_ctxt(BX)
	MOVQ	$0, gobuf_bp(BX)

	// 把 gp.sched.pc 值放入 BX 寄存器
	// PC 指针指向 gogo 退出时需要执行的函数地址
	MOVQ	gobuf_pc(BX), BX
	// 用 BX 寄存器里的值去修改 CPU 的 IP 寄存器，
	// 这样就可以根据 CS:IP 寄存器的段地址+偏移量跳转到 BX 寄存器里的地址，也就是 gp.sched.pc
	JMP	BX
```

`runtime.gogo` 函数接收 `gp.sched` 这个 `gobuf` 结构体实例，其中保存了函数栈寄存器 SP/PC/BP，如果熟悉操作系统原理的话可以知道这些寄存器是 CPU 进行函数调用和返回时切换对应的函数栈帧所需的寄存器，而 goroutine 的执行和函数调用的原理是一致的，也是 CPU 寄存器的切换过程，所以这里的几个寄存器当前存的就是 G 的函数执行栈，当 goroutine 在处理网络 I/O 之时，如果恰好处于 I/O 就绪的状态的话，则正常完成 `runtime.gogo`，并在最后跳转到特定的地址，那么这个地址是哪里呢？

我们知道 CPU 执行函数的时候需要知道函数在内存里的代码段地址和偏移量，然后才能去取来函数栈执行，而典型的提供代码段地址和偏移量的寄存器就是 CS 和 IP 寄存器，而 `JMP BX` 指令则是用 BX 寄存器去更新 IP 寄存器，而 BX 寄存器里的值是 `gp.sched.pc`，那么这个 PC 指针究竟是指向哪里呢？让我们来看另一处源码。

众所周知，启动一个新的 goroutine 是通过 `go` 关键字来完成的，而 go compiler 会在编译期间利用 `**[cmd/compile/internal/gc.state.stmt](https://link.zhihu.com/?target=https%3A//github.com/golang/go/blob/1984ee00048b63eacd2155cd6d74a2d13e998272/src/cmd/compile/internal/gc/ssa.go%23L1039)**` 和 `**[cmd/compile/internal/gc.state.call](https://link.zhihu.com/?target=https%3A//github.com/golang/go/blob/1984ee00048b63eacd2155cd6d74a2d13e998272/src/cmd/compile/internal/gc/ssa.go%23L4328)**` 这两个函数将 `go` 关键字翻译成 `**[runtime.newproc](https://link.zhihu.com/?target=https%3A//github.com/golang/go/blob/1984ee00048b63eacd2155cd6d74a2d13e998272/src/runtime/proc.go%23L3523)**` 函数调用，而 `runtime.newproc` 接收了函数指针和其大小之后，会获取 goroutine 和调用处的程序计数器，接着再调用 `**[runtime.newproc1](https://link.zhihu.com/?target=https%3A//github.com/golang/go/blob/1984ee00048b63eacd2155cd6d74a2d13e998272/src/runtime/proc.go%23L3548)**`：

```
// Create a new g in state _Grunnable, starting at fn, with narg bytes
// of arguments starting at argp. callerpc is the address of the go
// statement that created this. The caller is responsible for adding
// the new g to the scheduler.
//
// This must run on the system stack because it's the continuation of
// newproc, which cannot split the stack.
//
//go:systemstack
func newproc1(fn *funcval, argp unsafe.Pointer, narg int32, callergp *g, callerpc uintptr) *g {
  ...
  
  memclrNoHeapPointers(unsafe.Pointer(&newg.sched), unsafe.Sizeof(newg.sched))
 newg.sched.sp = sp
 newg.stktopsp = sp
 // 把 goexit 函数地址存入 gobuf 的 PC 指针里
 newg.sched.pc = funcPC(goexit) + sys.PCQuantum // +PCQuantum so that previous instruction is in same function
 newg.sched.g = guintptr(unsafe.Pointer(newg))
 gostartcallfn(&newg.sched, fn)
 newg.gopc = callerpc
 newg.ancestors = saveAncestors(callergp)
 newg.startpc = fn.fn
 if _g_.m.curg != nil {
  newg.labels = _g_.m.curg.labels
 }
 if isSystemGoroutine(newg, false) {
  atomic.Xadd(&sched.ngsys, +1)
 }
 casgstatus(newg, _Gdead, _Grunnable)
  
  ...
}
```

这里可以看到，`newg.sched.pc` 被设置了 `runtime.goexit` 的函数地址，`newg` 就是后面 `runtime.gogo` 执行的 goroutine，因此 `runtime.gogo` 最后的汇编指令 `JMP BX`是跳转到了 `runtime.goexit`，让我们来继续看看这个函数做了什么：

```
// The top-most function running on a goroutine
// returns to goexit+PCQuantum. Defined as ABIInternal
// so as to make it identifiable to traceback (this
// function it used as a sentinel; traceback wants to
// see the func PC, not a wrapper PC).
TEXT runtime·goexit<ABIInternal>(SB),NOSPLIT,$0-0
	BYTE	$0x90	// NOP
	CALL	runtime·goexit1(SB)	// does not return
	// traceback from goexit1 must hit code range of goexit
	BYTE	$0x90	// NOP
```

这个函数也是汇编实现的，但是非常简单，就是直接调用 `runtime·goexit1`：

```
// Finishes execution of the current goroutine.
func goexit1() {
 if raceenabled {
  racegoend()
 }
 if trace.enabled {
  traceGoEnd()
 }
 mcall(goexit0)
}
```

调用 `runtime.mcall`函数：

```
// func mcall(fn func(*g))
// Switch to m->g0's stack, call fn(g).
// Fn must never return. It should gogo(&g->sched)
// to keep running g.

// 切换回 g0 的系统堆栈，执行 fn(g)
TEXT runtime·mcall(SB), NOSPLIT, $0-8
	// 取入参 funcval 对象的指针存入 DI 寄存器，此时 fn.fn 是 goexit0 的地址
	MOVQ	fn+0(FP), DI

	get_tls(CX)
	MOVQ	g(CX), AX	// save state in g->sched
	MOVQ	0(SP), BX	// caller's PC
	MOVQ	BX, (g_sched+gobuf_pc)(AX)
	LEAQ	fn+0(FP), BX	// caller's SP
	MOVQ	BX, (g_sched+gobuf_sp)(AX)
	MOVQ	AX, (g_sched+gobuf_g)(AX)
	MOVQ	BP, (g_sched+gobuf_bp)(AX)

	// switch to m->g0 & its stack, call fn
	MOVQ	g(CX), BX
	MOVQ	g_m(BX), BX

	// 把 g0 的栈指针存入 SI 寄存器，后面需要用到
	MOVQ	m_g0(BX), SI
	CMPQ	SI, AX	// if g == m->g0 call badmcall
	JNE	3(PC)
	MOVQ	$runtime·badmcall(SB), AX
	JMP	AX

	// 这两个指令是把 g0 地址存入到 TLS 里，
	// 然后从 SI 寄存器取出 g0 的栈指针，
	// 替换掉 SP 寄存器里存的当前 g 的栈指针
	MOVQ	SI, g(CX)	// g = m->g0
	MOVQ	(g_sched+gobuf_sp)(SI), SP	// sp = m->g0->sched.sp

	PUSHQ	AX
	MOVQ	DI, DX

	// 入口处的第一个指令已经把 funcval 实例对象的指针存入了 DI 寄存器，
	// 0(DI) 表示取出 DI 的第一个成员，即 goexit0 函数地址，再存入 DI
	MOVQ	0(DI), DI
	CALL	DI // 调用 DI 寄存器里的地址，即 goexit0
	POPQ	AX
	MOVQ	$runtime·badmcall2(SB), AX
	JMP	AX
	RET
```

可以看到 `runtime.mcall` 函数的主要逻辑是从当前 goroutine 切换回 g0 的系统堆栈，然后调用 fn(g)，此处的 g 即是当前运行的 goroutine，这个方法会保存当前运行的 G 的 PC/SP 到 g->sched 里，以便该 G 可以在以后被重新恢复执行，因为也涉及到寄存器和堆栈指针的操作，所以也需要使用汇编实现，该函数最后会在 g0 系统堆栈下执行 `runtime.goexit0`:

```
func goexit0(gp *g) {
 _g_ := getg()

 casgstatus(gp, _Grunning, _Gdead)
 if isSystemGoroutine(gp, false) {
  atomic.Xadd(&sched.ngsys, -1)
 }
 gp.m = nil
 locked := gp.lockedm != 0
 gp.lockedm = 0
 _g_.m.lockedg = 0
 gp.preemptStop = false
 gp.paniconfault = false
 gp._defer = nil // should be true already but just in case.
 gp._panic = nil // non-nil for Goexit during panic. points at stack-allocated data.
 gp.writebuf = nil
 gp.waitreason = 0
 gp.param = nil
 gp.labels = nil
 gp.timer = nil

 if gcBlackenEnabled != 0 && gp.gcAssistBytes > 0 {
  // Flush assist credit to the global pool. This gives
  // better information to pacing if the application is
  // rapidly creating an exiting goroutines.
  scanCredit := int64(gcController.assistWorkPerByte * float64(gp.gcAssistBytes))
  atomic.Xaddint64(&gcController.bgScanCredit, scanCredit)
  gp.gcAssistBytes = 0
 }

 dropg()

 if GOARCH == "wasm" { // no threads yet on wasm
  gfput(_g_.m.p.ptr(), gp)
  schedule() // never returns
 }

 if _g_.m.lockedInt != 0 {
  print("invalid m->lockedInt = ", _g_.m.lockedInt, "\n")
  throw("internal lockOSThread error")
 }
 gfput(_g_.m.p.ptr(), gp)
 if locked {
  // The goroutine may have locked this thread because
  // it put it in an unusual kernel state. Kill it
  // rather than returning it to the thread pool.

  // Return to mstart, which will release the P and exit
  // the thread.
  if GOOS != "plan9" { // See golang.org/issue/22227.
   gogo(&_g_.m.g0.sched)
  } else {
   // Clear lockedExt on plan9 since we may end up re-using
   // this thread.
   _g_.m.lockedExt = 0
  }
 }
 schedule()
}
```

`runtime.goexit0` 的主要工作是就是

1.  利用 CAS 操作把 g 的状态从 `_Grunning` 更新为 `_Gdead`；
2.  对 g 做一些清理操作，把一些字段值置空；
3.  调用 `runtime.dropg` 解绑 g 和 m；
4.  把 g 放入 p 存储 g 的 `gfree` 链表作为缓存，后续如果需要启动新的 goroutine 则可以直接从链表里取而不用重新初始化分配内存。
5.  最后，调用 `runtime.schedule()` 再次进入调度循环去调度新的 goroutines，永不停歇。

另一方面，如果 goroutine 处于 I/O 不可用状态，我们前面已经分析过 netpoller 利用非阻塞 I/O + I/O 多路复用避免了陷入系统调用，所以此时会调用 `runtime.gopark` 并把 goroutine 暂时封存在用户态空间，并休眠当前的 goroutine，因此不会阻塞 `runtime.gogo` 的汇编执行，而是通过 `runtime.mcall` 调用 `runtime.park_m`：

```
func gopark(unlockf func(*g, unsafe.Pointer) bool, lock unsafe.Pointer, reason waitReason, traceEv byte, traceskip int) {
 if reason != waitReasonSleep {
  checkTimeouts() // timeouts may expire while two goroutines keep the scheduler busy
 }
 mp := acquirem()
 gp := mp.curg
 status := readgstatus(gp)
 if status != _Grunning && status != _Gscanrunning {
  throw("gopark: bad g status")
 }
 mp.waitlock = lock
 mp.waitunlockf = unlockf
 gp.waitreason = reason
 mp.waittraceev = traceEv
 mp.waittraceskip = traceskip
 releasem(mp)
 // can't do anything that might move the G between Ms here.
 mcall(park_m)
}

func park_m(gp *g) {
 _g_ := getg()

 if trace.enabled {
  traceGoPark(_g_.m.waittraceev, _g_.m.waittraceskip)
 }

 casgstatus(gp, _Grunning, _Gwaiting)
 dropg()

 if fn := _g_.m.waitunlockf; fn != nil {
  ok := fn(gp, _g_.m.waitlock)
  _g_.m.waitunlockf = nil
  _g_.m.waitlock = nil
  if !ok {
   if trace.enabled {
    traceGoUnpark(gp, 2)
   }
   casgstatus(gp, _Gwaiting, _Grunnable)
   execute(gp, true) // Schedule it back, never returns.
  }
 }
 schedule()
}
```

`runtime.mcall` 方法我们在前面已经介绍过，它主要的工作就是是从当前 goroutine 切换回 g0 的系统堆栈，然后调用 fn(g)，而此时 `runtime.mcall` 调用执行的是 `runtime.park_m`，这个方法里会利用 CAS 把当前运行的 goroutine -- gp 的状态 从 `_Grunning` 切换到 `_Gwaiting`，表明该 goroutine 已进入到等待唤醒状态，此时封存和休眠 G 的操作就完成了，只需等待就绪之后被重新唤醒执行即可。最后调用 `runtime.schedule()` 再次进入调度循环，去执行下一个 goroutine，充分利用 CPU。

至此，我们完成了对 Go netpoller 原理剖析的整个闭环。

### **Go netpoller 的问题**

Go netpoller 的设计不可谓不精巧、性能也不可谓不高，配合 goroutine 开发网络应用的时候就一个字：爽。因此 Go 的网络编程模式是及其简洁高效的，然而，没有任何一种设计和架构是完美的， `goroutine-per-connection` 这种模式虽然简单高效，但是在某些极端的场景下也会暴露出问题：goroutine 虽然非常轻量，它的自定义栈内存初始值仅为 2KB，后面按需扩容；海量连接的业务场景下， `goroutine-per-connection` ，此时 goroutine 数量以及消耗的资源就会呈线性趋势暴涨，虽然 Go scheduler 内部做了 g 的缓存链表，可以一定程度上缓解高频创建销毁 goroutine 的压力，但是对于瞬时性暴涨的长连接场景就无能为力了，大量的 goroutines 会被不断创建出来，从而对 Go runtime scheduler 造成极大的调度压力和侵占系统资源，然后资源被侵占又反过来影响 Go scheduler 的调度，进而导致性能下降。

### **Reactor 网络模型**

目前 Linux 平台上主流的高性能网络库 / 框架中，大都采用 Reactor 模式，比如 netty、libevent、libev、ACE，POE(Perl)、Twisted(Python) 等。

Reactor 模式本质上指的是使用 `I/O 多路复用(I/O multiplexing) + 非阻塞 I/O(non-blocking I/O)` 的模式。

通常设置一个主线程负责做 event-loop 事件循环和 I/O 读写，通过 select/poll/epoll_wait 等系统调用监听 I/O 事件，业务逻辑提交给其他工作线程去做。而所谓『非阻塞 I/O』的核心思想是指避免阻塞在 read() 或者 write() 或者其他的 I/O 系统调用上，这样可以最大限度的复用 event-loop 线程，让一个线程能服务于多个 sockets。在 Reactor 模式中，I/O 线程只能阻塞在 I/O multiplexing 函数上（select/poll/epoll_wait）。

Reactor 模式的基本工作流程如下：

*   Server 端完成在 `bind&listen` 之后，将 listenfd 注册到 epollfd 中，最后进入 event-loop 事件循环。循环过程中会调用 `select/poll/epoll_wait` 阻塞等待，若有在 listenfd 上的新连接事件则解除阻塞返回，并调用 `socket.accept` 接收新连接 connfd，并将 connfd 加入到 epollfd 的 I/O 复用（监听）队列。
*   当 connfd 上发生可读 / 可写事件也会解除 `select/poll/epoll_wait` 的阻塞等待，然后进行 I/O 读写操作，这里读写 I/O 都是非阻塞 I/O，这样才不会阻塞 event-loop 的下一个循环。然而，这样容易割裂业务逻辑，不易理解和维护。
*   调用 `read` 读取数据之后进行解码并放入队列中，等待工作线程处理。
*   工作线程处理完数据之后，返回到 event-loop 线程，由这个线程负责调用 `write` 把数据写回 client。

accept 连接以及 conn 上的读写操作若是在主线程完成，则要求是非阻塞 I/O，因为 Reactor 模式一条最重要的原则就是：I/O 操作不能阻塞 event-loop 事件循环。**实际上 event loop 可能也可以是多线程的，只是一个线程里只有一个 select/poll/epoll_wait**。

上面提到了 Go netpoller 在某些场景下可能因为创建太多的 goroutine 而过多地消耗系统资源，而在现实世界的网络业务中，服务器持有的海量连接中在极短的时间窗口内只有极少数是 active 而大多数则是 idle，就像这样（非真实数据，仅仅是为了比喻）：

![](https://pic2.zhimg.com/v2-a88718de089d5ad5e6564dc51ceec825_r.jpg)

那么为每一个连接指派一个 goroutine 就显得太过奢侈了，而 Reactor 模式这种利用 I/O 多路复用进而只需要使用少量线程即可管理海量连接的设计就可以在这样网络业务中大显身手了：

![](https://pic2.zhimg.com/v2-518bd82b20909d5d25d116568ab89be9_r.jpg)

在绝大部分应用场景下，我推荐大家还是遵循 Go 的 best practices，使用原生的 Go 网络库来构建自己的网络应用。然而，在某些极度追求性能、压榨系统资源以及技术栈必须是原生 Go （不考虑 C/C++ 写中间层而 Go 写业务层）的业务场景下，我们可以考虑自己构建 Reactor 网络模型。

### **gnet**

**[gnet](https://link.zhihu.com/?target=https%3A//gnet.host/)** 是一个基于事件驱动的高性能和轻量级网络框架。它直接使用 **[epoll](https://link.zhihu.com/?target=https%3A//en.wikipedia.org/wiki/Epoll)** 和 **[kqueue](https://link.zhihu.com/?target=https%3A//en.wikipedia.org/wiki/Kqueue)** 系统调用而非标准 Go 网络包：**[net](https://link.zhihu.com/?target=https%3A//golang.org/pkg/net/)** 来构建网络应用，它的工作原理类似两个开源的网络库：**[netty](https://link.zhihu.com/?target=https%3A//github.com/netty/netty)** 和 **[libuv](https://link.zhihu.com/?target=https%3A//github.com/libuv/libuv)**，这也使得`gnet` 达到了一个远超 Go **[net](https://link.zhihu.com/?target=https%3A//golang.org/pkg/net/)** 的性能表现。

`gnet` 设计开发的初衷不是为了取代 Go 的标准网络库：**[net](https://link.zhihu.com/?target=https%3A//golang.org/pkg/net/)**，而是为了创造出一个类似于 **[Redis](https://link.zhihu.com/?target=http%3A//redis.io/)**、**[Haproxy](https://link.zhihu.com/?target=http%3A//www.haproxy.org/)** 能高效处理网络包的 Go 语言网络服务器框架。

`gnet` 的卖点在于它是一个高性能、轻量级、非阻塞的纯 Go 实现的传输层（TCP/UDP/Unix Domain Socket）网络框架，开发者可以使用 `gnet` 来实现自己的应用层网络协议 (HTTP、RPC、Redis、WebSocket 等等)，从而构建出自己的应用层网络应用：比如在 `gnet` 上实现 HTTP 协议就可以创建出一个 HTTP 服务器 或者 Web 开发框架，实现 Redis 协议就可以创建出自己的 Redis 服务器等等。

**[gnet](https://link.zhihu.com/?target=https%3A//github.com/panjf2000/gnet)**，在某些极端的网络业务场景，比如海量连接、高频短连接、网络小包等等场景，**[gnet](https://link.zhihu.com/?target=https%3A//github.com/panjf2000/gnet)** 在性能和资源占用上都远超 Go 原生的 **[net](https://link.zhihu.com/?target=https%3A//golang.org/pkg/net/)** 包（基于 netpoller）。

`gnet` 已经实现了 `Multi-Reactors` 和 `Multi-Reactors + Goroutine Pool` 两种网络模型，也得益于这些网络模型，使得 `gnet` 成为一个高性能和低损耗的 Go 网络框架：

![](https://pic3.zhimg.com/v2-79180f7f9dc90dd776102dec874a0b02_r.jpg)![](https://pic4.zhimg.com/v2-e5570edbba9781e932a9f38e942a2823_r.jpg)

### **功能**

*   [x] **[高性能](https://zhuanlan.zhihu.com/write#-%E6%80%A7%E8%83%BD%E6%B5%8B%E8%AF%95)** 的基于多线程 / Go 程网络模型的 event-loop 事件驱动
*   [x] 内置 goroutine 池，由开源库 **[ants](https://link.zhihu.com/?target=https%3A//github.com/panjf2000/ants)** 提供支持
*   [x] 内置 bytes 内存池，由开源库 **[bytebufferpool](https://link.zhihu.com/?target=https%3A//github.com/valyala/bytebufferpool)** 提供支持
*   [x] 整个生命周期是无锁的
*   [x] 简单易用的 APIs
*   [x] 基于 Ring-Buffer 的高效且可重用的内存 buffer
*   [x] 支持多种网络协议 / IPC 机制：`TCP`、`UDP` 和 `Unix Domain Socket`
*   [x] 支持多种负载均衡算法：`Round-Robin(轮询)`、`Source-Addr-Hash(源地址哈希)` 和 `Least-Connections(最少连接数)`
*   [x] 支持两种事件驱动机制：**Linux** 里的 `epoll` 以及 **FreeBSD/DragonFly/Darwin** 里的 `kqueue`
*   [x] 支持异步写操作
*   [x] 灵活的事件定时器
*   [x] SO_REUSEPORT 端口重用
*   [x] 内置多种编解码器，支持对 TCP 数据流分包：LineBasedFrameCodec, DelimiterBasedFrameCodec, FixedLengthFrameCodec 和 LengthFieldBasedFrameCodec，参考自 **[netty codec](https://link.zhihu.com/?target=https%3A//netty.io/4.1/api/io/netty/handler/codec/package-summary.html)**，而且支持自定制编解码器
*   [x] 支持 Windows 平台，基于 IOCP 事件驱动机制 Go 标准网络库
*   [ ] 实现 `gnet` 客户端

**参考 & 延伸阅读**

*   **[The Go netpoller](https://link.zhihu.com/?target=https%3A//morsmachine.dk/netpoller)**
*   **[Nonblocking I/O](https://link.zhihu.com/?target=https%3A//copyconstruct.medium.com/nonblocking-i-o-99948ad7c957)**
*   **[epoll(7) — Linux manual page](https://link.zhihu.com/?target=https%3A//man7.org/linux/man-pages/man7/epoll.7.html)**
*   **[I/O Multiplexing: The select and poll Functions](https://link.zhihu.com/?target=https%3A//notes%253Ccode%253E.shich%253C/code%253Eao.io%253Ccode%253E/unp%253C/code%253E/ch6/)**
*   **[The method to epoll’s madness](https://link.zhihu.com/?target=https%3A//copyconstruct.medium.com/the-method-to-epolls-madness-d9d2d6378642)**
*   **[Scalable Go Scheduler Design Doc](https://link.zhihu.com/?target=https%3A//docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw)**
*   **[Scheduling In Go : Part I - OS Scheduler](https://link.zhihu.com/?target=https%3A//www.ardanlabs.com/blog/2018/08/scheduling-in-go-part1.html)**
*   **[Scheduling In Go : Part II - Go Scheduler](https://link.zhihu.com/?target=https%3A//www.ardanlabs.com/blog/2018/08/scheduling-in-go-part2.html)**
*   **[Scheduling In Go : Part III - Concurrency](https://link.zhihu.com/?target=https%3A//www.ardanlabs.com/blog/2018/12/scheduling-in-go-part3.html)**
*   **[Goroutines, Nonblocking I/O, And Memory Usage](https://link.zhihu.com/?target=https%3A//eklitzke.org/goroutines-nonblocking-io-and-memory-usage)**
*   **[IO 多路复用与 Go 网络库的实现](https://link.zhihu.com/?target=https%3A//ninokop.github.io/2018/02/18/go-net/)**
*   **[关于 select 函数中 timeval 和 fd_set 重新设置的问题](https://link.zhihu.com/?target=https%3A//blog.csdn.net/starflame/article/details/7860091)**
*   **[A Million WebSockets and Go](https://link.zhihu.com/?target=https%3A//www.freecodecamp.org/news/million-websockets-and-go-cc58418460bb/)**
*   **[Going Infinite, handling 1M websockets connections in Go](https://link.zhihu.com/?target=https%3A//speakerdeck.com/eranyanay/going-infinite-handling-1m-websockets-connections-in-go)**
*   [字节跳动在 Go 网络库上的实践](https://link.zhihu.com/?target=https%3A//mp.weixin.qq.com/s/wSaJYg-HqnYY4SdLA2Zzaw)

**更多干货尽在[腾讯技术](https://www.zhihu.com/org/teng-xun-ji-zhu-gong-cheng)，官方微信交流群已建立，交流讨论可加：Journeylife1900（备注腾讯技术） 。**