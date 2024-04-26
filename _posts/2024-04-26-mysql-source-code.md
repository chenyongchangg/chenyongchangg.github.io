# mysql source code read
# offset

## 1\. 准备工作

创建测试表：

```SQL
1CREATE TABLE `t1` (
2  `id` int unsigned NOT NULL AUTO_INCREMENT,
3  `str1` varchar(255) NOT NULL DEFAULT '',
4  `i1` int NOT NULL DEFAULT '0',
5  PRIMARY KEY (`id`) USING BTREE
6) ENGINE=InnoDB DEFAULT CHARSET=utf8mb3;
```

插入测试数据：

```SQL
1INSERT INTO t1(id, str1, i1) VALUES
2(1, 's1', 10),
3(2, 's2', 20),
4(3, 's3', 30),
5(4, 's4', 40),
6(5, 's5', 50),
7(6, 's6', 60),
8(7, 's7', 70),
9(8, 's8', 80);
```

示例 SQL：

```SQL
1select * from t1 limit 5, 2
```

## 2\. 整体介绍

我们先通过 explain 来看一下执行计划：

![](https://www.bigspring.cn/post/16809084580012/media/16809099286336.jpg)

从 explain 输出可以看到，执行计划比较简单，SQL 执行过程包含 2 个迭代器：

+   `Limit/Offset`，对应 LimitOffsetIterator 迭代器。

+   `Table scan`，对应 TableScanIterator 迭代器。


代码执行时堆栈如下：

```CPP
 1| > handle_connection(void*) sql/conn_handler/connection_handler_per_thread.cc:302
 2| + > do_command(THD*) sql/sql_parse.cc:1439
 3| + - > dispatch_command(...) sql/sql_parse.cc:2036
 4| + - x > dispatch_sql_command(THD*, Parser_state*) sql/sql_parse.cc:5322
 5| + - x = > mysql_execute_command(THD*, bool) sql/sql_parse.cc:4688
 6| + - x = | > Sql_cmd_dml::execute(THD*) sql/sql_select.cc:578
 7| + - x = | + > Sql_cmd_dml::execute_inner(THD*) sql/sql_select.cc:778
 8| + - x = | + - > Query_expression::execute(THD*) sql/sql_union.cc:1823
 9| + - x = | + - x > // 查询入口
10| + - x = | + - x > Query_expression::ExecuteIteratorQuery(THD*) sql/sql_union.cc:1770
11| + - x = | + - x = > // 实现 limit, offset
12| + - x = | + - x = > LimitOffsetIterator::Read() sql/iterators/composite_iterators.cc:128
13| + - x = | + - x = | > // 从存储引擎读取一条记录
14| + - x = | + - x = | > TableScanIterator::Read() sql/iterators/basic_row_iterators.cc:218
```

## 3\. 源码分析

TableScanIterator 迭代器用于从存储引擎读取记录，留到以后的文章介绍。

limit, offset 由 `LimitOffsetIterator` 迭代器实现，我们会介绍两个方法的代码：

+   `Query_expression::ExecuteIteratorQuery(THD*)`，这是查询入口方法，介绍了它，流程才算完整。

+   `LimitOffsetIterator::Read()`，limit, offset 的逻辑都在这个方法里实现。


### 3.1 ExecuteIteratorQuery()

```CPP
 1// sql/sql_union.cc
 2bool Query_expression::ExecuteIteratorQuery(THD *thd) {
 3  ...
 4  {
 5    ...
 6    for (;;) {
 7      // 从存储引擎读取一条记录
 8      int error = m_root_iterator->Read();
 9      DBUG_EXECUTE_IF("bug13822652_1", thd->killed = THD::KILL_QUERY;);
10
11      // 读取出错，直接返回
12      if (error > 0 || thd->is_error())  // Fatal error
13        return true;
14      // error < 0
15      // 表示已经读完了所有符合条件的记录
16      // 查询结束
17      else if (error < 0)
18        break;
19      // SQL 被客户端干掉了
20      else if (thd->killed)  // Aborted by user
21      {
22        thd->send_kill_message();
23        return true;
24      }
25      ...
26      // 发送数据给客户端
27      if (query_result->send_data(thd, *fields)) {
28        return true;
29      }
30      ...
31    }
32  }
33  ...
34}
```

从以上代码可以看到，select 查询入口方法的主体是一个无限 for 循环。

每一轮循环都会调用 `m_root_iterator->Read()` 方法从存储引擎读取一条记录。

对于示例 SQL 来说，m\_root\_iterator->Read() 就是 LimitOffsetIterator::Read()。

for 循环会一直执行，直到 m\_root\_iterator->Read() 的返回值命中以下任意一个条件才会结束：

+   `if (error > 0 || thd->is_error())`，读取出错了，以错误状态结束查询。

+   `if (error < 0)`，已经读完所有符合条件的记录，以正常状态结束查询。

+   `if (thd->killed)`，SQL 被客户端通过 kill <query\_id> 干掉了，中止查询。

    > `<query_id>` 为 `show processlist` 中的 Id 字段。


for 循环中，每次从存储引擎读取到一条记录，都会调用 `query_result->send_data(thd, *fields)` 方法。

对于示例 SQL 来说，这个方法的行为就是把记录发送给客户端。

### 3.2 LimitOffsetIterator::Read()

```CPP
 1// sql/iterators/composite_iterators.cc
 2int LimitOffsetIterator::Read() {
 3  // 这个 if 括号里的条件理解起来会有点困难
 4  // 所以被省略了，眼不见为净
 5  //【重点】只有读取第一条和最后一条记录时才会进入这个 if 分支
 6  if (...) {
 7    ...
 8    // m_needs_offset = true
 9    // 表示 SQL 语句中指定了 offset
10    if (m_needs_offset) {
11      ...
12      // 循环从存储引擎读取 m_offset 条记录
13      // 每读取到一条记录，直接丢弃
14      for (ha_rows row_idx = 0; row_idx < m_offset; ++row_idx) {
15        // 读取一条记录之后
16        // 如果没有出错，就接着读取下一条记录
17        int err = m_source->Read();
18        // 读取出错，直接返回错误码
19        if (err != 0) {
20          return err;
21        }
22        ...
23      }
24      // 读取 m_offset 条记录并丢弃之后
25      // 把 m_seen_rows 设置为已读取记录数 
26      m_seen_rows = m_offset;
27      // 然后把 m_needs_offset 设置为 false
28      // 表示不需要再处理 offset 逻辑了（因为已处理完成）
29      // 下次读取时也就不需要再跳过 m_offset 条记录了
30      m_needs_offset = false;
31      ...
32    }
33    // 如果已经读取了 m_limit 条记录
34    // 就返回 -1，表示读取结束
35    // m_limit = SQL 中的 limit + offset
36    if (m_seen_rows >= m_limit) {
37      ...
38      return -1;
39    }
40  }
41
42  // 读取需要返回给客户端的记录
43  const int result = m_source->Read();
44  ...
45  // 已读取记录数加 1
46  ++m_seen_rows;
47  // 返回当前读取的记录
48  // 给 Query_expression::ExecuteIteratorQuery() 方法
49  return result;
50}
```

除了处理 offset 逻辑之外，`LimitOffsetIterator::Read()` 每次只读取一条记录，这个方法的核心逻辑分为三部分：

**第 1 部分**：`if (m_needs_offset)`，SQL 语句中指定了 offset，返回第一条记录给客户端之前，需要读取 offset 条记录并丢弃，从第 `offset + 1` 条记录开始返回给客户端。

这部分的主要逻辑是一个 for 循环，会循环 offset 次，每次读取一条记录。

如果读取成功，就接着读取下一条记录，而不会对这条记录做任何操作，也就相当于丢弃了。

如果读取失败，直接返回错误码，读取结束，客户端会收到报错信息。

**第 2 部分**：`if (m_seen_rows >= m_limit)`，表示已经读取了 m\_limit 条记录，返回 -1 表示读取正常结束。

> m\_limit = SQL 中的 limit + offset。

**第 3 部分**：`result = m_source->Read()` 从存储引擎读取一条记录，然后，把结果返回给 `Query_expression::ExecuteIteratorQuery()` 方法。

## 4\. 总结

limit, offset 逻辑比较简单，全部由 `LimitOffsetIterator::Read()` 实现，核心逻辑总结如下：

+   从存储引擎读取返回给客户端的第 1 条记录之前，会先读取 offset 条记录并丢弃，然后再读取一条记录，用于返回给客户端。

+   从存储引擎读取第 2 ~ limit + offset 条记录时，每读取一条记录，都返回给 `Query_expression::ExecuteIteratorQuery()`，由该方法把记录返回给客户端。

+   读取 limit + offset 条记录之后，返回 -1 表示读取流程正常结束。


从 LimitOffsetIterator::Read() 的实现逻辑来看，offset 越大，读取之后被丢弃的记录就越多，读取这些记录所做的都是无用功。

为了提高 SQL 的执行效率，可以通过改写 SQL 让 offset 尽可能小，理想状态是 offset = 0。



# select

### **1\. 整体介绍**

对于 `select * from table` 中的星号，我们再熟悉不过了：它告诉 MySQL 返回表所有字段的内容。

MySQL 服务端收到 select 语句之后，会在 server 层把星号展开为表中的所有字段，然后告诉存储引擎返回这些字段的内容。

对于存储引擎来说，它只需要按照 server 层的要求返回指定字段的内容即可，它不知道（也不需要知道）客户端是要求返回表中所有字段，还是部分字段的内容。

`select *` 中的星号展开为表中所有字段涉及 2 个阶段：

+   **词法 & 语法分析阶段**：标记 select 字段列表中包含几个星号。
+   **查询准备阶段**：把星号展开为表中所有字段。

### **2\. 源码分析**

#### **2.1 Item\_asterisk::itemize()**

```javascript
// sql/item.cc
bool Item_asterisk::itemize(Parse_context *pc, Item **res) {
  ...
  pc->select->with_wild++;
  return false;
}
```

多表连接时，select 字段列表中可能会包含多个星号，词法 & 语法分析阶段，每碰到 select 字段列表中的一个星号，`Item_asterisk::itemize()` 就会给 `pc->select->with_wild` 属性加 1。

`pc->select` 是 `Query_block` 对象的指针，定义如下：

```javascript
// sql/parse_tree_node_base.h
struct Parse_context {
  ...
  Query_block *select; ///< Current Query_block object
  ...
};
```

后面 `Query_block::prepare()` 访问的 `with_wild` 属性就是这里的 `pc->select->with_wild`。

#### **2.2 Query\_block::prepare()**

```javascript
// sql/sql_resolver.cc
bool Query_block::prepare(THD *thd, mem_root_deque<Item *> *insert_field_list) {
  ...
  if (with_wild && setup_wild(thd)) return true;
  ...
}
```

prepare() 方法中，关于 `select *` 的逻辑比较简单，就这一行。

如果 with\_wild 大于 0，则调用 `setup_wild(thd)`，处理 select 字段列表中星号展开为表中所有字段的逻辑。

#### **2.3 Query\_block::setup\_wild()**

```javascript
// sql/sql_resolver.cc
bool Query_block::setup_wild(THD *thd) {
  ...
  // 从 select 字段列表中的第 1 个字段开始处理
  // 满足 2 个条件中的任意一个就结束循环：
  // 1. with_wild > 0 为 false，
  //    说明已处理完所有星号，结束循环
  // 2. it != fields.end() 为 false，
  //    说明已经处理了所有字段，结束循环
  for (auto it = fields.begin(); with_wild > 0 && it != fields.end(); ++it) {
    Item *item = *it;
    // item->hidden = true
    // 表示 select 字段列表中的这个字段
    // 是查询优化器给【偷偷】加上的
    // 肯定不会是星号，直接跳过
    if (item->hidden) continue;
    Item_field *item_field;
    // Item::FIELD_ITEM 说明当前循环的字段
    // 是个普通字段，不是函数、子查询等
    // 那它就有可能是星号，需要通过 item_field->is_asterisk() 
    // 进一步判断是否是星号
    if (item->type() == Item::FIELD_ITEM &&
        (item_field = down_cast<Item_field *>(item)) &&
        // 如果 item_field 对应的字段是星号
        // item_field->is_asterisk() 会返回 true
        item_field->is_asterisk()) {
      assert(item_field->field == nullptr);
      // 只有 create view as ... 中的 select 语句
      // any_privileges 为 true
      // 其它情况下，它的值为 false
      // insert_fields() 方法中会用到
      const bool any_privileges = item_field->any_privileges;
      // 如果当前 Query_block 对应的是子查询
      // master_query_expression()->item
      // 指向主查询中该子查询所属的 where 条件
      Item_subselect *subsel = master_query_expression()->item;
      ...
      // 当前 Query_block 是 exists 子查询
      // 并且子查询中不包含 having 子句
      // 则可以把子查询中的星号替换为常量
      if (subsel && subsel->substype() == Item_subselect::EXISTS_SUBS &&
          !having_cond()) {
        ...
        *it = new Item_int(NAME_STRING("Not_used"), 1,
                           MY_INT64_NUM_DECIMAL_DIGITS);
      } else {
      // 不满足 if 中的条件
      // 则需要调用 insert_fields()
      // 把星号展开为表中所有字段
        assert(item_field->context == &this->context);
        if (insert_fields(thd, this, item_field->db_name,
                          item_field->table_name, &fields, &it, any_privileges))
          return true;
      }

      // 每处理完 select 字段列表中的一个星号
      // with_wild 就减 1
      // 减到 0 之后，就说明所有星号都已经处理过了
      with_wild--;
    }
  }

  return false;
}
```

`Query_block::setup_wild()` 的主体逻辑是迭代 select 字段列表中的每个字段，遇到星号就处理，不是星号就忽略，星号的处理逻辑有 2 种：

**第 1 种**：满足 `if (subsel && ...)` 条件，说明 select 语句是 where 条件中的 exists 子查询，并且子查询中不包含 having 子句。这种场景下，select 字段列表中的星号可以被替换为常量，而不需要展开为表的所有字段。

`*it = new Item_int(...)` 创建了一个代表常量的字段对象，字段名为 `Not_used`，字段值为 `1`，用于替换 select 字段列表中的星号。

这种场景的示例 SQL 如下：

```javascript
select st1, i1 from t1 where exists(
  select * from t2 where t1.i1 = t2.i1
)
```

子查询只需要判断 t2 表中是否存在满足 `t1.i1 = t2.i1` 的记录，而不需要读取 t2 表的所有字段，因为读取了所有字段，也用不上，纯属浪费，所以，星号也就可以被替换成常量了。替换之后的 SQL 相当于这样：

```javascript
select st1, i1 from t1 where exists(
  select 1 from t2 where t1.i1 = t2.i1
)
```

> 实际上，子查询执行过程中，server 层会要求存储引擎返回 t2 表的 i1 字段内容，用于判断 t2 表中是否存在满足 `t1.i1 = t2.i1` 的记录。 这个逻辑是 server 层自主实现的，和 `select *` 中的星号展开为表中所有字段的逻辑不相关，我们知道有这个逻辑就可以，不展开介绍了。

**第 2 种**：不满足 `if (subsel && ...)` 条件，就需要调用 `insert_fields()`，把 select 字段列表中的星号展开为表的所有字段。

#### **2.4 insert\_fields()**

```javascript
// sql/sql_base.cc
bool insert_fields(THD *thd, Query_block *query_block, const char *db_name,
                   const char *table_name, mem_root_deque<Item *> *fields,
                   mem_root_deque<Item *>::iterator *it, bool any_privileges) {
  ...
  bool found = false;

  Table_ref *tables;
  // 按照 select 语句中表的出现顺序
  // 初始化表的迭代器
  Tables_in_user_order_iterator user_it;
  user_it.init(query_block, table_name != nullptr);

  while (true) {
    // 从迭代器中获取下一个需要处理的表
    tables = user_it.get_next();
    // tables == nullptr 说明迭代结束，结束循环
    if (tables == nullptr) break;
    // 表中的字段迭代器
    Field_iterator_table_ref field_iterator;
    TABLE *const table = tables->table;

    assert(tables->is_leaf_for_name_resolution());

    // if 进行 2 个条件判断，任何一个不满足则跳过当前表：
    // 1. table_name 不为 NULL 说明星号前面指定了表名
    //    比较星号前面的表名和当前迭代的表名是否相同
    // 2. db_name 不为 NULL 说明星号前面指定了数据库名
    //    比较星号前面的数据库名和当前迭代的表所属的数据库名是否相同
    if ((table_name &&
         my_strcasecmp(table_alias_charset, table_name, tables->alias)) ||
        (db_name && strcmp(tables->db, db_name))) 
      continue;

    // 以下 2 种情况都满足，需要检查
    // 当前连接用户是否有表中所有字段的 select 权限：
    // 1. !any_privileges 为 true
    //    说明当前 select 语句
    //    不是 create view as ... 中的 select 语句
    // 2. 当前连接用户没有表的 select 权限，
    if (!any_privileges && !(tables->grant.privilege & SELECT_ACL)) {
      field_iterator.set(tables);
      if (check_grant_all_columns(thd, SELECT_ACL, &field_iterator))
        return true;
    }
    ...
    // 初始化字段迭代器
    field_iterator.set(tables);
    // 迭代当前表的每一个字段
    for (; !field_iterator.end_of_fields(); field_iterator.next()) {
      // 根据字段对象创建 Item
      Item *const item = field_iterator.create_item(thd);
      if (!item) return true; /* purecov: inspected */
      assert(item->fixed);
      ...
      // found 的初始值为 false，
      // 表示这是表中第 1 个字段
      // 用该字段的 Item 对象替换星号的 Item 对象
      if (!found) {
        found = true;
        **it = item; /* Replace '*' with the first found item. */
      } else {
      // 表中第 2 个及以后的字段时，
      // Item 对象赋值给 it + 1 指向的位置
      // 也就是加入了 select 字段列表
        /* Add 'item' to the SELECT list, after the current one. */
        *it = fields->insert(*it + 1, item);
      }
      ...
    }
  }
  ...
```

`insert_fields()` 的主要逻辑如下：

按照 select 语句中表的出现顺序迭代每个表，每迭代一个表，都会判断该表名和星号前面的表名（`如果有`）是否相同，以及该表所属的数据库名和星号前面的数据库名是否相同（`如果有`）。

如果当前迭代的表名、表所属的数据库名和星号前面的表名、数据库名都相同，接下来会进行访问权限检查。

如果当前连接用户有表的 select 权限，说明它对表中的所有列都有查询权限，否则，需要调用 `check_grant_all_columns(...)`，检查它对表中每一个字段是否有 select 权限。

通过权限检查之后，就开始迭代表中的每个字段，每迭代一个字段，都根据该字段构造一个 Item 对象，并把 Item 对象加入 select 字段列表。

### **3\. 总结**

`select *` 中的星号展开为表中所有字段涉及词法 & 语法分析阶段、查询准备阶段，总结如下：

+   迭代 select 字段列表中的每个字段。
+   碰到星号会判断是否需要展开为表的所有字段。
+   如果需要展开，则按照 select 语句中表的出现顺序迭代每个表。
+   迭代每个表时，检查当前连接用户是否有该表或表中所有字段的 select 权限。
+   通过权限检查之后，把当前迭代的表的字段逐个加入 select 字段列表。
