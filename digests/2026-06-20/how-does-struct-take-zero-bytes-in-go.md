# How Does struct{} Take Zero Bytes in Go?

在学习 Go 的过程中，我遇到了 `struct{}`。最初是作为实现高效内存集合的方式，因为它占用零字节。后来在 goroutine 中又遇到了它，它是通过 channel 发送信号的标准方式。

两种用法都很合理。但我没有完全理解的是：一个占用零字节的东西，底层到底是怎么工作的？

剧透：答案在运行时。让我们深入探究。

## 零字节的说法

首先，让我们确认它确实占用零字节。我们可以用 `unsafe.Sizeof()` 来测量，它返回类型的字节大小。

```go
var i int
var s struct{}
fmt.Println(unsafe.Sizeof(i)) // 8
fmt.Println(unsafe.Sizeof(s)) // 0
```

我们可以看到 `struct{}` 占用零字节。正如开头提到的，它的一个用途是实现集合——因为值不占空间，map 只需要为键付出代价：

```go
b := map[string]bool{}
s := map[string]struct{}{}
fmt.Println(unsafe.Sizeof(b["a"])) // 1
fmt.Println(unsafe.Sizeof(s["a"])) // 0
```

> 值得注意的是：在现代 Go（1.24+）中，map 内部使用 Swiss Tables，这意味着 `map[string]struct{}` 和 `map[string]bool` 在实际中占用相同的内存。1 字节的节省不再适用。使用 `struct{}` 实现集合现在是风格选择，而非内存优化。

开头提到的另一个用例是通过 channel 发送信号。让我们创建一个 channel 并确认元素不占空间：

```go
i := make(chan int)
s := make(chan struct{})
fmt.Println(unsafe.Sizeof(<-i)) // 8
fmt.Println(unsafe.Sizeof(<-s)) // 0
```

由于 `struct{}` 不携带任何数据，它只是一个信号——元素不占空间。

## 底层原理

我们确认了 `struct{}` 占用零字节——但为什么？当你创建一个时，实际发生了什么？

```go
var s struct{}
```

当一个零大小的值确实被堆分配时，它会经过 `mallocgc`。它做的第一件事之一就是检查大小是否为零——如果是，则跳过分配，直接返回指向 `zerobase` 变量的指针：

```go
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
    // ...
    // Short-circuit zero-sized allocation requests.
    if size == 0 {
        return unsafe.Pointer(&zerobase)
    }
    // ...
}
```

`zerobase` 是一个硬编码的全局变量，直接内置在 Go 运行时中——一个固定的占位地址，用于所有零字节分配：

```go
// base address for all 0-byte allocations
var zerobase uintptr
```

这意味着你创建的每个 `struct{}` 变量都指向同一个地址。我们可以验证：

```go
var a struct{}
var b struct{}
var c [1000000]struct{}
fmt.Println(unsafe.Pointer(&a)) // 0x5a2c40
fmt.Println(unsafe.Pointer(&b)) // 0x5a2c40
fmt.Println(unsafe.Pointer(&c)) // 0x5a2c40
```

所以其中的奥秘很简单——在分配时，运行时只是返回指向全局 `zerobase` 变量的指针，每个空结构体都指向同一个地址。

## 等等，为什么在堆上？

为什么这里会调用 `mallocgc` 并在堆上分配一个空结构体？答案是 `fmt.Println`——它接受 `interface{}` 参数，这导致值逃逸到堆上——这个过程称为逃逸分析。这会触发一次 `mallocgc` 调用，然后立即返回 `zerobase`。

如果你使用内置的 `println`（它由运行时直接编译，不会导致逃逸），结构体会留在栈上：

```go
func noEscape() {
    var a struct{}
    var b struct{}
    println(unsafe.Pointer(&a)) // 0x1f6c6db0f38
    println(unsafe.Pointer(&b)) // 0x1f6c6db0f38
}

func withEscape() {
    var a struct{}
    var b struct{}
    fmt.Println(unsafe.Pointer(&a)) // 0x5a2c40
    fmt.Println(unsafe.Pointer(&b)) // 0x5a2c40
}
```

> 想自己看看吗？用 `-m` 标志编译，打印编译器的逃逸分析决策：
>
> ```
> go build -gcflags="-m" main.go
> ```
>
> 对于上面的代码片段，你会看到哪些变量移动到了哪里：
>
> ```
> ./main.go:9:6: moved to heap: a
> ./main.go:10:6: moved to heap: b
> ```

`noEscape()`——栈上的值地址大得多，因为栈位于地址空间的高位。两者共享同一个地址——它们在栈上也不占空间。

`withEscape()` 则使用一个静态全局变量——它根本不经过堆。它只是从 BSS 段（存放静态全局变量的地方）获取 `zerobase` 的运行时值。

栈、堆、BSS——我们三个都涉及到了。以下是它们如何组合在一起，以及空结构体在每种情况下的位置：

每个空结构体在内存中的位置

## 容易踩的坑

每个空结构体共享同一个地址是个巧妙的技巧，但 Go 规范只是允许这样做——并不承诺一定如此。这导致了两个值得了解的意外情况。

**指针比较不可靠。** 由于"两个不同的零大小变量可能具有相同的地址"，比较它们指针的结果取决于编译器——它可能将它们别名到 `zerobase`，也可能不会：

```go
var a, b struct{}
fmt.Println(&a == &b) // 可能打印 true 或 false
```

所以你不能用指针标识来区分两个零大小的值。不要构建依赖于此的逻辑。

**尾部字段陷阱。** 如果零大小字段是结构体的最后一个字段，运行时会为结构体填充一个额外的字。否则，指向最后一个字段的指针会指向分配区域之后的一个字节，这可能使下一个对象保持存活（或混淆 GC）。所以空结构体不再是免费的：

```go
type T struct {
    x int64
    z struct{}
}
fmt.Println(unsafe.Sizeof(T{})) // 16，不是 8 - 被填充了！
```

将零大小字段移到最后以外的任何位置，填充就会消失（`Sizeof` 回到 8）。虽然是小事，但它悄悄地破坏了使用 `struct{}` 的初衷。

## 总结

`struct{}` 不消耗任何资源，因为运行时从未真正分配它。它的大小始终为 `0`，所以在栈上不占空间——当值逃逸到堆时，运行时跳过分配，直接返回指向全局 `zerobase` 的指针。

两个值得记住的注意事项：

- **共享地址。** 空结构体的指针不是唯一的，所以不要依赖于比较它们。
- **尾部字段。** 作为结构体最后一个字段的 `struct{}` 会填充父结构体——悄悄地消耗你试图节省的字节。

这些都不会改变你的使用方式——`struct{}` 仍然是集合和 channel 信号的正确选择。它背后什么都没有，现在你知道原因了。
