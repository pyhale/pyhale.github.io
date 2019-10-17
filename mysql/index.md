## MySQL 索引优化

### 索引设计优化：

#### 索引建立的几大原则：

1） 最左前缀匹配原则

非常重要的原则，mysql会一直向右匹配直到遇到范围查询(>、<、between、like)就停止匹配，比如a = 1 and b = 2 and c > 3 and d = 4 如果建立(a,b,c,d)顺序的索引，d是用不到索引的，如果建立(a,b,d,c)的索引则都可以用到，a,b,d的顺序可以任意调整。

2） = 和 in 可以乱序

比如 a = 1 and b = 2 and c = 3 建立(a,b,c)索引可以任意顺序，MySQL 的查询优化器会帮你优化成索引可以识别的形式。

3） 尽量选择区分度高的列作为索引,区分度的公式是 count(distinct col)/count(*)

表示字段不重复的比例，比例越大我们扫描的记录数越少，唯一键的区分度是1，而一些状态、性别字段可能在大数据面前区分度就是0，那可能有人会问，这个比例有什么经验值吗？使用场景不同，这个值也很难确定，一般需要join的字段我们都要求是0.1以上，即平均1条扫描10条记录。

4） 索引列不能参与计算，保持列“干净”

比如 from_unixtime(create_time) = ’2014-05-29’ 就不能使用到索引，原因很简单，b+ 树中存的都是数据表中的字段值，但进行检索时，需要把所有元素都应用函数才能比较，显然成本太大。所以语句应该写成 create_time = unix_timestamp(’2014-05-29’);

5） 尽量的扩展索引，不要新建索引

比如表中已经有 a 的索引，现在要加 (a,b) 的索引，那么只需要修改原来的索引即可。

6） 定义有外键的数据列一定要建立索引

7）对于那些查询中很少涉及的列，重复值比较多的列不要建立索引

8）对于定义为text、image和bit的数据类型的列不要建立索引

9）对于经常存取的列避免建立索引

#### 索引使用的注意点：

一、 一般说来，索引应建立在那些将用于 JOIN,WHERE 判断和 ORDER BY 排序的字段上。尽量不要对数据库中某个含有大量重复的值的字段建立索引。对于一个ENUM类型的字段来说，出现大量重复值是很有可能的情况。

二、 应尽量避免在 where 子句中对字段进行 null 值判断，否则将导致引擎放弃使用索引而进行全表扫描。如：

~~~sql
select id from t where num is null
~~~

三、 最好不要给数据库留NULL，尽可能的使用 NOT NULL填充数据库。

备注、描述、评论之类的可以设置为 NULL，其他的，最好不要使用 NULL。

不要以为 NULL 不需要空间，比如：char(100) 型，在字段建立时，空间就固定了， 不管是否插入值（ NULL 也包含在内），都是占用 100个字符的空间的，如果是 varchar 这样的变长字段， null 不占用空间。
可以在 num 上设置默认值 0，确保表中 num 列没有 null 值，然后这样查询：

~~~sql
select id from t where num = 0
~~~

四、 应尽量避免在 where 子句中使用 != 或 <> 操作符，否则将引擎放弃使用索引而进行全表扫描。

五、 应尽量避免在 where 子句中使用 or 来连接条件，如果一个字段有索引，一个字段没有索引，将导致引擎放弃使用索引而进行全表扫描。如：

~~~sql
select id from t where num = 10 or Name = 'fuzhu'
~~~

可以这样查询，充分利用索引：

~~~sql
select id from t where num = 10
union all
select id from t where Name = 'fuzhu'
~~~

六、 in 和 not in 也要慎用，否则会导致全表扫描。

而且负向查询（not , not in, not like, <>, != ,!>,!< ） 不会使用索引

~~~sql
select id from t where num in(1,2,3)
~~~

对于连续的数值，能用 between 就不要用 in 了：

~~~sql
select id from t where num between 1 and 3
~~~

很多时候用 exists 代替 in 是一个好的选择，当然exists也不跑索引。
~~~sql
select num from a where num in(select num from b)
~~~
像上面的，用下面的语句替换：
~~~sql
select num from a where exists(select 1 from b where num = a.num)
~~~
七、 下面的模糊查询也将导致全表扫描：
~~~sql
select id from t where name like '%abc%'
~~~

一般情况下不鼓励使用 like 操作，如果非使用不可，如何使用也是一个问题。like “%aaa%” 不会使用索引，而 like “aaa%” 可以使用索引。

若要提高效率，可以考虑全文检索。

既然谈到模糊查询下使用索引，我们就顺便详细地讲讲吧。

1. like %keyword 索引失效，使用全表扫描。但可以通过翻转函数 + like 前模糊查询 + 建立翻转函数索引 = 走翻转函数索引，不走全表扫描。
2. like keyword% 索引有效。
3. like %keyword% 索引失效，也无法使用反向索引。
~~~sql
//可以拿我给出的数据库试一下嘛。然后用 explain 测试，就能测出有没有走索引了
select * from table where code like 'Classify_Description%'  
select * from table where code like '%Classify_Description%'  
select * from table where code like '%Classify_Description'  
~~~

八、 如果在 where 子句中使用参数，也会导致全表扫描。

因为 SQL 只有在运行时才会解析局部变量，但优化程序不能将访问计划的选择推迟到运行时；它必须在编译时进行选择。然而，如果在编译时建立访问计划，变量的值还是未知的，因而无法作为索引选择的输入项。如下面语句将进行全表扫描：
~~~sql
select id from t where num = @num
~~~
可以改为强制查询使用索引：
~~~sql
select id from t with(index(索引名)) where num = @num
~~~
应尽量避免在 where 子句中对字段进行表达式操作，这将导致引擎放弃使用索引而进行全表扫描。如：
~~~sql
select id from t where num/2 = 100
~~~
正上面的应改为：
~~~sql
select id from t where num = 100*2
~~~

九、 应尽量避免在where子句中对字段进行函数操作，这将导致引擎放弃使用索引而进行全表扫描。如：
~~~sql
select id from t where substring(name,1,3) = 'abc'       //name以abc开头的id
select id from t where datediff(day,createdate,'2005-11-30') = 0    //生成的id
~~~
应改为：
~~~sql
select id from t where name like 'abc%'
select id from t where createdate >= '2005-11-30' and createdate < '2005-12-1'
~~~
十、 不要在 where 子句中的“=”左边进行函数、算术运算或其他表达式运算，否则系统将可能无法正确使用索引。

十一、 在使用索引字段作为条件时，如果该索引是复合索引（多列索引），那么必须使用到该索引中的第一个字段作为条件时才能保证系统使用该索引，否则该索引将不会被使用，并且应尽可能的让字段顺序与索引顺序相一致。

十二、 索引并不是越多越好

索引固然可以提高相应的 select 的效率，但同时也降低了 insert 及 update 的效率，因为 insert 或 update 时有可能会重建索引，所以怎样建索引需要慎重考虑，视具体情况而定。一个表的索引数最好不要超过6个，若太多则应考虑一些不常使用到的列上建的索引是否有必要。

十三、 应尽可能的避免更新 clustered 索引数据列

十四、 尽量避免向客户端返回大数据量，若数据量过大，应该考虑相应需求是否合理。

十五、 MySQL查询只使用一个索引，因此如果where子句中已经使用了索引的话，那么 order by 中的列是不会使用索引的。

因此数据库默认排序可以符合要求的情况下不要使用排序操作；尽量不要包含多个列的排序，如果需要最好给这些列创建复合索引。


————————————————

版权声明：本文为CSDN博主「Jack__Frost」的原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/Jack__Frost/article/details/72571540