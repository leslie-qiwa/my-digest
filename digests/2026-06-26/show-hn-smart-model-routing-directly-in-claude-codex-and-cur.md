# Show HN: Smart model routing directly in Claude, Codex and Cursor

一个端点。每个模型。始终选对那一个。

一个面向 Anthropic、OpenAI 和 Gemini 的即插即用代理，它为每个请求挑选最合适的模型：使用本机上的小型嵌入器，而非凭感觉的提示词。

由 Weave 打造：排名第一的工程智能平台，深受 Robinhood、PostHog、Reducto 以及数百家其他公司的喜爱。

将 Claude Code、Codex、Cursor 或你自己的应用指向 `localhost:8080`。该路由器：

- 🎯 按请求路由。一个源自 Avengers-Pro 1 的聚类评分器，每一轮都从你已启用的提供商中挑选合适的模型。
- 🔌 兼容所有人的 API。Anthropic Messages、OpenAI Chat Completions、Gemini 原生格式。流式传输、工具、视觉，一应俱全。
- 🧠 也懂开源模型。通过 OpenRouter（或任何兼容 OpenAI 的端点）支持 DeepSeek、Kimi、GLM、Qwen、Llama、Mistral。
- 🔒 默认 BYOK（自带密钥）。提供商密钥留在你的机器上，静态加密保存。
- 📊 可观测。开箱即用的 OTLP 追踪。在 Weave 仪表盘（http://localhost:8080/ui/dashboard）中查看，或接入 Honeycomb、Datadog、Grafana 等任意工具。

最快的方式：用一条命令把 Claude Code、Codex 或 opencode 指向托管的 Weave Router。无需克隆、无需 Docker、无需 Postgres。

```
npx @workweave/router
```

就这么简单。安装程序会询问使用哪个工具（Claude Code、Codex 或 opencode），引导你完成作用域选择（用户级 vs. 项目级），获取一个路由器密钥，并写入正确的配置文件。其他用法：

```
npx @workweave/router --claude # 跳过选择器，直接 Claude Code
npx @workweave/router --codex # 跳过选择器，直接 OpenAI Codex CLI
npx @workweave/router --opencode # 跳过选择器，直接 opencode
npx @workweave/router --scope project # 按仓库，提交 settings.json（或 .codex/ / opencode.json）
npx @workweave/router --local # 自托管 localhost:8080
npx @workweave/router --base-url https://router.acme.internal
npx @workweave/router@0.1.0 # 锁定某个版本
```

需要 Node ≥ 18（Claude Code 和 opencode 路径还需要 `jq`）。完整的标志参考：install/npm/README.md。

如果你想在自己的机器上运行路由器（及仪表盘）：

```
# 1. 放入一个提供商密钥。推荐以 OpenRouter 作为基线。
echo "OPENROUTER_API_KEY=sk-or-v1-..." >> .env.local

# 2. 在 :8080 上启动 Postgres + 路由器，并初始化一个 rk_ 密钥。
make full-setup
```

路由器运行在 http://localhost:8080，仪表盘在 http://localhost:8080/ui/（密码：`admin`），你的 `rk_...` 密钥会打印在日志中。

```
# 像调用 Anthropic 那样调用
curl -sS http://localhost:8080/v1/messages \
-H "Authorization: Bearer rk_..." \
-d '{"model":"claude-sonnet-4-5","max_tokens":256,
"messages":[{"role":"user","content":"hi"}]}'

# ……或者像调用 OpenAI 那样
curl -sS http://localhost:8080/v1/chat/completions \
-H "Authorization: Bearer rk_..." \
-d '{"model":"gpt-4o-mini",
"messages":[{"role":"user","content":"hi"}]}'

# 在不实际代理的情况下查看路由决策
curl -sS http://localhost:8080/v1/route -H "Authorization: Bearer rk_..." -d '...'
```

Claude Code。运行 `make install-cc` 将 Claude Code 接入本地自托管路由器（在 `make full-setup` 结束时也会自动调用）。对于托管路由器，请使用上面的 `npx @workweave/router`。

Codex（OpenAI CLI）。`npx @workweave/router --codex` 会修补 `~/.codex/config.toml`（使用 `--scope project` 时则为 `<repo>/.codex/config.toml`），加入一个受管理的 `[model_providers.weave]` 块，并设置 `model_provider = "weave"`。Codex 现有的 `OPENAI_API_KEY` 会透传到 api.openai.com，用于基于套餐的直通；路由器密钥通过 `X-Weave-Router-Key` HTTP 头携带。重新安装以及 `--uninstall --codex` 只会重写/移除该受管理块，其余 Codex 配置保持不变。

opencode。`npx @workweave/router --opencode` 会将一个 `provider.weave` 条目合并进 `~/.config/opencode/opencode.json`（使用 `--scope project` 时则为 `<repo>/opencode.json`）。它使用 opencode 自带的 `@ai-sdk/anthropic` 提供商，指向路由器的 `/v1` 端点——路由器原生支持 Anthropic Messages API，因此 opencode 无需修改即可工作。路由器密钥和身份标识头会与提供商配置一同携带；重新安装只会重写受管理块，`--uninstall --opencode` 则会将其剥离。

Cursor（早期测试版，性能可能不是最佳）。设置 → 模型 → 覆盖 OpenAI Base URL → `http://localhost:8080/v1`，将 `rk_...` 粘贴为 API 密钥。

开关切换。安装后，`npx @workweave/router off --claude`（或 `--codex` / `--opencode`）会让该客户端直接路由回其提供商，而不丢弃路由器配置；`on` 将其翻转回来，`status` 报告当前指向哪一方。Claude Code 还提供 `/router-off`、`/router-on` 和 `/router-status` 斜杠命令。Cursor 通过上面相同的 设置 → 模型 覆盖来切换。参见 install/README.md。

两个密钥，别搞混了：

- `sk-or-...` / `sk-ant-...` / `sk-...` = 你的上游提供商密钥。存放在 `.env.local` 中。
- `rk_...` = 你的路由器密钥。客户端将其作为 Bearer 令牌发送。

| 端点 | 格式 |
|---|---|
| `POST /v1/messages` | Anthropic Messages，经路由 |
| `POST /v1/chat/completions` | OpenAI Chat Completions，经路由 |
| `POST /v1beta/models/:action` | Gemini `generateContent`，经路由 |
| `POST /v1/route` | 返回决策，不调用上游 |
| `GET /v1/models` · `POST /v1/messages/count_tokens` | Anthropic 直通 |
| `GET /health` · `GET /validate` | 存活检查 + 密钥校验 |

- 📐 配置参考：每个环境变量、BYOK 加密、OTel 旋钮、聚类路由。
- 🛠️ 贡献指南：分层规则、热重载开发、迁移、测试，整个工程循环。
- 🏗️ 架构：包布局、导入契约，以及添加端点／提供商／策略的实操方法。

脚注

1. Zhang, Y. 等。《超越 GPT-5：通过性能—效率优化路由让 LLM 更便宜也更优秀》（Avengers-Pro）。arXiv:2508.12631，2025。https://arxiv.org/abs/2508.12631 ↩
