# Awesome Go: ~3000 Categorized Go Resources

Awesome Go
我们使用 Golang Bridge 社区 Slack 进行即时交流，按此处的表单填写即可加入。
赞助：
特别鸣谢
Awesome Go 没有月费，但我们有辛勤工作以维持其运行的员工。借助筹集到的资金，我们可以回报每一位参与者的付出！你可以查看我们如何计算账单与分配，这一切对整个社区公开。想成为本项目的支持者，请点击此处。
一个精选的 Go 语言出色框架、库和软件清单。灵感来自 awesome-python。
贡献：
请先快速浏览一下贡献指南。感谢所有贡献者；你们太棒了！
如果你看到此处某个软件包或项目已不再维护或不再合适，请提交一个拉取请求来改进此文件。谢谢！
目录
展开目录
- Awesome Go
- 目录
- Actor 模型
- 人工智能
- 音频与音乐
- 认证与授权
- 区块链
- 机器人构建
- 构建自动化
- 命令行
- 配置
- 持续集成
- CSS 预处理器
- 数据集成框架
- 数据结构与算法
- 数据库
- 数据库驱动
- 日期与时间
- 分布式系统
- 动态 DNS
- 可嵌入脚本语言
- 错误处理
- 文件处理
- 金融
- 表单
- 函数式
- 游戏开发
- 生成器
- 地理
- Go 编译器
- Goroutine
- GUI
- 硬件
- 图像
- IoT（物联网）
- 任务调度器
- JSON
- 日志
- 机器学习
- 消息传递
- Microsoft Office
- 杂项
- 自然语言处理
- 网络
- OpenGL
- ORM
- 包管理
- 性能
- 查询语言
- 反射
- 资源嵌入
- 科学与数据分析
- 安全
- 序列化
- 服务器应用
- 流处理
- 模板引擎
- 测试
- 文本处理
- 第三方 API
- 实用工具
- UUID
- 验证
- 版本控制
- 视频
- Web 框架
- WebAssembly
- Webhooks 服务器
- Windows
- 工作流框架
- XML
- 零信任
- 代码分析
- 编辑器插件
- Go Generate 工具
- Go 工具
- 软件包
- 资源
Actor 模型
用于构建基于 actor 的程序的库。
- asyncmachine-go/pkg/machine - 图控制流库（AOP、actor、状态机）。
- Ergo - 一个基于 actor 的框架，具有网络透明性，用于在 Golang 中创建事件驱动架构。灵感来自 Erlang。
- Goakt - 快速且分布式的 Actor 框架，使用协议缓冲区（protocol buffers）作为 Golang 的消息。
- Hollywood - 用 Golang 编写的极速且轻量级的 Actor 引擎。
- ProtoActor - 面向 Go、C# 和 Java/Kotlin 的分布式 actor。
人工智能
用于构建利用 AI 的程序的库。
- AegisFlow - 用于在 10 多个提供商之间路由、保护和监控 LLM 流量的 AI 网关。兼容 OpenAI 的 API、WASM 策略插件、金丝雀发布、实时仪表板。
- Aetheris - 具备事件溯源、检查点恢复以及至多一次（At-Most-Once）执行保证的 AI Agent 执行运行时。用 Go 编写。
- agent-sdk-go - 用于在 Temporal 上构建持久化 AI agent 的 Go SDK，支持工具、MCP、人工审批以及子 agent 委派。
- ai - 一个 Go 工具包，用于跨多个提供商构建 AI agent 和应用，提供统一的 LLM、嵌入、工具调用以及 MCP 集成。
- chromem-go - 面向 Go 的可嵌入向量数据库，具有类似 Chroma 的接口且零第三方依赖。内存存储，可选持久化。
- dakera-go - Dakera 自托管 agent 内存服务器的官方 Go 客户端 SDK，提供用于内存存储/召回、会话管理、命名空间操作以及衰减配置的类型化接口。
- fun - 在 Go 中使用大型语言模型（LLM）的最简单却强大的方式。
- goai - 用于构建 AI 应用的 Go SDK。一个 SDK，20 多个提供商。灵感来自 Vercel AI SDK。
- hotplex - AI Agent 运行时引擎，为 Claude Code、OpenCode、pi-mono 及其他 CLI AI 工具提供长寿命会话。提供全双工流式传输、多平台集成以及安全沙箱。
- langchaingo - LangChainGo 是一个用于开发由语言模型驱动的应用的框架。
- langgraphgo - 一个用于构建有状态、多 actor LLM 应用的 Go 库，基于 LangGraph 的理念，内置大量 Agent 架构。
- LocalAI - 开源的 OpenAI 替代方案，自托管 AI 模型。
- localaik - 类似 LocalStack 的 OpenAI 和 Gemini API 本地仿真；单个 Docker 容器，llama.cpp + Gemma 3 后端。
- mcp-go - Model Context Protocol 的 Go 实现，用于在 Go 中构建 MCP 服务器和客户端。
- Ollama - 在本地运行大型语言模型。
- OllamaFarm - 管理、负载均衡以及对成组的 Ollama 进行故障转移。
- otellix - 面向成本受限生产环境的、原生 OpenTelemetry 的 LLM 可观测性与预算护栏。
- routex - 面向 Go 的 YAML 驱动的多 agent AI 运行时，具备 Erlang 风格的监督、MCP 工具服务器支持以及一个 CLI。
- trpc-agent-go - 用于构建基于 LLM 的多 agent 系统的框架。
- web-researcher-mcp - 一个 MCP 服务器，为 AI 助手提供网络搜索、内容提取以及多源研究能力。单个二进制文件，5 个搜索提供商带断路器故障转移，4 层抓取流水线。
- zenflow - 多 agent 编排与工作流引擎。声明式 YAML 工作流，带有中心辐射式邮箱的 LLM 协调器，竞态安全的投递。一个 YAML 文件，一个 Go 二进制文件。可在任何 goai 支持的提供商上运行。
音频与音乐
用于处理音频和音乐的库。
- beep - 一个用于播放和音频处理的简单库。
- flac - 原生 Go FLAC 编码器/解码器，支持 FLAC 流。
- gaad - 原生 Go AAC 比特流解析器。
- go-mpris - mpris dbus 接口的客户端。
- GoAudio - 原生 Go 音频处理库。
- gosamplerate - 面向 go 的 libsamplerate 绑定。
- id3v2 - 面向 Go 的 ID3 解码和编码库。
- malgo - 迷你音频库。
- minimp3 - 轻量级 MP3 解码器库。
- music-theory - Go 中的音乐理论模型。
- Oto - 一个用于在多个平台上播放声音的低层级库。
- PortAudio - PortAudio 音频 I/O 库的 Go 绑定。
- voxrai-ai - 采用 JSON 配置的 AI 语音 agent，通过 WebSocket 和 WebRTC 的 STT → LLM → TTS 流水线。
认证与授权
用于实现认证和授权的库。
- authboss - 面向 Web 的模块化认证系统。它尽可能地去除样板代码和"难做的事情"，这样每次你在 Go 中开始一个新的 Web 项目时，都可以插入它、配置它，并开始构建你的应用，而无需每次都构建一个认证系统。
- authgate - 一个轻量级的 OAuth 2.0 授权服务器，支持设备授权许可（RFC 8628）、带 PKCE 的授权码流程（RFC 6749 + RFC 7636），以及用于机器对机器认证的客户端凭证许可。
- branca - 面向 Golang 1.15+ 的 branca 令牌规范实现。
- casbin - 支持 ACL、RBAC、ABAC 等访问控制模型的授权库。
- cookiestxt - 提供 cookies.txt 文件格式的解析器。
- go-githubauth - 用于 GitHub 认证的实用工具：生成并使用 GitHub 应用和安装令牌。
- go-guardian - Go-Guardian 是一个 golang 库，提供一种简单、清晰且符合习惯的方式来创建强大的现代 API 和 Web 认证，支持 LDAP、Basic、Bearer 令牌以及基于证书的认证。
- go-iam - 开发者优先的身份与访问管理系统，带有简单的 UI。
- go-jose - 对 JOSE 工作组的 JSON Web Token、JSON Web Signatures 以及 JSON Web Encryption 规范相当完整的实现。
- go-jwt - 一个面向 Go 的 JWT（JSON Web Token）库。
- go-jwt - JWT 认证包，提供带指纹识别的访问令牌和刷新令牌、Redis 存储以及自动刷新能力。
- goiabada - 一个开源的认证与授权服务器，支持 OAuth2 和 OpenID Connect。
- gologin - 用于登录的可链式处理器
