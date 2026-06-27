# WordStar: A Writer's Word Processor (1996)

科幻作家
|
|
至于我为什么时至 21 世纪仍在使用并热爱 WordStar，请见下文。 |

"索耶下面那篇关于 WordStar 的长文极具洞见。" —马修·基尔申鲍姆（Matthew Kirschenbaum），《追踪修改：文字处理的文学史》（Track Changes: A Literary History of Word Processing）作者

"一款出色的文字处理程序，名叫 WordStar。它从不崩溃，从不出错，我对它爱得无以复加。" —迈克尔·夏邦（Michael Chabon），《卡瓦利与克雷的神奇冒险》（The Amazing Adventures of Kavalier & Clay）作者

"WordStar 有多好，连我都成了它的熟手，用它写了十几部长篇小说和数百篇短篇小说，足以证明这一点。这是个了不起的系统，尤其是跟 MS Word 相比。" —埃多·范贝尔科姆（Edo van Belkom），《尖叫女王》（Scream Queen）作者

"我很高兴向这些天才致意 [指 Rob Barnaby 和 Seymour Rubinstein，WordStar 的创造者]，是他们让我重获新生、再度提笔。我在 1978 年宣布退休，如今却有六本书在写，还有两本 [很可能要写]，全靠 WordStar。" —亚瑟·C·克拉克（Arthur C. Clarke），《2001：太空漫游》（2001: A Space Odyssey）作者

"我有一件秘密武器：我用 WordStar。它能完成我对文字处理程序的一切要求。" —乔治·R·R·马丁（George R.R. Martin），《权力的游戏》（A Game of Thrones）作者

"WordStar 美妙极了。我爱它。它逻辑严谨、优美、完美。相比之下，Microsoft Word 简直是一团疯狂。" —安妮·赖斯（Anne Rice），《夜访吸血鬼》（Interview with the Vampire）作者

许多科幻作家——包括我本人、罗杰·麦克布莱德·艾伦（Roger MacBride Allen）、杰拉尔德·布兰特（Gerald Brandt）、杰弗里·A·卡弗（Jeffrey A. Carver）、亚瑟·C·克拉克（Arthur C. Clarke）、大卫·杰罗德（David Gerrold）、特伦斯·M·格林（Terence M. Green）、詹姆斯·冈恩（James Gunn）、马修·休斯（Matthew Hughes）、唐纳德·金斯伯里（Donald Kingsbury）、埃里克·科塔尼（Eric Kotani）、保罗·莱文森（Paul Levinson）、乔治·R·R·马丁（George R. R. Martin）、冯达·麦金太尔（Vonda McIntyre）、基特·里德（Kit Reed）、詹妮弗·罗伯森（Jennifer Roberson）以及埃多·范贝尔科姆（Edo van Belkom）——至今仍把 DOS 版 WordStar 作为我们首选的写作工具。

尽管如此，我们当中大多数人多年来一直忍受着对这一选择的无脑批评，批评者通常是 WordPerfect 用户，尤其是那些除了那一款程序之外从未试过别的东西的 WordPerfect 用户。我用过 WordStar、WordPerfect、Word、MultiMate、Sprint、XyWrite，以及几乎所有其他 MS-DOS 和 Windows 文字处理软件包，而 WordStar 是我在键盘上进行创作时迄今为止最喜爱的选择。

这正是关键所在：辅助创作。为了说明 WordStar 在这方面如何比其他程序做得更好，让我先从一点历史讲起。

WordStar 于 1978 年首次发布，那时计算机键盘还没有任何标准化。当时，许多键盘缺少用于移动光标的方向键，也缺少用于发出命令的专用功能键。有些键盘甚至缺少诸如 `Tab`、`Insert`、`Delete`、`Backspace` 和 `Enter` 这样的按键。你能指望的，大概只有一套标准的 QWERTY 打字机字母数字键布局和一个 `Control` 键。`Control` 键是一个专门的换挡键。当它与某个字母键同时按下时，会让键盘生成一条特定的命令指令，而不是那个字母。这些控制码命名为 `Ctrl-A` 到 `Ctrl-Z`（还有几个标点键也能生成控制码）。在文本中，控制码常用在字母前加一个尖号（caret）来表示，就像这样：`^A`。

WordStar 最初的设计者赛默·鲁宾斯坦（Seymour Rubinstein）和罗布·巴纳比（Rob Barnaby）选定了五个控制码作为前缀，用来调出更多的功能菜单：`^O` 用于屏幕（On-screen）功能；`^Q` 用于快速（Quick）光标功能；`^P` 用于打印（Print）功能；`^K` 用于块（block）和文件功能；`^J` 用于帮助。

那么，前三个是按字母助记的。最后两个，`^K` 和 `^J`，乍看之下也许像是随意的选择。其实不然。看看打字机键盘。你会发现，对于盲打者而言，右手最有力的两根手指就停在主打字行上的 `^J` 和 `^K` 上。WordStar 认识到，最常用的功能应当最容易在物理上执行。

为了充当让光标向上、向左、向右或向下移动的方向键，WordStar 采用了 `^E`、`^S`、`^D` 和 `^X`。同样，看看打字机键盘就能明白其中的逻辑。这四个键在左手下方排成一个菱形：

```
  E
S   D
  X
```

这种基于位置（而非字母）的助记法，构成了 WordStar 界面的很大一部分。围绕 `E`/`S`/`D`/`X` 菱形，还聚集着更多的光标移动命令：

```
W E R
A S D F
Z X C
```

`^A` 和 `^F` 位于主打字行上，按词向左和向右移动光标。`^W` 和 `^Z` 位于光标上移和下移命令的左侧，按单行向上和向下滚动屏幕。`^R` 和 `^C` 位于光标上移和下移命令的右侧，每次一页地向上和向下滚动屏幕（这里的"页"是计算机意义上的一整屏文本）。

前面提到的快速光标移动菜单前缀 `^Q`，进一步扩展了这个菱形的威力。正如 `^E`、`^S`、`^D`、`^X` 按单个字符上、左、右、下移动光标一样，`^QE`、`^QS`、`^QD` 和 `^QX` 则把光标一直移到屏幕的顶部、左端、右端或底部。`^W` 向上滚动一行；`^QW` 连续向上滚动。`^Z` 向下滚动一行；`^QZ` 连续向下滚动。而既然 `^R` 和 `^C` 带你到屏幕的顶部和底部，`^QR` 和 `^QC` 就带你到文档的顶部和底部。`^Q` 命令还有很多，但我想从这一小部分示例中你就能看出，WordStar 的界面有一套内在的逻辑——这正是许多其他程序所严重欠缺的，尤其是 WordPerfect。

如今，对于其中许多功能，IBM PC 键盘上都有专用键。如果你愿意，WordStar 也允许你使用这些键。但盲打者发现，使用 WordStar 的 `Control` 键命令要高效得多，因为这些命令可以在主键行上直接敲出，无需在键盘别处搜寻特殊键。正因如此，许多应用程序——包括 dBase、SuperCalc、SideKick、CompuServe 的 TAPCIS 和 OzCis、Genie 的 Aladdin、Xtree Pro、Joe's Own Editor、VDE，甚至 Microsoft 随 MS-DOS 5.0 及更高版本附带的编辑器——都采用了部分或全部 WordStar 界面。

有些键盘把 `Control` 键放在字母 `A` 的左边。这让使用 WordStar 命令变得非常简单。另一些键盘则把 `CapsLock` 放在 `A` 旁边，而把 `Control` 键放在左 `Shift` 键下方，这就使得敲 WordStar 命令有点费劲。正因如此，WordStar 附带了一个名为 SWITCH.COM 的工具，可选择性地交换 `CapsLock` 键和 `Control` 键的功能。其他文字处理程序的问题之一在于，许多命令只能通过功能键和专用光标键方便地发出，而这些键的位置在不同键盘之间变化极大（例如，功能键有时排成左侧两列各五个，有时又排成顶部横贯一行；光标键有时聚成一个菱形，有时又排成一个倒 T 形；在笔记本电脑上，你可能不得不按一个特殊的 `Fn` 键再配合方向键，才能使用 `PgUp` 等功能，使得使用这些程序成了一种身体上的扭曲练习）。但要把任意键盘变成一个理想的 WordStar 键盘，你所要做的全部，就是在必要时运行那个 `CapsLock`/`Control` 切换工具。其他键的位置都无关紧要，因为用 WordStar 时你根本不需要它们。

另一方面，WordPerfect 的界面迫使盲打者不断地把手从主打字行移开，从而拖慢速度。要发出一条 WordPerfect 命令，你必须先按一个功能键，或者单独按，或者与 `Control`、`Shift` 或 `Alt` 键同时按。然后，对于许多功能，你还必须选择一个子功能。既然你的手现在已经移到了那排功能键上，你能用它们来选择子功能吗？不能。相反，你接下来必须把手重新移到数字键上，按编号选择你的子功能。最后，在继续打字之前，你还得把手重新对准主键行（较新版本的 WordPerfect 试图理顺这套折磨人的界面，但它仍然难用）。

事实上，在 WordPerfect 两者中
