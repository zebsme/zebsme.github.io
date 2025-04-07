+++

title = "Implement simple relational database"
date = 2025-03-10
authors = ["zeb"]

+++

- [ ] Logical Plan
- [ ] 

## Sql Support

### Tokenizer

第一步将SQL查询字符串转换为表示关键字、字面量、操作符。。。的tokens：

TODO

然后需要一个Tokenizer，给定输入SELECT a+b FROM C 我们期望输出如下：

TODO

### 

在一般的数据库系统中，一个sql从输入到执行的流程如下图所示：

lexer -> parser -> logical plan -> physical plan -> ...

###### 

从广义上来讲，query engine -> data source。

data source可以是in-memory 也可以是disk 也可以是



How query engine work

data type



How query engine works



disk-based 以及 memory

- [ ] 实现基于B+树的磁盘存储

type system

Logical Plan

interface{

​	schema()

​	children()

}

用零个或者多个logical plan作为输入。可以通过children()接口直接得到子逻辑计划，便于遍历。

## 实现query engine

实现一个语句，需要在执行流程的基础上进行考虑。以order by语句为例:

`[ORDER BY col_name [asc | desc] [, ...]]`



lexer添加DESC ASC ORDER BY关键字

ast中Select结构体添加order_by字段 Vec<String, Direction>

再增加executor中的Order node以及execute实现。

projection

lexer AS

Select添加select Vec<Expression，Option<String >>

Projection node

### Cross Join



```rust
Node::NestedLoopJoin {
	left,
	right,
}
```

笛卡尔积

### Inner Join



```rust
Node::NestedLoopJoin {
	left,
	right,
    predicate
}
```

判断equal条件。判断的时候有一个递归。

### outer join

right join和left join都属于outer join。build plan的时候，如果是right join的话，直接交换位置，execute的时候只用处理一种情况。

```rust
Node::NestedLoopJoin {
	left,
	right,
	predicate,
	outer,
}
```

group by



order by

过滤条件

having (对group by后的结果进行过滤)

索引支持

调研主流数据库的索引

MVCC

基于快照隔离

获取的当前活跃事务列表可以视为一个快照，描述事务在开始时的全局状态。

Txnwrite是中间状态，提交的时候要删掉txnwrite以及txnactive

rollback的时候删掉version

scan_prefix方法

commit和rollback的时候获取txnwrite的位置 commit直接删掉txnwrite。rollback删掉txnwrite对应的version(key) 获取活跃列表也需要scan_prefix

kv中list_table需要用table_name去扫描

### 索引的支持 

一行数据通过表名和主键值进行编码作为唯一id

table_name column_name column_value -> index(HashSet) 存放的是pk对应的value？

### HashJoin

###  



内存设计

磁盘设计

Sql引擎

## MVCC实现

version 是否可见

is_visible

if self.active_versions.contains(key) {

​	return false

} else{

​	version <= self.version

}

insert into

先去get_table

get table的流程

executor都通用的是txn的接口进行读写操作

txn.get(Key::Table(table_name))

txn.get中设定一个range，from 0 to当前txn id,MvccKey(key, 0) MvccKey(key, tnx.id)，这里的key是上面的Key::Table(table_name)，通过这个range在BTreeMap即keydir返回一个iterator。可以进行遍历，返回最大的visible version对应的value。再将value des为Table

Table{

name: String

columns: Vec<Column>

}

pub struct Column {
    pub name: String,
    pub datatype: DataType,
    pub nullable: bool,
    pub default: Option<Value>,
    pub primary_key: bool,
    pub index: bool,
}

这里的table 仅仅是schema

TODO index in create_row

再排列一下给出的列值以及列名

接着create row Key::Row(table_name, pk_id)来标志一个row。检查这个行的有效性字段是否匹配，检查这个row是否存在，如果存在且可见，dumplicate data。再txn.set。最后维护索引。

