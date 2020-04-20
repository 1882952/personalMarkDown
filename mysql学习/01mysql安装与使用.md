## 一：安装

现在都是用docker了，所以mysql的安装就使用docker安装，过程就是，

1. 拉取mysql镜像，docker pull mysql:[版本号]

2. 创建一个目录，用于挂载mysql volume，实际上就是创建一个保存mysql数据的目录，与docker中的虚拟mysql路径连接。 

3. 创建mysql容器，指令如下:

```bash
docker run -d --name mysql -p 3306:3306 \
 -v $(pwd):/var/lib/mysql -e MYSQL_ROOT_PASSWORD=123456 mysql:5.7.25
```

大概解释一下，run是启动执行一个容器， -d :可以后台常驻运行，--name：自定义一个容器名，  -p：设置端口映射。 -v: 后面就是挂载mysql volume具体的目录。  

 -e：这里就是设置root默认密码了。   最后是mysql镜像的版本号。

## 二：使用

docker的安装过程就是这么简单，软件的安装变成了容器的挂载，即插即拔，甚至结束软件时直接删除就行，再次使用时利用镜像重新创建一个容器就好。

> 但是这里有个问题，既然mysql容器在后台运行着，那么如何在后台运行时进入容器？使用如下指令：
> 
> ```bash
> docker exec -it 已经运行的容器名称 /bin/bash (/bin/bash 固定写法)
> ```

这样就可以进入后台运行的容器了。

### 访问拒绝问题。

在第一次远程连接mysql时，经常会发生访问拒绝的提示，1045，该怎么解决？

因为原因是远程访问该mysql用户没有权限，所以就需要设置权限。

解决办法是在服务器中登录mysql，然后授权。

```javascript
grant all privileges on *.* to '用户名'@'IP地址' identified by '密码';//（给特定ip授权）
//或者
eg:grant all privileges on *.* to root@"%" identified by 'root' with grant option;   //给所有用户授权

//然后刷新执行操作
flush privileges;
```



说完了安装与遇到的问题之后，下面就来使用mysql，因为前面学习过，所以这次只是总结。



## 三：操作

#### （1）：登录mysql

```bash
mysql -h localhost -P 3306 -u root -p
```

#### （2）：常用的mysql操作：

```mysql
SHOW PROCESSLIST -- 显示哪些线程正在运行
SHOW VARIABLES -- 显示系统变量信息
```

```bash
mysql -D 所选择的数据库名 -h 主机名 -u 用户名 -p
mysql> exit # 退出 使用 “quit;” 或 “\q;” 一样的效果
mysql> status;  # 显示当前mysql的version的各种信息
mysql> select version(); # 显示当前mysql的version信息
mysql> show global variables like 'port'; # 查看MySQL端口号
mysql>show variables like '%storage_engine%';#查看默认的存储引擎
mysql>show table status like "table_name" ; #查看表的存储引擎

```

```mysql
/* 数据库操作 */ ------------------
-- 查看当前数据库
    SELECT DATABASE();
-- 显示当前时间、用户名、数据库版本
    SELECT now(), user(), version();
-- 创建库
    CREATE DATABASE[ IF NOT EXISTS] 数据库名 数据库选项
    数据库选项：
        CHARACTER SET charset_name
        COLLATE collation_name
-- 查看已有库
    SHOW DATABASES[ LIKE 'PATTERN']
-- 查看当前库信息
    SHOW CREATE DATABASE 数据库名
-- 修改库的选项信息
    ALTER DATABASE 库名 选项信息
-- 删除库
    DROP DATABASE[ IF EXISTS] 数据库名
        同时删除该数据库相关的目录及其目录内容
```



## 四：练习操作：

### 创建数据库及相关操作：

```mysql
-- 创建一个名为 test_db 的数据库，数据库字符编码指定为 gbk

create database test_db character set gbk;

drop database test_db; -- 删除 库名为samp_db的库
show databases;        -- 显示数据库列表。
use test_db;     -- 选择创建的数据库samp_db
show tables;     -- 显示samp_db下面所有的表名字
describe 表名;    -- 显示数据表的结构
delete from 表名; -- 清空表中记录
```



然后就是创建表，增删查改，关于操作的相关知识，和oracle一样，关于sql的相关操作。



> 在docker中，比如进入了容器要退出， 使用exit指令， 如果不行，可以尝试ctrl c等。



### 总结：

今天大致总结了一下如何使用docker安装mysql，常见问题的解决方法等。
