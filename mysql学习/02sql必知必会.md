## 安装命令行客户端，myclic

```bash
#先安装最新版的python3，相关指令在网上找

#安装pip
yum install python-pip python-devel
#安装mycli
pip install mycli

## ，对应卸载
#  卸载指令(对应) easy_install -m mycli
#  yum remove [软件名]
```



然后就可直接使用mycli，这个命令行客户端的好处是有提示符。



## sql必知必会

将以前学习的sql操作复习一遍, 然后学习触发器索引等知识。

(刚开始学习sql是以oracle为主的，所以在一些操作上与mysql会有些不同，比如分页操作，存储过程的操作等)。

#### 创建一个表

```sql
--mysql中创建一个表
-- 如果数据库中存在user_accounts表，就把它从数据库中drop掉
DROP TABLE IF EXISTS `user_accounts`;
CREATE TABLE `user_accounts` (
  `id`             int(100) unsigned NOT NULL AUTO_INCREMENT primary key,
  `password`       varchar(32)       NOT NULL DEFAULT '' COMMENT '用户密码',
  `reset_password` tinyint(32)       NOT NULL DEFAULT 0 COMMENT '用户类型：0－不需要重置密码；1-需要重置密码',
  `mobile`         varchar(20)       NOT NULL DEFAULT '' COMMENT '手机',
  `create_at`      timestamp(6)      NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  `update_at`      timestamp(6)      NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),
  -- 创建唯一索引，不允许重复
  UNIQUE INDEX idx_user_mobile(`mobile`)
)
ENGINE=InnoDB DEFAULT CHARSET=utf8
COMMENT='用户表信息';
```

#### 基本操作指令

基本的增删查改操作，sql命令都是相同的。下面列举一些mysql中的操作。

```sql
UNION - 操作符用于合并两个或多个 SELECT 语句的结果集。


-- 列出所有在中国表（Employees_China）和美国（Employees_USA）的不同的雇员名
SELECT E_Name FROM Employees_China UNION SELECT E_Name FROM Employees_USA

```

#### 触发器：

> 语法： create trigger <触发器名称> { before | after} # 之前或者之后出发 insert | update | delete # 指明了激活触发程序的语句的类型 on <表名> # 操作哪张表 for each row # 触发器的执行间隔，for each row 通知触发器每隔一行执行一次动作，而不是对整个表执行一次。 <触发器SQL语句>



```sql
delimiter $
CREATE TRIGGER set_userdate BEFORE INSERT 
on `message`
for EACH ROW
BEGIN
  set @statu = new.status; -- 声明复制变量 statu
  if @statu = 0 then       -- 判断 statu 是否等于 0
    UPDATE `user_accounts` SET status=1 WHERE openid=NEW.openid;
  end if;
END
$
DELIMITER ; -- 恢复结束符号
```

OLD和NEW不区分大小写

- NEW 用NEW.col_name，没有旧行。在DELETE触发程序中，仅能使用OLD.col_name，没有新行。
- OLD 用OLD.col_name来引用更新前的某一行的列



### 添加索引

mysql中的索引分为普通索引，主键索引，唯一索引，全文索引。

#### 普通索引(INDEX)

> 语法：ALTER TABLE  `表名字`  ADD INDEX 索引名字 (  `字段名字`  )

```sql
-- –直接创建索引
CREATE INDEX index_user ON user(title)
-- –修改表结构的方式添加索引
ALTER TABLE table_name ADD INDEX index_name ON (column(length))
-- 给 user 表中的 name 字段 添加普通索引(INDEX)
ALTER TABLE `user` ADD INDEX index_name (name)
-- –创建表的时候同时创建索引
CREATE TABLE `user` (
    `id` int(11) NOT NULL AUTO_INCREMENT ,
    `title` char(255) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL ,
    `content` text CHARACTER SET utf8 COLLATE utf8_general_ci NULL ,
    `time` int(10) NULL DEFAULT NULL ,
    PRIMARY KEY (`id`),
    INDEX index_name (title(length))  ---添加了普通索引

)
-- –删除索引
DROP INDEX index_name ON table
```

#### 主键索引(PRIMARY key)

> 语法：ALTER TABLE  `表名字`  ADD PRIMARY KEY (  `字段名字`  )

```sql
-- 给 user 表中的 id字段 添加主键索引(PRIMARY key)
ALTER TABLE `user` ADD PRIMARY key (id);
```

#### 唯一索引（UNIQUE）

> 语法：ALTER TABLE   '表名' ADD  UNIQUE(字段名字) 

```sql
-- 给 user 表中的 creattime 字段添加唯一索引(UNIQUE)
ALTER TABLE `user` ADD UNIQUE (creattime);
```

#### 全文索引(FULLTEXT)

> 语法：ALTER TABLE  `表名字`  ADD FULLTEXT (`字段名字`)

```sql
-- 给 user 表中的 description 字段添加全文索引(FULLTEXT)
ALTER TABLE `user` ADD FULLTEXT (description);
```



#### 添加多列索引(也就是联合索引)

> 语法： ALTER TABLE  `table_name`  ADD INDEX index_name (  `column1`,  `column2`,  `column3`)

```sql
-- 给 user 表中的 name、city、age 字段添加名字为name_city_age的普通索引(INDEX)
ALTER TABLE user ADD INDEX name_city_age (name(10),city,age); 
```

#### 建立索引的时机

在`WHERE`和`JOIN`中出现的列需要建立索引，但也不完全如此：

- MySQL只对`<`，`<=`，`=`，`>`，`>=`，`BETWEEN`，`IN`使用索引
- 某些时候的`LIKE`也会使用索引。
- 在`LIKE`以通配符%和_开头作查询时，MySQL不会使用索引。

```sql
-- 此时就需要对city和age建立索引，
-- 由于mytable表的userame也出现在了JOIN子句中，也有对它建立索引的必要。
SELECT t.Name  
FROM mytable t LEFT JOIN mytable m ON t.Name=m.username 
WHERE m.age=20 AND m.city='上海';

SELECT * FROM mytable WHERE username like'admin%'; -- 而下句就不会使用：
SELECT * FROM mytable WHERE Name like'%admin'; -- 因此，在使用LIKE时应注意以上的区别。
```

索引的注意事项

- 索引不会包含有NULL值的列
- 使用短索引
- 不要在列上进行运算 索引会失效



### 创建后对于表的修改，alter命令

#### 添加列

> 语法：`alter table 表名 add 列名 列数据类型 [after 插入位置];`

示例:

```sql
-- 在表students的最后追加列 address: 
alter table students add address char(60);
-- 在名为 age 的列后插入列 birthday: 
alter table students add birthday date after age;
-- 在名为 number_people 的列后插入列 weeks: 
alter table students add column `weeks` varchar(5) not null default "" after `number_people`;
```

#### 修改列

> 语法：`alter table 表名 change 列名称 列新名称 新数据类型;`

```sql
-- 将表 tel 列改名为 telphone:
 alter table students change tel telphone char(13) default "-";
-- 将 name 列的数据类型改为 char(16):
 alter table students change name name char(16) not null;
-- 修改 COMMENT 前面必须得有类型属性
alter table students change name name char(16) COMMENT '这里是名字';
-- 修改列属性的时候 建议使用modify,不需要重建表
-- change用于修改列名字，这个需要重建表
alter table meeting modify `weeks` varchar(20) NOT NULL DEFAULT '' COMMENT '开放日期 周一到周日：0~6，间隔用英文逗号隔开';
-- `user`表的`id`列，修改成字符串类型长度50，不能为空，`FIRST`放在第一列的位置
alter table `user` modify COLUMN `id` varchar(50) NOT NULL FIRST ;
```

#### 删除列

> 语法：`alter table 表名 drop 列名称;`

```sql
-- 删除表students中的 birthday 列: 
alter table students drop birthday;
```

#### 重命名表

> 语法：`alter table 表名 rename 新表名;`

```sql
-- 重命名 students 表为 workmates: 
alter table students rename workmates;
```

#### 清空表数据

> 方法一：`delete from 表名;`  方法二：`truncate table "表名";`

- `DELETE:`1. DML语言;2. 可以回退;3. 可以有条件的删除;
- `TRUNCATE:`1. DDL语言;2. 无法回退;3. 默认所有的表内容都删除;4. 删除速度比delete快。

```sql
-- 清空表为 workmates 里面的数据，不删除表。 
delete from workmates;
-- 删除workmates表中的所有数据，且无法恢复
truncate table workmates;
```

#### 删除整张表

> 语法：`drop table 表名;`

```sql
-- 删除 workmates 表: 
drop table workmates;
```

#### 删除整个数据库

> 语法：`drop database 数据库名;`

```sql
-- 删除 samp_db 数据库: 
drop database samp_db;
```



### sql中删除重复记录

思路是一般先查出重复记录，然后删除。

> 语法：drop database 数据库名;

```sql
-- 查找表中多余的重复记录，重复记录是根据单个字段（peopleId）来判断
select * from people where peopleId in (select peopleId from people group by peopleId having count(peopleId) > 1)
-- 删除表中多余的重复记录，重复记录是根据单个字段（peopleId）来判断，只留有rowid最小的记录
delete from people 
where peopleId in (select peopleId from people group by peopleId having count(peopleId) > 1)
and rowid not in (select min(rowid) from people group by peopleId having count(peopleId )>1)
-- 查找表中多余的重复记录（多个字段）
select * from vitae a
where (a.peopleId,a.seq) in (select peopleId,seq from vitae group by peopleId,seq having count(*) > 1)
-- 删除表中多余的重复记录（多个字段），只留有rowid最小的记录
delete from vitae a
where (a.peopleId,a.seq) in (select peopleId,seq from vitae group by peopleId,seq having count(*) > 1) and rowid not in (select min(rowid) from vitae group by peopleId,seq having count(*)>1)
-- 查找表中多余的重复记录（多个字段），不包含rowid最小的记录
select * from vitae a
where (a.peopleId,a.seq) in (select peopleId,seq from vitae group by peopleId,seq having count(*) > 1) and rowid not in (select min(rowid) from vitae group by peopleId,seq having count(*)>1）
```



## 总结：

本篇学习了如何安装mycli，注意一点的是需要先安装python3，否则就会报错提示py2版本已过时，pip安装不了 install。

然后学习了mysql中如何创建表， 基本的sql操作与oracle没什么不同，区别是认识oracle的存储的最小单元是元数据，而mysql是按页为单位存储的，这就表现在了分页语句上，oracle是rownum总记录数， mysql分页只需要传入每页对应的条数就行。

然后学习了索引，索引，简单理解就是目录，对于表中的字段，可以创建对应的索引，mysql中的索引有普通索引（index），主键索引（PRIMARY key），唯一索引（unique），全文索引（FULLTEXT） ，可以为多个字段添加一个索引（也就是联合索引）。 需要掌握建立索引的时机，除了where、join外，

- MySQL只对`<`，`<=`，`=`，`>`，`>=`，`BETWEEN`，`IN`使用索引
- 某些时候的`LIKE`也会使用索引。
- 在`LIKE`以通配符%和_开头作查询时，MySQL不会使用索引。



接着学习了触发器，触发器可以行为单位的执行，比如每隔一行数据就执行一次。

最后学习了在创建表后如何更改表中的数据，看例子就行，sql的操作如果想熟练掌握，必须在实际中大量实践才行。










