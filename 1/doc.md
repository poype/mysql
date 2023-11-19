

# 安装完后修改user表

MySql安装完成后，就包含下面4个数据库实例。

![image-20231118230118172](D:\Workspace\mysql\1\img\image-20231118230118172.png)

在其中的mysql数据库实例中，包含一张user表，这张表记录了mysql的用户信息：

![image-20231118230331722](D:\Workspace\mysql\1\img\image-20231118230331722.png)

可以看到root用户的当前主机配置信息为localhost，Host列指定了允许用户登录所使用的IP，Host=localhost表示只能通过本机客户端去访问。如果 Host=%，表示所有IP都有连接权限。

执行下面两个命令允许root用户可以从任意IP地址连接到Mysql服务器：

```sql
update user set host = '%' where user ='root';

flush privileges;
```

![image-20231118230651870](D:\Workspace\mysql\1\img\image-20231118230651870.png)

`flush privileges`命令是使配置立即生效。

# 字符集

在MySQL 8.0版本之前，默认字符集为latin1，utf8字符集指向的是 utf8mb3 。

从MySQL 8.0开始，数据库的默认编码将改为utf8mb4 ，从而避免上述乱码的问题。

执行下面命令查看数据库的字符集：

```sql
show variables like '%char%';
```

如果在MySQL 8.0中，执行结果如下：

![image-20231119084551459](D:\Workspace\mysql\1\img\image-20231119084551459.png)

如果是MySQL 5.7中，执行结果如下：

![image-20231119084811661](D:\Workspace\mysql\1\img\image-20231119084811661.png)

MySQL 5.7 默认的服务器用了 latin1 ，不支持中文，保存中文会报错。

#### 各字符集配置项含义：

- character_set_server：服务器级别的字符集 
- character_set_database：当前数据库的字符集 
- character_set_client：服务器解码请求时使用的字符集 
- character_set_connection：服务器处理请求时会把请求字符串从character_set_client转为 character_set_connection
- character_set_results：服务器向客户端返回数据时使用的字符集

#### 每个字符集和比较规则的联系如下：

- 如果 **创建或修改列** 时没有显式的指定字符集和比较规则，则该列 默认用表的 字符集和比较规则 
- 如果 **创建表时** 没有显式的指定字符集和比较规则，则该表 默认用数据库的 字符集和比较规则 
- 如果 **创建数据库时** 没有显式的指定字符集和比较规则，则该数据库 默认用服务器的 字符集和比较规则

#### utf8 与 utf8mb4

- **utf8mb3：**阉割过的 utf8 字符集，只使用1～3个字节表示字符。 
- **utf8mb4：**正宗的 utf8 字符集，使用1～4个字节表示字符。

## 修改字符集

修改已创建数据库的字符集：

```sql
alter database dbtest1 character set 'utf8';
```

修改已创建数据表的字符集:

```sql
alter table t_emp convert to character set 'utf8';
```

# MySQL配置文件

/etc/my.cnf 是mysql的配置文件。



# 大小写敏感

在Linux环境下，MySQL对表名字的大小写是敏感的。

```sql
show VARIABLES LIKE '%lower_case_table_names%';
```

![image-20231119094009389](D:\Workspace\mysql\1\img\image-20231119094009389.png)

值为0表示对大小写敏感，值为1表示对大小写不敏感。

表名是小写的emp，如果换成EMO会报错“table doesn't exist”：

![image-20231119094114892](D:\Workspace\mysql\1\img\image-20231119094114892.png)

MySQL在Linux下数据库名、表名、列名、别名大小写规则是这样的： 

- 数据库名、表名、表的别名、变量名是严格区分大小写的；
- 关键字、函数名称在 SQL 中不区分大小写；
- 列名是忽略大小写的；

