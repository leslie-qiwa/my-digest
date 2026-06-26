# asciigraph 0.10: ASCII Line Graph Rendering Library

用于绘制轻量级 ASCII 折线图的 Go 包 ╭┈╯。

```
go get -u github.com/guptarohit/asciigraph@latest
```

```go
package main

import (
"fmt"

"github.com/guptarohit/asciigraph"
)

func main() {
data := []float64{3, 4, 9, 6, 2, 4, 5, 8, 5, 10, 2, 7, 2, 5, 6}
graph := asciigraph.Plot(data)

fmt.Println(graph)
}
```

运行该示例会渲染出如下图形：

```
10.00 ┤ ╭╮
 9.00 ┤ ╭╮ ││
 8.00 ┤ ││ ╭╮││
 7.00 ┤ ││ ││││╭╮
 6.00 ┤ │╰╮ ││││││ ╭
 5.00 ┤ │ │ ╭╯╰╯│││╭╯
 4.00 ┤╭╯ │╭╯ ││││
 3.00 ┼╯ ││ ││││
 2.00 ┤ ╰╯ ╰╯╰╯
```

```go
package main

import (
"fmt"

"github.com/guptarohit/asciigraph"
)

func main() {
data := [][]float64{{0, 1, 2, 3, 3, 3, 2, 0}, {5, 4, 2, 1, 4, 6, 6}}
graph := asciigraph.PlotMany(data)

fmt.Println(graph)
}
```

运行该示例会渲染出如下图形：

```
6.00 ┤ ╭─
5.00 ┼╮ │
4.00 ┤╰╮ ╭╯
3.00 ┤ │╭│─╮
2.00 ┤ ╰╮│ ╰╮
1.00 ┤╭╯╰╯ │
0.00 ┼╯ ╰
```

使用 `YAxisValueFormatter(...)` 来控制 Y 轴上打印数值的渲染方式。这对于人类可读的单位（如字节、时长或特定领域的标签）很有用。

```go
package main

import (
"fmt"

"github.com/guptarohit/asciigraph"
)

func main() {
data := []float64{
30 * 1024 * 1024 * 1024,
70 * 1024 * 1024 * 1024,
2 * 1024 * 1024 * 1024,
}
graph := asciigraph.Plot(data,
asciigraph.Height(5),
asciigraph.Width(45),
asciigraph.YAxisValueFormatter(func(v float64) string {
return fmt.Sprintf("%.2f GiB", v/1024/1024/1024)
}),
)

fmt.Println(graph)
}
```

运行该示例会渲染出如下图形：

```
70.00 GiB ┤ ╭──────╮
56.40 GiB ┤ ╭───────╯ ╰────╮
42.80 GiB ┤ ╭──────╯ ╰───╮
29.20 GiB ┼──╯ ╰────╮
15.60 GiB ┤ ╰───╮
 2.00 GiB ┤ ╰─
```

使用 `XAxisRange(min, max)` 在图形下方添加带标签的 X 轴。`XAxisTickCount(n)` 控制显示多少个刻度标记（默认 5，最小 2）。

```go
package main

import (
"fmt"

"github.com/guptarohit/asciigraph"
)

func main() {
data := []float64{3, 4, 9, 6, 2, 4, 5, 8, 5, 10, 2, 7, 2, 5, 6}
graph := asciigraph.Plot(data,
asciigraph.XAxisRange(0, 14),
asciigraph.XAxisTickCount(3),
)

fmt.Println(graph)
}
```

运行该示例会渲染出如下图形：

```
10.00 ┤ ╭╮
 9.00 ┤ ╭╮ ││
 8.00 ┤ ││ ╭╮││
 7.00 ┤ ││ ││││╭╮
 6.00 ┤ │╰╮ ││││││ ╭
 5.00 ┤ │ │ ╭╯╰╯│││╭╯
 4.00 ┤╭╯ │╭╯ ││││
 3.00 ┼╯ ││ ││││
 2.00 ┤ ╰╯ ╰╯╰╯
 └┬──────┬──────┬
 0 7 14
```

```go
package main

import (
"fmt"

"github.com/guptarohit/asciigraph"
)

func main() {
data := make([][]float64, 4)
for i := 0; i < 4; i++ {
for x := -20; x <= 20; x++ {
v := math.NaN()
if r := 20 - i; x >= -r && x <= r {
v = math.Sqrt(math.Pow(float64(r), 2)-math.Pow(float64(x), 2)) / 2
}
data[i] = append(data[i], v)
}
}
graph := asciigraph.PlotMany(data, asciigraph.Precision(0), asciigraph.SeriesColors(
asciigraph.Red,
asciigraph.Yellow,
asciigraph.Green,
asciigraph.Blue,
))

fmt.Println(graph)
}
```

运行该示例会渲染出如下图形：

`SeriesColorGradient` 不是为每个序列使用单一纯色，而是根据每个点的值沿着某个调色板着色——高值用暖色调，低值用冷色调。内置的 `HeatmapSpectrum` 提供了一个现成的冷到暖调色板，你也可以传入自定义的颜色停靠点（最低值优先）。

```go
package main

import (
"fmt"

"github.com/guptarohit/asciigraph"
)

func main() {
data := []float64{1, 2, 3, 4, 5, 6, 7, 8, 9, 8, 7, 6, 5, 4, 3, 2, 1}
graph := asciigraph.Plot(data,
asciigraph.Height(10),
asciigraph.SeriesColorGradient(asciigraph.HeatmapSpectrum...),
)

fmt.Println(graph)
}
```

运行该示例会渲染出如下图形：

`ColorAbove` 和 `ColorBelow` 会突出显示越过某个阈值的点——用于标记告警很有用，比如 CPU 使用率飙升或磁盘空间预警——而无需对整个序列重新着色。`ColorAbove` 为严格高于其阈值（value > threshold）的点着色，`ColorBelow` 为严格低于其阈值（value < threshold）的点着色。处于两者之间的点保持其正常颜色——即序列颜色，或在设置了 `SeriesColorGradient` 时的渐变颜色。它们优先于 `SeriesColorGradient` 和 `SeriesColors`；当两个阈值同时匹配同一个点时，`ColorAbove` 胜出。

```go
package main

import (
"fmt"

"github.com/guptarohit/asciigraph"
)

func main() {
data := []float64{42, 48, 55, 81, 85, 91, 87, 34, 12, 17, 10, 18, 55, 50}
graph := asciigraph.Plot(data,
asciigraph.Height(10),
asciigraph.Width(25),
asciigraph.LowerBound(0),
asciigraph.UpperBound(100),
asciigraph.Caption("CPU usage % (red: critical, green: idle)"),
asciigraph.ColorAbove(asciigraph.Red, 80),
asciigraph.ColorBelow(asciigraph.Green, 25),
)

fmt.Println(graph)
}
```

运行该示例会渲染出如下图形——高于 80% 的飙升点显示为红色，低于 25% 的下降点显示为绿色：

在命令行中，相同的阈值可通过 `-ca` 和 `-cb` 标志使用，每个标志接受一个 `color,value`（颜色,值）对：

```
seq 1 100 | asciigraph -h 10 -ca red,80 -cb green,25
```

图形可以为每个序列包含图例，使其更易于解读。

```go
package main

import (
"fmt"

"github.com/guptarohit/asciigraph"
"math"
)

func main() {
data := make([][]float64, 3)
for i := 0; i < 3; i++ {
for x := -12; x <= 12; x++ {
v := math.NaN()
if r := 12 - i; x >= -r && x <= r {
v = math.Sqrt(math.Pow(float64(r), 2)-math.Pow(float64(x), 2)) / 2
}
data[i] = append(data[i], v)
}
}
graph := asciigraph.PlotMany(data,
asciigraph.Precision(0),
asciigraph.SeriesColors(asciigraph.Red, asciigraph.Green, asciigraph.Blue),
asciigraph.SeriesLegends("Red", "Green", "Blue"),
asciigraph.Caption("Series with legends"))

fmt.Println(graph)
}
```

运行该示例会渲染出如下图形：

本包还附带了一个用于命令行使用的小工具。假设 `$GOPATH/bin` 在你的 `$PATH` 中，使用以下命令安装 CLI：

```
go install github.com/guptarohit/asciigraph/cmd/asciigraph@latest
```

或拉取 Docker 镜像：

```
docker pull ghcr.io/guptarohit/asciigraph:latest
```

或从发布页面下载二进制文件。

```
> asciigraph --help

Usage of asciigraph:
asciigraph [options]

Options:
 -ac axis color
 图形的 y 轴颜色
 -b buffer
 启用实时图形时的数据点缓冲区，默认等于 `width`
 -c caption
 图形的标题说明
 -ca above
 为高于某阈值的点着色："color,value"（例如 "red,4"）
 -cb below
 为低于某阈值的点着色："color,value"（例如 "green,2"）
 -cc caption color
 图形的标题说明颜色
 -d delimiter
 用于在输入流中分割数据点的数据分隔符（默认 ","）
 -f fps
 启用实时图形时，设置 fps 以控制图形渲染的频率（默认 24）
 -g gradient
 按值为点着色的渐变调色板："spectrum" 表示内置热力图，或以逗号分隔、由低到高的颜色停靠点（例如 "blue,cyan,green"）
 -h height
 以文本行数表示的高度，0 表示自动缩放
 -lb lower bound
 下界，设置纵轴的最小值（若序列包含更低的值则忽略）（默认 +Inf）
 -lc label color
 图形的 y 轴标签颜色
 -o offset
 标签的列偏移量（默认 3）
 -p precision
 y 轴上数据点标签的精度（默认 2）
 -r realtime
 为数据流启用实时图形
 -sc series colors
 与每个序列相对应的、以逗号分隔的序列颜色
 -sl series legends
 与每个序列相对应的、以逗号分隔的序列图例
 -sn number of series
 输入数据中的序列（列）数量（默认 1）
 -ub upper bound
 上界，设置纵轴的最大值（若序列包含更大的值则忽略）（默认 -Inf）
 -w width
 以列数表示的宽度，0 表示自动缩放
 -xmax value
 x 轴最大值（默认 NaN）
 -xmin value
 x 轴最小值（默认 NaN）
 -xt tick count
 x 轴刻度数量（默认 5，最小 2）
```

asciigraph 期望从标准输入（stdin）读取数据点。无效值会记录到标准错误（stderr）。

通过 stdin 向其输入数据点：

```
seq 1 72 | asciigraph -h 10 -c "plot data from stdin" -xmin 0 -xmax 40 -xt 5
```

或使用 Docker 镜像：

```
seq 1 72 | docker run -i --rm ghcr.io/guptarohit/asciigraph -h 10 -c "plot data from stdin" -xmin 0 -xmax 40 -xt 5
```

输出：

```
72.00 ┤ ╭────
64.90 ┤ ╭──────╯
57.80 ┤ ╭──────╯
50.70 ┤ ╭──────╯
43.60 ┤ ╭──────╯
36.50 ┤ ╭───────╯
29.40 ┤ ╭──────╯
22.30 ┤ ╭──────╯
15.20 ┤ ╭──────╯
 8.10 ┤ ╭──────╯
 1.00 ┼──╯
 └┬─────────────────┬─────────────────┬────────────────
```
