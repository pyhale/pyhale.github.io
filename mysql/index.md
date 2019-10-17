# MySQL 优化系列 -- 索引的使用、原理和设计优化

## 一、索引的概述和使用：

### （1）概述：

1. 什么是索引？

    **索引是一种特殊的文件(InnoDB 数据表上的索引是表空间的一个组成部分)，它们包含着对数据表里所有记录的引用指针。**

    更通俗的说，数据库索引好比是一本书前面的目录，能加快数据库的查询速度。在没有索引的情况下，数据库会遍历全部数据后选择符合条件的；而有了相应的索引之后，数据库会直接在索引中查找符合条件的选项。

    索引的性质分类：

    索引分为聚簇索引和非聚簇索引两种，聚簇索引是按照数据存放的物理位置为顺序的，而非聚簇索引就不一样了；聚簇索引能提高多行检索的速度，而非聚簇索引对于单行的检索很快。

2. 索引的优点：

    1. 通过创建唯一性索引，可以保证数据库表中每一行数据的唯一性。

    2. 可以大大加快数据的检索速度，这也是创建索引的最主要的原因。

    3. 可以加速表和表之间的连接，特别是在实现数据的参考完整性方面特别有意义。

    4. 在使用分组和排序 子句进行数据检索时，同样可以显著减少查询中分组和排序的时间。

    5. 通过使用索引，可以在查询的过程中，使用优化隐藏器，提高系统的性能。

3. 索引的缺点：

    1. 创建索引和维护索引要耗费时间，这种时间随着数据量的增加而增加。

    2. 索引需要占物理空间，除了数据表占数据空间之外，每一个索引还要占一定的物理空间，如果要建立聚簇索引，那么需要的空间就会更大。

    3. 当对表中的数据进行增加、删除和修改的时候，索引也要动态的维护，这样就降低了数据的维护速度。

4. 为什么需要索引：

    数据在磁盘上是以块的形式存储的。为确保对磁盘操作的原子性，访问数据的时候会一并访问所有数据块。磁盘上的这些数据块与链表类似，即它们都包含一个数据段和一个指针，指针指向下一个节点（数据块）的内存地址，而且它们都不需要连续存储（即逻辑上相邻的数据块在物理上可以相隔很远）。

    鉴于很多记录只能做到按一个字段排序，所以要查询某个未经排序的字段，就需要使用线性查找，即要访问N/2个数据块，其中N指的是一个表所涵盖的所有数据块。如果该字段是非键字段（也就是说，不包含唯一值），那么就要搜索整个表空间，即要访问全部N个数据块。（在某些情况下，索引可以避免排序操作。）

    然而，对于经过排序的字段，可以使用二分查找，因此只要访问 log2 N个数据块。同样，对于已经排过序的非键字段，只要找到更大的值，也就不用再搜索表中的其他数据块了。这样一来，性能就会有实质性的提升。

### （2）索引的使用（语法）：

1. 创建索引：（三种方式）

    第一种方式：
    ~~~sql
    //第一种方式：
    //在执行CREATE TABLE 时创建索引：（硬设一个id索引）
    CREATE TABLE `black_list` (
        `id` BIGINT(20) NOT NULL AUTO_INCREMENT,
        `black_user_id` BIGINT(20) NULL DEFAULT NULL,
        `user_id` BIGINT(20) NULL DEFAULT NULL,
        PRIMARY KEY (`id`)
        INDEX indexName (black_user_id(length))
    )
    COLLATE='utf8_general_ci'
    ENGINE=InnoDB
    ;
    ~~~
    第二种方式：使用ALTER TABLE命令去增加索引：
    
    ALTER TABLE 用来创建普通索引、UNIQUE 索引或 PRIMARY KEY 索引。
    ~~~sql
    //标准语句：
    ALTER TABLE table_name ADD INDEX index_name (column_list) //添加普通索引，索引值可出现多次。 
    ALTER TABLE table_name ADD UNIQUE (column_list) //这条语句创建的索引的值必须是唯一的(除了NULL外，NULL可能会出现多次)。 
    ALTER TABLE table_name ADD PRIMARY KEY (column_list) //该语句添加一个主键，这意味着索引值必须是唯一的，且不能为NULL。
    ALTER TABLE table_name ADD FULLTEXT index_name(olumu_name); //该语句指定了索引为FULLTEXT，用于全文索引。
    
    
    //针对上述数据库，增加商品分类的索引
    ALTER table commodity_list ADD INDEX classify_index  (Classify_Description)
    ~~~
    其中 table_name 是要增加索引的表名，column_list 指出对哪些列进行索引，多列时各列之间用逗号分隔。索引名index_name 可自己命名，缺省时，MySQL 将根据第一个索引列赋一个名称。另外，ALTER TABLE 允许在单个语句中更改多个表，因此可以在同时创建多个索引。
    
    第三种方式：使用 CREATE INDEX 命令创建
    
    CREATE INDEX 可对表增加普通索引或 UNIQUE 索引。
    
    ~~~sql
    //标准语句：
    CREATE INDEX index_name ON table_name (column_list)
    CREATE UNIQUE INDEX index_name ON table_name (column_list)
    //针对上述数据库：
    CREATE INDEX classify_index  ON commodity_list (Classify_Description)
    ~~~
    table_name、index_name 和 column_list 具有与 ALTER TABLE 语句中相同的含义，索引名不可选。另外，不能用CREATE INDEX 语句创建 PRIMARY KEY 索引。

2. 删除索引：

    删除索引可以使用 ALTER TABLE 或 DROP INDEX 语句来实现。DROP INDEX 可以在 ALTER TABLE 内部作为一条语句处理，其格式如下：
    ~~~sql
    DROP INDEX [indexName] ON [table_name];
    alter table [table_name] drop index [index_name];
    alter table [table_name] drop primary key;
    //针对上述数据库
    drop index classify_index on commodity_list;
    ~~~
    其中，在前面的两条语句中，都删除了 table_name 中的索引 index_name。而在最后一条语句中，只在删除 PRIMARY KEY 索引中使用，因为一个表只可能有一个 PRIMARY KEY 索引，因此不需要指定索引名。如果没有创建 PRIMARY KEY 索引，但表具有一个或多个 UNIQUE 索引，则 MySQL 将删除第一个 UNIQUE 索引。
    
    如果从表中删除某列，则索引会受影响。对于多列组合的索引，如果删除其中的某列，则该列也会从索引中删除。如果删除组成索引的所有列，则整个索引将被删除。

3. 查看索引：
    ~~~sql
    SHOW INDEX FROM [table_name];
    show keys from [table_name];
    ~~~
    Table：表的名称。
    
    Non_unique：如果索引不能包括重复词，则为0。如果可以，则为1。
    
    Key_name：索引的名称。
    
    Seq_in_index：索引中的列序列号，从1开始。
    
    Column_name：列名称。
    
    Collation：列以什么方式存储在索引中。在 MySQL 中，有值 ‘A’（升序）或 NULL（无分类）。
    
    Cardinality：索引中唯一值的数目的估计值。通过运行 ANALYZE TABLE 或 myisamchk -a 可以更新。基数根据被存储为整数的统计数据来计数，所以即使对于小型表，该值也没有必要是精确的。基数越大，当进行联合时，MySQL 使用该索引的机会就越大。
    
    Sub_part：如果列只是被部分地编入索引，则为被编入索引的字符的数目。如果整列被编入索引，则为 NULL。
    
    Packed：指示关键字如何被压缩。如果没有被压缩，则为 NULL。
    
    Null：如果列含有NULL，则含有YES。如果没有，则该列含有NO。
    
    Index_type：用过的索引方法（BTREE, FULLTEXT, HASH, RTREE）。

## 索引的基本原理：

举例解析基本原理：

整体性原理例子：

除了词典，生活中随处可见索引的例子，如火车站的车次表、图书的目录等。它们的原理都是一样的，通过不断的缩小想要获得数据的范围来筛选出最终想要的结果，同时把随机的事件变成顺序的事件，也就是我们总是通过同一种查找方式来锁定数据。

SQL 的应用场景会使用索引：

数据库也是一样，但显然要复杂许多，因为不仅面临着等值查询，还有范围查询(>、<、between、in)、模糊查询(like)、并集查询(or)等等。数据库应该选择怎么样的方式来应对所有的问题呢？我们回想字典的例子，能不能把数据分成段，然后分段查询呢？最简单的如果1000条数据，1到100分成第一段，101到200分成第二段，201到300分成第三段…这样查第250条数据，只要找第三段就可以了，一下子去除了90%的无效数据。

针对存储性质讲解：[此部分转载自此博主此博客](http://blog.csdn.net/kennyrose/article/details/7532032)

由于存储介质的特性，磁盘本身存取就比主存慢很多，再加上机械运动耗费，磁盘的存取速度往往是主存的几百分分之一，因此为了提高效率，要尽量减少磁盘I/O。为了达到这个目的，磁盘往往不是严格按需读取，而是每次都会预读，即使只需要一个字节，磁盘也会从这个位置开始，顺序向后读取一定长度的数据放入内存。这样做的理论依据是计算机科学中著名的局部性原理：当一个数据被用到时，其附近的数据也通常会马上被使用。程序运行期间所需要的数据通常比较集中。

由于磁盘顺序读取的效率很高（不需要寻道时间，只需很少的旋转时间），因此对于具有局部性的程序来说，预读可以提高I/O效率。

预读的长度一般为页（page）的整倍数。页是计算机管理存储器的逻辑块，硬件及操作系统往往将主存和磁盘存储区分割为连续的大小相等的块，每个存储块称为一页（在许多操作系统中，页得大小通常为4k），主存和磁盘以页为单位交换数据。当程序要读取的数据不在主存中时，会触发一个缺页异常，此时系统会向磁盘发出读盘信号，磁盘会找到数据的起始位置并向后连续读取一页或几页载入内存中，然后异常返回，程序继续运行。


