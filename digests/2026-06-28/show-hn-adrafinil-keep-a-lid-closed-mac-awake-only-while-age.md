# Show HN: Adrafinil – keep a lid-closed Mac awake only while agents work

rx 编号 006 ・ a·draf·i·nil /əˈdræfɪnɪl/ ・ 一款为机器准备的促清醒剂 ♡
清醒 ・ 有代理正在工作 |
睡眠 ・ 无代理，正常睡眠 |
服用注意 ・ 为那些在你入睡后仍在守望的机器而设。
现在是凌晨 3 点。你已经睡了。代理却没有——它仍在你几小时前开启的会话中沉思，而你像合上一只无法完全闭合的眼睑一样合上了它的盖子。
`caffeinate` 和 Amphetamine 是兴奋剂：它们让机器永远亢奋，无论是否有人在场。Adrafinil 则是促清醒剂。在代理获取它之前，它什么都不做；它只在工作存续期间让你的 Mac 在合盖状态下保持清醒，并在最后一个会话释放的那一刻清除。它只为工作而醒来——然后你们一同入睡。♡

仅在 AI 代理工作时才让你的 Mac 保持清醒。
Adrafinil 是一款 macOS 菜单栏应用，仅在 AI 编程代理拥有活动会话时才阻止系统睡眠——包括翻盖（合盖）睡眠。当没有代理工作时，睡眠行为不受影响：合上盖子，Mac 即正常睡眠。
它与 `caffeinate` 或 Amphetamine 这类始终开启的唤醒工具正相反。Adrafinil 只在代理（Claude Code、Codex、Cursor……）正处于任务中时介入，并在工作完成的那一刻让开。

⚠️ 特权睡眠控制。覆盖翻盖睡眠需要 root 权限。Adrafinil 将该权限隔离在一个微小、经过审计的辅助程序中，它只暴露 `setSleepBlocked(Bool)`——所有策略都位于一个非特权守护进程中。它为空闲睡眠持有一个标准的 `IOPMAssertion`，并使用 `pmset disablesleep` 处理翻盖（合盖）睡眠，此前已在设备上验证：更简洁的私有 `IOPMrootDomain` 路径无法让一台无外接显示器、合盖的 Mac 保持清醒。详见 Docs/ARCHITECTURE.md §2。

- 代理感知，而非始终开启。 仅当 ≥1 个代理会话持有断言时才阻止睡眠。零会话 → 正常睡眠，包括合盖。
- 9 款代理的钩子集成。 一键安装程序将 Adrafinil 接入 Claude Code、Codex、Cursor、Gemini CLI、Aider、Hermes、OpenCode、Cline 和 Pi 的钩子系统。
- 低于 50ms 的 CLI。 `adrafinil acquire` / `release` 由代理钩子调用，与守护进程往返耗时不到 50ms，因此绝不会拖慢代理的工作流程。
- 引用计数断言。 重叠的会话可以干净地叠加；只有当最后一个会话释放时，睡眠才会解除阻止。
- 热保护切断。 如果在合盖状态下表面/CPU 温度越过阈值，所有断言都会被强制释放，以免一台装在包里的 Mac 把自己烤熟。
- 空闲释放。 拥有者进程已死亡、或 CPU 空闲达 N 分钟的断言会被自动丢弃。
- 进程嗅探（可选）。 即使未安装钩子，守护进程也能在发现已知代理二进制文件运行时自动获取断言。
- 合盖音频提示 + 开盖摘要。 当你合上盖子时，提示音确认断言已被持有（屏幕已关闭，故无通知）；重新打开时会显示在你离开期间运行了什么、峰值温度，以及热保护切断是否触发。
- 干净卸载。 移除它在所有代理配置中添加的每一条钩子条目。

- macOS Tahoe 26.4。 这是我用于构建和测试的版本；它很可能也能在更早的 26.x 上运行，但我未在那些版本上测试过。
- Xcode 26+ 用于构建，并启用 Swift 6 严格并发。
- 标准安装需要管理员权限（特权辅助程序通过 `SMAppService` 安装）。非管理员安装路径会将 CLI 放入 `~/.local/bin` 而非 `/usr/local/bin`。

下载 Adrafinil——一个已签名、已公证的磁盘映像。打开它，将 Adrafinil 拖入"应用程序"，然后启动。首次启动会请求一次管理员权限以注册特权辅助程序。需要 macOS 26.4 或更高版本。
更喜欢自己构建？参见"构建"。

git clone https://github.com/kageroumado/adrafinil.git
cd adrafinil
open Adrafinil.xcodeproj

在 Xcode 中，选择 Adrafinil scheme 并运行。你需要设置一个开发团队用于代码签名——守护进程（LaunchAgent）和辅助程序（LaunchDaemon）已嵌入应用程序包中，并在应用启动时注册到系统。（源代码中未硬编码任何 Team ID；XPC 调用方检查会在运行时读取你自己的签名团队，因此在任何 Developer ID 下重新构建都会授权其自身组件，无需修改代码。）
若要在没有本地签名身份的情况下进行无头编译检查：

xcodebuild -project Adrafinil.xcodeproj -scheme Adrafinil -configuration Debug \
-destination 'generic/platform=macOS' \
CODE_SIGNING_ALLOWED=NO CODE_SIGNING_REQUIRED=NO CODE_SIGN_IDENTITY='' build

共享逻辑可作为 Swift 包独立构建和测试：

cd AdrafinilShared
swift test

代理不会直接与 Adrafinil 通信。每个代理的钩子系统都会调用捆绑的 CLI：

adrafinil acquire <session-key> --tool claude-code --reason "long build" # 当一个回合开始时
adrafinil release <session-key> # 当代理进入空闲时

持有是活动范围（activity-scoped）的，而非会话范围（session-scoped）的：Claude Code 在 `UserPromptSubmit` 时获取，在 `Stop` 时释放，因此只有在代理实际工作时才会保持 Mac 清醒——一个打开但在提示符处空闲的会话可以让它正常睡眠。
守护进程按会话键进行引用计数，并在计数非零时请求辅助程序阻止睡眠。
代理也可以为一个超出其回复时长的后台任务（如一次漫长的构建或部署）通过限时持有让 Mac 保持清醒——既可以直接调用 `adrafinil hold`，对于支持 MCP 的代理，也可以通过 `adrafinil mcp` 所提供的捆绑 MCP 工具：

adrafinil hold --for 30m --reason "deploy" # 保持清醒最多 30 分钟，随后自动释放
adrafinil mcp # 在 stdio 上讲 Model Context Protocol（供代理使用）

其他子命令：`status`、`install-hooks`、`uninstall-hooks`、`daemon-status`、`version`。

跨三个权限层级的四个产品（完整细节，包括 Xcode 项目布局，见 Docs/ARCHITECTURE.md）：

┌──────────────────────────────────────────────────────────────┐
│ Adrafinil.app（菜单栏应用，面向用户）                          │
│ • 状态项、设置、安装程序 GUI、开盖摘要                          │
└─────────────────────────────┬────────────────────────────────┘
                              │ XPC
                              ▼
┌──────────────────────────────────────────────────────────────┐
│ AdrafinilDaemon（LaunchAgent，以用户身份运行，始终开启）       │
│ • 引用计数的断言注册表                                          │
│ • 进程监视器（kqueue NOTE_EXIT + 周期性扫描）                  │
│ • 热监控（SMC） • 盖状态监控（IORegistry）                     │
│ • 合盖提示音 • 位于 …/Adrafinil/cli.sock 的 CLI 套接字         │
└─────────────────────────────┬────────────────────────────────┘
                              │ XPC（特权 Mach 服务）
                              ▼
┌──────────────────────────────────────────────────────────────┐
│ AdrafinilHelper（SMAppService LaunchDaemon，root）             │
│ • 唯一接触睡眠阻止 API 的组件                                   │
│ • setSleepBlocked(Bool) + 只读状态/版本                        │
│ • 验证调用方的代码签名要求                                      │
└──────────────────────────────────────────────────────────────┘

adrafinil（CLI，随 .app 一起发布，符号链接到 PATH）
• acquire / release / hold / mcp / status / install-hooks / uninstall-hooks
• 连接到守护进程套接字；<50ms 往返

- AdrafinilShared——一个在每个目标之间共享的 Swift 包：数据模型（`AgentKind`、`Assertion`）、IPC 线路格式、`AssertionRegistry`、`CallerVerifier`、钩子安装规范，以及 CLI 参数解析器。单元测试也位于此处。
- 辅助程序保持易于审计。 它不持有任何策略——引用计数、热、空闲和盖逻辑全部位于守护进程中。特权表面仅是一个可变端点加上只读自省。
- 守护进程是真相之源。 应用是纯视图层；它可以自由退出和重启，而不影响已持有的断言。
- 公共 IOPM 断言无法战胜翻盖睡眠。 使用公共类型的 `IOPMAssertionCreateWithName`（因此也包括 `caffeinate`）无法让一台合盖的 Mac 保持清醒。Adrafinil 的 v1 使用 `pmset disablesleep 1`，这一手段较为粗暴（它同时也禁用了空闲睡眠），且必须在关机时清除，否则会泄漏——辅助程序会在重生时先重置为 `disablesleep 0`，再重新应用状态。
- 守护进程的处理器在任意队列上运行。 XPC 和套接字回调
