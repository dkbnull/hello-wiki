# 字段类型差异

| **功能**         | **MySQL**                                               | **SQL Server**                                               |
| :--------------- | :------------------------------------------------------ | :----------------------------------------------------------- |
| **整数类型**     | `TINYINT`, `SMALLINT`, `MEDIUMINT`, `INT`, `BIGINT`     | `TINYINT`, `SMALLINT`, `INT`, `BIGINT`（无 `MEDIUMINT`）     |
| **自增字段**     | `AUTO_INCREMENT`                                        | `IDENTITY(1,1)`                                              |
| **字符串类型**   | `VARCHAR(n)`（最大 65535 字节）                         | `VARCHAR(n)`（最大 8000 字节）<br /> `VARCHAR(MAX)`（最大 2GB） |
| **日期时间类型** | `DATETIME`（精度到秒）<br />`TIMESTAMP`（自动时区转换） | `DATETIME`（精度到 3ms） <br />`DATETIME2`（更高精度，如 `DATETIME2(7)`） |
| **二进制大对象** | `BLOB`                                                  | `VARBINARY(MAX)`                                             |
| **布尔类型**     | `BOOLEAN`（实际是 `TINYINT(1)`）                        | 无原生布尔类型，通常用 `BIT`（0/1）                          |

------

# 常用函数差异

## 字符串函数

| **功能**       | **MySQL**                       | **SQL Server**                                               |
| :------------- | :------------------------------ | :----------------------------------------------------------- |
| **字符串拼接** | `CONCAT(str1, str2, ...)`       | `+` 运算符（需处理 NULL） `CONCAT(str1, str2)`（SQL Server 2012+） |
| **子字符串**   | `SUBSTRING(str, start, length)` | `SUBSTRING(str, start, length)`                              |
| **字符串替换** | `REPLACE(str, old, new)`        | `REPLACE(str, old, new)`                                     |
| **大小写转换** | `LOWER(str)`, `UPPER(str)`      | `LOWER(str)`, `UPPER(str)`                                   |
| **去空格**     | `TRIM(str)`                     | `LTRIM(RTRIM(str))`                                          |

## 日期函数

| **功能**         | **MySQL**                                     | **SQL Server**                               |
| :--------------- | :-------------------------------------------- | :------------------------------------------- |
| **当前日期时间** | `NOW()`                                       | `GETDATE()`                                  |
| **日期格式化**   | `DATE_FORMAT(date, format)` （如 `%Y-%m-%d`） | `FORMAT(date, format)` （如 `'yyyy-MM-dd'`） |
| **日期加减**     | `DATE_ADD(date, INTERVAL n DAY)`              | `DATEADD(DAY, n, date)`                      |
| **日期差**       | `DATEDIFF(unit, date1, date2)`                | `DATEDIFF(unit, date1, date2)`               |

## 条件函数

| **功能**       | **MySQL**                            | **SQL Server**                                         |
| :------------- | :----------------------------------- | :----------------------------------------------------- |
| **条件判断**   | `IF(condition, true_val, false_val)` | `CASE WHEN condition THEN true_val ELSE false_val END` |
| **空值处理**   | `IFNULL(expr, replacement)`          | `ISNULL(expr, replacement)`                            |
| **多条件判断** | `CASE ... WHEN ...`                  | `CASE ... WHEN ...`（语法相同）                        |

## 聚合函数

| **功能**       | **MySQL**           | **SQL Server**                                   |
| :------------- | :------------------ | :----------------------------------------------- |
| **字符串聚合** | `GROUP_CONCAT(...)` | `STRING_AGG(..., separator)`（SQL Server 2017+） |
| **分位数计算** | 无内置函数          | `PERCENTILE_CONT()` / `PERCENTILE_DISC()`        |

------

# 其他关键差异

## 分页查询

- **MySQL**：

  ```
SELECT * FROM table LIMIT 10 OFFSET 20;
  ```

- **SQL Server**：

  ```
SELECT * FROM table 
  ORDER BY column 
OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY;
  ```

## 取第一条数据

- **MySQL**：

  ```
  SELECT * FROM table LIMIT 1;
  ```

- **SQL Server**：

  ```
  SELECT top 1 * FROM table;
  ```

## 存储过程

- **MySQL**：使用 `DELIMITER` 定义存储过程。
- **SQL Server**：使用 `CREATE PROCEDURE`，无需修改分隔符。

## 系统函数

| **功能**           | **MySQL**          | **SQL Server**     |
| :----------------- | :----------------- | :----------------- |
| **获取最后插入ID** | `LAST_INSERT_ID()` | `SCOPE_IDENTITY()` |
| **当前用户**       | `CURRENT_USER()`   | `SUSER_NAME()`     |
| **数据库名称**     | `DATABASE()`       | `DB_NAME()`        |

------

# 隐式行为差异

| **场景**         | **MySQL**                           | **SQL Server**                            |
| :--------------- | :---------------------------------- | :---------------------------------------- |
| **隐式事务提交** | 默认自动提交（`autocommit=1`）      | 默认需显式提交（`BEGIN TRAN` + `COMMIT`） |
| **字符串比较**   | 不区分大小写（依赖排序规则）        | 区分大小写（依赖排序规则）                |
| **零日期处理**   | 允许 `0000-00-00`（严格模式可禁用） | 不允许 `0000-00-00`                       |

------

# 迁移注意事项

**数据类型转换：**

- 将 MySQL 的 `TEXT` 类型转换为 SQL Server 的 `VARCHAR(MAX)`。
- 将 `AUTO_INCREMENT` 替换为 `IDENTITY(1,1)`。

**函数替换：**

- MySQL 的 `IFNULL()` 改为 SQL Server 的 `ISNULL()`。
- 将 `LIMIT` 分页逻辑改为 `OFFSET FETCH`。

**示例：**

- **MySQL**：

  ```
CREATE TABLE users (
      id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50),
      created_at DATETIME DEFAULT NOW()
  );
  ```
  
- **SQL Server**：

  ```
CREATE TABLE users (
      id INT IDENTITY(1,1) PRIMARY KEY,
    name VARCHAR(50),
      created_at DATETIME DEFAULT GETDATE()
  );
  ```



**官方文档：**

[MySQL 数据类型](https://dev.mysql.com/doc/refman/8.0/en/data-types.html)

[SQL Server 数据类型](https://docs.microsoft.com/en-us/sql/t-sql/data-types/data-types-transact-sql)

