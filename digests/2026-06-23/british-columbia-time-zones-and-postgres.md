# British Columbia, Time Zones, and Postgres

# 不列颠哥伦比亚、时区与 PostgreSQL

阅读时长约 6 分钟

2026 年 3 月 8 日，不列颠哥伦比亚省将时钟调整为全年使用太平洋夏令时。三月份，他们照常将时钟拨快一小时至 UTC-7，但十一月将不再回拨至 UTC-8。此后，America/Vancouver 时区的 UTC 偏移量将永久保持为 UTC-7。

让我们借此机会讨论日期和时区的存储方式。在最基本的场景中，默认做法是存储 UTC 值，然后根据 UTC 计算本地时间。然而，使用日历系统的用户是以本地时间（即挂钟时间）来思考的，从不考虑 UTC。在修改时区数据后，某个地区从 UTC 计算出的时间将与用户输入的值不同。

如果你在 2026 年及以后为不列颠哥伦比亚的预约以 UTC 列存储时间戳，那么十一月到三月的预约可能会偏差一小时！

要知道，`timestamptz` 列并不存储本地时间。它们存储的是 UTC 时间，时区仅在插入和查询时用于 UTC 的转换。如果你在 America/Vancouver 时区下将一个未来预约存储为 `timestamptz`，它会使用存储时的规则转换为 UTC。当你稍后查询该预约时，它会使用当前规则转换回本地时间。如果规则在存储和查询之间发生了变化，你得到的本地时间就不是用户最初想要的。

如果你尚未更新 `tzdata` 包，那么 Postgres 不知道这个变化，会继续使用旧规则进行转换。Ubuntu 中的 `tzdata` 包多久更新一次？令人惊讶的是，每隔几个月就会更新。

如果你的列存储为 `timestamptz` 类型，并且服务于不列颠哥伦比亚的客户，可以使用以下 SQL 查询来判断 `tzdata` 包是否已更新：

```sql
SELECT
  to_char(
    '2026-12-01 10:00:00'::timestamp AT TIME ZONE 'America/Vancouver',
    'HH24:MI:SS OF'
  ) AS november_2026_vancouver_offset;
```

如果值为 `17:00:00 +00`，说明 `tzdata` 已更新。但这并不完全令人放心，因为需要翻阅日志才能知道未来的预约是在时区调整之前还是之后创建的。

如果值为 `18:00:00 +00`，那是好消息！你的 `tzdata` 尚未更新，数据不会在更新前后出现分裂。

## 时区偏移示例

今年早些时候，一位用户预订了 2026 年 11 月 10 日温哥华上午 10 点的预约。你将其存储为 `timestamptz`：

```sql
INSERT INTO appointments (patient_id, starts_at)
VALUES (42, '2026-11-10T10:00:00-08:00');
-- 存储为: 2026-11-10 18:00:00+00 (UTC)
```

2026 年 4 月，`tzdata` 更新发布，推送了新的时区规则。

2026 年 11 月 10 日，患者按照日历上记录的本地时间上午 10 点到达。但当你查询预约时，显示其预约时间为本地时间上午 11 点：

```sql
SELECT starts_at AT TIME ZONE 'America/Vancouver' AS local_time
FROM appointments
WHERE patient_id = 42;
-- 返回: 2026-11-10 11:00:00
```

注意，计算结果比最初输入的时间晚了一小时。

## 能够应对时区变化的模式：双列模式

顾名思义，双列模式将数据存储在两列中（实际上是三列）：

- 本地时间戳
- 本地时区
- UTC 时间戳

UTC 时间戳列应该是一个计算列。使用时间戳和时区来计算 UTC。该计算出的 UTC 值也会被存储和查询，以便后台任务发送通知，并简化约束检查（如预约冲突）。

当本地意图具有权威性时，双列模式是必要的：人员或快递在特定时间和地点、法律截止日期、日历事件等。

但也不要过度使用。当事件已经发生，或者确切的 UTC 时刻具有权威性时（日志条目、金融交易、传感器读数），使用普通的 `timestamptz` 即可。双列模式增加了成本和复杂性，只有在需要保留未来本地意图时才值得付出这个代价。

详细模式如下：

```sql
CREATE TABLE appointments (
  id        bigint PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
  local_time     timestamp   NOT NULL, -- 挂钟值
  timezone_name  text        NOT NULL, -- IANA 名称: 'America/Vancouver'
  starts_at_utc  timestamptz NOT NULL  -- 通过触发器计算
  ...
);
```

`local_time` 和 `timezone_name` 一起回答"用户的意图是什么？"，存储的是挂钟日期/挂钟时间/挂钟位置的值。这些值只应在用户请求时更改。它们将用于计算 `starts_at_utc`。

`starts_at_utc` 可以是你建立索引、查询和用于约束的列。它回答的是"这个预约目前对应的 UTC 时刻是什么？"拥有一个计算并存储的 UTC 值，应该能简化你当前使用 UTC 值的方式。

计算 `starts_at_utc` 有几种方式，可以使用应用程序或数据库。虽然计算列本是生成列的绝佳用例，但 Postgres 不允许 `timestamp with time zone` 列类型作为生成列，因为 `timestamptz` 由于时区规则的变化而未被归类为不可变的。因此，使用触发器在插入和更新时计算 `starts_at`：

```sql
CREATE OR REPLACE FUNCTION recompute_appointment_utc()
RETURNS TRIGGER AS $$
BEGIN
  NEW.starts_at_utc := NEW.local_time AT TIME ZONE NEW.timezone_name;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER ts_recompute_starts_at_utc
  BEFORE INSERT OR UPDATE ON appointments
  FOR EACH ROW
  EXECUTE FUNCTION recompute_appointment_utc();
```

## 双列模式下的时区变更处理

如果 `tzdata` 更新改变了某个时区的规则，数据库中已有的 `starts_at_utc` 值就会过时，需要重新计算。你可以用一条简单的 `UPDATE` 语句重新应用转换逻辑：

```sql
UPDATE appointments
SET starts_at_utc = local_time AT TIME ZONE timezone_name
WHERE timezone_name = 'America/Vancouver'
  AND starts_at_utc > now();
```

## RFC 9557 呢？

2024 年，RFC 9557 发布了一种新的时间戳格式，形如 `1996-12-19T16:39:57-08:00[America/Los_Angeles]`。2025 年 11 月，pgsql-general 论坛上对此进行了简短讨论。由于该标准仍然很新，大家在观望其采用情况，因此使用尚未推进。

然而，RFC 9557 明确指出它并不旨在解决：

> 以某个指定时区的本地时间给出的未来时间，其中该时区定义的变更（如实施或取消夏令时的政治决定）会影响该时间戳所代表的时刻；

因此，对于足够远的未来的现实世界时间，仍然应使用双列模式。

## 如果 tzdata 已经更新了怎么办？

如果你已经为新时区更新了 `tzdata` 包，并且你的列值被赋予了未知的 UTC 偏移，同时你的数据库为不列颠哥伦比亚的实体记录了未来时间，那你面前就有一个数据项目了。理想情况下，你应该：

- 查明或估计 `tzdata` 包的更新时间
- 找出所有可能不正确的记录
- 使用 `tzdata` 更新之后的 `updated_at` 时间戳识别可能受影响的行
- 制定通知用户时间偏移调整的计划，并提供选择退出或选择加入的方案
- 在非生产数据集上针对可能受影响的行测试时间偏移迁移
- 执行备份，然后在生产环境上运行时间偏移迁移
- 为受变更影响的日历项添加 UI 元素
- 当现已废除的十一月时间变更临近时，再次通知用户可能存在的时区问题

不列颠哥伦比亚拥有 580 万人口，其时区偏好变更将广泛影响某些数据集，而对另一些则毫无影响。不要被时区变化所困——`tzdata` 包更新的频率之高令人惊讶。
