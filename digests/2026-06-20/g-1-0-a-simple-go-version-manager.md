# g 1.0: A Simple Go Version Manager

简单的 Go 版本管理器，轻量无负担。

现有的版本管理器通常会从源代码编译 Go，要求特定的 shell，用 shim 和函数污染你的环境，或者带来你可能不需要的庞大插件生态系统。

`g` 的目标是尽可能地低侵入和高可移植性：一个可读的 shell 脚本，官方预编译的 Go 归档包，无需特权安装，只需设置一对惯用的 Go 环境变量即可开始使用。

`g` 的灵感来自 tj/n —— 我过去曾为其贡献过代码 —— 并借鉴了其部分代码。

- 使用安装脚本，一行命令即可开始使用 Go。
- 使用官方预编译的 Go 归档包快速安装新版本。
- 按需运行任何 Go 版本，无需更改当前激活的版本。
- 彩色交互式 TUI 界面，同时也可安全地用于管道和自动化脚本。
- 可信赖：以非特权用户身份安装和使用 Go 版本；无需 `sudo`。
- 发现新的 Go 版本，包括不稳定的 beta 和 RC 版本。
- 通过选择离你更近的镜像来加速下载。
- 一个单独的、可移植的 POSIX 脚本，无需 shim 即可在所有常见 shell 中工作。
- 透明：`g` 是一个你可以查看、复制和修改的 shell 脚本。

- macOS、Linux、BSD 或 WSL。
- 一个兼容 POSIX 的 shell。
- `curl` 或 `wget`。
- `sha256sum`、`shasum` 或 `openssl`。

你很可能已经全部具备了。

在运行任何脚本之前，请先阅读它：https://github.com/stefanmaric/g/releases/latest/download/install

然后使用 `curl` 安装：

```
curl -fsSL https://github.com/stefanmaric/g/releases/latest/download/install | sh
```

或使用 `wget`：

```
wget -qO- https://github.com/stefanmaric/g/releases/latest/download/install | sh
```

`g` 使用标准的 Go 环境变量。安装程序默认设置为：

`GOROOT=$HOME/.go`

`GOPATH=$HOME/go`

`$GOPATH/bin` 必须在 PATH 中，因为 `g` 和激活的 `go` 二进制文件都存放在那里。

安装程序将 `g` 下载到 `$GOPATH/bin/g`，使其可执行，并在你的 shell 启动文件中更新 `GOROOT`、`GOPATH` 以及将 `$GOPATH/bin` 添加到 `PATH`。

安装后请重启你的 shell，以便加载新的环境变量。

跳过所有提示并默认选择"是"，将立即安装最新的 Go 版本：

```
curl -fsSL https://github.com/stefanmaric/g/releases/latest/download/install | sh -s -- -y
```

配置特定的 shell：

```
curl -fsSL https://github.com/stefanmaric/g/releases/latest/download/install | sh -s -- zsh
```

配置多个 shell：

```
curl -fsSL https://github.com/stefanmaric/g/releases/latest/download/install | sh -s -- fish bash zsh
```

安装程序支持 `bash`、`zsh`、`fish`、`ash`、`dash`、`csh` 和 `tcsh`。

如果 `g` 已经被用作 shell 别名，安装程序可以配置一个替代别名。

要选择不同的默认值，请在安装前设置 `GOROOT` 和 `GOPATH`：

```
export GOROOT=$HOME/.local/share/golang
export GOPATH=$HOME/go-projects
curl -fsSL https://github.com/stefanmaric/g/releases/latest/download/install | sh
```

对于 `fish`：

```
set -gx GOROOT $HOME/.local/share/golang
set -gx GOPATH $HOME/go-projects
curl -fsSL https://github.com/stefanmaric/g/releases/latest/download/install | sh
```

- 在你的 shell 初始化脚本中配置相关变量
  - 设置 `GOROOT` 和 `GOPATH`。
  - 将 `$GOPATH/bin` 添加到 `PATH`。
- 将 `bin/g` 复制到 `$GOPATH/bin/g` 或 `PATH` 中的其他目录。
- 使其可执行：`chmod +x "$GOPATH/bin/g"`

安装最新的稳定版 Go 并将其设为激活版本：

```
g install latest
```

安装并使用特定版本：

```
g install 1.22.2
```

列出已安装的版本：

```
g list
```

列出所有可用的 Go 版本：

```
g list-all
```

运行特定版本而不更改当前激活的版本：

```
g run 1.22.2 version
```

移除旧版本：

```
g remove 1.21.9
```

不带命令运行 `g` 以打开已安装版本的交互式选择器。

```
用法：g [命令] [选项] [参数]

命令：
  g                         打开已下载版本的交互式界面
  g install latest          下载并设置最新的 Go 版本
  g install <version>       下载并设置 Go <version>
  g download <version>      下载 Go <version>
  g set <version>           切换到 Go <version>
  g run <version>           运行指定版本的 Go
  g which <version>         输出 <version> 的二进制路径
  g remove <version ...>    移除指定的版本
  g prune                   移除除当前版本外的所有版本
  g list                    输出已下载的 Go 版本
  g list-all                输出所有可用的远程 Go 版本
  g self-upgrade            将 g 升级到最新版本
  g help                    显示帮助信息，同 g --help

选项：
  -h, --help                显示帮助信息并退出
  -v, --version             输出 g 的当前版本并退出
  -q, --quiet               抑制几乎所有输出
  -c, --no-color            强制禁用彩色输出
  -y, --non-interactive     禁止交互提示
  -o, --os                  覆盖操作系统
  -a, --arch                覆盖系统架构
  -u, --unstable            在列表中包含不稳定版本
      --archive-url         覆盖 Go 归档基础 URL

别名：
  g install                 use
  g download                fetch
  g run                     exec
  g remove                  rm, uninstall
  g list                    ls
  g list-all                ls-remote, list-remote
  g self-upgrade            self-update
```

默认情况下，`g` 会从当前机器检测你的操作系统和架构。使用 `--os` 和 `--arch` 可以覆盖目标平台：

```
g install 1.22.2 --os linux --arch amd64
```

当你需要不同的 Go 构建版本时，这很有用，例如在 Apple Silicon 上使用 `amd64` 版本的 Go。如果相同版本已经安装，你使用不同的显式架构重新安装时，`g` 会替换它。

默认情况下，归档包从 https://dl.google.com/go 下载。

使用 `--archive-url` 从镜像站下载归档包：

```
g install latest --archive-url https://golang.google.cn/dl
```

或使用 `G_GO_ARCHIVE_URL` 设置默认镜像：

```
export G_GO_ARCHIVE_URL=https://golang.google.cn/dl
g install latest
```

仅提供归档文件的镜像站也可以使用，只要它们提供官方 Go 归档文件名：

```
export G_GO_ARCHIVE_URL=https://mirrors.aliyun.com/golang
g install 1.22.2
```

镜像下载仍然会在解压之前与官方 Go 校验和进行验证。

如果 `g` 是通过安装程序安装的，使用以下命令升级：

```
g self-upgrade
```

要仅移除 `g`，删除可执行文件即可：

```
# 如果你使用的是 bash、zsh 或其他 POSIX shell：
rm "$(command -v g)"

# 如果你使用的是 fish：
rm (command -v g)
```

要移除 `g` 安装的 Go 版本，在备份重要内容后删除 `$GOROOT`：

```
rm -r "$GOROOT"
```

如果你使用了安装程序，还需要从你的 shell 启动文件中删除 `g-install` 那一行。常见位置包括：

```
~/.bash_profile          macOS 上的 bash
~/.bashrc                Linux/BSD 上的 bash
~/.zshrc                 zsh
~/.config/fish/config.fish  fish
~/.cshrc                 csh
~/.tcshrc                tcsh
$ENV                     ash 或 dash
```

`g` 有意保持小巧：一个 POSIX shell 脚本，官方预编译归档包，校验和验证，以及最少的 shell 配置。如果你需要项目级别的版本文件、插件生态系统、Windows 优先支持或源代码编译，其他工具可能更适合你。

| 项目 | 适用场景 | 与 g 相比的取舍 |
|---|---|---|
| moovweb/gvm | 你需要一个功能齐全的 Go 环境管理器。 | 从源代码编译 Go，组件更多。 |
| syndbg/goenv | 你喜欢 `rbenv`/`pyenv` 风格。 | 使用 shim，shell 集成模型更重。 |
| hit9/oo | 你需要一个接口简洁的源代码编译 Go 管理器。 | 从源代码编译而非使用官方归档包。 |
| asdf-golang | 你已经在使用 `asdf`。 | 需要 asdf 插件系统。 |
| andrewkroh/gvm | 你需要 Bash、Batch 和 PowerShell 工作流。 | shell 通用性较差，使用不同的工作流。 |
| MakeNowJust/govm | 你需要一个极简管理器且不介意源代码编译。 | 从源代码编译，有额外的运行时要求。 |
| kevincobain2000/gobrew | 你需要一个用 Go 编写的管理器，支持 Windows 并提供额外便利功能。 | 工具更大，有自己的目录结构和 shell 配置。 |
| golang.org/dl | 你需要官方的特定 Go 版本包装命令。 | 每个版本安装一个包装器，不管理版本切换。 |

如果你想要一个贴近 Go 本身分发方式的工具，请使用 `g`：下载官方归档包，验证它们，然后轻松切换版本。

请阅读 CONTRIBUTING.md。♥

- 本项目的每一位贡献者。
- `n` 项目，`g` 的灵感来源和基础。
- `n-install` 项目，
