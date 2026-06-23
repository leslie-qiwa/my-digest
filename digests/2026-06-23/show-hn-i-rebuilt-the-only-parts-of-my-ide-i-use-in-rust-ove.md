# Show HN: I rebuilt the only parts of my IDE I use, in Rust, over a weekend

我不懂 Rust。

如今我几乎不再打开完整的 IDE——在 AI 出现之前，我每年要做数千次提交，现在我主要待在 IDE 的提交和差异视图里，这是为数不多"重"到让我愿意等 JVM 启动的功能。

所以 Kyde 就是这样一个东西。一个快速的原生提交与差异代码编辑器——一个 macOS 的 Git 客户端。（它应该支持 Windows 和 Linux，只是我注释掉了构建配置，因为我不会主动进行质量保证和维护这些发行版。）

~120fps 滚动浏览 37k 行的 package-lock.json
——视口虚拟化 + 离线程高亮。

- 快速。原生 GPU 渲染，低延迟。即使大文件也能 120fps。
- 熟悉。精心调校的暗色主题，让任何长期使用现代 IDE 的人都能感到自在。
- 并排差异对比，带词级高亮和中央装订线，可按块暂存/还原——`git add -p` 的可视化版本。
- 打开文件夹即可编辑，带 tree-sitter 语法高亮。

- gpui（Apache-2.0）——Zed 的原生 GPU 渲染 GUI 框架。没有 web，没有 Electron。
- git，通过 shell 调用。不使用 libgit2。
- similar（Apache-2.0）——行级和词级差异比较。

从零开始基于 gpui 构建，借鉴了现有编辑器的模式但不使用它们的代码。

手工调校的暗色调色板，可在运行时通过 `~/.config/kyde/theme.json` 配置。

- 无项目打开时的着陆页：可搜索的最近项目列表，显示分支和路径，持久化到 `~/.config/kyde/projects.json`。
- 通过原生文件夹选择器打开/新建项目。
- 文件夹树——可展开、可调整大小、文件类型图标、git 状态颜色。
- 文本编辑器——选择、撤销/重做、复制/剪切/粘贴、Tab/Shift-Tab 缩进、⌘-退格、行号、当前行高亮、输入法、自动保存。
- 查找与替换——`⌘F` 查找（`⌘G`/`⇧⌘G` 循环），`⌘R` 替换。
- 编辑器标签页，可滚动并跟随活动文件。
- PNG/JPG/GIF/WebP 图片预览。
- 通过 tree-sitter 语法高亮，按需从内置语言插件管理器安装。语言包：TypeScript/TSX、JavaScript、Rust、JSON、Markdown、Shell、CSS、SCSS、YAML、TOML、Python、HTML、Go、R、LaTeX——另有始终启用的 `.env` 和 `.gitignore` 高亮器，以及字体预览插件。每个语言包也是一个 Cargo feature，因此构建时可以只包含所需的语法（详情）。
- 语法支持的语言可进行代码折叠。
- Markdown 预览——编辑器旁的实时渲染面板。
- 内嵌多标签终端——基于真实 PTY 的 shell（alacritty 的 VTE 引擎），底部停靠，用 `⌃`` 切换。完整颜色支持、滚动回看、调整大小、多标签页。
- 鼠标选择 + 复制（`⌘C`）、粘贴（`⌘V`，支持括号粘贴模式）、`⌘`-点击 URL。
- 历史（`↑`）、Tab 补全、行编辑均可用——这就是你真正的 shell。
- 可选：使用 `--no-default-features` 完全移除（`terminal` Cargo feature），可减小约 2MB 二进制体积。

- 提交视图：更改文件列表 + 可编辑的并排差异——左侧为基准版本，右侧为实时工作副本，均有语法高亮。
- 从中央装订线按块暂存/还原，或整个文件操作；通过消息框提交。
- 原生窗口中的回滚——复选框树，可选删除新增文件，右键查看差异。
- 领先上游时可推送（状态栏按钮 + 上下文菜单）。
- 分支切换器——可搜索的树形结构，`/` 作为文件夹分隔，包含"最近"和"本地"根节点。
- 从文件树进行文件管理——新建文件、重命名、删除（带确认）。
- 跳转到文件（`⌘⇧O`/`⌘P`）和查找操作（`⌘⇧A`）模糊搜索。
- 在文件中查找（`⌘⇧F`）——跨仓库全文内容搜索（`git grep`），可直接跳转到匹配处。
- 临时文件——"Scratches"文件夹下的临时缓冲区。
- 状态栏中的面包屑导航。
- 岛屿布局——圆角面板、可拖动分隔线、活动导航栏、原生标题栏（双击缩放）、状态栏。
- 原生菜单栏——设置、FPS 监控切换、退出。
- 应用图标来自内置 logo。
- WebStorm/VSCode 预设，支持在 `~/.config/kyde/keymap.json` 中按操作覆盖。
- 首次启动时的键位方案选择器，可通过设置重新打开。
- 安装 shell 命令——可选将 `ky` 符号链接到 `~/.local/bin`。无需编辑 shell 配置文件，无需 sudo。

从 Releases 页面获取适用于你平台的最新构建。

macOS——下载 `kyde-macos.zip`，解压，将 Kyde.app 拖到 /Applications。应用尚未代码签名，因此首次启动会被 Gatekeeper 阻止——右键点击并选择"打开"（仅需一次），或清除隔离标记：

```
xattr -dr com.apple.quarantine /Applications/Kyde.app
```

不带参数启动进入项目视图，或指向一个仓库：

```
cargo run -- /path/to/repo
```

首次运行设置会提供安装 `ky` shell 命令（符号链接到 `~/.local/bin`——无需编辑 shell 配置，无需 `sudo`）；保持勾选即可从任意终端打开 Kyde：

```
ky              # 项目视图
ky /path/to/repo # 直接打开一个仓库
```

默认快捷键（WebStorm → VSCode）：

- 跳转到文件：`⌘⇧O` → `⌘P`
- 查找操作：`⌘⇧A`
- 文件内查找/替换：`⌘F`/`⌘R`
- 保存：`⌘S`
- 提交：`⌘K` → `⌘⏎`
- 提交视图：`⌘9` → `⌃⇧G`
- 浏览视图：`⌘1` → `⌘⇧E`
- 新建临时文件：`⌘⇧N`
- 设置：`⌘,`

需要 Rust 1.96+ 以及（macOS 上）Apple 的 Metal 工具链，gpui 使用它来编译着色器——如果在全新机器上报错"missing Metal Toolchain"，运行 `xcodebuild -downloadComponent MetalToolchain`。

```
cargo build --release  # 完整构建——所有语言语法均内置（默认）
cargo test             # 逻辑测试、性能守护和无头 gpui 冒烟测试
```

每个语言包是一个 Cargo feature，因此可以从二进制文件中完全移除未使用的语法（更小的镜像 + 常驻内存）：

```
cargo build --release --no-default-features --features rust,json,toml
```

大文件通过视口虚拟化（每帧仅对屏幕可见行进行排版）和异步高亮（大文件立即以纯文本打开，然后在后台线程高亮）保持流畅。一个 37k 行的 `package-lock.json` 可以以约 120fps 滚动。

通过 `perf_*` 时间预算测试、无头 gpui 冒烟测试（渲染每个界面，任何 panic 即失败）以及可切换的屏幕 FPS 监控器来保障性能。

- 预构建发布版仅限 macOS。我在 macOS 上开发，不会主动测试 Linux/Windows，因此只发布签名 + 公证的 macOS 构建，而非我无法保证质量的二进制文件。代码本身是跨平台的——gpui 在三个平台上都能运行，Linux/Windows 的打包已经在 `scripts/` 中存在（只是未连接到发布流程）。重新启用它们对于使用这些平台的贡献者来说是一个不错的入门 issue。在此之前，Linux/Windows 用户可以使用 `cargo build --release`。
- 尚不支持软换行或光标跟随滚动。编辑器使用扁平的 `String`；基于 rope 的缓冲区将在后续版本中支持超大文件编辑。
- 文件监控目前仅在获得焦点时刷新。实时更新可能会使用 `notify`。

如果你提交 PR，请友善一些——解释惯用的写法，不要嘲笑我的 `.clone()`。我会阅读每一条评论，然后悄悄去搜索什么是生命周期。

如果某个功能会增加大量体积，它应该作为插件实现。
