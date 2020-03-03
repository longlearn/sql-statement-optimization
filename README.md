# sql-statement-optimization
MySQL的sql语句优化
### 常用sql语句优化
> 1. 不要写一些没有意义的查询，如需要生成一个空表结构：
  >> ``` sql
  >> select col1,col2 into #t from t where 1=0
  >> ```
  >> 这类代码不会返回任何结果集，但是会消耗系统资源的，应改成这样：
  >> ``` sql 
  >> create table #t(...)
  >> ```
> 2. 很多时候用 exists 代替 in 是一个好的选择,但要分具体情况：
  >> 如果两个表中一个较小，一个大表，则子查询表大的用exists，子表小的用in。
  >> 例如：表A（小表），表B(大表）
  >> ```sql
  >> select * from A where cc in (select cc from B)
  >> ```
  >> 效率低，用到了A表cc列的索引
  >> ``` sql
  >> select * from A where cc exists (select cc from B where cc=A.cc)
  >> ```
  >> 效率高，用到了B表cc列的索引  
  >> 反之，若A为大表，情况相反
> 3. 无论哪个表大，用 `not exists` 都比 `not in` 效率要高
> 4. 应尽量避免在 `where` 子句中使用 `or` 来连接条件，否则将导致引擎放弃使用索引而进行全表扫描，如：
  >> ``` sql
  >> select id from t where num = 10 or num =20
  >> ```
  >> 应该为：
  >> ``` sql
  >> select id from t where num =10 
  >> union all
  >> select id from t where num =20 
  >> ```
> 5. 应尽量避免在 `where` 子句中对字段使用表达式操作，这会导致引擎放弃索引，进行全表扫描。如：
  >> ``` sql
  >> select id from t where num/12 = 100 
  >> ```
  >> 应该为:
  >> ``` sql 
  >> select id from t where num = 100 * 2 
  >> ```
> 6. 应尽量避免在 ·where· 子句中对字段进行函数操作，这会导致引擎放弃索引，进行全表扫描。如：
  >> ``` sql
  >> select id from t where substring(name,1,3) = 'abc'
  >> //查询name字段以abc开头的id
  >> select id form t where datediff(day,createdate,'2020-03-03') = 0
  >> //查询createdate字段在2020-03-03生成的id
  >> ```
  >> 应该为：
  >> ``` sql
  >> select id from t where name like 'abc%'
  >> select id from t where createdate >= '2020-03-03' and createdate < '2020-03-04'
  >> ```
  >> 注意：下面的查询也会导致全表扫描 
  >> ``` sql
  >> select id from t where name like  '%abc%'
  >> ```
  >> 对于 `like` ，匹配字符不能以 `%` 开头
> 7. 对查询进行优化，应尽量避免全表扫描，首先应先考虑在 `where` 和 `order by` 涉及的列上建立索引
> 8. 应尽量避免在where子句中使用 `!=` 或 `<>` 操作符,否住将进行全表扫描
> 9. 尽量使用表变量来代替临时表。如果表变量包含大量数据，请注意索引非常有限(只有主键索引)。
> 10. 应尽量避免频繁创建和删除临时表，以减少系统表资源的消耗
> 11. 应尽量避免在 where 子句中对字段进行 null 值 判断，否则将导致引擎放弃使用索引而进行全表扫描，如：
  >> ``` sql
  >> select id from t where num is null
  >> ```
  >> 可以在num上设置默认值0，确保表中num列没有null值，然后这样查询：
  >> ``` sql 
  >> select id from t where num=0
  >> ```
> 12. 并不是所有索引对查询都有效，SQL是根据表中数据来进行查询优化的，当索引列有大量数据重复时，SQL查询可能不会去利用索引.  
      如一表中有字段sex，male、female几乎各一半，那么即使在sex上建了索引也对查询效率起不了作用。
> 13. 在新建临时表时，如果一次性插入数据量很大，那么可以使用 `select into` 代替 `create table`，避免造成大量 log ，以提高速度;
      如果数据量不大，为了缓和系统表的资源，应先 `create table`，然后 `insert`
> 14. 如果使用到了临时表，在存储过程的最后务必将所有的临时表显式删除，先 `truncate table` ，然后 `drop table` ，
      这样可以避免系统表的较长时间锁定.
> 15. 绝对不要轻易用 `order by rand()` ，很可能会导致mysql的灾难！！
> 16. 尽量使用数字型字段，若只含数值信息的字段尽量不要设计为字符型，这会降低查询和连接的性能，并会增加存储开销。  
      这是因为引擎在处理查询和连接时会逐个比较字符串中每一个字符，而对于数字型而言只需要比较一次就够了
> 17. 应尽可能的避免更新 clustered 索引数据列，
      因为 clustered 索引数据列的顺序就是表记录的物理存储顺序，一旦该列值改变将导致整个表记录的顺序的调整，会耗费相当大的资源。
      若应用系统需要频繁更新 clustered 索引数据列，那么需要考虑是否应将该索引建为 clustered 索引
> 18. 索引并不是越多越好，索引固然可以提高相应的 select 的效率，但同时也降低了 insert 及 update 的效率，
      因为 insert 或 update 时有可能会重建索引，所以怎样建索引需要慎重考虑，视具体情况而定。
      一个表的索引数最好不要超过6个，若太多则应考虑一些不常使用到的列上建的索引是否有必要.
> 19. 使用基于游标的方法或临时表方法之前，应先寻找基于集的解决方案来解决问题，基于集的方法通常更有效。
> 20. 永远别要用复杂的mysql语句来显示你的聪明。就我来说，看到一次关联了三，四个表的语句，只会让人觉得很不靠谱。
> 21. 拆分大的 DELETE 或 INSERT 语句。因为这两个操作是会锁表的，表一锁住了，别的操作都进不来了，就我来说 有时候我宁愿用for循环来一个个执行这些操作。
> 22. 尽可能使用 `varchar/nvarchar` 代替 `char/nchar`，因为首先变长字段存储空间小，节省存储空间，其次对查询来说，在一个相对较小的字段内效率显然更高。
> 23. 任何情况都不要使用到 `select * from t`,用具体的字段列表。
### MySQL无法使用索引的情况总结
> 1. 字段使用函数，将无法使用索引
> 2. `join` 语句中 `join` 条件字段类型不一致的时候，无法使用
> 3. 复合索引的情况下，如果查询条件不包含索引列的最左边部分，即不满足 `最左前缀原则`，则不会使用索引
> 4. 扫描数据超过30%，都会走全表扫描
> 5. 以 `%` 开头的like查询
> 6. 数据类型出现隐式转换的时候也不会使用索引，特别是当列类型是字符串，那么一定记得在where条件中把字符串常量值用引号引起来，
     否则即便这个列上有索引，MySQL也不会用到，因为MySQL默认把输入的常量值进行转换以后才进行检索
> 7. 用or分割开的条件，如果 or前的条件中的列有索引，而后面的列中没有索引，那么涉及的索引都不会被用到
