### 查询缓存

MySQL可以把处理过的查询请求及其结果缓存起来，这样如果下一次有同样的请求过来，就能够直接从缓存中查找结果，无需再去底层的表中查找了。

但如果两个查询请求有任何字符上的不同(例如，空格、注释、大小写)，都会导致查询缓存不命中。

如果某个表的结构或数据被修改，比如对该表使用了INSERT、UPDATE、DELETE、ALTER TABLE等语句，则与该表有关的**所有**查询缓存都将变为无效并从查询缓存中删除！

### InnoDB VS MyISAM

InnoDB: 支持事务、行级锁、外键

MyISAM：主要的非事务处理存储引擎

### InnoDB 页

InnoDB存储引擎将数据划分为若干个页，以页作为磁盘和内存之间交互的基本单位。页的大小一般为16KB。

**一次最少从磁盘中读取16KB的内容到内存中，一次最少把内存中的16KB内容刷新到磁盘中。**

系统变量**innodb_page_size**表明了InnoDB存储引擎中的页大小，默认值是16384字节，也就是16KB。可以在第一次初始化MySQL数据目录时指定，之后就再也不能更改了。

### InnoDB行格式

COMPACT, REDUNDANT, DYNAMIC, COMPRESSED. 

MySQL 5.7版本默认使用的行格式是 DYNAMIC

##### 1. 变长字段长度

像varchar这种变长字段占用的存储空间分为两部分：

- 真正的数据内容
- 该数据占用的字节数

##### 2. Null值列表

Null值列表就是一个bitmap，用于记录一行记录中有哪些值时Null。

##### 3. 记录头信息

**delete_flag：**该属性用来标记当前记录是否被删除，占用1比特。一条记录被删除后只是利用delete_flag属性打上一个被删除的标记，并没有真正从磁盘中移除。

被删除的记录之所以不从磁盘中移除，是因为移除它们之后，还需要在磁盘上重新排列其它记录，这回带来性能消耗，只打上一个删除标记就可以避免这个问题。

所有被删除掉的记录会组成一个垃圾链表，垃圾链表中的空间被称为可重用空间。之后若有新记录插入到表中，新记录就可能覆盖掉被删除掉的记录占用的存储空间。

##### 4. 溢出列

如果某一列中的数据非常多，则在本记录中只存储该列前768字节的数据以及一个指向其它页的指针，然后把剩余的数据放到其它页中。

这些存储768字节之外的数据的页面就称为溢出页。

### InnoDB数据页结构

一个数据页中包含**User Record**和**Free Space**两块区域。

我们自己存储的记录会按照指定的行格式存储到User Record部分。但在一开始的时候，并没有User Record部分，每当insert一条记录时，都会从Free Space部分申请一个记录大小的空间，并将这个空间划分给User Record部分。

当Free Space部分的空间全部被User Record部分替代掉之后，页就意味着这个页使用完了。此时如果再有新的记录，就需要去申请新的页了。





在一个数据页内部，各个行记录是通过**单向链表数据结构**进行组织的。

每个数据页会自动包含两条记录Infimum和Suprenum，Infimum代表页中最小的记录，Suprenum代表页中最大的记录。

Infimum作为单向链表的头节点，Suprenum作为单向链表的尾节点。

![image-20231207215316591](.\img\image-20231207215316591.png)

如果从表中删除一条记录，则那个记录会从这个单向链表中删除。

![image-20231207215824267](.\img\image-20231207215824267.png)

无论如何对页中的记录进行增删改操作，InnoDB始终会维护记录的一个单向链表，**链表中的各个节点是按照primary key由小到大的顺序链接起来的**。

### Page Directory

在单链表中查找一条记录，只能从链表头节点开始，一个一个向后遍历每个节点，这样查找效率肯定不高。

为了提高查找效率，链表中的所有正常记录(不包括垃圾链表中的记录)会被划分为几个组，每个组的最后一条记录相当于这个组的”带头大哥“，”带头大哥“头信息中的**n_owned**属性表示该组内共有几条记录。各个组的”带头大哥“在页面中的地址偏移量会被提取出来，**按顺序**存储到页面尾部的区域。这个区域就叫做Page Directory。那些”带头大哥“的地址偏移量称为Slot。

![image-20231208083155992](.\img\image-20231208083155992.png)

![image-20231208083306083](.\img\image-20231208083306083.png)

有了Page Directory，在一个数据页中查找指定主键值的记录时，过程分为两步。

1. 通过二分法在Page Directory中查找，确定该记录所在分组对应的Slot，然后找到该Slot所在分组中主键值最小的那条记录。
2. 通过记录的next_record指针依次遍历该组中的各个记录。

Page Directory就像一个稀疏索引一样。







各个数据页之间是以双向链表的数据结构进行组织的：

![image-20231208084043513](.\img\image-20231208084043513.png)



### 总结一下：

各个数据页可以组成一个双向链表，而每个数据页中的记录会按照主键值从小到大的顺序组成一个单向链表。

每个数据页都会为存储在它里面的记录生成一个页目录，在通过主键值查找某条记录的时候可以在页目录中使用二分法快速定位到对应的槽，然后再遍历改槽对应分组中的记录。

![image-20231209210121477](.\img\image-20231209210121477.png)

页a、页b、页c……页n这些页可以不在物理结构上相连，只需要通过双向链表相关联即可。

下一个数据页中记录的主键值必须大于上一个页中用户记录的主键值。

### 页分裂

假设一页最多只能存放三条记录，下图展示了增加一条新记录分配新页的过程：

![image-20231209220506363](.\img\image-20231209220506363.png)

上图有两个地方需要注意，1. 页与页之间可能是不连续的，一个是10页，一个是28页(但InnoDB会尽量让这些页面相邻)。2. 后一页记录的主键值没有大于前一页中所有记录的主键值，所以在插入主键值4的记录时需要伴随着一次记录移动，即把主键值为5的记录移动到28页中，再把主键值为4的记录插入到页10中。这个过程如下图所示：

![image-20231209221009464](.\img\image-20231209221009464.png)

在对页中的记录进行增删改操作的过程中，我们必须通过一些诸如记录移动的操作来始终保证这个状态一直成立：下一个数据页中用户记录的主键值必须大于上一个页中用户记录的主键值。这个过程页称作**页分裂**。

### B+树

![image-20231209222832310](D:\Workspace\mysql\4\img\image-20231209222832310.png)

如果B+树有3层，就能支持一亿条记录。

如果B+树有4层，就能支持一千亿条记录

##### 聚簇索引

##### 二级索引

聚簇索引只能在搜索条件是主键值时才能发挥作用，原因是聚簇索引B+树中的数据都是按照主键进行排序的。

我们可以多建几棵B+树，并且不同的B+树中的数据采用不同的排序规则。这些就是二级索引。

**二级索引B+树中的叶子节点并不是完整的用户记录，而只是索引列+主键列**。

### 回表

由于二级索引的叶子节点中并包含全部列的信息，而只包含索引列和主键列。所以在利用二级索引查询**所有**列信息的时候，需要先在二级索引B+树中查找主键列的值，再通过主键列的值到聚簇索引中查找到完整的用户记录。这个通过携带主键信息到聚簇索引中重新定位完整的用户记录的过程称为回表。