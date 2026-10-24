> TimescaleDB 是构建于 PostgreSQL 之上的时序数据库扩展，三大核心特性正好对应时序数据的完整生命周期：自动分片（Chunk）解决海量写入与查询裁剪，列式压缩（Compression）可将存储成本降低 90% 以上，保留策略（Retention）让过期数据自动清理、无需人工运维。本文基于 TimescaleDB 2.x，系统讲解三者的原理、配置与示例 SQL。

---

## 一、TimescaleDB 与 hypertable

### 1.1 核心概念

| 概念 | 英文 | 说明 |
|------|------|------|
| 超级表 | hypertable | 面向用户的逻辑表，增删改查与普通表完全一致 |
| 分片 | chunk | hypertable 底层的物理子表，按时间范围（可选叠加空间维度）自动切分 |

用户始终对 hypertable 操作，TimescaleDB 自动把数据路由到对应的 chunk，整个过程对应用透明：

```
hypertable: metrics（逻辑表，用户可见）
      │
      ├── chunk _hyper_1_1_chunk   [10-01, 10-08)   热数据，未压缩
      ├── chunk _hyper_1_2_chunk   [09-24, 10-01)   未压缩
      ├── chunk _hyper_1_3_chunk   [09-17, 09-24)   已压缩（列式存储）
      └── ...超过保留期的 chunk 由保留策略自动 DROP
```

### 1.2 与 PostgreSQL 原生分区表的对比

PostgreSQL 自带声明式分区，但分区需要手工管理，也没有压缩和自动清理。TimescaleDB 相当于"自动化的分区 + 列式压缩 + 生命周期管理"：

| 对比项 | PostgreSQL 声明式分区 | TimescaleDB hypertable |
|--------|----------------------|------------------------|
| 分区创建 | 手工执行（或借助 pg_partman） | 按时间自动创建，无需干预 |
| 分区裁剪 | 支持 | 支持且可跨 chunk 并行扫描 |
| 索引维护 | 需在每个分区上维护 | 建在 hypertable 上，自动下发到所有 chunk |
| 数据压缩 | 无原生列式压缩 | 内置列式压缩，通常 10 倍以上 |
| 过期清理 | 手写定时任务 DROP 分区 | 一条保留策略自动完成 |

### 1.3 环境准备

```sql
-- 安装扩展包后，在库中启用（需超级用户）
CREATE EXTENSION IF NOT EXISTS timescaledb;
```

> 提示：时间列推荐使用 `TIMESTAMPTZ`（带时区）类型，避免时区换算导致的分片边界问题。

---

## 二、分片（Chunk）

### 2.1 创建 hypertable

先建普通表，再转换为 hypertable：

```sql
-- 1. 创建普通表
CREATE TABLE metrics (
    time        TIMESTAMPTZ      NOT NULL,
    device_id   BIGINT           NOT NULL,
    cpu         DOUBLE PRECISION DEFAULT 0,
    memory      DOUBLE PRECISION DEFAULT 0
);

-- 2. 按时间列自动分片，每 7 天一个 chunk
SELECT create_hypertable('metrics', 'time',
    chunk_time_interval => INTERVAL '7 days',
    if_not_exists       => TRUE
);
```

注意事项：

- **唯一约束必须包含时间列**：数据分散在不同 chunk 中，无法保证跨 chunk 的全局唯一，因此主键/唯一索引必须带上时间列：

```sql
-- 错误：仅 device_id 无法保证跨 chunk 唯一
-- 正确：带上时间列
CREATE UNIQUE INDEX ON metrics (device_id, time DESC);
```

- 表中已有数据时，加 `migrate_data => TRUE` 可自动迁移
- 其他表可以引用普通表，但**不能外键引用 hypertable**（反向允许）

### 2.2 chunk_time_interval 怎么选

官方经验法则：**单个 chunk（含索引）的大小控制在数据库可用内存的 25% 以内，并保证热数据所在 chunk 能整体驻留内存**。间隔过小会导致 chunk 数量爆炸、规划器开销上升；间隔过大则裁剪粒度变粗、数据倾斜。

| 数据特征 | 起步建议 |
|---------|---------|
| 高频写入（每秒万级以上） | 1 天 ~ 7 天 |
| 中频（分钟级采样） | 1 周 ~ 1 个月 |
| 低频（小时/天级） | 1 个月 ~ 1 年 |

先按经验值设置，再根据实际 chunk 大小调整；**调整只影响新建 chunk，不影响已有 chunk**：

```sql
-- 动态调整 chunk 间隔（只对新 chunk 生效）
SELECT set_chunk_time_interval('metrics', INTERVAL '1 day');
```

### 2.3 空间分片（可选，慎用）

可以在时间维度之外，再按某一列（如 `device_id`）把每个时间 chunk 切成 N 个空间分区：

```sql
SELECT create_hypertable('metrics', 'time',
    chunk_time_interval => INTERVAL '7 days',
    partitioning_column => 'device_id',
    number_partitions   => 4
);
```

官方明确建议：**单机场景一般不要启用空间分片**。它主要服务于超大规模数据集，普通规模下只会让 chunk 数量成倍增加，加大规划与合并开销。

### 2.4 查看与管理 chunk

```sql
-- 查看 hypertable 的所有 chunk
SELECT
    chunk_schema,
    chunk_name,
    range_start,
    range_end,
    is_compressed
FROM timescaledb_information.chunks
WHERE hypertable_name = 'metrics'
ORDER BY range_start;

-- 只列出 30 天前的 chunk
SELECT show_chunks('metrics', older_than => INTERVAL '30 days');

-- 查看某行数据落在哪个 chunk 里
SELECT tableoid::regclass AS chunk
FROM metrics
ORDER BY time DESC
LIMIT 1;
```

在 hypertable 上创建的索引会自动应用到已存在与未来创建的所有 chunk：

```sql
-- 典型查询索引：按设备查最近数据（自动下发到每个 chunk）
CREATE INDEX idx_metrics_device_time ON metrics (device_id, time DESC);
```

---

## 三、压缩（Compression）

### 3.1 压缩原理

TimescaleDB 将 chunk 整体转换为**列式存储**：

- 每 1000 行组成一个 batch，按列编码存放：数值列做增量压缩，字符串列做字典/游程编码
- `compress_segmentby` 指定的列，值相同的行会聚合存放（类似按该列 GROUP BY），查询时可整段跳过无关数据
- `compress_orderby` 决定批内物理排序（默认按时间倒序），利于时间范围扫描

| 设置项 | 作用 | 选择建议 |
|--------|------|---------|
| `timescaledb.compress_segmentby` | 分组存储列 | 查询 WHERE 中最常出现的等值过滤列，如 `device_id` |
| `timescaledb.compress_orderby` | 批内排序列 | 默认 `time DESC`，可追加次级排序列 |

压缩属于"旧数据归档"手段：压缩后的数据仍可正常 SELECT，但写更新成本变高（见 3.6）。

### 3.2 启用压缩

```sql
ALTER TABLE metrics SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'device_id',
    timescaledb.compress_orderby   = 'time DESC'
);
```

该语句只设置压缩参数，**不会**立即压缩任何数据。

### 3.3 手动压缩

```sql
-- 压缩指定 chunk
SELECT compress_chunk('_timescaledb_internal._hyper_1_2_chunk');

-- 一次性压缩所有超过 7 天的 chunk
SELECT compress_chunk(chunk)
FROM show_chunks('metrics', older_than => INTERVAL '7 days') AS chunk;
```

### 3.4 自动压缩策略

```sql
-- 7 天前的 chunk 自动压缩（策略默认每 12 小时调度一次）
SELECT add_compression_policy('metrics', older_than => INTERVAL '7 days');
```

> 版本兼容：TimescaleDB 2.13 及以上参数名为 `older_than`；更早版本写作 `compress_after`（新版本中旧参数名仍作为兼容别名保留）。

### 3.5 查看压缩效果

```sql
-- 每个 chunk 的压缩前后大小
SELECT
    chunk_name,
    compression_status,
    pg_size_pretty(before_compression_total_bytes) AS "压缩前",
    pg_size_pretty(after_compression_total_bytes)  AS "压缩后"
FROM timescaledb_information.chunk_compression_stats('metrics');

-- 整体压缩率
SELECT
    pg_size_pretty(sum(before_compression_total_bytes)) AS "压缩前总量",
    pg_size_pretty(sum(after_compression_total_bytes))  AS "压缩后总量",
    round(100 * (1 - sum(after_compression_total_bytes)::numeric
                  / NULLIF(sum(before_compression_total_bytes), 0)), 1) AS "压缩率(%)"
FROM timescaledb_information.chunk_compression_stats('metrics');
```

时序数据重复度高，实测压缩率通常在 **90% ~ 97%**（约 10 ~ 30 倍）。

### 3.6 更新压缩数据

```sql
-- 手动解压某个 chunk（修改历史数据的经典做法）
SELECT decompress_chunk('_timescaledb_internal._hyper_1_2_chunk');

-- 解压后即可正常 UPDATE / DELETE
UPDATE metrics
SET cpu = 0.5
WHERE device_id = 1
  AND time < '2026-10-01';
```

版本差异：

- 旧版本（2.10 及以前）压缩 chunk 完全只读，更新前必须解压
- 2.11+ 支持直接向压缩 chunk INSERT
- 2.13+ 进一步支持对压缩 chunk 直接 UPDATE / DELETE（批量修数时仍建议先解压，性能更好）

---

## 四、数据过期删除（Retention）

### 4.1 drop_chunks 与 DELETE 的区别

| 对比项 | `DELETE FROM ... WHERE time < ...` | `drop_chunks` |
|--------|-----------------------------------|---------------|
| 执行方式 | 逐行标记删除 | 直接 DROP 整个 chunk 物理表 |
| 速度 | 慢（与行数成正比） | 极快（与行数无关） |
| 空间回收 | 需 VACUUM，有膨胀风险 | 立即释放 |
| 索引维护 | 产生死元组 | 无 |
| 适用场景 | 精确删除少量数据 | 按时间窗口批量清理 |

### 4.2 手动删除：drop_chunks

```sql
-- 删除 90 天前的所有 chunk（无论是否已压缩）
SELECT drop_chunks('metrics', older_than => INTERVAL '90 days');

-- 指定具体时间点
SELECT drop_chunks('metrics', older_than => '2026-07-01'::timestamptz);
```

**边界说明**：drop_chunks 按 chunk 整体删除——只有 chunk 的时间范围完全早于阈值才会被删。假设 chunk 间隔为 7 天、保留期为 90 天，实际保留的数据会在 90 ~ 97 天之间浮动。**把保留期设置为 chunk 间隔的整数倍，可以让删除边界更整齐**。

### 4.3 自动策略：add_retention_policy

```sql
-- 只保留 90 天数据，超期自动删除（策略默认每天调度一次）
SELECT add_retention_policy('metrics', older_than => INTERVAL '90 days');
```

> 版本兼容：2.13 及以上参数名为 `older_than`；更早版本写作 `drop_after`。

### 4.4 策略管理

压缩策略和保留策略本质上都是注册到后台调度的 job：

```sql
-- 查看该表上的所有后台任务
SELECT
    job_id,
    application_name,
    schedule_interval,
    config
FROM timescaledb_information.jobs
WHERE hypertable_name = 'metrics';

-- 查看任务执行统计
SELECT job_id, total_successes, total_failures
FROM timescaledb_information.job_stats;

-- 调整调度频率
SELECT alter_job(1001, schedule_interval => INTERVAL '2 hours');

-- 移除策略
SELECT remove_compression_policy('metrics');
SELECT remove_retention_policy('metrics');

-- 彻底删除某个 job
SELECT delete_job(1001);
```

### 4.5 注意事项

- drop_chunks 与保留策略**不区分 chunk 是否压缩**，压缩数据同样会被自动清理
- 删除的是**整个 chunk**，存在 4.2 所述的边界浮动
- **压缩 + 保留是黄金组合**：保留窗口内仍需查询的数据全部压缩，把存储成本降到最低后再过期删除

---

## 五、完整实战：一张 IoT 指标表的全生命周期

```sql
-- 1. 启用扩展
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- 2. 建表（时间列用 TIMESTAMPTZ）
CREATE TABLE metrics (
    time        TIMESTAMPTZ      NOT NULL,
    device_id   BIGINT           NOT NULL,
    cpu         DOUBLE PRECISION DEFAULT 0,
    memory      DOUBLE PRECISION DEFAULT 0
);

-- 3. 转为 hypertable：按天分片
SELECT create_hypertable('metrics', 'time',
    chunk_time_interval => INTERVAL '1 day',
    if_not_exists       => TRUE
);

-- 4. 常用查询索引（自动下发到每个 chunk）
CREATE INDEX idx_metrics_device_time ON metrics (device_id, time DESC);

-- 5. 开启压缩：按设备聚合、按时间排序
ALTER TABLE metrics SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'device_id',
    timescaledb.compress_orderby   = 'time DESC'
);

-- 6. 自动压缩 7 天前的数据
SELECT add_compression_policy('metrics', older_than => INTERVAL '7 days');

-- 7. 自动删除 90 天前的数据
SELECT add_retention_policy('metrics', older_than => INTERVAL '90 days');

-- 8. 验证：两个策略已在后台运行
SELECT job_id, application_name, schedule_interval
FROM timescaledb_information.jobs
WHERE hypertable_name = 'metrics';
```

运行后的数据生命周期：

```
时间轴 ─────────────────────────────────────────────────────▶
 现在                          -7天              -90天
  │  可写可查（未压缩）          │   已压缩          │   自动 DROP
  ▼                            ▼                  ▼
 INSERT/SELECT ── 压缩策略 ─────┤── 保留策略 ───────┤──► 磁盘空间立即释放
```

日常查询无需任何修改，压缩与分片对用户完全透明：

```sql
-- time_bucket 为 TimescaleDB 内置的时间分桶函数
SELECT
    time_bucket('1 hour', time) AS bucket,
    avg(cpu)                    AS avg_cpu
FROM metrics
WHERE device_id = 1001
  AND time > now() - INTERVAL '24 hours'
GROUP BY bucket
ORDER BY bucket;
```

---

## 六、最佳实践与常见问题

### 6.1 最佳实践

1. **时间列用 TIMESTAMPTZ**，避免时区边界问题
2. **唯一/主键约束必须包含时间列**
3. **chunk 大小控制在内存 25% 以内**，间隔宁大勿小
4. **segmentby 选查询过滤列**（如 `device_id`）；没有过滤列就只设 `orderby`
5. **先压缩、后删除**：保留期内数据全部压缩，存储成本最小
6. **保留期设为 chunk 间隔的整数倍**，删除边界整齐
7. **监控 job_stats 的 total_failures**：策略执行失败不会自动告警
8. 保留策略配错会批量删数据且**不可恢复**，务必做好备份

### 6.2 常见问题

**Q1：压缩后查询会变慢吗？**
分析型查询（扫描大量行、只取少数列）通常更快——列裁剪加批量解码；精确点查可能略慢。segmentby 选得合适，扫描量会大幅减少。

**Q2：chunk_time_interval 调整后，已有数据会重新分片吗？**
不会，只影响新建 chunk，历史 chunk 保持原样。

**Q3：drop_chunks 删除的数据比预期多？**
chunk 是整体删除的，且边界按 chunk 时间范围对齐，实际删除量在"保留期 ± 一个 chunk 间隔"之间浮动。

**Q4：可以只删除某几个设备的数据吗？**
drop_chunks 只按时间删。按业务维度删除需要 DELETE（可先 `decompress_chunk` 对应 chunk），或按维度分表。

**Q5：修改压缩参数后，已压缩的旧 chunk 会按新参数重压吗？**
不会。需要先 `decompress_chunk` 解压，再重新 `compress_chunk`。

---

## 总结

| 特性 | 一句话 | 核心命令 |
|------|--------|---------|
| 分片 | 按时间自动切表，透明分区裁剪 | `create_hypertable` |
| 压缩 | chunk 转列式存储，节省 90% 以上空间 | `ALTER TABLE SET (timescaledb.compress, ...)` + `add_compression_policy` |
| 过期删除 | 整 chunk 秒删，免 VACUUM | `drop_chunks` / `add_retention_policy` |

三者组合起来，就是时序数据的标准生命周期管理方案：新数据高速写入、温数据自动压缩、冷数据自动过期，全程无需人工干预。

---

<p align="center">
    <a href="https://github.com/dkbnull/hello-wiki/blob/main/40-Database/09-TimescaleDB%E5%88%86%E7%89%87%E5%8E%8B%E7%BC%A9%E4%B8%8E%E6%95%B0%E6%8D%AE%E8%BF%87%E6%9C%9F%E5%88%A0%E9%99%A4.md" target="_blank">
        <img src="https://img.shields.io/badge/GitHub-访问地址-blue?logo=github">
    </a>
    <a href="https://gitee.com/dkbnull/hello-wiki/blob/main/40-Database/09-TimescaleDB%E5%88%86%E7%89%87%E5%8E%8B%E7%BC%A9%E4%B8%8E%E6%95%B0%E6%8D%AE%E8%BF%87%E6%9C%9F%E5%88%A0%E9%99%A4.md" target="_blank">
        <img src="https://img.shields.io/badge/Gitee-访问地址-red?logo=gitee">
    </a>
    <a href="https://mp.weixin.qq.com/s/Pewm94GTG4dGn-dlvR0OBw" target="_blank">
       <img src="https://img.shields.io/badge/微信公众号-访问地址-brightgreen?logo=wechat">
    </a>
    <a href="https://juejin.cn/post/7699871906405892138" target="_blank">
       <img src="https://img.shields.io/badge/掘金-访问地址-blue?logo=juejin">
    </a>
    <a href="https://zhuanlan.zhihu.com/p/2097444507369394319" target="_blank">
       <img src="https://img.shields.io/badge/知乎-访问地址-blue?logo=zhihu">
    </a>
</p>
