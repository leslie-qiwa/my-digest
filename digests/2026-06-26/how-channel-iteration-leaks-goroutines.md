# How Channel Iteration Leaks Goroutines

Channel 遍历与 goroutine 泄漏

我在编写一个自定义的 cron 调度器时,遇到了经典的「range 遍历 channel」泄漏问题。我以前在生产环境中调试过很多次这种问题,但当自己在一小段代码里亲手写一个时,我才再次意识到:即便你知道这种坑,写出这类 bug 仍然是多么容易。

如下:

- 在每一次 tick,调度器分发到期的任务
- 每个任务通过一个 channel 上报其执行结果
- 一个收集器(collector)对该 channel 进行 range,以记录每次运行

```go
// cron/scheduler.go
func tick(due []Job) []outcome {
results := make(chan outcome)
var wg sync.WaitGroup
for _, j := range due {
wg.Add(1)
go func() {
results <- outcome{job: j.Name, err: j.Run()} // (1)
}()
}
var log []outcome
go func() {
for r := range results { // (2)
log = append(log, r)
wg.Done()
}
}()
wg.Wait()
// (3) no close(results)
return log
}
```

- (1) 每个到期的任务在这个无缓冲 channel 上发送其执行结果
- (2) 收集器对 `results` 进行 range,记录每个结果并标记其完成(done)
- (3) 一旦每个任务都已上报,`wg.Wait` 解除阻塞,`tick` 返回

生产者(producers)是没问题的。每一次发送都与收集器的一次接收相匹配,所以每个任务 goroutine 发送一次后就退出了。问题出在收集器上。在最后一个结果之后,它循环回到 range 并等待下一个值,但没有任何东西去关闭这个 channel。于是它在那次接收上一直阻塞,直到进程结束。每一次 tick 都会再泄漏一个。

手动去 drain(抽干)同一个 channel 就永远不会泄漏。发送三个值,然后正好取三个:

```go
ch := make(chan int)
go func() {
<-ch
<-ch
<-ch
}()
ch <- 1
ch <- 2
ch <- 3
```

三次接收,然后 goroutine 返回。把这些接收换成 range 就会泄漏:

```go
ch := make(chan int)
go func() {
for range ch { // never ends: ch is never closed
}
}()
ch <- 1
ch <- 2
ch <- 3
```

这两种形式停止的条件不同。三次显式的接收在第三个值之后会自行停止。而 `range` 会一直读取,直到 channel 被关闭。回到调度器里,没有任何东西去关闭 `results`,所以那个进行 range 的收集器会阻塞在一次永远无法完成的接收上。

修复方法就是有 bug 的版本所缺失的那一行:在每个任务都已上报后 close 掉 `results`。range 结束,收集器返回:

```go
// cron/scheduler.go
// ...
wg.Wait()
close(results) // ends the range, the collector returns
return log
```

> **警告**
> 改用一个带缓冲的 channel 并不能解决这个问题。`range` 只有在 channel 被关闭时才会结束。无论缓冲区有多大,接收方都会一直等待那个永远不会到来的 close。

这是一个相当有据可查的泄漏。Uber 把它称为 channel iteration misuse(channel 遍历误用)。

通常你会用 goleak 来捕捉这类泄漏:

- 接入 goleak
- 在一个测试中执行那条会泄漏的路径
- 测试会因为那个卡住的 goroutine 的堆栈而失败

我在 early return leak(提前返回泄漏)那篇文章里写过 goleak 的工作流。但 goleak 只有在某个测试执行了有 bug 的路径时才能捕捉到泄漏,而我的调度器测试从来没有跑过那条路径。所以 goleak 从未发现它。

真正捕捉到它的,是 Go 1.27 新增的 leak profile(泄漏剖析)。我在写这篇文章时正好在自己的代码上运行它,而它根本不需要测试。它借助垃圾回收器来找出那些阻塞在某个永远无法被触达的东西上的 goroutine,并只报告这些。在 `debug=2` 下运行它,那个卡住的收集器就会被标记上 `(leaked)` 显示出来:

```
goroutine 25 [chan receive (leaked)]:
main.tick.func2()
leaky-tick/main.go:43 +0x60
created by main.tick in goroutine 1
leaky-tick/main.go:42 +0x15c
```

`main.tick.func2` 就是那个收集器,停在第 43 行的 range 上。该 profile 能确定性地找出这类泄漏,没有误报,而且无需任何测试去执行那条路径。

关闭 channel 阻止了泄漏,但它留下了一个别扭之处:`WaitGroup` 在统计任务数,而收集器却在调用 `Done`。

收集器应该只负责 drain results。任务的完成,应当归属于运行该任务的那个 goroutine。一旦每个任务都返回,一个 waiter 就可以 close 掉 `results`,range 也就能正常结束。

使用 `wg.Go`,修正后的版本变为:

```go
// cron/scheduler.go
func tick(due []Job) []outcome {
results := make(chan outcome)
var wg sync.WaitGroup
for _, j := range due {
wg.Go(func() { // (1)
results <- outcome{job: j.Name, err: j.Run()}
})
}
go func() {
wg.Wait()
close(results) // (2)
}()
var log []outcome
for r := range results { // (3)
log = append(log, r)
}
return log
}
```

- (1) `wg.Go` 运行每个任务,并在其返回时调用 `Done`,所以每个任务标记自己的完成
- (2) 一个单独的 goroutine 等待每个任务,然后 close 掉 `results`,这样 range 才能结束
- (3) `tick` 自己 drain `results`,因此不存在单独的收集器 goroutine

在这里如果忘了 `close`,那么 `tick` 会在最后一个结果之后阻塞在 range 上。所有生产者都已退出,所以没有人能再发送另一个值。这还是同一个缺失的 close,但现在它表现为一个死锁(deadlock),而不是泄漏一个后台收集器。

代码可在示例仓库(example repo)中获取。
