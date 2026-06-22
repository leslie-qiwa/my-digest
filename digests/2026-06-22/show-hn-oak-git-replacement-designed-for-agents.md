# Show HN: Oak – Git replacement designed for agents

# Oak

本仓库是 Oak 的开源核心：**以代理速度运行的版本控制**。它以 Cargo 工作区形式开发：一个可复用的 VCS 库加上供代理驱动的 `oak` 命令行客户端。

自带你的代理（Claude Code、Codex、Cursor……）；Oak 是代理进行读取、写入、分支和协作的基础。其底层设计围绕代理的实际工作方式——以"每会话一分支"作为工作单元，用分支描述替代逐次提交消息，以及内容寻址的惰性挂载让代理在几秒内即可编辑任何仓库。由于采用内容寻址并按需加载，它在代理工作负载上也比 git 快得多——但速度是设计的结果，而非卖点。

| Crate | 路径 | crates.io | 简介 |
|---|---|---|---|
| oakvcs-core | core/ | oakvcs-core | VCS 基础：BLAKE3 内容哈希、内容定义分块、差异/合并、Blob/Manifest/Commit/Tree 数据模型，以及可选的客户端本地仓库（SQLite + git 后端）。 |
| oakvcs-cli | cli/ | oakvcs-cli | 基于 oakvcs-core 构建的 `oak` 二进制程序。 |

## 在你的项目中使用该库

`oakvcs-core` 可独立使用——例如将 Oak 集成到其他工具或引擎中。关闭默认功能即可仅引入内容寻址数据模型和哈希（不含 SQLite/git）：

```toml
[dependencies]
oakvcs-core = { version = "0.99.0", default-features = false }
```

该 crate 发布名为 `oakvcs-core`，但导入时使用 `oak_core`。

如需使用磁盘上的 `Repository`（SQLite + 只读 git）后端，请添加默认的 `local-repo` 功能。

## 安装 CLI

Oak 目前处于公开测试阶段（v0.99.0）。最快的方式是使用预编译的 `oak` 二进制文件：

```bash
curl -fsSL oak.space/install | sh
```

安装器支持 macOS（Apple Silicon）和 Linux（x86_64）。安装后，`oak upgrade` 可原地更新二进制文件。

### Windows (x86_64)

`curl … | sh` 安装器仅适用于 Unix。在 Windows 上，请从最新的 GitHub 发布页面下载预编译的 `oak-windows-x86_64.exe`（重命名为 `oak.exe` 并放入 `PATH`），或通过 crates.io 构建：`cargo install oakvcs-cli`。之后 `oak upgrade` 可原地更新。

Windows 上的 `oak mount` 使用投影文件系统（ProjFS），这是一个可选的 Windows 功能。在提升权限的 PowerShell 中启用一次即可：

```powershell
Enable-WindowsOptionalFeature -Online -FeatureName Client-ProjFS -NoRestart
```

（或通过 设置 → 应用 → 可选功能 → "Windows 投影文件系统"）。其他功能——clone、push、pull、commit——无需此设置即可工作。

更倾向于从 crates.io 构建？改用 Cargo 安装（支持 macOS、Linux 和 Windows——TLS 层使用 rustls + `ring`，因此无需 C/NASM 构建工具链）：

```bash
cargo install oakvcs-cli # 构建并安装 `oak` 二进制文件
```

## 从源码构建

```bash
cargo build --workspace  # 构建 oak-core + oak 二进制文件
cargo test -p oakvcs-cli # CLI 测试（含 wiremock HTTP 测试）
make build               # 发布构建 + CLI 发布工具
make release-proof       # 非变更式发布/上线就绪证明
```

CLI 通过工作区内路径依赖 `oak-core`，因此直接 `cargo build` 即可基于本地 `core/` 检出构建，无需额外设置。

参见 `docs/release-readiness.md` 了解发布证明和 crates.io 发布顺序检查。

## 许可证

Apache-2.0。详见 LICENSE。

## AI

本仓库几乎完全由 AI 在人工监督下编写。如果你发现任何需要修复的问题或希望参与贡献，请发送邮件至 [email protected] 或通过 Discord 联系。
