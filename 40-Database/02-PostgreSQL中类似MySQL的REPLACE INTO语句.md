# 使用 `INSERT ... ON CONFLICT`

PostgreSQL 提供了 **UPSERT** 功能（`INSERT ... ON CONFLICT`），支持在唯一键冲突时更新数据，语法如下：

```sql
INSERT INTO 表名 (字段列表)
VALUES (值列表)
ON CONFLICT (冲突字段) 
DO UPDATE SET 字段 = EXCLUDED.字段, ...;
```

示例：

```sql
INSERT INTO table_name (
	id,
	name,
	remark)
VALUES (
:id,
:name,
:remark) 
ON CONFLICT(id) DO UPDATE SET
	name = :name,
	remark = :remark
```

# 手动实现 `DELETE + INSERT`

如果需完全删除旧行并插入新行（完全模拟 MySQL 的 `REPLACE INTO`），可显式使用事务：

```sql
WITH 
  deleted AS (
    DELETE FROM 表名 
    WHERE 条件
    RETURNING *
  )
INSERT INTO 表名 (字段列表)
VALUES (值列表);
```

示例：

```sql
BEGIN;

-- 先尝试删除（需明确指定条件）
DELETE FROM table_name 
WHERE id = 1;

-- 再插入新数据
INSERT INTO table_name (id,	name, remark)
VALUES (1, 'test', 'remark');

COMMIT;
```

