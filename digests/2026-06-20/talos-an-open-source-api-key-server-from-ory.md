# Talos: An Open Source API Key Server from Ory

Chat · 讨论 · 新闻通讯 · 文档 · 试用 Ory Network · 招聘

Ory Talos 是一个可扩展且安全的 API 密钥服务器，专为低延迟验证、水平扩展和可预测的运维而优化。它遵循 API 密钥的既定安全最佳实践，可发行、验证、撤销和派生 API 密钥及短期令牌，适用于高吞吐量系统。

- 什么是 Ory Talos？
- 部署选项
- 快速入门
- 谁在使用 Ory Talos
- 生态系统
- 文档
- 开发 Ory Talos
- 安全
- 遥测

Ory Talos 是一个用于发行、验证和管理 API 密钥的服务器。它遵循云架构最佳实践，专注于：

- 大规模发行、验证和撤销 API 密钥
- 导入外部发行的 API 密钥以实现统一验证
- 从长期密钥派生短期 JWT 和 macaroon 令牌
- 以 Sidecar 方式部署实现快速 API 密钥验证
- 通过缓存和最终撤销机制实现低延迟验证
- 通过结构化日志、指标和链路追踪实现可预测的运维

我们建议从 Ory Talos 文档开始，了解更多关于其架构、功能集以及与其他系统比较的信息。

Ory Talos 的设计目标是：

- 以单一二进制文件运行，支持三种部署模式：管理端、自助服务端或一体化模式
- 通过数据库验证 API 密钥并利用缓存实现低延迟，而派生的 JWT 和 macaroon 令牌可以离线验证，无需数据库查询
- 将管理端和自助服务端分离，使密钥创建、撤销、派生和验证可以独立于持有证明自撤销进行扩展和安全保护
- 通过外部数据库（Postgres、MySQL、CockroachDB）和可选的分布式缓存实现水平扩展
- 适配 Kubernetes 和托管平台等现代云原生环境
- 离线铸造缩小范围的短期令牌，使代理、CI/CD 任务和服务无需在每次请求时调用服务器
- 保持凭据路由、哈希和验证的集中化和常量时间处理

您可以通过两种主要方式运行 Ory Talos：

- 作为 Ory Network 上的托管服务
- 作为您自行控制的自托管服务，可选择是否使用 Ory 企业许可证

Ory Network 是在生产环境中使用 Ory Talos 最快的方式。

Ory Network 提供：

- 具有低延迟全球边缘节点的 API 密钥发行、验证和派生
- 用于单点登录、API 访问和机器对机器授权的 OAuth2 和 OpenID Connect
- 可扩展至数十亿用户和设备的身份和凭据管理
- 支持通行密钥、生物识别、社交登录、SSO 和多因素认证的注册、登录和账户管理流程
- 预构建的登录、注册和账户管理页面及组件
- 基于 Zanzibar 模型和 Ory 权限语言的低延迟权限检查
- 符合 GDPR 要求的存储，兼顾数据本地化和合规性
- 用于管理和运维的基于 Web 的 Ory 控制台和 Ory CLI
- 与开源服务器兼容的云原生 API
- 公平的基于用量的定价

注册免费开发者账户即可开始使用。

您可以自行运行 Ory Talos，完全掌控基础设施、部署和定制化。

安装指南说明了如何：

- 在 Linux、macOS、Windows 和 Docker 上安装 Ory Talos
- 配置 SQLite、PostgreSQL、MySQL 和 CockroachDB 等数据库
- 部署到 Kubernetes 和其他编排系统

开源发行版以单实例方式运行，使用内嵌的 SQLite 数据库。它非常适合个人、研究人员、黑客以及希望在没有服务级别协议（SLA）的情况下进行实验、原型开发或运行低流量工作负载的公司。

如果您将 Ory Talos 作为业务关键系统的一部分运行，例如在热路径上进行 API 密钥验证，您应该使用商业协议来降低运维和安全风险。Ory 企业许可证（OEL）在自托管 Ory Talos 之上提供：

- 由外部数据库（Postgres、MySQL、CockroachDB）支持的多节点部署
- 多租户、分布式缓存、速率限制执行和边缘验证节点
- 定期安全更新，包括带 SLA 的 CVE 补丁
- 高级扩展和复杂部署支持
- 高级支持选项，包括响应 SLA、直接联系工程师和入职协助
- 访问私有 Docker 镜像仓库，提供频繁的、经过审核的企业构建版本

如需获得有保障的 CVE 修复、最新企业构建版本、高级功能和生产支持，您需要有效的 Ory 企业许可证并访问 Ory 企业 Docker 镜像仓库。如需了解更多，请联系 Ory 团队。

安装 Ory CLI 并使用托管的 Ory Network，或使用 Docker Compose 在本地运行 Ory Talos。

# 如果尚未安装 Ory CLI，请先安装：
bash <(curl https://raw.githubusercontent.com/ory/meta/master/install.sh) -b . ory
sudo mv ./ory /usr/local/bin/

# 登录或注册
ory auth

# 创建新项目
ory create project --create-workspace "Ory Open Source" --name "GitHub Quickstart" --use-project

在本地运行 Ory Talos：

# 开源版本（SQLite，单节点）
docker-compose -f docker-compose.oss.yaml up --build

API 将在 http://localhost:4420 上可用。

有关发行、验证和撤销密钥的端到端演练，请参阅快速入门指南和发行与验证。

Ory 社区建立在个人、公司和维护者的共同努力之上。Ory 团队感谢每一位参与者——从提交错误报告和功能请求，到贡献补丁和文档。Ory 社区拥有超过 50,000 名成员，并且仍在增长。Ory 技术栈每天在数千家公司中保护超过 70 亿次 API 请求。如果没有你们每一个人，这一切都不可能实现！

如果您希望在 Ory Talos 上线 Network 后在此展示，请联系 office@ory.com。

衷心感谢所有个人贡献者

在架构设计方面，我们基于以下指导原则构建 Ory：

- 最小化依赖
- 随处运行
- 轻松扩展
- 最小化人为和网络错误的空间

Ory 的架构设计为在 Kubernetes、CloudFoundry、OpenShift 等容器编排系统上以最佳方式运行。二进制文件体积小巧，适用于所有主流处理器类型（ARM、AMD64、i386）和操作系统（FreeBSD、Linux、macOS、Windows），且无需系统依赖（Java、Node、Ruby、libxml 等）。

Ory Kratos 是一个 API 优先的身份和用户管理系统，按照云架构最佳实践构建。它实现了几乎每个软件应用都需要处理的核心用例：自助登录和注册、多因素认证（MFA/2FA）、账户恢复和验证、个人资料和账户管理。

Ory Hydra 是一个经过 OpenID 认证™ 的 OAuth2 和 OpenID Connect 提供者，通过编写一个小型"桥接"应用程序即可轻松连接到任何现有的身份系统。它提供对用户界面和用户体验流程的完全控制。

Ory Oathkeeper 是一个 BeyondCorp/零信任身份与访问代理（IAP），可为您的 Web 服务配置认证、授权和请求变更规则：认证 JWT、访问令牌、API 密钥、mTLS；检查所包含的主体是否被允许执行该请求；将结果内容编码为自定义头信息（`X-User-ID`）、JSON Web 令牌等！

Ory Keto 是一个策略决策点。它使用一组访问控制策略（类似于 AWS IAM 策略），以确定某个主体（用户、应用程序、服务、汽车等）是否被授权对某个资源执行特定操作。

Ory Talos 文档位于 www.ory.com/docs/talos。

有关以下信息，请参阅 CONTRIBUTING.md：

- 贡献指南
- 前提条件和开发环境设置
- 运行开源版和商业版的测试
- 生成 protobuf、SQL 和 SDK 产物
- 构建 Docker 镜像

Ory Talos
