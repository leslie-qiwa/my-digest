# PgQue: A Pure SQL + PL/pgSQL Zero-Bloat Postgres Queue

零膨胀 Postgres 队列。安装只需一个 SQL 文件，用 pg_cron 或 pg_timetable 驱动定时。

在 Hacker News 上的讨论。

适用于希望在 Postgres 内部拥有持久事件流的团队。其模型更接近 Kafka（日志），而非 ActiveMQ 或 RabbitMQ（任务消息队列）。共享事件日志，每个消费者独立游标，持续负载下零膨胀。纯 SQL 和 PL/pgSQL，适用于任何 Postgres 14+ —— 托管或自托管，无需旁路守护进程。本 README 的其余部分将介绍支持上述主张的历史、对比和安装路径。

- 为什么选择 PgQue
- 延迟权衡
- 三种延迟
- 对比
- 安装
- 角色与权限
- 项目状态
- 文档
- 快速入门
- 客户端库
- 基准测试
- 子消费者 / 协作式消费者
- 架构
- 路线图
- 贡献
- 许可证

PgQue 复兴了 PgQ —— 生产环境中运行时间最长的 Postgres 队列架构之一 —— 以一种可在任何 Postgres 平台（包括托管服务商）上运行的形式呈现。

PgQ 于 2006 年在 Skype 设计，用于为数亿用户运行消息系统，并在大型自管理 Postgres 部署上运行了十余年。标准 PgQ 依赖于一个 C 扩展（pgq）和一个外部守护进程（pgqd），这两者在大多数托管 Postgres 服务商上均无法运行。

PgQue 用纯 PL/pgSQL 重建了这个经过实战检验的引擎，使零膨胀队列模式可以在任何能运行 SQL 的地方工作 —— 无需向您的技术栈添加另一个分布式系统。

它是同一个引擎 ── PgQ ── 为托管 Postgres 重新打包，并提供了 TypeScript、Python 和 Go 的客户端库。

反扩展理念。纯 SQL + PL/pgSQL，适用于任何 Postgres 14+ —— 包括 RDS、Aurora、Cloud SQL、AlloyDB、Supabase、Neon 以及大多数其他托管服务商。无需 C 扩展，无需 shared_preload_libraries，无需服务商审批，无需重启。

历史背景，两份演示文稿：

外部报道：

- PgQue: Two Snapshots and a Diff，作者 Christophe Pettus —— 对快照/差分机制的详细讲解：两个连续的 tick 快照如何决定事件可见性，以及为什么这种方式避免了行级锁和死元组膨胀。
- HN 讨论

大多数 Postgres 队列依赖 SKIP LOCKED 加上 DELETE 和/或 UPDATE。这在简单示例中表现良好，但在持续负载下会演变为死元组、VACUUM 压力、索引膨胀和性能衰退。

PgQue 完全避免了这类问题。它使用基于快照的批处理和基于 TRUNCATE 的表轮换，而非逐行删除。热路径保持可预测：

- 设计上零膨胀 —— 主队列路径无死元组
- 无性能衰退 —— 不会因为运行了数月而变慢
- 为高负载系统构建 —— 原始 PgQ 架构所设计的持续负载场景
- 真正的 Postgres 保证 —— ACID 事务、事务性入队/消费、WAL、备份、复制、SQL 可见性
- 适用于托管 Postgres —— 无需自定义构建、无需 C 扩展、无需独立守护进程

PgQue 在 Postgres 内部为您提供队列语义，具备 Postgres 的持久性和事务行为，且没有大多数数据库内队列最终会遇到的膨胀代价。

PgQue 围绕基于快照的批处理构建，而非逐行声明。这正是它在热路径上实现零膨胀、持续负载下行为稳定以及在 Postgres 内部实现干净 ACID 语义的原因。

权衡在于端到端投递延迟 —— 即 send 和消费者能够 receive 事件之间的时间间隔。PgQue 默认每秒 tick 10 次（每 100 毫秒），因此等待下一次 tick 的时间平均约为 tick 周期的一半。一个已提交的基准测试（benchmark/tick-rate/）测量出在默认 100 毫秒 tick 下，端到端投递中位数约为 52 毫秒，最大值大约为一个 tick 周期（在已提交的多次运行中约为 105–145 毫秒），加上消费者的轮询间隔。send / receive / ack 调用本身单独执行都很快。详见 docs/latency-and-tuning.md 了解完整分析。

降低投递延迟的方法：调整 tick 周期（例如 pgque.set_tick_period_ms(50) 实现每秒 20 次 tick；可接受的周期为 1000 毫秒的精确因子）和队列阈值；使用 force_next_tick() 用于测试和演示或强制立即生成批次。未来版本可能会添加基于逻辑解码的唤醒机制，在不增加 WAL 写入的情况下实现亚毫秒级投递。

如果您的首要目标是个位数毫秒的调度延迟，PgQue 不是合适的工具。如果您的首要目标是在负载下保持稳定且无膨胀，那正是 PgQue 的用武之地。

"队列延迟"是三个数字，而非一个：

- 生产者延迟 —— send / insert_event。单次执行很快。
- 订阅者延迟 —— 在预构建的批次上执行 next_batch。单次执行很快。
- 端到端投递延迟 —— send → 消费者可见。平均约为 tick 周期的一半（默认 100 毫秒 tick），在已提交的 benchmark/tick-rate/ 基准测试中测量出中位数约为 52 毫秒。可通过 pgque.set_tick_period_ms(ms) 从 1 毫秒调整至 1000 毫秒。不随负载增长。

详见 docs/latency-and-tuning.md 了解分析、tick 节奏权衡表以及与基于 UPDATE/DELETE 设计的对比。

| 特性 | PgQue | PgQ | PGMQ | River | Que | pg-boss |
|---|---|---|---|---|---|---|
| 基于快照的批处理（无行锁） | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| 持续负载下零膨胀 | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| 无需外部守护进程或工作进程二进制文件 | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ |
| 纯 SQL 安装，托管 Postgres 就绪 | ✅ | ❌ | ✅ | ✅ | ✅ | ✅ |
| 语言无关的 SQL API | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| 共享日志扇出：每个消费者看到每个事件 | ✅ | ✅ | ❌ | ❌ | ❌ | |
| 内置带退避的重试机制 | ✅ | ✅ | ✅ | ✅ | ✅ | |
| 内置死信队列 | ✅ | ❌ | ❌ | ✅ |

图例：✅ 是 · ❌ 否 ·

注：

- PgQ 是 PgQue 衍生自的 Skype 时代队列引擎（约 2007 年）。相同的快照/轮换架构，但需要 C 扩展和外部守护进程（pgqd）—— 在托管 Postgres 上不可用。PgQue 移除了这两个限制。
- 无需外部守护进程：PgQue 使用 pg_cron（或您自己的调度器）进行 tick；PGMQ 使用可见性超时。River、Que 和 pg-boss 需要 Go / Ruby / Node.js 工作进程二进制文件。
- Que 使用咨询锁（非 SKIP LOCKED）—— 声明时无死元组，但已完成的任务仍会被 DELETE。Brandur 关于膨胀的文章讲的就是 Heroku 上的 Que。仅支持 Ruby。
- PGMQ 的重试是基于可见性超时的重新投递（read_ct 追踪）—— 无可配置的退避或最大尝试次数。
- PGMQ 消费者：PGMQ 支持多个生产者和多个竞争消费者/工作进程。扇出行"❌"表示它不提供 PgQ 风格的独立消费者游标，即每个注册消费者从共享日志接收每个事件。
- pg-boss 的扇出是按队列复制的 publish()/subscribe()，而非具有独立游标的共享事件日志。
- 类别：River、Que 和 pg-boss（以及 Oban、graphile-worker、solid_queue、good_job）是任务队列框架。PgQue 是为高吞吐量流式处理和扇出优化的事件/消息队列。

1. 设计上零事件表膨胀。SKIP LOCKED 队列（PGMQ、River、pg-boss、Oban、graphile-worker）对行进行 UPDATE + DELETE，产生需要 VACUUM 的死元组。在持续负载下，这会导致有据可查的故障：

- Brandur/Heroku（2015）—— 一小时内积压 6 万条。
- PlanetScale（2026）—— 在同时运行 OLAP 的情况下，800 任务/秒时进入死亡螺旋。
- River issue #59 —— 自动清理饥饿。

Oban Pro 推出了表分区来缓解此问题；PGMQ 提供了激进的自动清理设置。PgQue 的 TRUNCATE 轮换从结构上产生零死元组。无需调优。对 xmin 视界固定免疫。

2. 原生扇出。每个注册的消费者在共享事件日志上维护自己的游标，独立接收所有事件。这与竞争消费者（SKIP LOCKED）不同，后者每个任务只发送给一个工作进程。pg-boss 有扇出功能，但它是按队列复制的（每个订阅者每个事件一次 INSERT）。PgQue 的模型是共享日志上的位置 —— 无数据重复，原子批次边界，后来的订阅者可以追赶。更接近 Kafka 主题而非任务队列。

- 当您需要事件驱动的扇出时，选择 PgQue
