# Tracing HTTP Requests with net/http/httptrace

# 使用 Go 的 net/http/httptrace 追踪 HTTP 请求

`net/http/httptrace` 自 Go 1.7 起就已包含在标准库中，但我接触过的大多数 Go 开发者从未使用过它。它为发出的 HTTP 请求中那些通常从传输层外部无法观察到的关键节点提供了钩子：DNS 解析、连接获取、TLS 握手、字节写入网络的时刻，以及收到第一个响应字节的时刻。

有趣的是它的接入方式。`http.Client` 上没有 `Tracer` 接口，也没有需要注册的中间件。你将一个 `ClientTrace` 附加到 `context.Context` 上，传输层在需要的节点通过 `httptrace.ContextClientTrace` 将其取出。我想先讲解这个设计决策，因为它解释了该包如何与标准库的其余部分组合使用，然后用它构建两个东西：一个类似 `curl --trace` 风格的命令行工具，以及一个可复用的 `http.RoundTripper`，为每个请求记录耗时信息。

## 为什么用 Context 而不是接口

请求追踪的显而易见的设计方式是定义一个 `Tracer` 接口，在 `http.Client` 或 `http.Transport` 上添加一个 `Tracer` 字段，然后在传输层内部调用其方法。大多数语言大致就是这样处理的。

Go 的标准库不是这样工作的。`httptrace.WithClientTrace` 返回一个携带 `*ClientTrace` 的新 context，你通过 `req.WithContext(ctx)` 将该 context 附加到请求上，传输层在需要的节点通过 `httptrace.ContextClientTrace` 将 trace 取出。

```go
trace := &httptrace.ClientTrace{
    DNSStart: func(info httptrace.DNSStartInfo) {
        fmt.Printf("DNS start: %s\n", info.Host)
    },
    DNSDone: func(info httptrace.DNSDoneInfo) {
        fmt.Printf("DNS done: %v\n", info.Addrs)
    },
}
ctx := httptrace.WithClientTrace(context.Background(), trace)
req, _ := http.NewRequestWithContext(ctx, http.MethodGet, "https://example.com", nil)
http.DefaultClient.Do(req)
```

这种方式不太常见，但收益明显。trace 随请求传递，因此任何转发 context 的中间件都会自动传播追踪功能。客户端上没有共享的可变状态，所以同一个 `http.Client` 发出的并发请求可以携带不同的 trace。而且如果没有人附加 trace，传输层会完全忽略它，因此未使用时的开销仅仅是一次 nil 检查。

`ClientTrace` 本身是一个包含可选函数字段的结构体：

```go
type ClientTrace struct {
    GetConn              func(hostPort string)
    GotConn              func(GotConnInfo)
    PutIdleConn          func(err error)
    GotFirstResponseByte func()
    Got100Continue       func()
    Got1xxResponse       func(code int, header textproto.MIMEHeader) error
    DNSStart             func(DNSStartInfo)
    DNSDone              func(DNSDoneInfo)
    ConnectStart         func(network, addr string)
    ConnectDone          func(network, addr string, err error)
    TLSHandshakeStart    func()
    TLSHandshakeDone     func(tls.ConnectionState, error)
    WroteHeaderField     func(key string, value []string)
    WroteHeaders         func()
    Wait100Continue      func()
    WroteRequest         func(WroteRequestInfo)
}
```

你只需设置关心的字段。未设置的字段为 nil，会被跳过。这也是为什么该包可以在不破坏任何人的情况下添加新钩子——已有代码只是让新字段保持 nil。

## 构建一个 curl --trace

首先值得构建的是一个命令行工具，它接收一个 URL 并打印类似 `curl -w` 的耗时分解，但具有 `httptrace` 所提供的更细粒度。技巧在于在每个钩子中记录时间戳，然后计算相对于起始时间的持续时长。

```go
package main

import (
    "context"
    "crypto/tls"
    "fmt"
    "net/http"
    "net/http/httptrace"
    "os"
    "time"
)

type timings struct {
    start        time.Time
    dnsStart     time.Time
    dnsDone      time.Time
    connectStart time.Time
    connectDone  time.Time
    tlsStart     time.Time
    tlsDone      time.Time
    gotConn      time.Time
    firstByte    time.Time
    done         time.Time
}

func (t *timings) elapsed(at time.Time) time.Duration {
    return at.Sub(t.start)
}
```

这个 `ClientTrace` 是机械式的——在每个钩子中捕获 `time.Now()`：

```go
func newTrace(t *timings) *httptrace.ClientTrace {
    return &httptrace.ClientTrace{
        DNSStart: func(_ httptrace.DNSStartInfo) {
            t.dnsStart = time.Now()
        },
        DNSDone: func(_ httptrace.DNSDoneInfo) {
            t.dnsDone = time.Now()
        },
        ConnectStart: func(_, _ string) {
            t.connectStart = time.Now()
        },
        ConnectDone: func(_, _ string, _ error) {
            t.connectDone = time.Now()
        },
        TLSHandshakeStart: func() {
            t.tlsStart = time.Now()
        },
        TLSHandshakeDone: func(_ tls.ConnectionState, _ error) {
            t.tlsDone = time.Now()
        },
        GotConn: func(_ httptrace.GotConnInfo) {
            t.gotConn = time.Now()
        },
        GotFirstResponseByte: func() {
            t.firstByte = time.Now()
        },
    }
}
```

然后 main 函数将其串联起来并打印分解结果：

```go
func main() {
    url := os.Args[1]

    t := &timings{start: time.Now()}
    trace := newTrace(t)

    ctx := httptrace.WithClientTrace(context.Background(), trace)
    req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
    if err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }

    res, err := http.DefaultClient.Do(req)
    if err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
    res.Body.Close()
    t.done = time.Now()

    fmt.Printf("DNS 查询:    %v\n", t.dnsDone.Sub(t.dnsStart))
    fmt.Printf("TCP 连接:    %v\n", t.connectDone.Sub(t.connectStart))
    fmt.Printf("TLS 握手:    %v\n", t.tlsDone.Sub(t.tlsStart))
    fmt.Printf("服务器处理:  %v\n", t.firstByte.Sub(t.gotConn))
    fmt.Printf("内容传输:    %v\n", t.done.Sub(t.firstByte))
    fmt.Printf("总计:        %v\n", t.done.Sub(t.start))
}
```

对任意 URL 运行它，输出会告诉你时间花在了哪里。DNS 查询慢、TLS 握手慢、或者服务器产生第一个字节耗时过长，都会在分解结果中作为单独的一行显示出来。无需代理，无需 APM 代理，依赖图中也不需要任何插桩库。

有几点需要了解。`DNSStart` 和 `DNSDone` 仅在 Go 的解析器执行查询时触发——如果地址已存在于内核的 DNS 缓存中，或者你直接传入了 IP 地址，这些钩子不会触发。`TLSHandshakeStart` 和 `TLSHandshakeDone` 仅在 HTTPS 时触发。`GotConn` 无论连接是新建的还是复用的都会触发，`GotConnInfo` 结构体中的 `Reused` 字段会告诉你具体是哪种情况。

## 构建一个 RoundTripper

一次性的命令行工具很有用，但大多数时候你希望通过给定的 `http.Client` 发出的每个请求都自动被追踪。这正是 `http.RoundTripper` 的用途。包装默认传输层，在委托之前将 trace 附加到 context 上，请求完成后记录结果。

```go
type TracingTransport struct {
    Base http.RoundTripper
    Log  func(req *http.Request, t *timings)
}

func (tt *TracingTransport) RoundTrip(req *http.Request) (*http.Response, error) {
    base := tt.Base
    if base == nil {
        base = http.DefaultTransport
    }

    t := &timings{start: time.Now()}
    trace := newTrace(t)
    req = req.WithContext(httptrace.WithClientTrace(req.Context(), trace))

    res, err := base.RoundTrip(req)
    t.done = time.Now()

    if tt.Log != nil {
        tt.Log(req, t)
    }
    return res, err
}
```

使用方式很简单：

```go
client := &http.Client{
    Transport: &TracingTransport{
        Log: func(req *http.Request, t *timings) {
            log.Printf("%s %s dns=%v tls=%v ttfb=%v total=%v",
                req.Method, req.URL,
                t.dnsDone.Sub(t.dnsStart),
                t.tlsDone.Sub(t.tlsStart),
                t.firstByte.Sub(t.gotConn),
                t.done.Sub(t.start),
            )
        },
    },
}
client.Get("https://example.com")
```

这里有一个容易踩坑的细节。如果调用方已经在请求的 context 上附加了一个 `ClientTrace`，再次调用 `httptrace.WithClientTrace` 不会替换它——而是组合。`ContextClientTrace` 返回最近附加的 trace，但两者的钩子都会触发。对于 `RoundTripper` 来说这通常正是你想要的：调用方的追踪被保留，你的追踪与之并行运行。如果你想替换已有的 trace，需要自己检查 context。

另一点需要了解的是，`RoundTrip` 在响应头读取完毕时就返回了，而不是在响应体消费完毕时。这里的 `t.done` 测量的是 TTFB 加上头部读取的时间，而非包含响应体的完整请求时间。要测量包含响应体传输的总时间，需要将 `res.Body` 包装在一个在 `Close` 时记录 `time.Now()` 的 reader 中：

```go
type timedBody struct {
    io.ReadCloser
    onClose func()
}

func (tb *timedBody) Close() error {
    tb.onClose()
    return tb.ReadCloser.Close()
}

// ...在
```
