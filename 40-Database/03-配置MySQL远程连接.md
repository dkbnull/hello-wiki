以下为通过创建远程访问用户方式授权远程访问

# 创建用户并赋予权限

格式：`grant 权限 on 数据库对象 to 用户@IP(或者相应正则)`
注：可以赋予 `select,delete,update,insert,index` 等权限精确到某一个数据库某一个表。

> **注意：MySQL 8.0 及以上版本不再支持在 `GRANT` 语句中直接使用 `IDENTIFIED BY` 创建用户，需要先使用 `CREATE USER` 创建用户，再使用 `GRANT` 授权。**

```sql
-- 创建远程访问用户（MySQL 8.0+ 推荐方式）
CREATE USER '用户名'@'%' IDENTIFIED BY '密码';

-- 赋予该用户所有数据库所有表的全部权限（*.*表示所有表的权限），% 表示所有IP地址
GRANT ALL PRIVILEGES ON *.* TO '用户名'@'%' WITH GRANT OPTION;
```

示例

```sql
-- 创建远程访问用户（示例用户：remote_user，密码：password）
CREATE USER 'remote_user'@'%' IDENTIFIED BY 'password';

-- 授予所有数据库的所有权限，允许所有IP访问
GRANT ALL PRIVILEGES ON *.* TO 'remote_user'@'%' WITH GRANT OPTION;
```

> MySQL 5.7 及以下版本也可以使用一行语句完成创建用户并授权：
> ```sql
> GRANT ALL PRIVILEGES ON *.* TO 'remote_user'@'%' IDENTIFIED BY 'password' WITH GRANT OPTION;
> ```

# 刷新权限

```sql
-- 刷新权限
FLUSH PRIVILEGES;
```

# 查看权限

```sql
SELECT user,host FROM mysql.user; 
```

