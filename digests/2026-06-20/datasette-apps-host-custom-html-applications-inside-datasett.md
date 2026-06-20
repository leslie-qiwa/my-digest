# Datasette Apps: Host custom HTML applications inside Datasette

标题：Datasette Apps：在 Datasette 中托管自定义 HTML 应用

URL 来源：https://simonwillison.net/2026/Jun/18/datasette-apps/

发布时间：2026年6月20日（周六）00:04:23 GMT

Markdown 内容：

2026年6月18日

今天我们为 Datasette 发布了一个新插件 [datasette-apps](https://github.com/datasette/datasette-apps)，并在 Datasette 项目博客上发布了[这篇发布公告](https://datasette.io/blog/2026/datasette-apps/)。那篇文章介绍了"是什么"，但我打算在这里稍作展开，说明一下"为什么"。

#### 概要[#](https://simonwillison.net/2026/Jun/18/datasette-apps/#the-tl-dr)

Datasette Apps 是自包含的 HTML+JavaScript 应用程序，运行在你的 Datasette 应用所托管的严格约束的 `<iframe>` 沙箱中。它们可以使用 JavaScript 对 Datasette 中的数据执行只读 SQL 查询，如果你通过[一些存储查询](https://datasette.io/blog/2026/sql-write-queries/)进行配置，还可以执行写入查询。

这里有一个[非常简单的示例](https://agent.datasette.io/-/apps/01kvdp1d26g8trye3r4gc3yy9c)和一个[更复杂的自定义时间线示例](https://agent.datasette.io/-/apps/01ktvyaejhk07zskdx2tewxppe)——后者的界面如下：

![图片 1：一个网页应用的截图，标题为"Datasette timeline"，右上方有"All apps"、"Edit app"和"Pin"按钮，下方有一个"Full screen"按钮。在一个带边框的面板内，标题"Datasette timeline"下方有一个搜索框，显示"Search news, blog posts and releases…"，以及三个已勾选的复选框，分别标记为 News、Blog 和 Releases。下方文字显示"Showing 200 of 1,953 items"，后面是一个可滚动的时间线条目列表。每个条目都有一个彩色标签（蓝色"BLOG"或绿色"RELEASE"）、一个日期、一个蓝色链接标题和一段描述文字。可见的条目包括一篇日期为 2026-06-11 的"BLOG"文章，标题为"Datasette 1.0a33 with JSON extras in the API"，一个日期为 2026-06-11 的"RELEASE"，标题为"datasette 1.0a33"，以及一个日期为 2026-06-09 的"RELEASE"，标题为"llm 0.32a3"，每个都有正文内容和一个"▶ Show more"展开按钮。底部一个单独的面板显示一个折叠的"▶ 2 log entries"展开按钮。](https://static.simonwillison.net/static/2026/datasette-timeline-app.jpg)

Apps 可以运行 JavaScript 并渲染 HTML 和 CSS。它们的访问权限受到限制——它们运行所在的 `<iframe sandbox="allow-scripts allow-forms">` 阻止了对 cookies 或 localStorage 的访问，同时还注入了一个 CSP 头（感谢[这项研究](https://simonwillison.net/2026/Apr/3/test-csp-iframe-escape/)），防止它们向外部主机发起 HTTP 请求，从而阻止恶意或有缺陷的应用泄露私密数据。

Datasette Apps 最初是我尝试为 [Datasette Agent](https://datasette.io/blog/2026/datasette-agent/) 构建一个类似 Claude Artifacts 的机制，但我很快意识到这种沙箱模式的意义远不止在聊天界面中添加自定义应用，因此将其提升为 Datasette 生态系统中的一个顶层概念。

它们也是一种有趣的方式，可以把我[多年来在氛围编码 HTML 工具上的实验](https://tools.simonwillison.net/)变成我主项目的核心功能！

你可以通过 GitHub 登录 [agent.datasette.io](https://agent.datasette.io/) 演示实例来试用 Datasette Apps。

#### 为什么要做这个？[#](https://simonwillison.net/2026/Jun/18/datasette-apps/#why-build-this-)

从最初的版本开始，Datasette 就通过其 JSON API 提供了一个灵活的后端，用于创建自定义 HTML 应用。

我最早的 Datasette 项目之一是我在 Eventbrite 工作时做的一个内部文档搜索引擎——它的工作方式是通过 cron 任务将不同系统的文档导入 SQLite，然后通过一个 Datasette 实例提供服务，配以一个自定义的 HTML+JavaScript 搜索界面，直接查询 Datasette API。

我使用客户端 JavaScript 来构建 SQL 查询，这最初是作为一个工程上的玩笑，但后来证明是一种_非常高效_的应用迭代方式！

那个项目，加上我[构建 HTML 工具集](https://simonwillison.net/2025/Dec/10/html-tools/)的经验以及我的 [Claude Artifacts 实验](https://simonwillison.net/2024/Oct/21/claude-artifacts/)，让我确信：将 Datasette 风格的后端与自包含的 HTML 前端相结合，是一种极其强大的组合。

想象一下，如果 Claude Artifacts 能够访问一个持久化的关系数据库，那会有多大的用处。这就是我用 Datasette Apps 所构建的东西！

#### Datasette Apps 中的巧妙想法[#](https://simonwillison.net/2026/Jun/18/datasette-apps/#neat-ideas-in-datasette-apps)

以下是我在构建过程中总结出的一些我认为具有持久价值的想法和模式。

##### `<iframe sandbox="allow-scripts" srcdoc="...">` + `<meta http-equiv="Content-Security-Policy" content="default-src 'none'; script-src 'unsafe-inline'; style-src 'unsafe-inline'; img-src data: blob:;">`[#](https://simonwillison.net/2026/Jun/18/datasette-apps/#iframe-sandbox-allow)

这是使 Datasette Apps 得以实现的关键组合。我需要在一个高度敏感的域上运行不受信任的 HTML 和 JavaScript——一个经过身份验证的 Datasette 实例可能包含各种私密数据。`sandbox=` 属性使我能够以一种无法与父应用交互的方式运行不受信任的代码——它无法读取 DOM，无法访问 cookies，也无法从 `localStorage` 中窃取机密。但它仍然可以使用 `fetch()` 等方法从其他域加载内容（或泄露数据）。不过……事实证明，如果你在 HTML 页面_开头_放置一个 `<meta http-equiv="Content-Security-Policy">` 头，你就可以[设置额外的策略](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CSP)来锁定对其他域的访问。我曾担心恶意 JavaScript 能够更新或移除该头，但事实证明[这行不通](https://github.com/simonw/research/tree/main/test-csp-iframe-escape#readme)——一旦设置，CSP 策略对于该框架的内容就是不可变的。

##### 使用 `postMessage()` 和 `MessageChannel()` 的锁定 API[#](https://simonwillison.net/2026/Jun/18/datasette-apps/#locked-down-apis-with-postmessage-and-messagechannel-)

在将这些 iframe 锁定到几乎无法做任何有意义的事情之后，挑战在于重新开放它们，使其能够执行一组白名单操作，首先是对指定数据库的只读 SQL 查询。

我最初使用 `postMessage()` 构建了第一个版本，它允许子 iframe 向父窗口发送消息。我创建了一个简单的协议来请求父窗口运行 SQL 查询——父窗口可以在执行之前验证查询是否针对白名单中的数据库。

某个 LLM 工具，我记得是 GPT-5.5，提出如果 iframe 以某种方式从不受信任的域加载了额外代码，单独使用 `postMessage()` 可能会被利用。我认为这不适用于 Datasette Apps，但我也信奉纵深防御的原则，所以我[让 GPT-5.5 帮我](https://gist.github.com/simonw/0b29f301c2007808314eb04675c66916)迁移到了基于 [MessageChannel()](https://developer.mozilla.org/en-US/docs/Web/API/MessageChannel) 的传输方式。

`MessageChannel()` 的优势在于，如果页面导航到其他地方，通道会自动关闭，从而消除了执行来自不受信任外部页面命令的可能性。

##### 可见的日志，用于查询和错误[#](https://simonwillison.net/2026/Jun/18/datasette-apps/#visible-logs-for-queries-and-errors)

如果你访问[时间线演示](https://agent.datasette.io/-/apps/01ktvyaejhk07zskdx2tewxppe)并搜索字符串 `usercontent`，你会拉取到一些嵌入了来自 `user-images.githubusercontent.com` 域图片的搜索结果。该域不在 CSP 白名单中，因此会触发错误。

这些错误
