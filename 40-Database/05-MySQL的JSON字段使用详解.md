> MySQL 5.7.8+ 起原生支持 `JSON` 类型；MySQL 8.0 大幅增强了 JSON 函数与索引能力。本文以 MySQL 8.0 为基准，系统讲解 JSON 字段的创建、增删改查、路径表达式、全部内置函数、索引方案、性能优化与避坑指南。

---

## 一、为什么需要 JSON 字段

### 1.1 传统关系模型的痛点

当业务存在以下场景时，固定表结构会变得笨重：

- **字段频繁变动**：商品属性（颜色、尺寸、材质）随品类不同而不同，每加一个属性就 `ALTER TABLE` 代价高。
- **半结构化数据**：日志、配置、第三方接口返回的嵌套 JSON，强行拆成多表 JOIN 成本高。
- **弹性扩展**：用户画像标签、风控规则、动态表单等需求字段不定。

### 1.2 JSON 字段的优势

| 特性 | 说明 |
|------|------|
| **原生类型** | `JSON` 不是字符串，而是二进制存储（BSON 风格），读取时无需解析文本 |
| **自动校验** | 写入时校验 JSON 合法性，非法 JSON 直接报错 |
| **路径查询** | 通过 JSONPath 表达式精准提取嵌套值 |
| **函数丰富** | 30+ 内置函数覆盖增删改查、聚合、搜索、表展开 |
| **可索引** | 通过生成列（Generated Column）或多值索引（Multi-Valued Index）支持索引加速 |
| **保留类型** | 数字、字符串、布尔、null、数组、对象在 JSON 内部有独立类型，区分 `"123"` 与 `123` |

### 1.3 何时该用、何时别用

✅ **建议使用**：
- 字段结构动态、可变，且不需要对这些字段做高频复杂关联
- 配置类、属性类、扩展类数据
- 日志/事件存储
- 与 MongoDB 类似场景但希望事务一致性仍在 MySQL 内

❌ **不建议使用**：
- 字段需要参与高频 `JOIN`、`GROUP BY`、复杂 `WHERE` 过滤，且字段固定 → 应拆为正式列
- 把 JSON 当万能字段，所有业务数据塞进去 → 破坏关系模型，可维护性骤降
- 对字段做范围扫描且数据量大极大但未建索引 → 全表扫描性能差

> **原则**：JSON 是"逃生舱"，不是"主战场"。能用普通列就用普通列，JSON 用于真正的弹性字段。

---

## 二、创建表与 JSON 字段

### 2.1 基本建表

```sql
CREATE TABLE `t_user_profile` (
    `id`           BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '主键',
    `user_name`    VARCHAR(64)     NOT NULL COMMENT '用户名',
    `profile`      JSON            NULL     COMMENT '用户画像（JSON）',
    `ext_attrs`    JSON            NULL     COMMENT '扩展属性（JSON）',
    `create_time`  DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    `update_time`  DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    `is_deleted`   TINYINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '逻辑删除：0未删除 1已删除',
    PRIMARY KEY (`pk_user_profile` (`id`)),
    KEY `idx_user_name` (`user_name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户画像表';
```

### 2.2 默认值

MySQL 8.0.13 起，JSON 列允许设置默认值（之前只能为 `NULL`）：

```sql
ALTER TABLE `t_user_profile`
    MODIFY `ext_attrs` JSON NULL DEFAULT (JSON_OBJECT());
```

### 2.3 字符集与排序规则

JSON 列内部始终使用 UTF8MB4（`utf8mb4_bin` 排序），与表本身的字符集无关，无需特别设置。

---

## 三、JSON 路径表达式（JSONPath）

几乎所有 JSON 函数都依赖路径表达式，先理解路径语法是基础。

### 3.1 路径语法

路径以 `$` 开头，表示当前 JSON 文档根。

| 语法 | 含义 | 示例 |
|------|------|------|
| `$.key` | 对象成员 | `$.name` |
| `$.key.subkey` | 嵌套成员 | `$.address.city` |
| `$[i]` | 数组第 i 个元素（从 0 开始） | `$.tags[0]` |
| `$[*]` | 数组所有元素 | `$.tags[*]` |
| `$.**` | 递归下降，匹配所有层级 | `$.**.city` |
| `$.key[$[0 to 2]]` | 数组范围 | `$.tags[0 to 2]` |
| `$.key[last]` | 数组最后一个元素 | `$.tags[last]` |
| `$.key[last-1 to last]` | 倒数两个 | `$.tags[last-1 to last]` |

### 3.2 路径示例

给定 JSON：

```json
{
  "name": "张三",
  "age": 28,
  "address": { "city": "北京", "zip": "100000" },
  "tags": ["vip", "active", "high-value"],
  "orders": [
    { "id": "A001", "amount": 199.00 },
    { "id": "A002", "amount": 88.50 }
  ]
}
```

| 路径 | 结果 |
|------|------|
| `$.name` | `"张三"` |
| `$.address.city` | `"北京"` |
| `$.tags[1]` | `"active"` |
| `$.tags[*]` | `["vip", "active", "high-value"]` |
| `$.orders[*].id` | `["A001", "A002"]` |
| `$.orders[last].amount` | `88.50` |
| `$**.amount` | `[199.00, 88.50]` |

### 3.3 路径不存在

如果路径不存在，函数返回 `NULL`，而不是报错（除非显式断言）：

```sql
SELECT JSON_EXTRACT('{"a":1}', '$.b');  -- 返回 NULL
```

---

## 四、插入与更新 JSON 数据

### 4.1 插入数据

JSON 列接受字符串字面量，会自动校验并转为二进制存储：

```sql
INSERT INTO `t_user_profile` (`user_name`, `profile`, `ext_attrs`)
VALUES (
    'zhangsan',
    '{"age":28,"gender":"M","address":{"city":"北京","zip":"100000"},"tags":["vip","active"]}',
    '{}'
);
```

更推荐用 `JSON_OBJECT` / `JSON_ARRAY` 构造，避免字符串拼接错误：

```sql
INSERT INTO `t_user_profile` (`user_name`, `profile`, `ext_attrs`)
VALUES (
    'lisi',
    JSON_OBJECT(
        'age', 25,
        'gender', 'F',
        'address', JSON_OBJECT('city', '上海', 'zip', '200000'),
        'tags', JSON_ARRAY('new', 'active')
    ),
    JSON_OBJECT()
);
```

### 4.2 批量插入

```sql
INSERT INTO `t_user_profile` (`user_name`, `profile`) VALUES
    ('u1', JSON_OBJECT('age', 20, 'level', 1)),
    ('u2', JSON_OBJECT('age', 30, 'level', 2)),
    ('u3', JSON_OBJECT('age', 40, 'level', 3));
```

### 4.3 非法 JSON 报错

```sql
INSERT INTO `t_user_profile` (`user_name`, `profile`)
VALUES ('bad', '{age:28}');   -- 缺引号，非法 JSON
-- ERROR 3140 (22032): Invalid JSON text
```

### 4.4 整体替换更新

```sql
UPDATE `t_user_profile`
SET `profile` = JSON_OBJECT('age', 29, 'gender', 'M')
WHERE `user_name` = 'zhangsan';
```

### 4.5 局部更新（推荐）

使用 `JSON_SET` / `JSON_INSERT` / `JSON_REPLACE` / `JSON_REMOVE` 进行局部修改，避免整体覆盖：

```sql
-- 修改 age 并新增 last_login
UPDATE `t_user_profile`
SET `profile` = JSON_SET(`profile`, '$.age', 29, '$.last_login', '2026-06-27')
WHERE `user_name` = 'zhangsan';

-- 仅在路径不存在时插入
UPDATE `t_user_profile`
SET `profile` = JSON_INSERT(`profile`, '$.reg_time', '2026-01-01')
WHERE `user_name` = 'zhangsan';

-- 仅在路径存在时替换
UPDATE `t_user_profile`
SET `profile` = JSON_REPLACE(`profile`, '$.age', 30)
WHERE `user_name` = 'zhangsan';

-- 删除指定路径
UPDATE `t_user_profile`
SET `profile` = JSON_REMOVE(`profile`, '$.tags[0]')
WHERE `user_name` = 'zhangsan';
```

四个函数的区别：

| 函数 | 路径存在时 | 路径不存在时 |
|------|-----------|-------------|
| `JSON_SET` | 覆盖 | 创建 |
| `JSON_INSERT` | 不变 | 创建 |
| `JSON_REPLACE` | 覆盖 | 不变 |
| `JSON_REMOVE` | 删除 | 不变 |

---

## 五、查询 JSON 数据

### 5.1 提取值：`JSON_EXTRACT` 与 `->` / `->>`

```sql
-- 函数形式
SELECT JSON_EXTRACT(`profile`, '$.age')        AS age,
       JSON_EXTRACT(`profile`, '$.address.city') AS city
FROM `t_user_profile`
WHERE `user_name` = 'zhangsan';

-- 等价的操作符 -> （返回带类型）
SELECT `profile`->'$.age'          AS age,
       `profile`->'$.address.city' AS city
FROM `t_user_profile`
WHERE `user_name` = 'zhangsan';

-- 操作符 ->> 返回去除引号的字符串（等同于 JSON_UNQUOTE(JSON_EXTRACT(...))）
SELECT `profile`->>'$.address.city' AS city
FROM `t_user_profile`
WHERE `user_name` = 'zhangsan';
```

> **`->` 与 `->>` 的区别**：
> - `->` 返回 JSON 值，字符串带双引号：`"北京"`
> - `->>` 返回纯字符串：`北京`

### 5.2 在 WHERE 中使用

```sql
-- 查 age > 25 的用户
SELECT `user_name`, `profile`->>'$.age' AS age
FROM `t_user_profile`
WHERE `profile`->'$.age' > 25;

-- 查 city = '北京'
SELECT `user_name`
FROM `t_user_profile`
WHERE `profile`->>'$.address.city' = '北京';

-- 查标签包含 vip
SELECT `user_name`
FROM `t_user_profile`
WHERE JSON_CONTAINS(`profile`->'$.tags', '"vip"');
```

### 5.3 注意类型隐式转换

JSON 内部保留类型，比较时需注意：

```sql
-- profile.age 是 JSON number，与字符串比较时会隐式转换，可能不走索引
SELECT * FROM `t_user_profile` WHERE `profile`->'$.age' = 28;      -- 推荐
SELECT * FROM `t_user_profile` WHERE `profile`->>'$.age' = '28';   -- 字符串比较，性能差
```

---

## 六、JSON 内置函数全览

MySQL 8.0 提供了 30+ 个 JSON 函数，按用途分类如下。

### 6.1 创建类函数

| 函数 | 作用 | 示例 |
|------|------|------|
| `JSON_OBJECT(k, v, ...)` | 构造对象 | `JSON_OBJECT('a', 1, 'b', 2)` → `{"a": 1, "b": 2}` |
| `JSON_ARRAY(v, ...)` | 构造数组 | `JSON_ARRAY(1, 'x', NULL)` → `[1, "x", null]` |
| `JSON_QUOTE(str)` | 把字符串转为 JSON 字符串字面量 | `JSON_QUOTE('a"b')` → `"a\"b"` |
| `JSON_UNQUOTE(json)` | 去除 JSON 字符串的引号 | `JSON_UNQUOTE('"a"')` → `a` |

```sql
SELECT JSON_OBJECT('name', '张三', 'age', 28) AS obj;
-- {"name": "张三", "age": 28}

SELECT JSON_ARRAY(1, 2, JSON_OBJECT('k', 'v'));
-- [1, 2, {"k": "v"}]
```

### 6.2 查询提取类函数

| 函数 | 作用 |
|------|------|
| `JSON_EXTRACT(doc, path, ...)` | 按路径提取，返回 JSON 值 |
| `JSON_UNQUOTE(json)` | 去除引号得到原始字符串 |
| `->` / `->>` | `JSON_EXTRACT` / `JSON_EXTRACT`+`JSON_UNQUOTE` 的简写 |
| `JSON_KEYS(doc[, path])` | 返回对象所有 key 的数组 |
| `JSON_LENGTH(doc[, path])` | 返回 JSON 长度（对象=成员数，数组=元素数，标量=1） |
| `JSON_DEPTH(doc)` | 返回 JSON 最大嵌套深度 |
| `JSON_TYPE(json_val)` | 返回 JSON 值类型字符串 |

```sql
SET @j = '{"a": [10, 20], "b": {"c": 30}}';

SELECT JSON_KEYS(@j);          -- ["a", "b"]
SELECT JSON_KEYS(@j, '$.b');   -- ["c"]
SELECT JSON_LENGTH(@j);        -- 2（顶层有 a、b 两个 key）
SELECT JSON_LENGTH(@j, '$.a'); -- 2
SELECT JSON_DEPTH(@j);         -- 3
SELECT JSON_TYPE(@j);          -- OBJECT
SELECT JSON_TYPE(@j->'$.a');   -- ARRAY
```

### 6.3 修改类函数

| 函数 | 作用 |
|------|------|
| `JSON_SET(doc, path, val, ...)` | 存在则覆盖，不存在则新增 |
| `JSON_INSERT(doc, path, val, ...)` | 仅在不存在时插入 |
| `JSON_REPLACE(doc, path, val, ...)` | 仅在存在时替换 |
| `JSON_REMOVE(doc, path, ...)` | 删除指定路径 |
| `JSON_ARRAY_APPEND(doc, path, val, ...)` | 在数组末尾追加 |
| `JSON_ARRAY_INSERT(doc, path, val, ...)` | 在数组指定位置插入 |
| `JSON_MERGE_PATCH(doc, ...)` | RFC 7396 合并（同 key 后者覆盖前者，null 删除 key） |
| `JSON_MERGE_PRESERVE(doc, ...)` | 合并但保留同名数组（追加而非覆盖） |

```sql
SET @j = '{"a": 1, "b": [2, 3]}';

SELECT JSON_SET(@j, '$.a', 10, '$.c', 30);
-- {"a": 10, "b": [2, 3], "c": 30}

SELECT JSON_INSERT(@j, '$.a', 99, '$.d', 40);
-- {"a": 1, "b": [2, 3], "d": 40}    -- a 已存在不变

SELECT JSON_REPLACE(@j, '$.a', 99, '$.d', 40);
-- {"a": 99, "b": [2, 3]}             -- d 不存在不变

SELECT JSON_REMOVE(@j, '$.b[0]');
-- {"a": 1, "b": [3]}

SELECT JSON_ARRAY_APPEND(@j, '$.b', 4);
-- {"a": 1, "b": [2, 3, 4]}

SELECT JSON_ARRAY_INSERT(@j, '$.b[1]', 99);
-- {"a": 1, "b": [2, 99, 3]}

-- 合并示例
SELECT JSON_MERGE_PATCH('{"a":1,"b":2}', '{"b":3,"c":4}');
-- {"a": 1, "b": 3, "c": 4}

SELECT JSON_MERGE_PRESERVE('{"a":[1,2]}', '{"a":[3,4]}');
-- {"a": [1, 2, 3, 4]}
```

> **`JSON_MERGE_PATCH` vs `JSON_MERGE_PRESERVE`**：
> - `PATCH`：对象按 key 覆盖；数组整体覆盖；值为 `null` 时删除 key
> - `PRESERVE`：对象 key 不存在则添加，存在则递归合并；数组追加

### 6.4 搜索类函数

| 函数 | 作用 |
|------|------|
| `JSON_CONTAINS(target, candidate[, path])` | 是否包含候选值 |
| `JSON_CONTAINS_PATH(doc, one\|all, path, ...)` | 是否存在指定路径 |
| `JSON_SEARCH(doc, one\|all, str[, escape[, path]...])` | 按字符串值搜索，返回路径 |
| `JSON_EXTRACT` | 也可视为搜索提取 |

```sql
SET @j = '{"a":[1,2,{"x":3}],"b":"hello"}';

-- 是否包含数字 2
SELECT JSON_CONTAINS(@j, '2');                 -- 1
-- 是否在 a 中包含对象 {"x":3}
SELECT JSON_CONTAINS(@j, '{"x":3}', '$.a');     -- 1

-- 是否存在路径 $.a 或 $.c
SELECT JSON_CONTAINS_PATH(@j, 'one', '$.a', '$.c');  -- 1（任一存在）
SELECT JSON_CONTAINS_PATH(@j, 'all', '$.a', '$.c');  -- 0（不全存在）

-- 搜索值为 "hello" 的路径
SELECT JSON_SEARCH(@j, 'one', 'hello');         -- "$.b"

-- 搜索数组中的值 3，返回所有匹配路径
SELECT JSON_SEARCH('{"a":[3,3]}', 'all', '3');  -- ["$.a[0]", "$.a[1]"]

-- 支持 % 与 _ 通配符
SELECT JSON_SEARCH(@j, 'one', 'hel%');          -- "$.b"
```

### 6.5 聚合类函数

| 函数 | 作用 |
|------|------|
| `JSON_ARRAYAGG(col_or_expr)` | 把一列值聚合成 JSON 数组 |
| `JSON_OBJECTAGG(key, value)` | 把两列聚合成 JSON 对象 |

```sql
-- 把所有用户名聚合成数组
SELECT JSON_ARRAYAGG(`user_name`) AS users
FROM `t_user_profile`;
-- ["zhangsan", "lisi", "u1", "u2", "u3"]

-- 按 level 聚合用户名
SELECT
    `profile`->>'$.level' AS level,
    JSON_ARRAYAGG(`user_name`) AS users
FROM `t_user_profile`
WHERE `profile`->>'$.level' IS NOT NULL
GROUP BY `profile`->>'$.level';

-- 构造 user_name -> age 的映射对象
SELECT JSON_OBJECTAGG(`user_name`, `profile`->'$.age') AS name_age_map
FROM `t_user_profile`;
-- {"zhangsan": 29, "lisi": 25, ...}
```

### 6.6 工具/校验类函数

| 函数 | 作用 |
|------|------|
| `JSON_VALID(json_str)` | 是否合法 JSON，返回 0/1 |
| `JSON_PRETTY(json)` | 格式化为易读多行字符串 |
| `JSON_STORAGE_SIZE(doc)` | JSON 列存储字节数 |
| `JSON_STORAGE_FREE(doc)` | 局部更新后释放的字节数 |
| `JSON_TABLE(...)` | 把 JSON 展开为关系表（见第七节） |
| `JSON_SCHEMA_VALID(schema, doc)` | 是否符合 JSON Schema（8.0.17+） |
| `JSON_SCHEMA_VALIDATION_REPORT(schema, doc)` | 返回校验报告 |
| `JSON_VALUE(doc, path)` | 提取标量并按指定 SQL 类型返回（8.0.21+） |

```sql
SELECT JSON_VALID('{"a":1}');       -- 1
SELECT JSON_VALID('{a:1}');         -- 0

SELECT JSON_PRETTY('{"a":1,"b":[2,3]}');
/*
{
  "a": 1,
  "b": [
    2,
    3
  ]
}
*/

-- JSON_VALUE 支持指定返回类型
SELECT JSON_VALUE('{"price":"99.5"}', '$.price' RETURNING DECIMAL(10,2));
-- 99.50

-- JSON Schema 校验
SET @schema = '{"type":"object","required":["name"],"properties":{"name":{"type":"string"}}}';
SELECT JSON_SCHEMA_VALID(@schema, '{"name":"张三"}');      -- 1
SELECT JSON_SCHEMA_VALID(@schema, '{"age":28}');           -- 0
```

---

## 七、JSON_TABLE：把 JSON 展开为表

`JSON_TABLE` 是 MySQL 8.0 最重要的 JSON 功能之一，它能把 JSON 数组展开成关系表，让 JSON 与 SQL 完美融合。

### 7.1 语法

```sql
JSON_TABLE(
    json_doc,
    path COLUMNS (
        column_name column_type PATH json_path
            [DEFAULT default_value ON EMPTY]
            [DEFAULT default_value ON ERROR]
            [column_list],
        ...
        NESTED PATH path COLUMNS ( ... )   -- 嵌套展开
        ...
    )
) [AS] alias
```

### 7.2 基本示例

```sql
SET @orders = '[
  {"id":"A001","amount":199.00,"items":["book","pen"]},
  {"id":"A002","amount":88.50,"items":["cup"]},
  {"id":"A003","amount":1200.00,"items":["phone","case","charger"]}
]';

SELECT *
FROM JSON_TABLE(
    @orders,
    '$[*]' COLUMNS (
        order_id   VARCHAR(10)  PATH '$.id',
        amount     DECIMAL(10,2) PATH '$.amount',
        item_count INT          PATH '$.items[*]'  -- 仅取数量需用其他方式
    )
) AS t;
```

### 7.3 NESTED PATH 嵌套展开

```sql
SELECT order_id, amount, item
FROM JSON_TABLE(
    @orders,
    '$[*]' COLUMNS (
        order_id VARCHAR(10)   PATH '$.id',
        amount   DECIMAL(10,2) PATH '$.amount',
        NESTED PATH '$.items[*]' COLUMNS (
            item VARCHAR(50) PATH '$'
        )
    )
) AS t;
```

结果：

| order_id | amount | item   |
|----------|--------|--------|
| A001     | 199.00 | book   |
| A001     | 199.00 | pen    |
| A002     | 88.50  | cup    |
| A003     | 1200.00| phone  |
| A003     | 1200.00| case   |
| A003     | 1200.00| charger|

### 7.4 从表中查询并展开

```sql
-- 假设 profile 中有 orders 数组
SELECT u.`user_name`, ot.order_id, ot.amount
FROM `t_user_profile` u,
JSON_TABLE(
    u.`profile`,
    '$.orders[*]' COLUMNS (
        order_id VARCHAR(20)    PATH '$.id',
        amount   DECIMAL(10,2)  PATH '$.amount'
    )
) AS ot
WHERE u.`is_deleted` = 0;
```

### 7.5 ON EMPTY / ON ERROR

```sql
SELECT *
FROM JSON_TABLE(
    '[{"a":1},{"a":"x"}]',
    '$[*]' COLUMNS (
        a_int INT PATH '$.a'
            DEFAULT '0' ON EMPTY
            DEFAULT '-1' ON ERROR
    )
) AS t;
-- 第一行 a_int=1，第二行 a_int=-1（字符串无法转 int）
```

---

## 八、JSON 索引方案

JSON 列本身**不能直接建索引**，但 MySQL 提供了三种间接方案。

### 8.1 方案一：生成列 + 索引（最常用）

把 JSON 中常用查询路径抽取为生成列（Generated Column），再在该列上建索引：

```sql
ALTER TABLE `t_user_profile`
    ADD COLUMN `age` INT AS (`profile`->'$.age') STORED,
    ADD COLUMN `city` VARCHAR(64) AS (`profile`->>'$.address.city') STORED,
    ADD KEY `idx_age` (`age`),
    ADD KEY `idx_city` (`city`);
```

- `STORED`：把计算结果物理存储，查询时直接读，性能最佳。
- `VIRTUAL`：不存储，只在读取/索引时计算，节省空间但索引读时需重算。

```sql
-- 查询时让优化器自动识别生成列
SELECT * FROM `t_user_profile`
WHERE `profile`->'$.age' > 25;
-- 优化器会自动使用 idx_age 索引
```

> **注意**：`STORED` 列表达式必须确定性，且与查询表达式**完全一致**才能被优化器识别为索引可替换。

### 8.2 方案二：多值索引（Multi-Valued Index，8.0.17+）

针对 JSON 数组中的元素查询，传统索引无能为力。多值索引可以为数组每个元素建立索引项。

```sql
-- 为 tags 数组建多值索引
ALTER TABLE `t_user_profile`
    ADD KEY `idx_tags` ((CAST(`profile`->'$.tags[*]' AS CHAR(20) ARRAY)));
```

使用场景：

```sql
-- MEMBER OF：判断元素是否属于数组
SELECT * FROM `t_user_profile`
WHERE 'vip' MEMBER OF (`profile`->'$.tags');

-- JSON_OVERLAPS：判断两数组是否有交集
SELECT * FROM `t_user_profile`
WHERE JSON_OVERLAPS(`profile`->'$.tags', '["vip","new"]');

-- JSON_CONTAINS：判断数组是否包含候选值（也可走多值索引）
SELECT * FROM `t_user_profile`
WHERE JSON_CONTAINS(`profile`->'$.tags', '"vip"');
```

三个相关函数：

| 函数 | 语义 |
|------|------|
| `value MEMBER OF (json_array)` | value 是否是 json_array 元素 |
| `JSON_OVERLAPS(a, b)` | 两个 JSON 是否有共同元素 |
| `JSON_CONTAINS(target, candidate)` | target 是否包含 candidate |

### 8.3 方案三：函数索引（Functional Index，8.0.13+）

直接对 JSON 路径表达式建函数索引，无需额外生成列：

```sql
ALTER TABLE `t_user_profile`
    ADD KEY `idx_func_age` ((CAST(`profile`->'$.age' AS UNSIGNED))),
    ADD KEY `idx_func_city` ((CAST(`profile`->>'$.address.city' AS CHAR(64))));
```

查询时使用**完全一致的表达式**即可命中：

```sql
SELECT * FROM `t_user_profile`
WHERE CAST(`profile`->'$.age' AS UNSIGNED) > 25;
```

> **优劣对比**：
> - 生成列：直观、可复用、有列可查；缺点是占空间
> - 函数索引：不占列、灵活；缺点是表达式必须严格一致
> - 多值索引：唯一支持数组元素查询；缺点是仅 8.0.17+ 且语法较特殊

---

## 九、性能考量与最佳实践

### 9.1 存储与读取成本

- JSON 列以二进制格式存储（BLOB 变体），读取时无需解析文本，但**字段内子元素检索仍需遍历**
- 单个 JSON 文档大小上限默认为 1GB，但**强烈建议单文档 < 1MB**，否则读写性能显著下降
- 局部更新（`JSON_SET` 等）会尝试就地更新，若空间不足则整体重写并产生碎片

### 9.2 查询性能要点

1. **避免在 WHERE 中对 JSON 函数包裹运算**：`WHERE JSON_EXTRACT(...,'$.age') + 1 > 26` 索引失效，应改为 `WHERE JSON_EXTRACT(...,'$.age') > 25`
2. **优先用生成列 + 索引**：对高频过滤字段必须建索引
3. **避免 `SELECT *` 拉取大 JSON**：仅取必要路径
4. **避免对 JSON 列做 `ORDER BY` / `GROUP BY`**：除非已建索引，否则需全表扫描并解析
5. **`JSON_TABLE` 展开大数组会膨胀行数**：注意 LIMIT 与过滤条件

### 9.3 写入性能要点

- 局部更新（`JSON_SET`）虽然只改局部，但仍可能触发整行重写
- 高频写入的字段不要塞进 JSON，应单独建列
- 大 JSON 文档频繁更新会带来 binlog 膨胀

### 9.4 EXPLAIN 验证

```sql
EXPLAIN SELECT * FROM `t_user_profile` WHERE `profile`->'$.age' > 25;
-- 检查 type、key、rows，确认是否命中 idx_age
```

关注 `Extra` 字段：
- 出现 `Using index` → 命中索引覆盖
- 出现 `Using where` → 在服务器层过滤
- 出现 `Using filesort` / `Using temporary` → 排序或分组未命中索引

---

## 十、常见陷阱与避坑指南

### 10.1 类型陷阱

JSON 内部区分 `number` / `string` / `boolean` / `null`：

```sql
-- 数字与字符串不相等
SELECT JSON_EXTRACT('{"a":1}', '$.a') = JSON_EXTRACT('{"a":"1"}', '$.a');  -- 0

-- 布尔与数字不相等
SELECT JSON_EXTRACT('{"a":true}', '$.a') = 1;  -- 0

-- null 与 SQL NULL 不同
SELECT JSON_EXTRACT('{"a":null}', '$.a') IS NULL;  -- 0（JSON null 不是 SQL NULL）
SELECT JSON_EXTRACT('{"a":null}', '$.a') = CAST('null' AS JSON);  -- 1
```

### 10.2 字符串引号陷阱

`->` 返回带引号的 JSON 字符串，`->>` 返回去引号字符串，两者不等价：

```sql
SET @j = '{"name":"张三"}';
SELECT @j->'$.name';    -- "张三"（带引号）
SELECT @j->>'$.name';   -- 张三（无引号）

-- 在 WHERE 中比较字符串，必须用 ->> 或 JSON_UNQUOTE
SELECT * FROM t WHERE `profile`->'$.name' = '张三';      -- 错误，匹配不上
SELECT * FROM t WHERE `profile`->>'$.name' = '张三';     -- 正确
SELECT * FROM t WHERE `profile`->'$.name' = '"张三"';    -- 正确但写法丑
```

### 10.3 大小写敏感性

JSON 路径中的 key **区分大小写**，且与列排序规则无关：

```sql
SELECT JSON_EXTRACT('{"Name":"张三"}', '$.name');  -- NULL
SELECT JSON_EXTRACT('{"Name":"张三"}', '$.Name');  -- "张三"
```

### 10.4 NULL 处理

- JSON 列本身可为 `NULL`（SQL NULL）
- JSON 文档内的 `null` 是 JSON null，与 SQL NULL 不同
- 对 NULL 列调用 `JSON_EXTRACT` 返回 SQL NULL

```sql
SELECT JSON_EXTRACT(NULL, '$.a');               -- NULL
SELECT JSON_EXTRACT('{"a":null}', '$.a');       -- null（JSON null）
SELECT JSON_EXTRACT('{"a":null}', '$.a') IS NULL;  -- 0
```

### 10.5 索引命中条件

生成列 / 函数索引要被命中，**查询表达式必须与索引定义表达式严格一致**：

```sql
-- 假设 idx_age 是 (profile->'$.age') 的生成列
SELECT * FROM t WHERE `profile`->'$.age' > 25;        -- ✅ 命中
SELECT * FROM t WHERE JSON_EXTRACT(`profile`, '$.age') > 25;  -- ✅ 命中（等价）
SELECT * FROM t WHERE CAST(`profile`->'$.age' AS SIGNED) > 25; -- ❌ 不命中（多了一层 CAST）
SELECT * FROM t WHERE `profile`->>'$.age' > '25';      -- ❌ 不命中（用了 ->> 且类型不匹配）
```

### 10.6 数组索引越界

```sql
SELECT JSON_EXTRACT('{"a":[1,2,3]}', '$.a[5]');  -- NULL，不报错
```

### 10.7 局部更新不触发触发器

JSON 列的局部更新（`JSON_SET`）在 binlog 中记录的是**整列新值**，binlog 大小不会因"局部"而减小。基于行复制的下游会收到完整新 JSON。

### 10.8 不要把 JSON 当字符串处理

```sql
-- 错误：用 LIKE 在 JSON 文本里搜索
SELECT * FROM t WHERE `profile` LIKE '%张三%';
-- 1) 性能差（全表扫描 + 解析）
-- 2) 可能匹配到 key 名而非 value
-- 3) 无法利用索引

-- 正确：用 JSON_SEARCH 或抽取后比较
SELECT * FROM t WHERE JSON_SEARCH(`profile`->'$.tags', 'one', '张三') IS NOT NULL;
```

---

## 十一、完整实战示例

### 11.1 场景描述

电商商品表，不同品类的属性差异大，用 JSON 存储扩展属性。

### 11.2 建表与索引

```sql
CREATE TABLE `t_product` (
    `id`          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '主键',
    `product_no`  VARCHAR(32)     NOT NULL COMMENT '商品编号',
    `name`        VARCHAR(128)    NOT NULL COMMENT '商品名称',
    `category`    VARCHAR(32)     NOT NULL COMMENT '品类',
    `price`       DECIMAL(10,2)   NOT NULL COMMENT '售价',
    `attrs`       JSON            NOT NULL COMMENT '扩展属性',
    `create_time` DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `update_time` DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    `is_deleted`  TINYINT UNSIGNED NOT NULL DEFAULT 0,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_product_no` (`product_no`),
    KEY `idx_category` (`category`),
    -- 抽取常用查询字段建生成列索引
    KEY `idx_brand` ((CAST(`attrs`->>'$.brand' AS CHAR(32)))),
    KEY `idx_color` ((CAST(`attrs`->>'$.color' AS CHAR(16))))
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='商品表';
```

### 11.3 插入数据

```sql
INSERT INTO `t_product` (`product_no`, `name`, `category`, `price`, `attrs`) VALUES
('P001', 'iPhone 15', 'phone', 6999.00,
    JSON_OBJECT('brand', 'Apple', 'color', '黑色', 'storage', '256G', 'tags', JSON_ARRAY('5G','旗舰'))),
('P002', '小米14', 'phone', 3999.00,
    JSON_OBJECT('brand', 'Xiaomi', 'color', '白色', 'storage', '128G', 'tags', JSON_ARRAY('5G','性价比'))),
('P003', 'MacBook Pro', 'laptop', 14999.00,
    JSON_OBJECT('brand', 'Apple', 'cpu', 'M3', 'ram', '16GB', 'tags', JSON_ARRAY('办公','旗舰'))),
('P004', 'ThinkPad X1', 'laptop', 11999.00,
    JSON_OBJECT('brand', 'Lenovo', 'cpu', 'i7', 'ram', '32GB', 'tags', JSON_ARRAY('办公','商务')));
```

### 11.4 常用查询

```sql
-- 1. 查所有 Apple 品牌商品（命中 idx_brand 索引）
SELECT `product_no`, `name`, `price`
FROM `t_product`
WHERE `attrs`->>'$.brand' = 'Apple';

-- 2. 按颜色筛选
SELECT `product_no`, `name`
FROM `t_product`
WHERE `attrs`->>'$.color' = '黑色';

-- 3. 查标签包含 "5G" 的商品（命中多值索引）
SELECT `product_no`, `name`
FROM `t_product`
WHERE '5G' MEMBER OF (`attrs`->'$.tags');

-- 4. 查标签同时包含 "5G" 和 "旗舰" 的商品
SELECT `product_no`, `name`
FROM `t_product`
WHERE JSON_CONTAINS(`attrs`->'$.tags', '["5G","旗舰"]');

-- 5. 按品牌分组统计
SELECT `attrs`->>'$.brand' AS brand, COUNT(*) AS cnt
FROM `t_product`
WHERE `category` = 'phone'
GROUP BY `attrs`->>'$.brand';

-- 6. 把所有手机商品的标签聚合
SELECT JSON_ARRAYAGG(`attrs`->'$.tags[*]') AS all_tags
FROM `t_product`
WHERE `category` = 'phone';
```

### 11.5 更新属性

```sql
-- 给 P001 新增 "重量" 字段，修改颜色
UPDATE `t_product`
SET `attrs` = JSON_SET(`attrs`, '$.weight', '174g', '$.color', '银色')
WHERE `product_no` = 'P001';

-- 给 P002 的 tags 追加一个 "热销" 标签
UPDATE `t_product`
SET `attrs` = JSON_ARRAY_APPEND(`attrs`, '$.tags', '热销')
WHERE `product_no` = 'P002';

-- 删除 P003 的 cpu 字段
UPDATE `t_product`
SET `attrs` = JSON_REMOVE(`attrs`, '$.cpu')
WHERE `product_no` = 'P003';
```

### 11.6 展开属性为行

```sql
-- 把每个商品的所有属性展开为 (product_no, attr_key, attr_value) 三列
SELECT p.`product_no`, jt.`key`, jt.`value`
FROM `t_product` p,
JSON_TABLE(
    JSON_KEYS(p.`attrs`),
    '$[*]' COLUMNS (`key` VARCHAR(64) PATH '$')
) AS k,
JSON_TABLE(
    p.`attrs`,
    '$' COLUMNS (
        `value` VARCHAR(255) PATH CONCAT('$.', k.`key`)  -- 注：实际需用动态 SQL 或固定字段
    )
) AS jt
WHERE p.`is_deleted` = 0;
```

> 注：`JSON_TABLE` 路径不能直接引用其他表的列做动态拼接。展开所有键值对的实用写法是用存储过程或在应用层处理。下面是一个更实用的固定字段展开示例：

```sql
SELECT
    p.`product_no`,
    p.`name`,
    p.`attrs`->>'$.brand'   AS brand,
    p.`attrs`->>'$.color'   AS color,
    p.`attrs`->>'$.storage' AS storage,
    p.`attrs`->>'$.cpu'     AS cpu,
    p.`attrs`->>'$.ram'     AS ram
FROM `t_product` p
WHERE p.`is_deleted` = 0;
```

---

## 十二、版本特性对照

| 特性 | 5.7 | 8.0 | 8.0.13+ | 8.0.17+ | 8.0.21+ |
|------|-----|-----|---------|---------|---------|
| `JSON` 类型 | ✅ | ✅ | ✅ | ✅ | ✅ |
| `JSON_TABLE` 表函数 | ❌ | ✅ | ✅ | ✅ | ✅ |
| `->` / `->>` 操作符 | 部分 | ✅ | ✅ | ✅ | ✅ |
| 函数索引 | ❌ | ❌ | ✅ | ✅ | ✅ |
| 多值索引（Multi-Valued Index） | ❌ | ❌ | ❌ | ✅ | ✅ |
| `MEMBER OF` / `JSON_OVERLAPS` | ❌ | ❌ | ❌ | ✅ | ✅ |
| `JSON_VALUE` 带 RETURNING | ❌ | ❌ | ❌ | ❌ | ✅ |
| JSON Schema 校验 | ❌ | ❌ | ❌ | ✅（8.0.17） | ✅ |
| JSON 列默认值 | ❌ | ❌ | ✅ | ✅ | ✅ |

---

## 十三、总结

### 13.1 何时该用 JSON

- 字段结构动态、可变，且更新频率低
- 配置、属性、扩展信息类数据
- 与外部 JSON 数据天然契合（接口返回、日志）

### 13.2 何时该拆为正式列

- 字段需要参与高频 `WHERE` / `JOIN` / `GROUP BY` / `ORDER BY`
- 字段含义固定，且对全体行都有意义
- 字段需要强类型约束与外键关系

### 13.3 关键要点速查

1. **JSON 是二进制存储的原生类型，不是字符串**
2. **路径表达式是核心**：`$.key` / `$[i]` / `$[*]` / `$.**`
3. **`->` 返回带类型 JSON，`->>` 返回去引号字符串**
4. **JSON 列不能直接建索引**，用生成列 / 函数索引 / 多值索引三种方案
5. **局部更新仍可能整行重写**，高频更新字段别塞 JSON
6. **类型严格**：JSON number 与 string 不相等，JSON null ≠ SQL NULL
7. **`JSON_TABLE` 是连接 JSON 与关系世界的桥梁**
8. **单文档建议 < 1MB**，避免读写性能下降

### 13.4 参考文档

- [MySQL 8.0 官方文档 - JSON Functions](https://dev.mysql.com/doc/refman/8.0/en/json-functions.html)
- [MySQL 8.0 官方文档 - The JSON Data Type](https://dev.mysql.com/doc/refman/8.0/en/data-type-json.html)
- [MySQL 8.0 官方文档 - Multi-Valued Indexes](https://dev.mysql.com/doc/refman/8.0/en/create-index.html#create-index-multi-valued)
- [RFC 7396 - JSON Merge Patch](https://datatracker.ietf.org/doc/html/rfc7396)

---

<p align="center">
    <a href="https://mp.weixin.qq.com/s/QRwXxoWXl99Dm8bycT3UqQ" target="_blank">
       <img src="https://img.shields.io/badge/微信公众号-访问地址-brightgreen?logo=wechat">
    </a>
    <a href="https://juejin.cn/spost/7655992822656630834" target="_blank">
       <img src="https://img.shields.io/badge/掘金-访问地址-blue?logo=juejin">
    </a>
    <a href="https://zhuanlan.zhihu.com/p/2054692930275873707" target="_blank">
       <img src="https://img.shields.io/badge/知乎-访问地址-blue?logo=zhihu">
    </a>
</p>