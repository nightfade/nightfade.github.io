layout: post
date: 2015-08-02 18:55
tags: 
- database
- SQLite
title: SQLite中的常用SQL语法、锁机制和WAL
---

本文主要参考《SQLite权威指南》。

# 创建表
创建表的通用形式如下：
```sql
CREATE [TEMP|TEMPORARY] TABLE table_name (column_definitions [, constraints]);
```
- `TEMP/TEMPORARY`关键字表示声明的表是临时表，一旦当前数据库断开链接，就自动销毁。
- `table_name`是表名。
- `column_definitions`是由逗号分隔的字段列表组成。

SQLite有5中本地类型：`INTEGER`，`REAL`，`TEXT`，`BLOB`，`NULL`。

下面是一个例子：
```sql
CREATE TABLE contacts (id INTEGER PRIMARY KEY,
                       name TEXT NOT NULL COLLATE NOCASE,
                       phone TEXT NOT NULL DEFAULT 'UNKNOWN',
                       UNIQUE (name, phone));
```
# 修改表
修改表的一般格式为：
```sql
ALTER TABLE table_name { RENAME TO name | ADD COLUMN column def };
```
例如：
```sql
ALTER TABLE contacts
ADD COLUMN email TEXT NOT NULL DEFAULT '' COLLATE NOCASE;
```

# 数据库查询
`select`命令的通用形式如下：
```sql
SELECT [DISTINCT] heading
FROM tables
WHERE predicate
GROUP BY columns
HAVING predicate
ORDER BY columns [ASC|DESC]
LIMIT count
OFFSET offset;
```
理解`select`命令的最好方法是将其当成处理关系的`管道`。

## 过滤
```sql
SELECT * FROM dogs WHERE color='purple' AND grin='toothy';
```
数据库会获取表`dogs`中的所有行，然后用`where`子句形成预测逻辑，如果为真，该行将会包含在结果集里。
## 限定和排序
可以用`limit`和`offset`关键字限定结果集的大小和范围。例如：
```sql
SELECT * FROM foods WHERE name LIKE 'B%' ORDER BY type_id DESC, name LIMIT 1 OFFSET 2;
```
# 函数（Function）和聚合（Aggregate）
SQLite提供多种内置的函数和聚合。
函数的种类包括：
- 数学函数，例如计算绝对值的`ABS()`。
- 字符串格式函数，例如将字符串转化为大写或小写的`UPPER()`和`LOWER()`。

函数可以是任意表达式的一部分，可以用在`WHERE`子句中。例如：
```sql
SELECT id, UPPER(name), LENGTH(name) FROM foods WHERE LENGTH(name) < 5 LIMIT 5;
```
聚合是一类特殊的函数，它从一组记录中计算聚合值。标准的聚合函数包括：`SUM()`，`AVG()`，`COUNT()`，`MIN()`，`MAX()`。例如：
```sql
SELECT COUNT(*) FROM foods WHERE type_id=1;
```
聚合不仅可以聚合字段，也可以聚合任意表达式，标扩函数。例如：
```sql
SELECT AVG(LENGTH(name)) FROM foods;
```
# 分组（Grouping）
聚合不只是能够计算整个结果集的聚合至，还可以把结果集分成多个组，然后计算每个组的聚合值，方法是使用`GROUP BY`子句，例如：
```sql
SELECT type_id, COUNT(*) FROM foods GROUP BY type_id;
```
从操作上讲，`GROUP BY`介于`WHERE`和`SELECT`子句中间。`GROUP BY`接收`WHERE`的输出，并将其分割成共享某个字段（或多个字段）上同等值的小组，这些组再传递给`SELECT`子句。
使用`GROUP BY`时，`SELECT`子句*对每组单独应用聚合，而不是对整个结果进行聚合*。因此，聚合对魅族生成一个值，并将这些组的行作为单行。

`HAVING`是一个可以应用到`GROUP BY`的断言。它从`GROUP BY`中过滤组的方式与`WHERE`子句从`FROM`中过滤行的方式相同。唯一不同的是，`WHERE`子句的预测是针对单个行的，而`HAVING`的断言是针对聚合值的。例如：
```sql
SELECT type_id, COUNT(*) from foods
GROUP BY type_id HAVING COUNT(*) < 20;
```
`GROUP BY`接收`WHERE`子句的约束，将结果行分成共享某值的组。`HAVING`对每组应用过滤，通过过滤的组传递给`SELECT`子句来做聚合和映射。

# 去掉重复

`DISTINCT`处理`SELECT`的结果并过滤掉其中重复的行。例如：
```sql
SELECT DISTINCT type_id FROM foods;
```
该语句的工作流程：`WHERE`子句返回表`foods`的所有行，`SELECT`子句取出其中的`type_id`字段，最终`DISTINCT`删除重复行。

# 多表链接
连接(JOIN)是多表关系数据工作的关键，它是`SELECT`命令的第一个操作，连接操作的结果作为输入，供`SELECT`语句的其他部分过滤处理。

例如：
```sql
SELECT foods.name, food_types.name
FROM foods, food_types
WHERE foods.type_id=food_types.id
LIMIt 10;
```
要实现连接，数据库需要找出匹配的行。对于第一个表中的每一行，数据库都要查询的第二个表的所有行，寻找那些链接字段具有相同值的行，然后将他们包含到输入关系中。

SQLite支持六种不同类型的连接，其中一种称为*内连接*的连接是最普遍的。

## 内连接

内连接就是通过表中的两个字段进行连接。内连接找出两边集合那些包含相似值的行，然后结合两者形成结果集。
内连接只返回满足给定字段关系的行，也称为连接条件。例如：
```sql
SELECT *
FROM foods INNER JOIN food_types ON foods.id = food_types.id;
```

## 交叉连接
如果没有连接条件，`SELECT`会产生一种更基础的连接，称为交叉连接、笛卡尔积或者交叉乘积。例如：
```sql
SELECT * FROM foods, food_types;
```
`FROM`在缺乏其他条件时产生交叉连接，`foods`中的每一行都与`food_types`的所有行组合在一起。

## 外连接
内连接是根据给定关系选择表中的行。外连接选择内连接的所有行外加一些关系之外的行。
三种外连接类型分别是：左外连接、右外连接和全外连接。例如：
```sql
SELECT *
FROM foods LEFT OUTER JOIN foods_episodes ON foods.id = foods_episodes.food_id;
```
左外连接试图将`foods`中的所有行与`foods_episodes`的所有行进行行连接关系的匹配，所有匹配的行都包含在结果集中。但是，如果我们在`foods`表中注册一些食品，这些食品没有在`foods_episodes`中出现，`foods`表中没有匹配`food_episodes`的剩余行仍然会出现在结果集中，foods_episodes没有提供相应的行，它会以null补充。

右外连接的工作方式类似，不管是否匹配，右表的行都包含在结果集中。

全外连接是左外连接和右外连接的结合。它包含所有的匹配航，然后是右边和左边表的不匹配行。

目前SQLite不支持右外连接和全外连接，但是右外连接可以用左外连接代替，全外连接可以通过使用复合查询执行。

## 自然连接
自然连接是内连接的另一种形式，自然连接通过表中共有的字段名称将两个表连接起来，因此，使用自然连接时，不用添加连接条件就可以获得内连接的结果。
如果表的设计一直在改变，最好清晰的定义查询中的连接条件，不要使用自然连接。

## 语法偏好
语法上讲，可以通过各种方式制定连接，`foods`和`food_types`的内连接可以通过`WHERE`子句隐含实现：
```sql
SELECT * FROM foods, food_types WHERE foods.id = food_types.food_id;
```
这种隐式形式虽然简洁，但是是一种应该避免的果实的语法形式。SQL中正确的表达连接的方法是使用连接关键字：
```sql
SELECT heading FROM left_table JOIN_TYPE right_table ON join_condition;
```

# 名称和别名
举例说明：
```sql
SELECT f.name, t.name
FROM foods f, food_types t
WHERE f.type_id = t.id
LIMIT 10;
```

```sql
SELECT f.name AS food, e1.name, e2.season, e2.name, e2.season
FROM episodes e1, foods_episodes fe1, foods f,
     episodes e2, foods_episodes fe2
WHERE e1.id = fe1.episode_id AND e1.season = 4 AND fe1.food_id = f.id 
      AND fe1.food_id = fe2.food_id
      AND fe2.episode_id = e2.id AND e2.season != e1.season
ORDER BY f.name;
```

# 子查询
子查询最常用的地方是`WHERE`子句，特别是`IN`操作符中：
```sql
SELECT COUNT(*) 
FROM foods
WHERE type_id IN
    (SELECT id 
     FROM food_types
     WHERE name='Bakery' OR name='Cereal');
```
可以用来从其他表向结果集添加额外数据：
```sql
SELECT name,
       (SELECT COUNT(id) FROM foods_episodes WHERE food_id=f.id) AS count
FROM foods f ORDER BY count DESC LIMIT 10;
```
子查询中可以使用`ORDER BY`子句：
```sql
SELECT * FROM foods f
ORDER BY (SELECT COUNT(type_id) FROM foods WHERE type_id=f.type_id) DESC;
```
有时可能想与其他结果进行连接：
```sql
SELECT f.name, types.name FROM foods f
INNER JOIN (SELECT * FROM food_types WHERE id=6) types ON f.type_id=types.id;
```
# 复合查询
复合查询与子查询相反，它是使用三种特殊的关系操作符处理多个查询的结果：`UNION`、`INTERSECT`、`EXCEPT`。
复合查询操作需要如下一些条件：
- 涉及的关系的字段数目必须相同。
- 只能有一个`ORDER BY`子句，并且处在复合查询的最末尾，对联合结果进行排序。

`UNION`联合两个`SELECT`语句的结果，默认情况下，`UNION`会消除重复数据，如果想在结果中保留重复数据，可以使用`UNION ALL`。
例如：
```sql
SELECT f.*, top_foods.count FROM foods f
INNER JOIN (SELECT food_id, COUNT(food_id) AS count from foods_episodes
            GROUP BY food_id
            ORDER BY COUNT(food_id) DESC LIMIT 1) top_foods
    ON f.id=top_foods.food_id
UNION
select f.*, bottom_foods.count FROM foods f
INNER JOIN (SELECT food_id, count(food_id) AS count FROM foods_episodes
            GROUP BY food_id
            ORDER BY COUNT(food_id) LIMIT 1) bottom_foods
    ON f.id=bottom_foods.food_id
ORDER BY top_foods.count DESC; 
```

`INTERSECT`操作输入两个关系A和B，选择那些既在A也在B的行。例如：
```sql
SELECT f.* FROM foods f
    INNER JOIN
        (SELECT food_id, COUNT(food_id) AS count
         FROM foods_episodes
         GROUP BY food_id
         ORDER BY COUNT(food_id) DESC LIMIT 10) top_foods
    ON f.id=top_foods.food_id
INTERSECT
SELECT f.* FROM foods f
    INNER JOIN foods_spisodes fe ON f.id=fe.food_id
    INNER JOIN episodes e ON fe.episode_id=e.id
    WHERE e.season BETWEEN 3 AND 5
    ORDER BY f.name;
```

`EXCEPT`操作输入两个关系A和B，找出所有在A但不在B的行。例如：
```sql
SELECT f.* FROM foods f
    INNER JOIN
        (SELECT food_id, COUNT(food_id) AS count FROM foods_episodes
         GROUP BY food_id
         ORDER BY COUNT(food_id) DESC LIMIT 10) top_foods
    ON f.id=top_foods.food_id
EXCEPT
SELECT f.* from foods f
    INNER JOIN foods_episodes fe ON f.id=fe.food_id
    INNER JOIN foods_episodes e ON fe.episode_id=e.id
    WHERE e.season BETWEEN 3 AND 5
ORDER BY f.name;
```
# 条件结果
`CASE`表达式允许在`SELECT`语句中处理各种情况。
1. 接受静态值并列出各种情况下`CASE`返回值：
```sql
SELECT name || CASE type_id
                   WHEN 7 THEN ' is a drink'
                   WHEN 8 THEN ' is a fruit'
                   WHEN 9 THEN ' is junkfood'
                   WHEN 13 THEN ' is seafood'
                   ELSE NULL
               END AS description
FROM foods
WHERE description IS NOT NULL
ORDER BY name
LIMIT 10;
```
2. `WHEN`条件中有表达式：
```sql
SELECT name, (SELECT CASE
                  WHEN COUNT(*) > 4 THEN 'VERY HIGH'
                  WHEN COUNT(*) = 4 THEN 'HIGH'
                  WHEN COUNT(*) in (2, 3) THEN 'Moderate'
                  ELSE 'Low'
              END
              FROM foods_episodes
              WHERE food_id=f.id) AS frequency
FROM foods f
WHERE frequency like '%HIGH';
```

# 修改数据
## 插入记录
使用`INSERT`命令向表中插入记录，`INSERT`语句的一般格式为：
```sql
INSERT INTO table_name (column_list) VALUES (value_list);
```
插入一行：
```sql
INSERT INTO foods (name, type_id) VALUES ('Cinnamon Bobka', 1);
```
插入一组行：
```sql
INSERT INTO foods
VALUES (NULL,
        (SELECT id FROM food_types WHERE name='Bakery'),
        'Blackberry Bobka');
```
```sql
INSERT INTO foods
SELECT last_insert_rowid()+1, type_id, name FROM foods
WHERE name='Chocolate Bobka';
```
只要`SELECT`子句的字段数目与要插入的表的字段数目匹配，或者与提供的字段列表匹配，`INSERT`语句就可以正常工作。

## 更新记录
`UPDATE`命令用于更新表中的记录，该命令可以修改一个表中的一行或多行中的一个或多个字段。`UPDATE`语句的一般格式为：
```sql
UPDATE table_name SET update_list WHERE predicate;
```
`update_list`是一个或多个“字段赋值”的列表，字段赋值的格式为`column_name=value`。例如：
```sql
UPDATE foods SET name='CHOCOLATE BOBKA'
WHERE name='Chocolate Bobka';
```

## 删除记录
使用`DELETE`命令可以删除表中的记录，`DELETE`语句的一般格式为：
```sql
DELETE FROM table_name WHERE predicate;
```
例如：
```sql
DELETE FROM foods WHERE name='CHOCOLATE BOBKA';
```

# 数据完整性
数据完整性用于定义和保护表内部或表之间的数据关系。一般有四种完整性：域完整性、实体完整性、引用完整性和用户自定义完整性。
- 域完整性：控制字段内的值。
- 实体完整性：表中的行。
- 引用完整性：表之间的行，也就是外键关系。
- 用户自定义完整性：其他。

## 实体完整性
行必须在某种方式上是唯一的，这就是**主键**的功能。
主键由至少带有`UNIQUE`约束的一个或一组字段组成。
唯一性约束：一个`UNIQUE`约束要求一个或一组字段的所有值互不相同，如果视图插入一个重复值，或者将一个值更新成一个已存在的值，数据库将引发一个约束非法，并终止操作。
主键约束：定义一个表时总要确定一个主键，不管自己有没有定义。这个字段是一个64bit的整形字段，称为`rowid`，还有两个别名`_rowid_`和`oid`。
```sql
CREATE TABLE contacts (
    id INTEGER PRIMARY KEY AUTOINCREMENT
    name TEXT NOT NULL COLLATE nocase,
    phone TEXT NOT NULL DEFAULT 'UNKNOWN',
    UNIQUE(name, phone));
```

## 域完整性
1. 默认值：如果用`INSERT`语句插入记录时没有为该字段指定值，关键字`DEFAULT`将为字段提供一个默认值。
`DEFAULT`还可以接受3中预定义格式的保留字，用于生成日期和时间。`current_time`将生成（HH:MM:SS）的当前时间，`current_date`将生成(YYYY-MM-DD)格式的当前日期，`current_timestamp`将生成一个日期时间的组合(YYYY-MM-DD HH:MM:SS)。
```sql
CREATE TABLE times (id INT,
                    date NOT NULL DEFAULT current_date,
                    time NOT NULL DEFAULT current_time,
                    timestamp NOT NULL DEFAULT current_timestamp);
```
2. `NOT NULL`约束：确保该字段值不为`NULL`。
3. `CHECK`约束：允许自定义表达式来测试要插入或更新的字段值。
```sql
CREATE TABLE contacts (id INTEGER PRIMARY KEY,
                       name TEXT NOT NULL COLLATE NOCASE,
                       phone TEXT NOT NULL DEFAULT 'UNKNOWN',
                       UNIQUE(name, phone),
                       CHECK(LENGTH(name) >= 7));
CREATE TABLE foo (x INTEGER,
                  y INTEGER CHECK (y > x),
                  z INTEGER CHECK (z > ABS(y)));
```
4. 外键约束：确保一个表中的关系值必须从另一个表中引用，且该数据必须在另一个表中实际存在，语法如下：
```sql
CREATE TABLE table_name (column_definitions REFERENCES foreign_table (column_name) on {DELETE|UPDATE} integrity_action [not] deferrable [INITIALLY {DEFERED|IMMEDIATE},]
);
```
举例：
```sql
CREATE TABLE foods (
    id INTEGER PRIMARY KEY,
    type_id INTEGER REFERENCES food_types(id) 
    ON DELETE RESTRICT
    DEFERRABLE INITIALLY DEFERRED,
    name TEXT
)
```

`DELETE RESTRICT`阻止任何破坏完整性的行为。其他的选项有：
- `SET NULL`：如果父值被删除或者不存在了，子值设置为`NULL`。
- `SET DEFAULT`：如果赋值不存在了就设置为默认值。
- `CASCADE`：更新父值时，更新所有匹配的子值。删除父值时，删除所有子值。
- `RESTRICT`：如果更新或删除父值可能出现孤立的子值，就阻止事务。
- `NO ACTION`：不干涉操作，只是在整个语句的结尾报错。

`DEFERRABLE`子句控制定义的约束是立即强制实施还是延迟到整个事务结束时。

5. 排序规则：SQLite内置三种排序规则，默认的是二进制排序规则，使用C函数`memcmp()`逐字节比较文本值。第二种，`NOCASE`是拉丁字母中26个ASCII字符的非大小写敏感排序算法。第三种，`REVERSE`排序规则，和二进制排序规则相反。

# 视图
视图即虚拟表，也称为派生表，因为它们的内容都派生自其他表的查询结果。视图的内容是在使用时动态产生的。
```sql
CREATE VIEW name AS select-stmt;
```
例如：
```sql
CREATE VIEW details AS
SELECT f.name AS fd, ft.name AS tp, e.name AS ep, e.season AS ssn
FROM foods f
INNER JOIN food_types ft ON f.type_id=ft.id
INNER JOIN foods_spisodes fe ON f.id=fe.food_id
INNER JOIN episodes e ON fe.episode_id=e.id;

SELECT fd AS FOOD, ep AS Episode FROM details WHERE ssn=7 AND tp LIKE 'Drinks';
```
视图的内容是动态生成的，每次使用`details`时，基于数据库对当前数据执行相关的SQL语句，产生结果。
使用命令`DROP VIEW`删除视图：
```sql
DROP VIEW name;
```

# 索引
索引是一种用来在某种条件下加速查询的结构，例如如下查询：
```sql
SELECT * FROM foods WHERE name='JujyFruit';
```
当数据库搜索匹配行时，执行这种查询的默认方法是调用顺序扫描。如果表`foods`非常大，使用索引的方法查找数据就更有意义了。
创建索引的命令如下：
```sql
CREATE INDEX [UNIQUE] index_name ON table_name (columns);
```
关键字`UNIQUE`会在索引上添加约束，索引中的所有值必须是唯一的。
例：
```sql
CREATE INDEX UNIQUE foo_index ON foo (a, b);
```
删除索引：
```sql
DROP INDEX index_name;
```

排序规则：索引中的每个字段都有相关的排序规则，例如，要在`foods.name`上创建大小写不敏感的索引，可以使用如下命令：
```sql
CREATE INDEX foods_name_idx ON foods (name COLLATE NOCASE);
```

使用索引
对于下面会在`WHERE`子句中出现的表达式，SQLite将使用单个字段索引：
```sql
column {=|>|>=|<=|<} expression
expression {=|>|>=|<=|<} column
column IN (expression-list)
column IN (subquery)
```

多字段索引有更复杂的条件，它从左到右智能的使用字段。

# 触发器
当具体的表发生特定的数据库事件时，触发器执行对应的SQL命令。创建触发器的一般命令如下：
```sql
CREATE [TEMP|TEMPORARY] TRIGGER name
[BEFORE|AFTER] [INSERT|DELETE|UPDATE|UPDATE OF columns] ON TABLE
action;
```
例如：
```sql
CREATE TEMP TRIGGER foods_update_log UPDATE OF name ON foods
BEGIN
    INSERT INTO log VALUES ('updated foods: new name = ' || new.name);
END;
```
在`UPDATE`触发器中，未更新行可以引用为`old`，已更新行可以引用为`new`。

错误处理：SQLite提供一个特殊的SQL函数`raise()`供触发器调用，该函数允许在触发器内产生错误，`raise()`定义如下：
```python
raise(resolution, error_message);
```
第一个参数是冲突解决策略（`abort`, `fail`, `ignore`, `rollback`等），第二个参数是错误消息。

可更新视图：可利用触发器实现可更新视图。
例如：
```sql
CREATE TRIGGER on_update_foods_view
INSTEAD OF UPDATE ON foods_view
FOR EACH ROW
BEGIN
    UPDATE foods SET name=new.fname WHERE id=new.fid;
    UPDATE food_types SET name=new.tname WHERE id=new.tid;
END;
```

# 事务
事务定义了一组SQL命令的边界，这组命令或者作为一个整体被全部执行，或者都不执行。
事务由3个命令控制：`BEGIN`、`COMMIT`和`ROLLBACK`。
- `BEGIN`开始一个事务，`BEGIN`之后的所有操作都可以取消，如果连接终止前没有发出`COMMIT`，也会被取消。
- `COMMIT`提交事务开始后所执行的所有操作。
- `ROLLBACK`还原`BEGIN`之后的所有操作。

例如：
```sql
BEGIN;
DELETE FROM foods;
ROLLBACK;
SELECT COUNT(*) FROM foods; -- output: 412
```
## 冲突解决
违反约束会导致事务终止，在对数据进行很多修改的过程中，命令终止会简单的将前面所做的修改全部取消。
通过冲突解决可以指定不同的方式处理约束违反的情况。
SQLite提供5种可能的冲突解决方案或策略：
- `REPLACE`：违反唯一性约束时，将造成这种违反的记录删除，以插入或修改的新纪录替代。违反`NOT NULL`约束时，则使用该字段的默认值代替`NULL`，如果没有默认值，则应用`ABORT`策略。
- `IGNORE`：当违反约束发生时，允许命令继续执行，违反约束的行保持不变，命令继续处理其他且不报错。
- `FAIL`：当违反约束发生时，终止命令，但是不回复约束违反之前已经修改的记录，违反约束之后的都不回继续处理了。
- `ABORT`：违反约束发生时，恢复当前命令所有的所有改变并终止命令。这是SQLite中所有操作的默认解决办法。
- `ROLLBACK`：违反约束发生时，执行回滚。当前命令所做的改变和事务中之前命令的改变都回滚。

冲突解决方法既可以在SQL命令中指定，也可以在表盒索引的定义中执行。具体来说，可以在`INSERT`、`UPDATE`、`CREATE TABLE`和`CREATE INDEX`中指定。
在`INSERT`和`UPDATE`中的语法形式如下：
```sql
INSERT OR *RESOLUTION_TYPE* INTO table_name (column_list) VALUES (value_list);
UPDATE OR *RESOLUTION_TYPE* table_name SET (value_list) WHERE predicate;
```
表内定义时，可以为单个字段指定冲突解决方法，例如：
```sql
CREATE TABLE cast (name TEXT UNIQUE ON CONFLICT ROLLBACK);
```

## 数据库锁
SQLite使用锁逐步提升机制，为了写数据库，连接需要逐级获得排它锁。SQLite有5种不同的锁状态：未加锁(`UNLOCKED`)、共享(`SHARED`)、预留(`RESERVED`)、未决(`PENDING`)和排他(`EXCLUSIVE`)。
- 最初的状态是`UNLOCK`状态，此状态下连接还没有访问数据库。
- 为了能从数据库**读**数据，连接必须首先进入`SHARED`状态。多个连接可以同时获得并保持共享锁，只要有一个共享锁没有释放，就不允许任何连接写数据库。
- 如果一个连接想要写数据库，必须首先获得一个预留锁(`RESERVED`)。一个数据库只能有一个预留锁，该预留锁可以与共享锁共存，它不阻止其他拥有共享锁的连接继续读数据库，也不阻止其他连接获得新的共享锁。
- 一旦连接获得预留锁，就可以开始处理数据库的修改操作，此时修改只能在缓冲区中进行，不实际写磁盘。
- 当连接想要**提交修改**时，需要将预留锁提升为排他锁(`EXCLUSIVE`)。为了得到排他锁，必须首先将预留锁提升为未决锁(`PENDING`)。**获得未决锁之后，其他连接不能再获得新的共享锁。**此时，拥有未决锁的连接等待其他拥有共享锁的连接完成工作并释放共享锁。
- 一旦所有其他的共享锁都被释放，拥有未决锁的连接就可以将其锁提升至排它锁(`EXCLUSIVE`)，此时就可以自由对数据库进行修改，所有以前所缓存的修改都会被写到数据库文件中。

## 事务的类型
SQLite有三种不同的事务类型，它们以不同的锁状态启动事务。事务可以开始于：`DEFERED`、`IMMEDIATE`或`EXCLUSIVE`。事务类型在`BEGIN`命令中制定：
```sql
BEGIN [ DEFERRED | IMMEDIATE | EXCLUSIVE ] TRANSACTION;
```
- 一个`DEFERED`知道必须使用时才获取锁。对于延迟事务，`BEGIN`本身不会做什么事情——它从未锁定状态开始。只是默认的情况。多个连接可以在同一时刻未创建任何锁的情况下开始延迟事务。第一个对数据库的读操作获取共享锁，第一个对数据库的写操作试图获取预留锁。
- 由`BEGIN`开始的`IMMEDIATE`事务在`BEGIN`执行时视图获取预留锁。如果成功，`BEGIN IMMEDIATE`保证没有其他连接可以写数据库。
- `EXCLUSIVE`事务会试着获取对数据库的排它锁。一旦成功，`EXCLUSIVE`事务保证数据库中没有其他活动连接，可以对数据库进行任意读写操作。

如果两个连接都用`DEFERED`事务，可能会出现死锁：都想写数据库，但是都没有放弃自己的锁（例如一个获取了未决锁，一个保持了共享锁）。但是如果都以`BEGIN IMMEDIATE`或者`BEGIN EXCLUSIVE`开始事务，就不会死锁。
基本准则是：如果数据库没有其他连接，用`BEGIN`就足够。如果有其他连接会进行写操作，需要使用`BEGIN IMMEDIATE`或者`BEGIN EXCLUSIVE`开始事务。

# 数据库管理

## 附加数据库
```sql
ATTACH [database] filename AS database_name;
DETACH [database] database_name;
```
可以用`main`全名引用主数据库中的对象。
例子：
```sql
ATTACH database '/tmp/db' AS db2;
SELECT * FROM db2.foo;
SELECT * FROM main.foods LIMIT 2;
```

## 数据库清理

SQLite有两个命令用于清理数据库——`REINDEX`和`VACUUM`。
`REINDEX`用于重建索引：
```sql
REINDEX collation_name;
REINDEX table_name|index_name;
```
第一种形式重建所有使用指定排序名称的索引，当腰改变用户定义的排序行为时才需要这种形式。要重构表中所有索引（或指定名称的索引），可以使用第二种形式的命令。

`VACUUM`通过重构数据库文件清理那些未使用的空间。

## 系统目录
`sqlite_master`表是系统表，包含数据库中所有表、视图、索引和触发器信息。
```sql
SELECT type, name, rootpage FROM sqlite_master;
```
`type`字段说明对象的类型，`name`字段是对象的名称，`rootpage`指对象的第一个B-tree页面在数据库文件中的位置。

## 查看查询计划
使用`EXPLAN QUERY PLAN`命令查看SQLite执行查询的方法：
```sql
EXPLAN QUERY PLAN SELECT * FROM foods WHERE id = 145;
```

# Write-Ahead Logging
SQLite处理`Atomic Commit`和`Rollback`传统方法叫做`Rollback Journal`。当进程想要修改数据库的时候，首先把未修改的这部分内容记录在`Rollback Journal`里，这个时候原始数据库还没有改变。当`COMMIT`数据的时候，SQLite首先确保没有其他数据库连接正在读数据库，将修改直接写入数据库文件，删除`Rollback Journal`。如果这期间发生了crash或者`ROLLBACK`，再利用`Rollback Journal`记录的原始内容`REVERT`之前的变更。

`Write-Ahead Logging`方法反其道而行之，把原始内容保存在数据库文件里，把修改记录在单独的`WAL`文件里，多个`Transaction`会按顺序追加在文件末尾。要读取数据库的时候，首先会查找`WAL`文件里最后一个合法的`COMMIT`记录，并查看要读取的页面是不是出现在这次记录里，如果出现在记录里，就从`WAL`读取修改以后的数据，否则就从原数据库文件里读取。因此在`WAL`模式下，从原始数据库文件中读数据和提交数据修改可以同时进行，但是写事务之间依然不可并行（要保证`COMMIT`的数据按顺序一次追加到`WAL`文件末尾）。在之后的某个时间点(checkpoint)，所有的修改事务再写回数据库。