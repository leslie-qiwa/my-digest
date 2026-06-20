# Finding a Needle in a 4 GB Haystack: From 0.75 GB/s to 49 GB/s

我有一个 4 GiB 的文件，几乎全是零，唯一一个非零的 `int64` 藏在偏移量 `Size - 8`（最后一个对齐槽位）。任务是：用 Go 在 Linux 上尽可能快地找到那个偏移量。

这是一个刻意设计的简单问题。没有解析，没有索引，算法层面没有任何花招。它唯一衡量的是每秒能让多少数据流过 CPU。正是这种微任务暴露了技术栈的每一层：Go 运行时、标准库、内核、页面缓存、内存层次结构，以及 SIMD——包括 Go 1.26 全新的 `simd/archsimd` 包，它允许你用纯 Go 编写 AVX-512。

从最直白的 `os.ReadFile` + `for range` 开始，我们得到 0.75 GB/s。十三个变体之后达到 49 GB/s，66 倍的加速，而且我们将清楚地知道每一面墙在哪里、为什么会撞上它。

## 实验环境

测试机器：

- AMD Ryzen 5 9600X（Zen 5，6 核 / 12 线程，AVX2 和 AVX-512）
- 15 GiB DDR5
- WSL2 / Linux 6.6 / ext4 / NVMe SSD
- Go 1.26

数据文件大小恰好为 4 GiB（`4 << 30` 字节）。我先用 `pwrite` 将整个文件写满零，确保磁盘块实际被分配（否则稀疏文件的读取是免费且无意义的），然后在偏移量 `Size - 8` 处植入一个固定的 `magic` `int64`。指针位置固定不变——每次运行、每次程序调用都在同一个位置、同一个值，因此不存在运气因素，迭代之间页面缓存状态也不会被干扰。

对于每个变体，我运行 5 次计时迭代，去掉最慢和最快的，取剩余三次的平均值。所有主要测量数据均为热缓存（4 GiB 文件可以轻松放入内存，每个变体运行前预读一次）。最后我还会展示使用 `posix_fadvise(POSIX_FADV_DONTNEED)` 的冷缓存数据。

每个变体的完整源码和基准测试框架可在 GitHub 上浏览。

## V1：最朴素的方式

将整个文件读入内存，逐字节遍历：

```go
func (S) Search(path string) (int64, error) {
    data, err := os.ReadFile(path)
    if err != nil { return -1, err }
    for i, b := range data {
        if b != 0 {
            return int64(i) &^ 7, nil
        }
    }
    return -1, nil
}
```

`os.ReadFile` 分配一个 4 GiB 的 `[]byte`，然后通过 `copy_to_user` 将整个文件复制进去。接着一个紧凑的 Go 循环遍历 40 亿字节，寻找第一个非零字节。

执行时间大约 72% 花在扫描循环中，28% 花在 `os.ReadFile` 内部（主要是 `runtime.makeslice` 对 4 GiB 堆内存的零初始化）。这段朴素代码做了双倍的工作：分配、复制，然后扫描。

**结果：749 MB/s。** 大部分时间花在分配 + 内核复制上：让 Go 运行时每次运行都扩展 4 GiB 堆空间并不免费，而且一旦工作集超过 L3 缓存，分配器的压力就会清晰地显现出来。这是我们的基准线。

## V2：bufio（教科书式答案）

每个"如何在 Go 中读取大文件"的 Stack Overflow 回答都会这样写：

```go
r := bufio.NewReaderSize(f, 1<<16) // 64 KiB
for {
    b, err := r.ReadByte()
    ...
}
```

约 77% 的 CPU 时间仅花在 `bufio.(*Reader).ReadByte` 上。二十亿次调用，二十亿次边界检查，二十亿次偏移量递增。实际的字节比较反而是最便宜的部分。

**结果：755 MB/s，** 基本与朴素的 `os.ReadFile`（[#v1](#v1)）持平。

`bufio.Reader.ReadByte` 每个字节一次 Go 函数调用。四十亿次调用。编译器无法内联穿透它，调用本身的开销远远超过查看一个字节的开销。朴素的 `os.ReadFile`（[#v1](#v1)）预先支付了巨额分配代价；`bufio` 则支付了巨额函数调用代价。总耗时几乎一模一样。

**教训：** 如果能避免，就不要为每个字节支付函数调用开销。

## V3：更大的块，每次扫描 8 字节

让我们将文件以 1 MiB 为单位流式读入可复用的缓冲区，并将每个块作为 `[]uint64` 处理：

```go
buf := make([]byte, 1<<20)
for {
    n, err := io.ReadFull(f, buf)
    words := unsafe.Slice((*uint64)(unsafe.Pointer(&buf[0])), n/8)
    for i, w := range words {
        if w != 0 { return off + int64(i)*8, nil }
    }
    off += int64(n)
    if err != nil { break }
}
```

两个变化：

- 每兆字节一次系统调用，而不是每字节一次。内核通过一次 `copy_to_user` 传输 256 个页面缓存页。
- 内层循环每次迭代处理 8 字节。分支和比较次数与朴素的 `os.ReadFile`（[#v1](#v1)）相同，但每次迭代的工作量是 8 倍。

大约 60% 的时间花在 `internal/poll.(*FD).Read`（read 系统调用 + `copy_to_user`）中，40% 在扫描循环中。我们已经干净地摊销了系统调用开销；内核复制现在是主要成本。

**结果：13.7 GB/s，** 提升 18 倍。这已经相当不错了。大多数人会到此为止。

我们不会。

## V4：mmap，显而易见的下一步

既然内核可以直接给我们一个指向页面缓存的窗口，为什么还要从内核空间往用户空间复制字节呢？

```go
data, _ := unix.Mmap(int(f.Fd()), 0, size, unix.PROT_READ, unix.MAP_SHARED)
defer unix.Munmap(data)
words := unsafe.Slice((*uint64)(unsafe.Pointer(&data[0])), size/8)
for i, w := range words {
    if w != 0 { return int64(i) * 8, nil }
}
```

假设：与分块 uint64（[#v3](#v3)）相同的扫描循环，但去掉了巨大的 `copy_to_user`。应该更快。

几乎 100% 的时间都在我们那个单独的 `S.Search` 帧内。但看起来像是"扫描时间"的，实际上大部分是循环触及每个新的 4 KiB 页面时懒惰发生的次要页面错误。pprof 将时间归属于触发陷阱的用户空间函数，掩盖了内核侧的开销。

**结果：13.2 GB/s，** 比分块 uint64（[#v3](#v3)）略慢。出人意料。

原因是次要页面错误。mmap 并不真正映射页面——它只设置地址空间元数据。当用户空间循环第一次触及每个 4 KiB 页面时，CPU 陷入内核，内核找到已缓存的页面并更新页表。对于 4 GiB 文件，这意味着超过 100 万次页面错误。每次大约在微秒级别。仅错误开销就有整整一秒，而循环本身已经在全速扫描了。

所以 mmap 省掉了内核到用户空间的复制，却增加了一个逐页的税。

## V5：提示内核：madvise

也许内核只是需要知道我们在做什么：

```go
_ = unix.Madvise(data, unix.MADV_SEQUENTIAL)
_ = unix.Madvise(data, unix.MADV_WILLNEED)
```

- `MADV_SEQUENTIAL`："我正在顺序扫描，请积极预取，并可以丢弃已过的页面。"
- `MADV_WILLNEED`："现在就把这些页面调入；别让我等冷读。"

与 mmap（[#v4](#v4)）无法区分：火焰图上是 `S.Search` 的一个高高的单栈。`madvise` 调用在左侧显示为一小片，但没带来任何收益，因为缓存已经是热的。

**结果：13.2 GB/s，** 与 mmap（[#v4](#v4)）基本相同。为什么？

因为在热缓存上没有什么可预取的，页面已经在内存中了。当所有内容都已缓存时，`WILLNEED` 是空操作，而 `SEQUENTIAL` 的预读只对冷读有帮助。瓶颈不在 I/O，而在那 100 万+ 次页面错误，而 `madvise` 对此无能为力。

（`madvise` 对冷缓存读取很有用，我们在最后会看到。）

## V6：并行化 mmap 扫描

单个 goroutine 做标量 uint64 加载大约会达到 DDR5 单线程带宽的上限。通过多核心，我们可以发出更多并行的缓存行请求，驱动更多正在传输中的 DRAM 通道：

```go
nWorkers := runtime.NumCPU() // 本机为 12
per := (totalWords + nWorkers - 1) / nWorkers
var found atomic.Int64; found.Store(-1)
var wg sync.WaitGroup
for w := range nWorkers {
    start, end := ... // 分片边界
    wg.Add(1)
    go func() {
        defer wg.Done()
        words := unsafe.Slice(..., end-start)
        for i, x := range words {
            if x != 0 {
                // CAS 更新最小找到的偏移量
                ...
                return
            }
        }
    }()
}
wg.Wait()
```

约 90% 的时间分布在各 goroutine 的 `Search.func1` 帧中执行扫描。有趣的一小部分是约 8% 在 `runtime.asyncPreempt` 中——运行时在循环中途停止 goroutine 以服务调度器。当 12 个热 goroutine 争夺 12 个逻辑 CPU 时，抢占成本变得可见。

**结果：28.8 GB/s，** 是 mmap + madvise（[#v5](#v5)）的 2.2 倍，是朴素 `os.ReadFile`（[#v1](#v1)）的 38 倍。

扩展情况：从 11 GB/s（1 个工作线程）翻倍到 6 核时的 27 GB/s（6 倍核心带来 2.4 倍提升），然后趋于平稳。超过 6 个工作线程后，增加更多 goroutine 不再有帮助，有时反而有害。瓶颈不在 CPU 或内存带宽；而在每个进程的
