# gookit/color: Terminal Color Rendering Library

一个支持 16 色/256 色/真彩色的命令行颜色库，提供通用 API 方法和 Windows 支持。

基本颜色预览：

现在，256 色和 RGB 颜色也已支持在 Windows CMD 和 PowerShell 中使用：

- 使用简单，零依赖
- 支持丰富的颜色输出：16 色（4 位）、256 色（8 位）、真彩色（24 位，RGB）
- 16 色输出是最常用且支持最广泛的，可在任何 Windows 版本上运行
- 自 `v1.2.4` 起，256 色（8 位）、真彩色（24 位）支持 Windows CMD 和 PowerShell
- 有关真彩色支持的信息，请参阅此 gist
- 支持将 `HEX` `HSL` 值转换为 RGB 颜色
- 通用 API 方法：`Print`、`Printf`、`Println`、`Sprint`、`Sprintf`
- 支持 HTML 标签风格的颜色渲染，例如 `<green>message</> <fg=red;bg=blue>text</>`。
  - 除了使用内置标签外，还支持自定义颜色属性
  - 自定义颜色属性支持使用 16 色颜色名称、256 色颜色值、RGB 颜色值和十六进制颜色值
  - 支持在 Windows `cmd` 和 `powerShell` 终端中使用
- 基本颜色：`Bold`、`Black`、`White`、`Gray`、`Red`、`Green`、`Yellow`、`Blue`、`Magenta`、`Cyan`
- 附加样式：`Info`、`Note`、`Light`、`Error`、`Danger`、`Notice`、`Success`、`Comment`、`Primary`、`Warning`、`Question`、`Secondary`
- 支持通过设置 `NO_COLOR` 来禁用颜色，或使用 `FORCE_COLOR` 强制开启颜色渲染。
- 支持 RGB、256、16 色之间的转换

查看 GitHub 的 godoc

```
go get github.com/gookit/color
```

```go
package main

import (
	"fmt"

	"github.com/gookit/color"
)

func main() {
	// 快速使用包函数
	color.Redp("简单易用的颜色")
	color.Redln("简单易用的颜色")
	color.Greenp("简单易用的颜色\n")
	color.Cyanln("简单易用的颜色")
	color.Yellowln("简单易用的颜色")

	// 像 fmt.Print* 一样快速使用
	color.Red.Println("简单易用的颜色")
	color.Green.Print("简单易用的颜色\n")
	color.Cyan.Printf("简单易用的 %s\n", "颜色")
	color.Yellow.Printf("简单易用的 %s\n", "颜色")

	// 像函数一样使用
	red := color.FgRed.Render
	green := color.FgGreen.Render
	fmt.Printf("%s 行 %s 库\n", red("命令"), green("颜色"))

	// 自定义颜色
	color.New(color.FgWhite, color.BgBlack).Println("自定义颜色样式")

	// 也可以：
	color.Style{color.FgCyan, color.OpBold}.Println("自定义颜色样式")

	// 内部主题/样式：
	color.Info.Tips("消息")
	color.Info.Prompt("消息")
	color.Info.Println("消息")
	color.Warn.Println("消息")
	color.Error.Println("消息")

	// 使用样式标签
	color.Print("<suc>你</><comment>好</>, <cyan>欢</><red>迎</>\n")
	// 自定义标签属性：支持使用 16 色颜色名称、256 色颜色值、RGB 颜色值和十六进制颜色值
	color.Println("<fg=11aa23>你</><bg=120,35,156>好</>, <fg=167;bg=232>欢</><fg=red>迎</>")

	// 应用样式标签
	color.Tag("info").Println("info 样式文本")

	// 提示消息
	color.Info.Prompt("提示样式消息")
	color.Warn.Prompt("提示样式消息")

	// 提示消息
	color.Info.Tips("提示样式消息")
	color.Warn.Tips("提示样式消息")
}
```

运行示例：`go run ./_examples/demo.go`

支持任何 Windows 版本。提供通用 API 方法：`Print`、`Printf`、`Println`、`Sprint`、`Sprintf`

```go
color.Bold.Println("粗体消息")
color.Cyan.Println("黄色消息")
color.Yellow.Println("黄色消息")
color.Magenta.Println("黄色消息")

// 仅使用前景色
color.FgCyan.Printf("简单易用的 %s\n", "颜色")
// 仅使用背景色
color.BgRed.Printf("简单易用的 %s\n", "颜色")
```

运行示例：`go run ./_examples/color_16.go`

```go
// 完全自定义：前景色、背景色、选项
myStyle := color.New(color.FgWhite, color.BgBlack, color.OpBold)
myStyle.Println("自定义颜色样式")

// 也可以：
color.Style{color.FgCyan, color.OpBold}.Println("自定义颜色样式")
```

自定义设置控制台配置：

```go
// 设置控制台颜色
color.Set(color.FgCyan)

// 打印消息
fmt.Print("消息")

// 重置控制台设置
color.Reset()
```

提供通用 API 方法：`Print`、`Printf`、`Println`、`Sprint`、`Sprintf`

使用预定义样式打印消息：

```go
color.Info.Println("信息消息")
color.Notice.Println("通知消息")
color.Error.Println("错误消息")
// ...
```

运行示例：`go run ./_examples/theme_basic.go`

提示样式

```go
color.Info.Tips("信息提示消息")
color.Notice.Tips("通知提示消息")
color.Error.Tips("错误提示消息")
color.Secondary.Tips("次要提示消息")
```

运行示例：`go run ./_examples/theme_tips.go`

提示样式

```go
color.Info.Prompt("信息提示消息")
color.Notice.Prompt("通知提示消息")
color.Error.Prompt("错误提示消息")
// ...
```

运行示例：`go run ./_examples/theme_prompt.go`

块样式

```go
color.Danger.Block("危险块消息")
color.Warn.Block("警告块消息")
// ...
```

运行示例：`go run ./_examples/theme_block.go`

256 色自 `v1.2.4` 起支持 Windows CMD、PowerShell 环境

```go
color.C256(val uint8, isBg ...bool) Color256
```

```go
c := color.C256(132) // 前景色
c.Println("消息")
c.Printf("格式化 %s", "消息")

c := color.C256(132, true) // 背景色
c.Println("消息")
c.Printf("格式化 %s", "消息")
```

可同时设置前景色和背景色。

```go
S256(fgAndBg ...uint8) *Style256
```

```go
s := color.S256(32, 203)
s.Println("消息")
s.Printf("格式化 %s", "消息")
```

带选项：

```go
s := color.S256(32, 203)
s.SetOpts(color.Opts{color.OpBold})
s.Println("带选项的样式")
s.Printf("带 %s 的样式\n", "选项")
```

运行示例：`go run ./_examples/color_256.go`

RGB 颜色自 `v1.2.4` 起支持 Windows `CMD`、`PowerShell` 环境

预览：

运行示例：

运行示例：`go run ./_examples/color_rgb.go`

示例：

```go
color.RGB(30, 144, 255).Println("消息。使用 RGB 数值")

color.HEX("#1976D2").Println("蓝色加深")
color.HEX("#D50000", true).Println("红色强调。使用 HEX 样式")

color.RGBStyleFromString("213,0,0").Println("红色强调。使用 RGB 数值")
color.HEXStyle("eee", "D50000").Println("深紫色颜色")
```

```go
color.RGB(r, g, b uint8, isBg ...bool) RGBColor
```

```go
c := color.RGB(30,144,255) // 前景色
c.Println("消息")
c.Printf("格式化 %s", "消息")

c := color.RGB(30,144,255, true) // 背景色
c.Println("消息")
c.Printf("格式化 %s", "消息")
```

从十六进制颜色字符串创建样式：

```go
color.HEX(hex string, isBg ...bool) RGBColor
```

```go
c := color.HEX("ccc") // 也可以："cccccc" "#cccccc"
c.Println("消息")
c.Printf("格式化 %s", "消息")

c = color.HEX("aabbcc", true) // 作为背景色
c.Println("消息")
c.Printf("格式化 %s", "消息")
```

可同时设置前景色和背景色。

```go
color.NewRGBStyle(fg RGBColor, bg ...RGBColor) *RGBStyle
```

```go
s := color.NewRGBStyle(RGB(20, 144, 234), RGB(234, 78, 23))
s.Println("消息")
s.Printf("格式化 %s", "消息")
```

从十六进制颜色字符串创建样式：

```go
color.HEXStyle(fg string, bg ...string) *RGBStyle
```

```go
s := color.HEXStyle("11aa23", "eee")
s.Println("消息")
s.Printf("格式化 %s", "消息")
```

带选项：

```go
s := color.HEXStyle("11aa23", "eee")
s.SetOpts(color.Opts{color.OpBold})
s.Println("带选项的样式")
s.Printf("带 %s 的样式\n", "选项")
```

`Print`、`Printf`、`Println` 函数支持自动解析和渲染颜色标签。

```go
text := `
<mga1>gookit/color:</>
一个 <green>命令行</>
<cyan>颜色库</> 支持 <fg=167;bg=232>256 色</>
和 <fg=11aa23;op=bold>真彩色</>，
<fg=mga;op=i>通用 API</> 方法
以及 <cyan>Windows</> 支持。
`
color.Print(text)
```

预览，代码请参阅 _examples/demo_tag.go：

标签格式：

- 使用内置标签：`<TAG_NAME>CONTENT</>`，例如：`<info>message</>`
- 自定义标签属性：`<fg=VALUE;bg=VALUE;op=VALUES>CONTENT</>`，例如：`<fg=167;bg=232>wel</>`

支持 Windows `cmd.exe` `PowerShell`。

示例：

```go
// 使用样式标签
color.Print("<suc>你</><comment>好</>, <cyan>欢</><red>迎</>")
color.Println("<suc>你好</>")
color.Println("<error>你好</>")
color.Println("<warning>你好</>")

// 自定义颜色属性
color.Print("<fg=yellow;bg=black;op=underscore;>你好，欢迎</>\n")

// 自定义标签属性：支持使用 16 色颜色名称、256 色颜色值、RGB 颜色值和十六进制颜色值
color.Println("<fg=11aa23>你</><bg=120,35,156>好</>, <fg=167;bg=232>欢</><fg=red>迎</>")
```

标签属性格式：
