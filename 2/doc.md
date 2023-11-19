# 数据库文件的存放路径

在`/etc/my.cnf`配置文件中指定了MySql的数据库文件会放在`/var/lib/mysql`目录下：

![image-20231119102453849](D:\Workspace\mysql\2\img\image-20231119102453849.png)

也可以在数据库中通过下面的命令查看数据文件所在的路径：

```sql
show variables like 'datadir';
```

![image-20231119102708709](D:\Workspace\mysql\2\img\image-20231119102708709.png)

## 数据库和文件系统的关系

![image-20231119105455374](D:\Workspace\mysql\2\img\image-20231119105455374.png)

除了information_schema这个系统数据库外，其它数据库在`/var/lib/mysql`路径下都对应着一个目录：

![image-20231119105556686](D:\Workspace\mysql\2\img\image-20231119105556686.png)

可以看到`dbtest1`和`dbtest2`两个数据库实例分别在`/var/lib/mysql`路径下对应着两个目录。

dbtest1数据库中包含了dep和emp两张表：

![image-20231119105858847](D:\Workspace\mysql\2\img\image-20231119105858847.png)

每个表在dbtest1目录中分别对应一个 **ibd** 文件：

![image-20231119110024165](D:\Workspace\mysql\2\img\image-20231119110024165.png)

## MySQL5.7 和 MySQL8.0的区别

在MySQL5.7 中， 为了保存表结构， InnoDB 在数据库子目录下创建了一个专门用于描述表结构的文 件 ，文件名是这样：

```
表名.frm
```

但在MySQL8.0中不再单独提供`表名.frm`文件，而是把表结构数据合并在`表名.ibd`文件中。

# 系统表空间 和 独立表空间

##### 系统表空间（system tablespace）

默认情况下，InnoDB会在数据目录下创建一个名为 ibdata1 、大小为 12M 的文件，这个文件就是对应 的 **系统表空间** 在文件系统上的表示。怎么才12M？注意这个文件是 自扩展文件 ，当不够用的时候它会自 己增加文件大小。

##### 独立表空间(file-per-table tablespace

在MySQL5.6.6以及之后的版本中，InnoDB并不会默认的把各个表的数据存储到系统表空间中，而是为 每 一个表建立一个独立表空间 ，也就是说我们**创建了多少个表，就有多少个独立表空间**。使用 独立表空间 来 存储表数据的话，会在该表所属数据库对应的子目录下创建一个**表示该独立表空间的文件**，文件名和表名相同，只不过添加了一个 .ibd 的扩展名而已。