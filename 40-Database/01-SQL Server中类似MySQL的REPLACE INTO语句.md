SQL Server中，没有MySQL中的REPLACE INTO语句，可以通过以下两种方法实现

# 使用 `MERGE` 语句

`MERGE` 语句能根据条件执行插入或更新操作，语法如下：

```sql
MERGE INTO 表名 AS target
USING (VALUES (参数列表)) AS source (字段列表)
ON target.匹配字段 = source.匹配字段
WHEN MATCHED THEN
    UPDATE SET 字段 = source.字段, ...
WHEN NOT MATCHED THEN
    INSERT (字段列表) VALUES (source.字段列表);
```

**示例**

```sql
MERGE INTO mch_info AS target
USING (VALUES (@mchId, @mchName)) AS source (mchId, mchName)
ON target.mchId = source.mchId
WHEN MATCHED THEN
    UPDATE SET mchName = source.mchName
WHEN NOT MATCHED THEN
    INSERT (mchId, mchName) 
    VALUES (source.mchId, source.mchName);
```

# 使用 `UPDATE` + `INSERT` 组合

即通过不存在时插入数据，存在时更新数据的方法来实现（insert when/where not exists），语法如下：

```sql
INSERT INTO 表名 (字段列表)
    SELECT * FROM ( VALUES
        (参数列表)
    ) AS source (字段列表)
    WHERE NOT EXISTS (
    SELECT * FROM 表名 AS target WHERE source.匹配字段= target.匹配字段
    );
    
UPDATE 表名
	SET target.字段 = source.字段
    FROM 表名 AS target,
    (SELECT * FROM ( VALUES 
		(参数列表)
    ) AS source_temp (字段列表)) AS source
    WHERE target.匹配字段 = source.匹配字段
```

**示例**

~~~sql
INSERT INTO table (id, name)
    SELECT * FROM ( VALUES
        (1,'name1'),
        (2,'name2')
    ) AS table_temp (id, name)
    WHERE NOT EXISTS (
    SELECT * FROM table WHERE table.id= table_temp.id
    );
    
UPDATE table 
	SET table.name = table_temp.name
    FROM table,
    (SELECT * FROM ( VALUES
        (1,'name1'),
        (2,'name2')
    ) AS temp (id, name)) AS table_temp
    WHERE table.id = table_temp.id
~~~



如果只是单条数据，即使使用业务代码来判断需要执行插入还是更新操作都不会太复杂，但是对于批量插入/更新数据时，再使用业务代码来判断将会大幅增加耗时，参照使用 `UPDATE` + `INSERT` 组合，可以使用如下方式实现：

~~~xml
<insert id="insertListMssql">
    INSERT INTO mch_info(<include refid="Base_Column_List"/>)
    SELECT * FROM ( VALUES
    <foreach collection="list" index="i" separator="," item="item">
        (
        #{item.mchId},
        #{item.mchName},
        )
    </foreach>
    ) AS mch_info_temp (<include refid="Base_Column_List"/>)
    WHERE NOT EXISTS (SELECT 1 FROM mch_info WHERE mch_info.mchId = mch_info_temp.mchId)
</insert>

<insert id="updateListMssql">
    UPDATE mch_info SET
    mch_info.mchName = mch_info_temp.mchName
    FROM mch_info ,(SELECT * FROM ( VALUES
    <foreach collection="list" index="i" separator="," item="item">
        (
        #{item.mchId},
        #{item.mchName}
        )
    </foreach>
    ) AS temp (<include refid="Base_Column_List"/>)) AS mch_info_temp
    WHERE mch_info.mchId = mch_info_temp.mchId
</insert>
~~~