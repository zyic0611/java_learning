explain 执行计划。 mysql就不会去执行这条sql 而是返回一个报告。

1. type 查询类型 
2. possible_keys mysql优化器可能用到的树
3. key 世纪用到的索引 为null 说明全盘扫描

### type：

- all 全盘扫描 
- index 全索引扫描 没扫全表 但是查全部索引树 通常查询字段都在索引树上 就直接全部扫索引树了 不需要回表
- range 范围查找 between > < in()
- ref 索引的等值查找 可能不是唯一的 会查出多条数据 
- eq_ref  连接查询 在使用join的时候 主表在从表中只会匹配到一条记录 最多 通常是 从表使用了主键/唯一索引 作为关联条件：SELECT * FROM a JOIN b ON a.id = b.a_id;
- const 极速匹配 通过主键或者唯一索引 由于是唯一确定的 扫描到一条就停止
- system 表里只有一行数据 const的特例

性能从低到高排序

### using 

- using index: 覆盖索引 没有回表
- using index condition 索引下推 回表了 但是在二级索引上过滤一批
- using where 在mysql 的 server做where过滤  不一定没用索引 有可能用了索引  但是可能只过滤出来了部分条件 还需要丢给server层走where条件再过滤。
- using filesort ：mysql无法利用索引排序 只能把数据读取到内存中 在应用层进行一次排序 非常耗CPU和内存
  - SELECT * FROM user WHERE age = 20 ORDER BY create_time
  - 只有age是单列索引 扫出来数据后 create_time是乱序的
  - 建立联合索引 就天然有序了
- using temporary 使用临时表 为了完成查询 得建立一张内部临时表来存放中间结果 用完之后要删除
  - group by 、distinct 、多表join 再order都会引发
  - 让group by 的列走索引 比如分成 group by 分成3个 就需要建立三个临时表 但是如果走索引 顺序读取就好了 天然有序

## 总结
如果type是all 或者index 
extra是 filesort/temporary
检查where的条件 和order by排序
设计联合索引、遵循最左原则 绝大多数都会优化
联合索引 where字段在前 order by在后 必须连续匹配 才能直接索引排序 消除文件排序或者中间表
