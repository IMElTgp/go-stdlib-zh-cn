# 函数

## func JoinHostPort

```go
func JoinHostPort(host, port string) string
```

`JoinHostPort`将`host`和`port`组合成符合`"host:port"`格式的网络地址。如果`host`包含冒号（如字面量IPv6地址中），`JoinHostPort`返回`"[host]:port"`。

关于对`host`和`port`参数的介绍，见[func Dial](#func-dial)。

## func SplitHostPort

```go
func SplitHostPort(hostport string) (host, port string, err error)
```

`SplitHostPort`将`"host:port"`、`host%zone:port`、`"[host]:port"`或`"[host%zone]:port"`格式的网络地址拆解为`host`或`host%zone`，以及`port`。

`hostport`中的字面量IPv6地址必须由中括号包裹，如`"[::1]:80"`、`"[::1%1o0]:80"`。

关于对`host`和`port`参数以及`host`和`port`结果的介绍，见[func Dial](#func-dial)。

# 类型

## type Conn

```go
type Conn interface {
    // Read reads data from the connection.
    // Read can be made to time out and return an error after a fixed
    // time limit; see SetDeadline and SetReadDeadline.
    Read(b []byte) (n int, err error)

    // Write writes data to the connection.
    // Write can be made to time out and return an error after a fixed
    // time limit; see SetDeadline and SetWriteDeadline.
    Write(b []byte) (n int, err error)

    // Close closes the connection.
    // Any blocked Read or Write operations will be unblocked and return errors.
    Close() error

    // LocalAddr returns the local network address, if known.
    LocalAddr() Addr

    // RemoteAddr returns the remote network address, if known.
    RemoteAddr() Addr

    // SetDeadline sets the read and write deadlines associated
    // with the connection. It is equivalent to calling both
    // SetReadDeadline and SetWriteDeadline.
    //
    // A deadline is an absolute time after which I/O operations
    // fail instead of blocking. The deadline applies to all future
    // and pending I/O, not just the immediately following call to
    // Read or Write. After a deadline has been exceeded, the
    // connection can be refreshed by setting a deadline in the future.
    //
    // If the deadline is exceeded a call to Read or Write or to other
    // I/O methods will return an error that wraps os.ErrDeadlineExceeded.
    // This can be tested using errors.Is(err, os.ErrDeadlineExceeded).
    // The error's Timeout method will return true, but note that there
    // are other possible errors for which the Timeout method will
    // return true even if the deadline has not been exceeded.
    //
    // An idle timeout can be implemented by repeatedly extending
    // the deadline after successful Read or Write calls.
    //
    // A zero value for t means I/O operations will not time out.
    SetDeadline(t time.Time) error

    // SetReadDeadline sets the deadline for future Read calls
    // and any currently-blocked Read call.
    // A zero value for t means Read will not time out.
    SetReadDeadline(t time.Time) error

    // SetWriteDeadline sets the deadline for future Write calls
    // and any currently-blocked Write call.
    // Even if write times out, it may return n > 0, indicating that
    // some of the data was successfully written.
    // A zero value for t means Write will not time out.
    SetWriteDeadline(t time.Time) error
}
```

`Conn`是一种面向流的网络连接。

多个goroutine可以并发调用`Conn`实现的方法。

## func Dial

```go
func Dial(network, address string) (Conn, error)
```

调用`Dial`可与命名网络的目标地址建立连接。

知名网络类型包括：`"tcp"`、`"tcp4"`（仅IPv4）、`"tcp6"`（仅IPv6）、`"udp"`、`"udp4"`（仅IPv4）、`"udp6"`（仅IPv6）、`"ip"`、`"ip4"`（仅IPv4）、`"ip6"`（仅IPv6）、`"unix"`、`"unixgram"`、`"unixpacket"`。

对于`TCP`和`UDP`网络类型，地址采用`"host:port"`格式。其中`host`必须是字面值IP地址或可解析为IP地址的主机名（host name）。`port`必须是字面值端口号或服务名（service name）。如果`host`为字面值IPv6地址，则必须被中括号包裹，如`"[2001:db8::1]:80"`或`"[fe80::1%zone]:80"`。区域标识符（zone）用于指定字面量IPv6地址的作用域（scope），其定义遵循[RFC 4007](https://rfc-editor.org/rfc/rfc4007.html)标准。[`JoinHostPort`](#func-joinhostport)和[`SplitHostPort`](#func-splithostport)函数操作符合该地址格式的一对`host`和`port`。使用`TCP`协议且主机名解析到多个IP地址时，`Dial`将逐个尝试与这些IP地址建立连接直到成功建立连接。

例：

```go
Dial("tcp", "golang.org:http")
Dial("tcp", "192.0.2.1:http")
Dial("tcp", "198.51.100.1:80")
Dial("udp", "[2001:db8::1]:domain")
Dial("udp", "[fe80::1%lo0]:53")
Dial("tcp", ":80")
```

对于IP网络，其类型必须为`"ip"`、`"ip4"`或`"ip6"`，后接冒号以及字面值协议号或协议名，且地址必须符合`"host"`的格式。`host`必须是字面量IP地址或含作用域的字面量IPv6地址。操作系统对`0`或`255`等非常规协议号的处理机制取决于具体系统实现。

例：

```go
Dial("ip4:1", "192.0.2.1")
Dial("ip6:ipv6-icmp", "2001:db8::1")
Dial("ip6:58", "fe80::1%lo0")
```

对于`TCP`、`UDP`以及`IP`协议网络，如果主机名为空或为未指定IP地址的字面值（如`TCP`/`UDP`的`":80"`、`"0.0.0.0:80"`、`"[::]:80"`或`IP`的`""`、`"0.0.0.0"`、`"::"`），系统将默认绑定到本地所有可用接口。

对于Unix网络，地址必须为文件系统路径。

## func DialTimeout

```go
func DialTimeout(network, address string, timeout time.Duration) (Conn, error)
```

`Dial`的作用类似于[`Dial`](#func-dial)但增加了超时控制（timeout）。

超时期限包含必要的域名解析时间。当使用`TCP`协议且地址参数中的主机名解析到多个IP地址时，超时时间将按比例分配给每个连续的连接尝试，确保每次拨号（dial）获得相应的连接时间片。

关于对`network`和`address`参数的介绍，见[func Dial](#func-dial)。

## func FileConn

```go
func FileConn(f *os.File) (c Conn, err error)
```

`FileConn`返回对应于已打开文件`f`的网络连接的一份拷贝。完成时由调用者负责关闭`f`。关闭`c`和关闭`f`互不影响。
