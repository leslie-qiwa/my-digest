# gopls's Model Context Protocol (MCP) Server

Gopls：模型上下文协议支持

Gopls 包含一个实验性的内置模型上下文协议（MCP）服务器，允许它以 MCP 工具的形式向 AI 助手公开其部分功能。

## 运行 MCP 服务器

该服务器有两种运行模式："附加模式"和"分离模式"。在附加模式下，MCP 服务器在活跃的 gopls LSP 会话上下文中运行，因此能够与你的 LSP 会话共享内存并观察当前未保存的缓冲区状态。在分离模式下，gopls 与一个无头 LSP 会话交互，因此只能看到磁盘上已保存的文件。

### 附加模式

要使用"附加"模式，请使用 `-mcp.listen` 标志运行 gopls。例如：

```
gopls serve -mcp.listen=localhost:8092
```

这会使用服务器发送事件传输（SSE）暴露一个基于 HTTP 的 MCP 服务器，可通过 `http://localhost:8092/sessions/1` 访问（假设你的 gopls 实例上只有一个会话）。

### 分离模式

要使用"分离"模式，请运行 `mcp` 子命令：

```
gopls mcp
```

这会运行一个独立的 gopls 实例，通过 stdin/stdout 进行 MCP 通信。

## 模型指令

该 gopls MCP 服务器包含使用说明的模型指令，描述了使用其可用工具与 Go 代码交互的工作流程。为避免强制客户端进入特定工作流程，这些指令不会在初始化期间自动发布。

你可以通过使用 `-instructions` 标志将其打印出来，作为 AI 辅助会话中的附加上下文加载：

```
gopls mcp -instructions > /path/to/contextFile.md
```

## 编码助手设置

要将 gopls MCP 服务器与基于 LLM 的编码助手配合使用，你需要配置助手的 MCP 客户端以与 `gopls` 通信。

以下是主流助手的配置指南。

首先确保已安装 gopls v0.20 或更高版本，并且可在你的 `$PATH` 中找到。使用以下命令安装最新版本：

```
$ go install golang.org/x/tools/gopls@latest
```

### Gemini CLI

对于 Gemini CLI，请将以下配置添加到 `$HOME/.gemini/extensions/go/gemini-extension.json`：

```json
{
  "name": "go",
  "version": "0.0.1",
  "mcpServers": {
    "go": {
      "command": "gopls",
      "args": ["mcp"]
    }
  },
  "contextFileName": "GEMINI.md"
}
```

然后运行以下命令输出模型指令：

```
$ gopls mcp -instructions > ~/.gemini/extensions/go/GEMINI.md
```

要验证连接，请在 Gemini 会话中使用 `/mcp list` 命令查看可用工具。或者，从 shell 中运行 `gemini mcp list` 来检查活跃的 MCP 服务器和连接状态。

### Claude Code

对于 Claude Code，运行以下命令将 `gopls` 添加为 MCP 服务器：

```
$ claude mcp add gopls -- gopls mcp
```

要验证连接并浏览可用工具：

- 在 Claude 会话中输入 `/mcp` 查看提供的工具列表。
- 运行 `claude mcp list` 检查当前活跃的 MCP 服务器。
- 运行 `claude mcp get gopls` 检查服务器的配置详情。

## 安全注意事项

gopls MCP 服务器是对 gopls 通常通过语言服务器协议（LSP）公开的功能的封装。因此，gopls 的工具可能执行 gopls 正常执行的任何操作，包括：

- 从文件系统读取文件，并在工具结果中返回其内容（例如提供上下文时）；
- 执行 `go` 命令以加载包信息，这可能导致对 https://proxy.golang.org 的调用以下载 Go 模块，以及对 go 缓存的写入；
- 写入 gopls 的缓存或持久化配置文件；以及
- 如果你已选择加入 Go 遥测，则上传每周遥测数据。

gopls MCP 服务器不会执行 gopls 在普通 IDE 会话中未曾执行的任何操作。与大多数 LSP 服务器一样，gopls 通常不会直接写入你的源代码树，但它可能会指示客户端应用编辑。它也不会通过网络发起任意请求，但可能会向某些服务（如 Go 模块镜像或 Go 漏洞数据库）发起范围有限的请求，这些请求不易被混淆代理利用为数据泄露的途径。然而，在作为 AI 系统的一部分使用时，这些功能可能需要额外的考量。

本文档的源文件位于 golang.org/x/tools/gopls/doc 下。
