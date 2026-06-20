# go-pretty: Pretty Print Tables, Lists and Text on the Terminal

美化控制台输出的工具集，支持表格、列表、进度条、文本等，高度强调自定义和灵活性。

```
go get github.com/jedib0t/go-pretty/v6
```

导入所需的包：

```go
import (
    "github.com/jedib0t/go-pretty/v6/table"
    "github.com/jedib0t/go-pretty/v6/list"
    "github.com/jedib0t/go-pretty/v6/progress"
    "github.com/jedib0t/go-pretty/v6/text"
)
```

注意：当前主版本为 v6。详见 Go 模块版本管理。

支持颜色、自动合并、排序、分页及多种输出格式（ASCII、HTML、Markdown、CSV、TSV）的表格美化打印。

```
+-----+------------+-----------+--------+-----------------------------+
|   # | FIRST NAME | LAST NAME | SALARY |                             |
+-----+------------+-----------+--------+-----------------------------+
|   1 | Arya       | Stark     |   3000 |                             |
|  20 | Jon        | Snow      |   2000 | You know nothing, Jon Snow! |
| 300 | Tyrion     | Lannister |   5000 |                             |
+-----+------------+-----------+--------+-----------------------------+
|     |            | TOTAL     |  10000 |                             |
+-----+------------+-----------+--------+-----------------------------+
```

试试嵌套彩色表格演示：

```
go run github.com/jedib0t/go-pretty/v6/cmd/demo-table@latest colors
```

跟踪一个或多个任务的进度，支持预计完成时间（ETA）、速度计算、不确定状态指示器及可自定义样式。

支持多层级、缩进及多种输出格式（ASCII、HTML、Markdown）的层级列表美化打印。

```
╭─ Game Of Thrones
│  ├─ Winter
│  ├─ Is
│  ╰─ Coming
│     ├─ This
│     ├─ Is
│     ╰─ Known
╰─ The Dark Tower
   ╰─ The Gunslinger
```

用于操作字符串/文本的实用函数，完整支持 ANSI 转义序列。在本仓库的其他包中被广泛使用。

功能包括：对齐（水平/垂直）、颜色与格式化、光标控制、文本转换（大小写、JSON、时间、URL）、字符串操作（填充、修剪、换行）等。
