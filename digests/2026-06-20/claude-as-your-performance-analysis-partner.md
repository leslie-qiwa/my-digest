# Claude as Your Performance Analysis Partner

性能分析涉及通过测量硬件计数器、CPU 性能剖析和跟踪等数据来识别和解决应用程序瓶颈。这些数据文件通常很大（数百兆字节），CPU 性能剖析包含大量指令开销细节。手动检查这些大型文件——包括在浏览器中放大跟踪数据以发现模式和依赖关系——是一项费力且容易出错的工作。

我尝试使用 Claude 来看看性能分析这项任务是否能从令人疲惫变为令人愉悦。结果非常令人鼓舞。在本文中，我将讨论如何使用 Claude 通过 CPU 性能剖析和跟踪来进行性能分析。我以 Go 语言新的 Green Tea 垃圾收集器（GC）作为重点代码，在 POWER10 架构上使用 sweet 基准测试套件评估其优化机会。

## CPU 性能剖析分析

Go 提供了使用 Go 工具 `pprof` 为二进制文件生成 CPU 性能剖析的功能。这些剖析可以识别性能瓶颈，甚至可以精确到汇编指令级别。Claude 负责在 `pprof` 标记的热点中寻找优化建议。以 bleve 索引基准测试为例，耗时最多的前十个函数如下所示：

```
go tool pprof BleveIndexBatch100-1212217505-cpu.prof
File: bleve-index-bench
Build ID: 81f203eb5714d9755a42c833024d5f60afd94e03
Type: cpu
Time: 2026-03-10 16:10:56 IST
Duration: 5.40s, Total samples = 15.18s (280.91%)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top 10
Showing nodes accounting for 8830ms, 58.17% of 15180ms total
Dropped 208 nodes (cum <= 75.90ms)
Showing top 10 nodes out of 103
flat flat% sum% cum cum%
3170ms 20.88% 20.88% 4260ms 28.06% runtime.tryDeferToSpanScan
1100ms 7.25% 28.13% 2970ms 19.57% runtime.scanObjectSmall
840ms 5.53% 33.66% 900ms 5.93% github.com/blevesearch/segment.segmentWords#
710ms 4.68% 38.34% 7180ms 47.30% runtime.scanSpan#
670ms 4.41% 42.75% 2940ms 19.37% runtime.scanObjectsSmall
520ms 3.43% 46.18% 970ms 6.39% github.com/blevesearch/bleve/index/store/gtreap.itemCompare
520ms 3.43% 49.60% 3610ms 23.78% github.com/steveyen/gtreap.(*Treap).union
440ms 2.90% 52.50% 2070ms 13.64% github.com/steveyen/gtreap.(*Treap).split
430ms 2.83% 55.34% 450ms 2.96% runtime.extractHeapBitsSmall
430ms 2.83% 58.17% 1690ms 11.13% runtime.mallocgcSmallScanNoHeader
```

## 原子操作示例

在最热的函数 `runtime.tryDeferToSpanScan` 中，Claude 快速识别出一段涉及原子操作的代码，其中存在显著的优化机会。

该代码对每个对象执行 2 次原子操作：

- `atomic.Load8()`：检查是否已被标记（从剖析数据看耗时 1.10 秒）
- `atomic.Or8()`：设置标记位（耗时 690 毫秒）

总计：仅原子操作就耗费 1.79 秒

### 建议的优化方案

**单次原子测试并设置（推荐）：** Go `atomic` 包中的 `atomic.Or8()` 函数存在局限性，因为它不返回旧值，这一点不同于 `atomic.Or32()`。由于此限制，当前实现必须先使用 `Load8` 来确定对象是否已被标记。鉴于原子指令在 PowerPC 架构上开销较大，Claude 建议使用仅需一次原子指令 `Or32` 的替代实现。

```go
// 使用单次原子操作设置标记位
idx, mask := objIndex/8, uint8(1)<<(objIndex%8)
// 对对齐的 32 位字使用 Or32 以获取旧值
byteOffset := uintptr(idx) & 0b11
wordPtr := (*uint32)(unsafe.Pointer(uintptr(unsafe.Pointer(&q.marks[idx])) &^ 0b11))
shift := byteOffset * 8
if goarch.BigEndian {
    shift = 32 - shift - 8
}
oldWord := atomic.Or32(wordPtr, uint32(mask)<<shift)
if (oldWord>>shift)&uint32(mask) != 0 {
    return true // 已被标记
}
```

预期节省：约 1.10 秒（消除了单独的 `Load8`）

Claude 还指出，这与 `spanScanOwnership.or()`（第 95-110 行）中使用的模式完全相同！

虽然该建议方案看似合理，但实际导致了性能回退。这主要是由于垃圾收集过程中的伪共享（false sharing）问题。多个线程访问堆簿记区域导致竞争：

- 当多个线程访问同一个 4 字节字中的不同字节时，锁被应用于整个字，迫使访问变为串行。
- Claude 证实了这一问题，并指出了该实现的以下问题：
  - **伪共享：** 使用 `Or32` 意味着四个相邻字节共享同一个原子操作。
  - **高竞争：** 多个 GC 线程标记同一个 span 中的不同对象时，会竞争同一个 32 位字。
  - **缓存行抖动：** 每次 `Or32` 操作都会使不同 CPU 上的缓存行失效。性能剖析数据证实了这一点：

```
atomic.Or32：0.43 秒 → 1.95 秒（+1.52 秒）
tryDeferToSpanScan 本身有所改善（4.53 秒 → 2.61 秒），但 Or32 的开销吞噬了所有收益
```

尽管该优化在垃圾收集的场景中无法带来帮助，但在其他应用中，当对同一个字的竞争不激烈时，它可能会产生效果。

## 算术优化

在同一函数 `tryDeferToSpanScan` 中，Claude 还在 objIndex 计算过程中建议了一个有趣的 2 的幂次除法优化。

```go
// 对于 2 的幂次大小，使用移位代替魔数除法
elemsize := gc.SizeClassToSize[q.class.sizeclass()]
if isPowerOfTwo(elemsize) {
    objIndex = uint16((p - base) >> log2(elemsize))
} else {
    objIndex = uint16((uint64(p-base) * uint64(gc.SizeClassToDivMagic[q.class.sizeclass()])) >> 32)
}
```

此更改带来的性能提升有限，因为它仅对处理大小为 2 的幂次的对象的程序有效。在其他情况下，还需要额外的条件检查。因此，该建议的有效性高度依赖于特定的基准测试特征，不被认为是普遍有效的。

## 底层代码分析

在 biogo-igor 基准测试中，Claude 建议将 `q.class.sizeclass` 变量缓存起来作为一项潜在优化。当被质疑编译器应该能处理这种简单的窥孔优化时，在随后查看汇编转储后，Claude 确认该优化并未被执行。此外，模型还能够精确定位可能的原因：一个原子调用访问了该变量的一部分，这很可能阻止了编译器应用该优化。

这表明 Claude 可以用来仔细研究汇编代码，发现那些由于汇编格式难以阅读而容易遗漏的洞察。

汇编证据（igor.dump 第 70772-70831 行）：

**第一次访问：计算 objIndex（第 70772-70789 行）**

```asm
# 源码: objIndex := uint16((uint64(p-base) * uint64(gc.SizeClassToDivMagic[q.class.sizeclass()])) >> 32)
70773→ 0x440e3a MOVZX 0x7f(DX), SI #① 从内存加载 q.class（DX 中偏移 0x7f）
70775→ 0x440e3e SHRL $0x1, SI      #② 计算 sizeclass() - 右移 1 位
70777→ 0x440e41 MOVSX SI, SI       #③ 符号扩展 sizeclass 结果
...
70784→ 0x440e4a CMPQ SI, $0x44     #④ 边界检查：sizeclass < 68？
70785→ 0x440e4e JAE 0x441012       #⑤ 越界则跳转
70786→ 0x440e54 LEAQ gc.SizeClassToDivMagic(SB), R8 # ⑥ 加载数组基地址
70787→ 0x440e5b MOVL 0(R8)(SI*4), CX #⑦ 加载 gc.SizeClassToDivMagic[sizeclass]
70788→ 0x440e5f IMULQ DI, CX       #⑧ 乘法：(p-base) * divMagic
70789→ 0x440e63 SHRQ $0x20, CX     #⑨ 右移 32 位 → objIndex
```

操作：
- 内存加载：从偏移 0x7f 读取 `q.class`
- 计算：`sizeclass()`（右移）
- 数组访问：`SizeClassToDivMagic[sizeclass]`

**第二次访问：在 noscan 路径中（第 70814-70831 行）**

```asm
# 源码: gcw.bytesMarked += uint64(gc.SizeClassToSize[q.class.sizeclass()])
70815→ 0x440ead MOVZX 0x7f(DX), CX # ① 再次从内存加载 q.class！（相同偏移 0x7f）
70817→ 0x440eb1 TESTL $0x1, CL     # ② 检查 noscan 位
...
70823→ 0x440eb8 SHRL $0x1, CL      # ③ 再次计算 sizeclass()！（右移）
70825→ 0x440eba MOVSX CL, CX       # ④ 再次符号扩展
...
70827→ 0x440ec0 CMPQ CX, $0x44     # ⑤ 再次边界检查：sizeclass < 68？
70828→ 0x440ec4 JAE 0x440ffa       # ⑥ 再次越界跳转
70
```
