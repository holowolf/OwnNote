## Golang Basic

### net.Conn interface function

#### 发送端上面

发送字符串需要三个步骤:
1. 打开对应接收进程的连接
2. 写字符串
3. 关闭连接

net包提供了一对实现这个功能的方法.
1. ResolveTcpAddr() : 该函数返回TCP终端地址.
2. DialTCP() : 类似于TCP网络的拨号.

两方法在go源码中src/net/tcpsock.go中定义的.
```go
func ResolveTCPAddr(network, address string) (*TCPAddr, error) {
	switch network {
	case "tcp", "tcp4", "tcp6":
	case "": // a hint wildcard for Go 1.0 undocumented behavior
		network = "tcp"
	default:
		return nil, UnknownNetworkError(network)
	}
	addrs, err := DefaultResolver.internetAddrList(context.Background(), network, address)
	if err != nil {
		return nil, err
	}
	return addrs.forResolve(network, address).(*TCPAddr), nil
}
```

ResolveTCPAddr() 接收两个字符串参数
* network 必须式tcp 网络名 tcp, tcp4, tcp6
* address TCP地址字符串, 如果它不是字面量的ip地址或端口号不是字面量的端口号.
ResolveTCPAddr会将传入的地址解决成TCP的终端地址. 否则将一对字面量IP地址和端口数字作为地址.
address参数可以使用host名称, 但是不推荐这样做，因为它最多会返回host名字的一个IP地址。

ResolveTCPAddr()接收带代表TCP地址的字符串(localhost:80, 127.0.0.1:80,或 [::1]:80)都代表本机80端口)
返回net.TCPAddr指针 nil 如果地址不能有效的解析为tcp地址就会返回(nil error).

```go
func DialTCP(network string, laddr, raddr *TCPAddr) (*TCPConn, error) {
    switch network {
    case "tcp", "tcp4", "tcp6":
    default:
        return nil, &OpError{Op: "dial", Net: network, Source: laddr.opAddr(), Addr: raddr.opAddr(), Err: UnknownNetworkError(network)}
    }
    if raddr == nil {
        return nil, &OpError{Op: "dial", Net: network, Source: laddr.opAddr(), Addr: nil, Err: errMissingAddress}
    }
    c, err := dialTCP(context.Background(), network, laddr, raddr)
    if err != nil {
        return nil, &OpError{Op: "dial", Net: network, Source: laddr.opAddr(), Addr: raddr.opAddr(), Err: err}
    }
    return c, nil
}
```

DialTCP()函数接收三个参数
* network:这个参数和ResolveTCPAddr的network参数一样, 必须式TCP网络名.
* laddr: TCPAddr类型的指针, 代表本地TCP地址
* raddr: TCPAddr类型的指针, 代表式远程TCP地址

他会连接拨号两个TCP地址, 并返回这个连接作为net.TCPConn对象返回(连接失败返回error).
如果我们不需要对Dial设置有过多控制, 那么我们就可以使用Dial()代替.

```go
func Dial(network, address string) (Conn, error) {
    var d Dialer
    return d.Dial(network, address)
}
```

Dial()函数接收一个TCP地址, 返回一个一般的net.Conn.如果需要在TCP上可用功能可以返回TCP变体(DialTCP, TCPConn, TCPAddr).
成功拨号后, 如上所述新的连接可以与其他的输入输出流同时对待.
可以将连接包装进buffio.ReadWrite中, 这样可以使用各种ReadWriter方法,例如ReadString(), ReadBytes,WriteString等.


```go
func Open(addr string) (*bufio ReadWriter, error) {
	conn, err := net.Dial("tcp", addr)
	if err != nil {
		return nil, errors.Wrap(err, "Dialing"+addr+"failed")
	}
	// 将net.Conn对象包装到bufio.ReadWriter中
	return bufio.NewReadWrite(bufio.NewReader(conn), bufio.NewWriter(conn)), nil
}
```

Write在写入后需要调用Flush()方法，这样所有的数据才会刷到底层网络中
最后每个连接对象都有一个close方法来终止通信

微调(fine tuning)

Dialer结构
```go
type Dialer struct {
    Timeout time.Duration
    Deadline time.Time
    LocalAddr Addr
    DualStack bool
    FallbackDelay time.Duration
    KeepAlive time.Duration
    Resolver *Resolver
    Cancel <-chan struct{}
}
```

* Timeout: 拨号等待连接结束的最大时间数。如果同时设置了Deadline, 可以更早失败。默认没有超时。 当使用TCP并使用多个IP地址拨号主机名，超时会在它们之间划分。使用或不使用超时，操作系统都可以强迫更早超时。例如，TCP超时一般在3分钟左右。
* Deadline: 是拨号即将失败的绝对时间点。如果设置了Timeout, 可能会更早失败。0值表示没有截止期限， 或者依赖操作系统或使用Timeout选项。
* LocalAddr: 是拨号一个地址时使用的本地地址。这个地址必须是要拨号的network地址完全兼容的类型。如果为nil, 会自动选择一个本地地址。
* DualStack: 这个属性可以启用RFC 6555兼容的"欢乐眼球(Happy Eyeballs)"拨号，当network是tcp时，address参数中的host可以被解析被IPv4和IPv6地址。这样就允许客户端容忍(tolerate)一个地址家族的网络规定稍微打破一下。
* FallbackDelay: 当DualStack启用的时候, 指定在产生回退连接之前需要等待的时间。如果设置为0， 默认使用延时300ms。
* KeepAlive: 为活动网络连接指定保持活动的时间。如果设置为0，没有启用keep-alive。不支持keep-alive的网络协议会忽略掉这个字段。
* Resolver: 可选项，指定使用的可替代resolver。
* Cancel: 可选通道，它的闭包表示拨号应该被取消。不是所有的拨号类型都支持拨号取消。 已废弃，可使用DialContext代替。

有两个选项可以微调

因此Dialer借口提供了可以微调的两个方面的选项.

* DeadLine和Timeout选项: 用于不成功拨号的超时设置。
* KeepAlive选项: 管理连接的使用寿命(life span)。

```go
type Conn interface {
    Read(b []byte) (n int, err error)
    Write(b []byte) (n int, err error)
    Close() error
    LocalAddr() Addr
    RemoteAddr() Addr
    SetDeadline(t time.Time) error
    SetReadDeadline(t time.Time) error
    SetWriteDeadline(t time.Time) error
}
```

net.Conn接口是面向流的一般的网络连接，它具有下面这些接口方法：

* Read(): 从连接上读取数据。
* Write(): 向连接上写入数据。
* Close(): 关闭连接。
* LocalAddr(): 返回本地网络地址。
* RemoteAddr(): 返回远程网络地址。
* SetDeadline(): 设置连接相关的读写最后期限。等价于同时调用SetReadDeadline()和SetWriteDeadline()。
* SetReadDeadline(): 设置将来的读调用和当前阻塞的读调用的超时最后期限。
* SetWriteDeadline(): 设置将来写调用以及当前阻塞的写调用的超时最后期限。

Conn接口也有deadline设置; 有对整个连接的(SetDeadLine())，也有特定读写调用的(SetReadDeadLine()和SetWriteDeadLine())。

注意deadline是(wallclock)时间固定点。和timeout不同，它们新活动之后不会重置。因此连接上的每个活动必须设置新的deadline。

下面的样本代码没有使用deadline, 因为它足够简单，我们可以很容易看到什么时候会被卡住。Ctrl-C时我们手动触发deadline的工具。

#### 接收端上

接收端步骤
1. 对本地端口打开监听.
2. 当请求到来时, 产生(spawn)goroutine来处理请求
3. 当goroutine中, 读取数据. 也可以选择性的发送响应.
4. 关闭连接

监听需要指定本地监听的端口号。一般来说，监听应用程序(也叫server)宣布监听的端口号，如果提供标准服务, 那么使用这个服务对应的相关端口。
例如，web服务通常监听80来伺服HTTP, 443端口伺服HTTPS请求。 SSH守护默认监听22端口, WHOIS服务使用端口43。

```go
type Listener interface {
    // Accept waits for and returns the next connection to the listener.
    Accept() (Conn, error)

    // Close closes the listener.
    // Any blocked Accept operations will be unblocked and return errors.
    Close() error

    // Addr returns the listener's network address.
    Addr() Addr
}
```

```go
func Listen(network, address string) (Listener, error) {
    addrs, err := DefaultResolver.resolveAddrList(context.Background(), "listen", network, address, nil)
    if err != nil {
        return nil, &OpError{Op: "listen", Net: network, Source: nil, Addr: nil, Err: err}
    }
    var l Listener
    switch la := addrs.first(isIPv4).(type) {
    case *TCPAddr:
        l, err = ListenTCP(network, la)
    case *UnixAddr:
        l, err = ListenUnix(network, la)
    default:
        return nil, &OpError{Op: "listen", Net: network, Source: nil, Addr: la, Err: &AddrError{Err: "unexpected address type", Addr: address}}
    }
    if err != nil {
        return nil, err // l is non-nil interface containing nil pointer
    }
    return l, nil
}
```

net包实现服务端的核心部分：
net.Listen() 在给定的本地网络地址上来创建新的监听器. 如果只传端口号给它, 如":51000"那么监听器
会监听所有可以的网络接口. 这样相当方便, 因为计算机通常至少提供两个活动接口, 回环接口和最少一个真实网卡. 
这个函数成功的话返回Listener.

Listener接口有一个Accept()方法用来等待请求进来。然后它接受请求，并给调用者返回新的连接。
Accept()一般来说都是在循环中调用，能够同时服务多个连接。每个连接可以由一个单独的goroutine处理，正如下面代码所示的。

















