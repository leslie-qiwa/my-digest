# Codex logging bug may write TBs to local SSDs

Codex SQLite 反馈日志每年可写入约 640 TB 数据，快速消耗 SSD 写入寿命

## 问题

Codex 持续向本地 SQLite 反馈日志数据库写入大量数据：

- `~/.codex/logs_2.sqlite`
- `~/.codex/logs_2.sqlite-wal`
- `~/.codex/logs_2.sqlite-shm`

在我的机器上，大约 21 天的运行时间后，主 SSD 已写入约 37 TB。进程/文件级别的检查显示 Codex SQLite 日志是主要的持续写入源。

推算下来大约每年 640 TB。对于 1 TB 的 SSD，这意味着每年约 640 次全盘写入。一些消费级 SSD 的额定写入量约为 600 TBW，因此这可能在不到一年内耗尽整个驱动器的保修写入寿命。

## 证据

`logs_2.sqlite` 中当前保留的行数：

| 指标 | 值 |
|------|------|
| 保留行数 | 681,774 |
| 估计保留日志内容 | 1,035.6 MiB |

级别分布：

| 级别 | 估计 MiB | 字节占比 |
|------|----------|----------|
| TRACE | 732.5 | 70.7% |
| INFO | 266.5 | 25.7% |
| DEBUG | 30.6 | 3.0% |
| WARN | 5.9 | 0.6% |

最大的 target+level 组合：

| target | 级别 | 估计 MiB |
|--------|------|----------|
| codex_api::endpoint::responses_websocket | TRACE | 527.4 |
| codex_otel.log_only | INFO | 141.2 |
| codex_otel.trace_safe | INFO | 121.2 |
| log | TRACE | 97.4 |
| codex_client::transport | TRACE | 60.1 |
| codex_core::stream_events_utils | DEBUG | 27.5 |
| codex_api::sse::responses | TRACE | 19.1 |

主要来源大多是全局 TRACE 日志、镜像遥测日志和原始 WebSocket/SSE 载荷日志。仅 `TRACE` 就占保留字节的约 70.7%。`codex_otel.log_only` + `codex_otel.trace_safe` 又增加了 25.3%。过滤这些类别应能在不完全禁用反馈日志的情况下移除该样本中约 96% 的保留日志字节。

### 来自最频繁 TRACE 源的脱敏示例：`target=log`

这些是高频保留样本。原始 WebSocket/SSE 载荷正文因可能包含私人对话内容而未列出。

```
128,764x TRACE log: inotify event: ... mask: OPEN, name: Some("ld.so.cache")
 37,982x TRACE log: inotify event: ... mask: OPEN, name: Some("locale.alias")
 23,843x TRACE log: inotify event: ... mask: OPEN, name: Some("passwd")
  3,639x TRACE log: <tokio-tungstenite checkout>/src/compat.rs:131 AllowStd.with_context
  3,505x TRACE log: <tokio-tungstenite checkout>/src/lib.rs:245 WebSocketStream.with_context
  3,362x TRACE log: <tokio-tungstenite checkout>/src/compat.rs:154 Read.read
  3,356x TRACE log: <tokio-tungstenite checkout>/src/compat.rs:157 Read.with_context read -> poll_read
  3,230x TRACE log: <tokio-tungstenite checkout>/src/lib.rs:294 Stream.poll_next
  3,227x TRACE log: <tokio-tungstenite checkout>/src/lib.rs:304 Stream.with_context poll_next -> read()
  3,213x TRACE log: inotify event: ... mask: OPEN, name: Some("nsswitch.conf")
  2,001x TRACE log: WouldBlock
  1,217x TRACE log: Masked: false
  1,169x TRACE log: Opcode: Data(Text)
  1,169x TRACE log: First: 11000001
```

### 来自频繁 INFO 源的脱敏示例

主要的 INFO 来源大多是重复的 OpenTelemetry 镜像事件。ID 已脱敏。

```
843x INFO codex_client::custom_ca:
  using system root certificates because no CA override environment variable was selected ...

334x INFO codex_otel.trace_safe:
  session_loop{thread_id=<已脱敏>}:submission_dispatch{...}

333x INFO codex_otel.log_only:
  session_loop{thread_id=<已脱敏>}:submission_dispatch{...}

332x INFO codex_otel.log_only:
  session_loop{thread_id=<已脱敏>}:submission_dispatch{...}

332x INFO codex_otel.trace_safe:
  session_loop{thread_id=<已脱敏>}:submission_dispatch{...}
```

## 写入放大

保留的数据库大小掩盖了真实的写入量。在 15 秒的采样中：

| 指标 | 之前 | 之后 |
|------|------|------|
| 保留行数 | 681,774 | 681,774 |
| 最大行 ID | 5,003,347,015 | 5,003,383,226 |

15 秒内插入了约 36,211 行，而保留行数保持不变。这表明存在持续的插入-清除写入放大：行被插入、索引、写入 WAL，然后被清除。

## 可能原因

SQLite 反馈日志接收器使用全局 TRACE 默认级别安装：

```rust
Targets::new().with_default(Level::TRACE)
```

这默认以 TRACE 级别持久化所有目标，包括依赖库/内部日志和大型原始协议载荷。

## 建议修复

保持反馈日志启用，但收紧默认持久化范围：

- 不要对 SQLite 反馈日志接收器使用全局 TRACE。
- 丢弃或提高低价值依赖噪声的阈值，特别是 `target=log`、`hyper_util`、tokio-tungstenite 内部日志、inotify 垃圾信息和低级 OpenTelemetry SDK 日志。
- 默认不持久化完整的原始 WebSocket/SSE 载荷。改为存储摘要信息：事件类型、持续时间、成功/错误、token 用量和载荷字节长度。
- 除非对反馈调试明确有用，否则避免持久化镜像的 `codex_otel.log_only` / `codex_otel.trace_safe` 事件。
- 添加全局日志数据库大小/写入上限。仅按线程限制在多线程/多进程场景下不够。

可选的逃生出口（如 `sqlite_logs_enabled = false`）仍然有用，但主要修复应该是改善默认过滤。
