# Finding Leaked Goroutines in Go 1.27

已接受的提案：Go 标准库中的 goroutine 泄漏分析

目录

Go 1.27 将在 `runtime/pprof` 中加入 goroutine 泄漏检测器。该提案已于四月被接受。

## 几种常见的 goroutine 泄漏

当 goroutine 阻塞在一个永远不会被释放的 channel 或锁上时，它就会泄漏，从而在进程的整个生命周期中一直存留。我一直使用 uber-go/goleak 在测试中捕获它们。

一种是提前返回导致发送方被搁置的情况，我在《提前返回与 goroutine 泄漏》中介绍过。代码如下：

```go
func run(tasks []func() error) error {
	errs := make(chan error) // 无缓冲
	var wg sync.WaitGroup
	for _, task := range tasks {
		wg.Go(func() { errs <- task() }) // (1)
	}
	for range tasks {
		if err := <-errs; err != nil {
			return err // (2)
		}
	}
	wg.Wait()
	return nil
}
```

这里：

- (1) 每个任务通过 `wg.Go` 在无缓冲 channel 上发送其结果
- (2) 第一个错误导致提前返回，因此仍在排队等待发送的任务将永远阻塞

给 `errs` 一个足够容纳所有任务的缓冲区，或者在返回前排空所有结果，可以防止发送操作阻塞。

一种相关的泄漏出现在你向多个副本发送请求并只保留第一个响应时：

```go
func replicate(replicas []func() string) string {
	results := make(chan string) // 无缓冲
	for _, r := range replicas {
		go func() { results <- r() }() // (1)
	}
	return <-results // (2)
}
```

这里：

- (1) 每个副本竞争在无缓冲 channel 上发送其响应
- (2) 第一个响应被返回后，较慢的副本将永远阻塞在其发送操作上

与之前一样，一个大小与副本数量匹配的缓冲区可以让较慢的副本完成发送并退出。

另一种是忘记 `close`：

```go
func stream(work []int) {
	out := make(chan int)
	go func() {
		for v := range out { // (1)
			handle(v)
		}
	}()
	for _, v := range work {
		out <- v
	}
	// (2) 没有 close(out)
}
```

这里：

- (1) range 持续从 `out` 中拉取数据，直到它被关闭
- (2) `stream` 返回时没有执行 `close(out)`，因此 range 永远不会结束，goroutine 发生泄漏

修复方法是在最后一次发送后执行 `close(out)`，这会结束 range 循环并让 goroutine 返回。

一旦你发现了它们就很明显，但在提前返回或周围代码膨胀后很容易被忽略。goleak 在测试中能捕获它们。在生产环境中，你有常规的 `/debug/pprof/goroutine` 分析。它显示每个 goroutine 阻塞在什么上面，但不能判断它是否会被解除阻塞，所以你只能猜测哪些是永久阻塞的，哪些只是处于空闲状态。

这个列表远非详尽，而且并非所有泄漏都在你自己的代码中。一个依赖库，或它的传递依赖，也可能造成泄漏。Uber 对其 Go 单体仓库中的模式进行了编目。

## 标准库泄漏分析现在可以找到它们

它来自 Uber，与 goleak 同源，由 Vlad Saioc 和 Milind Chabbi 设计。检测依赖于垃圾回收器。当一个 goroutine 阻塞在一个 channel 或锁上，且没有任何可运行的 goroutine 能直接或通过其可解除阻塞的其他 goroutine 到达它时，该 goroutine 就是泄漏的。没有什么能唤醒它。GC 将其标记出来。

> **注意**
>
> 可以将其理解为可达性测试。如果一个 goroutine 阻塞在原语 `P` 上，而 `P` 从任何可运行的 goroutine 或从这些可运行 goroutine 能够解除阻塞的任何 goroutine 都不可达，那么 `P` 就无法被解除阻塞。该 goroutine 永远无法被唤醒。

goleak 和泄漏分析回答的是不同的问题：

|  | goleak | goroutineleak 分析 |
|---|---|---|
| 问的是 | 有什么还在运行但你没预期的 | 有什么永远无法再运行 |
| 判断方式 | 快照，无证明 | 可达性证明，通过 GC |
| 适用于 | 测试，在清理阶段 | 运行中的进程 |
| 误报 | 有，在实际服务器上 | 无，只报告可证明阻塞的 goroutine |

区别在于各自的运行场景。在测试的清理阶段，不应该有任何东西还在运行。直接返回仍存在的内容正是你期望 goleak 做的。运行中的服务器则恰恰相反。它的大多数 goroutine 是故意阻塞的，等待下一个请求，而 goleak 无法区分这些和真正的泄漏。

泄漏分析改用证明的方式。它从仍可运行的 goroutine 开始，沿着它们能到达的路径追踪，并拯救那些 channel 或锁仍在使用中的被阻塞 goroutine。剩下的就是没有任何东西能触及的。它们永久阻塞了。Uber 之前已经尝试过使用采样工具进行生产环境的泄漏检测，但采样基于启发式判断，会产生误报。GC 遍历只报告它能证明被阻塞的 goroutine。这就是零误报的保证。

该分析发布时不包含 goleak 的 `VerifyNone(t)` 或 `VerifyTestMain(m)`。测试部分展示了如何自行实现。

API 非常精简。没有新的类型或函数，只有一个名为 `goroutineleak` 的分析。它在注册后即可使用，标准的 `pprof` 工具可以像读取其他分析一样读取它。

## 你可以通过常见的四种方式获取该分析

> **注意**
>
> 目前该分析隐藏在构建标志之后。使用 `GOEXPERIMENT=goroutineleakprofile` 运行以下示例，否则 `pprof.Lookup("goroutineleak")` 会返回 nil。Go 1.27 将正式开放并取消该标志。

### 从你自己的代码中

你获取分析并自行写入输出。从 `debug=0` 开始，它将 gzip 压缩的 protobuf 转储到文件：

```go
func main() {
	f, _ := os.Create("leak.pb.gz")
	pprof.Lookup("goroutineleak").WriteTo(f, 0)
}
```

`pprof.Lookup` 返回分析，`WriteTo` 在写入之前运行一个用于泄漏检测的 GC 周期。使用 `go tool pprof` 打开该文件，与 CPU 或堆分析相同。

`WriteTo` 的第二个参数是 `debug` 级别。`1` 和 `2` 提供文本输出，你可以直接发送到 `os.Stdout`。在 `debug=1` 时，信号处理器允许 `kill -USR1 <pid>` 按需转储泄漏信息：

```go
// kill -USR1 <pid> 按需转储泄漏信息
sig := make(chan os.Signal, 1)
signal.Notify(sig, syscall.SIGUSR1)
go func() {
	for range sig {
		pprof.Lookup("goroutineleak").WriteTo(os.Stdout, 1)
	}
}()
```

文本直接指向泄漏的 goroutine：

```
goroutineleak profile: total 2
1 @ ...
# 0x... main.leakSend.func1+0x27 formats/main.go:15
1 @ ...
# 0x... main.leakRange.func1+0x33 formats/main.go:21
```

`debug=2` 是完整的 goroutine 转储，泄漏的 goroutine 标记有 `(leaked)`：

```
goroutine 7 [chan send (leaked)]:
main.leakSend.func1()
	formats/main.go:15 +0x28
created by main.leakSend in goroutine 1
	formats/main.go:15 +0x6c

goroutine 8 [chan receive (leaked)]:
main.leakRange.func1()
	formats/main.go:21 +0x34
created by main.leakRange in goroutine 1
	formats/main.go:20 +0x6c
```

正常的转储显示 `[chan send]` 和 `[chan receive]`。`(leaked)` 后缀是该分析新增的。

### 在测试中

一个辅助函数运行检测并返回被阻塞的内容：

```go
func leaked() (string, bool) {
	p := pprof.Lookup("goroutineleak")
	if p == nil {
		return "", false // 实验特性关闭，无需检测
	}
	var b bytes.Buffer
	p.WriteTo(&b, 1)
	return b.String(), p.Count() > 0
}
```

对于单个测试，将其封装在一个你 `defer` 的 `verifyNone` 中：

```go
// verifyNone 对应 goleak.VerifyNone。
func verifyNone(t *testing.T) {
	t.Helper()
	if report, ok := leaked(); ok {
		t.Fatalf("leaked goroutines:\n%s", report)
	}
}

func TestRun(t *testing.T) {
	defer verifyNone(t)
	// ... 执行被测代码 ...
}
```

对于整个测试套件，编写一个 `verifyTestMain` 并在 `TestMain` 中调用它：

```go
// verifyTestMain 对应 goleak.VerifyTestMain。
func verifyTestMain(m *testing.M) {
	code := m.Run()
	if code == 0 {
		if report, ok := leaked(); ok {
			fmt.Fprintf(os.Stderr, "leaked goroutines:\n%s", report)
			code = 1
		}
	}
	os.Exit(code)
}

func TestMain(m *testing.M) {
	verifyTestMain(m)
}
```

### 通过 HTTP

导入 `net/http/pprof` 会将其注册到默认多路复用器上。提供该多路复用器服务（下面的 `nil` handler），端点即可使用，无需额外代码：

```go
// 在 http.DefaultServeMux 上注册 /debug/pprof/goroutineleak
import _ "net/http/pprof"

http.ListenAndServe("localhost:6060", nil)
```

然后从端点读取分析：

```
$ curl 'localhost:6060/debug/pprof/goroutineleak?debug=1'
goroutineleak profile: total 1
1 @ ...
# 0x... main.main.func1+0x27 server/main.go:14
```

### 使用 go tool pprof

`go tool pprof` 可以像读取其他分析一样读取它，指向该端点或保存的 `debug=0` 转储：

```
$ go tool
```
