以下为通过创建远程访问用户方式授权远程访问

# 赋予某个用户权限

格式：`grant 权限 on 数据库对象 to 用户@IP(或者相应正则)`
注：可以赋予 `select,delete,update,insert,index` 等权限精确到某一个数据库某一个表。

```sql
-- 赋予该用户所有数据库所有表的全部权限（*.*表示所有表的权限），% 表示所有IP地址
GRANT ALL PRIVILEGES ON *.* TO '用户名'@'%' IDENTIFIED BY '密码' WITH GRANT OPTION;
```

示例

```sql
-- 授予所有数据库的所有权限，允许所有IP访问（示例用户：remote_user，密码：password）
GRANT ALL PRIVILEGES ON *.* TO 'remote_user'@'%' IDENTIFIED BY 'password' WITH GRANT OPTION;
```

# 刷新权限

```sql
-- 刷新权限
FLUSH PRIVILEGES;
```

# 查看权限

```sql
SELECT user,host FROM mysql.user; 
```

