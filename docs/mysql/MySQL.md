### 文件结构

**InnoDB 引擎的数据表底层只有.frm 和 .ibd 两种格式的文件。**

- .frm 文件用于存放表的数据结构
- .ibd 文件用于存储数据和索引的文件

**MyISAM 引擎的表底层有 .MYD，.MYI 和 .frm 三种格式的文件。**

- .frm 文件用于存放表的数据结构
- .MYD 全称为 .mydata 用于存放数据的文件
- .MYI 全称为 .myIndex 用于存放索引的文件

**InnoDB 的数据和索引是存放在同一个文件中的而 MyISAM 的数据和索引是分开存储的，这也导致两者的索引的实现方式的不同**。

---

### 索引

#### 索引类型

##### B+Tree 索引

InnoDB 的 B+Tree 索引分为主键索引和辅助索引，主键索引的叶子节点存储的是完整的数据，这种索引方式被称为聚簇索引。因为无法把数据行存放在两个不同的地方，所以一个表只能有一个聚簇索引。
辅助索引的叶子节点存储的是主键的值。所以如果在使用辅助索引进行查询数据的时候是会有回表的操作，所以我们尽量用主键进行查询。另外如果建表的时候没有设置主键的话，InnoDB 引擎会自动生成一个 rowid 作为表的主键。

MyISAM 的索引也是采用 B+ 树实现的，与 InnoDB 不同的是，MyISAM 主键索引的叶子节点和辅助索引保存的都是数据的地址，主键索引并没有保存完整的数据，所以都需要根据数据地址再次查询数据。

##### Hash 索引

哈希索引能以 O(1) 时间进行查找，但是失去了有序性:

- 无法用于排序与分组
- 只支持精确查找，无法用于部分查找和范围查找。

InnoDB 存储引擎有一个特殊的功能叫“自适应哈希索引”，当某个索引值被使用的非常频繁时，会在 B+Tree 索引之 上再创建一个哈希索引，这样就让 B+Tree 索引具有哈希索引的一些优点，比如快速的哈希查找。

##### 全文索引

MyISAM 存储引擎支持全文索引，用于查找文本中的关键词，而不是直接比较是否相等。

查找条件使用 MATCH AGAINST，而不是普通的 WHERE。

全文索引使用倒排索引实现，它记录着关键词到其所在文档的映射。

InnoDB 存储引擎在 MySQL 5.6.4 版本中也开始支持全文索引。

##### 空间数据索引

MyISAM 存储引擎支持空间数据索引(R-Tree)，可以用于地理数据存储。空间数据索引会从所有维度来索引数据， 可以有效地使用任意维度来进行组合查询。

必须使用 GIS 相关的函数来维护数据。

#### B+树原理

##### 数据结构

B Tree 指的是 Balance Tree，也就是平衡树。平衡树是一颗查找树，并且所有叶子节点位于同一层。

B+ Tree 是基于 B Tree 和叶子节点顺序访问指针进行实现，它具有 B Tree 的平衡性，并且通过顺序访问指针来提高区间查询的性能。

##### 操作

进行查找操作时，首先在根节点进行二分查找，找到一个 key 所在的指针，然后递归地在指针所指向的节点进行查找。直到查找到叶子节点，然后在叶子节点上进行二分查找，找出 key 所对应的 data。

插入删除操作会破坏平衡树的平衡性，因此在插入删除操作之后，需要对树进行一个分裂、合并、旋转等操作来维护平衡性。

##### 与红黑树的比较

红黑树等平衡树也可以用来实现索引，但是文件系统及数据库系统普遍采用 B+ Tree 作为索引结构，主要有以下两个原因:

(一)更少的查找次数

平衡树查找操作的时间复杂度和树高 h 相关，O(h)=O(logdN)，其中 d 为每个节点的出度。

红黑树的出度为 2，而 B+ Tree 的出度一般都非常大，所以红黑树的树高 h 很明显比 B+ Tree 大非常多，查找的次数也就更多。

(二)利用磁盘预读特性

为了减少磁盘 I/O 操作，磁盘往往不是严格按需读取，而是每次都会预读。预读过程中，磁盘进行顺序读取，顺序读取不需要进行磁盘寻道，并且只需要很短的磁盘旋转时间，速度会非常快。

操作系统一般将内存和磁盘分割成固定大小的块，每一块称为一页，内存与磁盘以页为单位交换数据。数据库系统将索引的一个节点的大小设置为页的大小，使得一次 I/O 就能完全载入一个节点。并且可以利用预读特性，相邻的节点也能够被预先载入。

#### 分类

1. 主键索引
2. 唯一索引
3. 普通索引
4. 全文索引
5. 组合索引

#### 技术名词

1. 回表

   > InnoDB**普通索引**的叶子节点存储主键值。
   >
   > 先定位主键，再通过主键定位数据行
   >
   > staffs表 id为主键 name, age 为组合索引
   >
   > ` select name, age, pos from staffs where name = 'July' and age = 25; `
   >
   > 可以看到查询列为name, age ,pos 由于pos 列没有建立索引,此时需要根据已经检索出的id，再去查找对应的数据行，这个过程叫做回表。

2. 覆盖索引

   > 基本介绍
   >
   > > 1. 如果一个索引包含所有需要查询的字段的值，我们称之为覆盖索引
   > > 2. 不是所有类型的索引都可以称为覆盖索引，覆盖索引必须要存储索引列的值
   > > 3. 不同的存储实现覆盖索引的方式不同，不是所有的引擎都支持覆盖索引，memory不支持覆盖索引
   >
   > 优势
   >
   > > 1. 索引条目通常远小于数据行大小，如果只需要读取索引，那么mysql就会极大的较少数据访问量
   > > 2. 因为索引是按照列值顺序存储的，所以对于IO密集型的范围查询会比随机从磁盘读取每一行数据的IO要少的多
   > > 3. 一些存储引擎如MYISAM在内存中只缓存索引，数据则依赖于操作系统来缓存，因此要访问数据需要一次系统调用，这可能会导致严重的性能问题
   > > 4. 由于INNODB的聚簇索引，覆盖索引对INNODB表特别有用

3. 最左匹配

4. 索引下推

   > staffs表	组合索引(name, age)
   >
   > ` select * from staffs where name like 'J%' and age > 25; `
   >
   > > 执行该语句有两种可能
   > >
   > > 1. 根据组合索引(name, age) 查询所有满足name以J开头的索引，然后回表查询出相应的全行数据，然后再筛选出age大于25的数据。
   > >
   > > 2. 根据组合索引(name, age) 查询所有满足name以J开头的索引，然后直接在筛选出age大于25的索引，之后再回表查询出相应的全行数据。
   > >
   > > 很明显第二种需要回表查询的全行数据比较少，这就是索引下推。MySQL默认启用索引下推。

#### 匹配方式

   1. 全值匹配

      > 全值匹配指的是和索引中的所有列进行匹配
      >
      > ` explain select * from staffs where name = 'July' and age = 23 and pos = 'dev'; `

   2. 匹配最左前缀

      > 只匹配前面的几列
      >
      > `explain select * from staffs where name = 'July' and age = 23; `
      >
      > `explain select * from staffs where name = 'July'; `

   3. 匹配列前缀

      > 可以匹配某一列的值的开头部分
      >
      > `explain select * from staffs where name like 'J%'; `

   4. 匹配范围值

      > 可以查找某一个范围的数据
      >
      > ` explain select * from staffs where name > 'Mary';`

   5. 精确匹配某一列并范围匹配另外一列

      > 可以查询第一列的全部和第二列的部分
      >
      > `explain select * from staffs where name = 'July' and age > 25;`

   6. 只访问索引的查询

      > 查询的时候只需要访问索引，不需要访问数据行本质上就是覆盖索引
      >
      > `explain select name, age,pos from staffs where name = 'July' and age = 25 and post = 'dev';`

#### 索引失效情况

1. like 以%开头，索引无效；当like前缀没有%，后缀有%时，索引有效。
2. or语句前后没有同时使用索引。当or左右查询字段只有一个是索引，该索引失效，只有当or左右查询字段均为索引时，才会生效。
3. 组合索引，不是使用第一列索引，索引失效。
4. 数据类型出现隐式转化。如varchar不加单引号的话可能会自动转换为int型，使索引无效，产生全表扫描。
5. 在索引列上使用 IS NULL 或 IS NOT NULL操作。
6. 在索引字段上使用not，<>，!=。不等于操作符是永远不会用到索引的，因此对它的处理只会产生全表扫描。 优化方法： key<>0 改为 key>0 or key<0。
7. 对索引字段进行计算操作、字段上使用函数。
8. 当全表扫描速度比索引速度快时，mysql会使用全表扫描，此时索引失效。


---

### 锁

1. InnoDB 支持表级锁和行级锁，MyISAM 只支持表级锁。
2. InnoDB 的锁分为共享锁 S 和独占锁 X，意向共享锁 IS和意向独占锁 IX，记录锁，间隙锁以及 next-key 锁。

#### InnoDB 常见的几种锁机制

1. **共享锁和独占锁**（Shared and Exclusive Locks），InnoDB 通过共享锁和独占所两种方式实现了标准的行锁。共享锁（S 锁）：允许事务获得锁后去读数据，独占锁（X 锁）：允许事务获得锁后去更新或删除数据。一个事务获取的共享锁 S 后，允许其他事务获取 S 锁，此时两个事务都持有共享锁 S，但是不允许其他事务获取 X 锁。如果一个事务获取的独占锁（X），则不允许其他事务获取 S 或者 X 锁，必须等到该事务释放锁后才可以获取到。
2. **意向锁**（Intention Locks），上面说过 InnoDB 支持行锁和表锁，意向锁是一种表级锁，用来指示接下来的一个事务将要获取的是什么类型的锁（共享还是独占）。意向锁分为意向共享锁（IS）和意向独占锁（IX），依次表示接下来一个事务将会获得共享锁或者独占锁。意向锁不需要显示的获取，在我们获取共享锁或者独占锁的时候会自动的获取，意思也就是说，如果要获取共享锁或者独占锁，则一定是先获取到了意向共享锁或者意向独占锁。 意向锁不会锁住任何东西，除非有进行全表请求的操作，否则不会锁住任何数据。存在的意义只是用来表示有事务正在锁某一行的数据，或者将要锁某一行的数据。
3. **记录锁**（record Locks），锁住某一行，如果表存在索引，那么记录锁是锁在索引上的，如果表没有索引，那么 InnoDB 会创建一个隐藏的聚簇索引加锁。所以我们在进行查询的时候尽量采用索引进行查询，这样可以降低锁的冲突。
4. **间隙锁**（Gap Locks），间隙锁是一种记录行与记录行之间存在空隙或在第一行记录之前或最后一行记录之后产生的锁。间隙锁可能占据的单行，多行或者是空记录。通常的情况是我们采用范围查找的时候，比如在学生成绩管理系统中，如果此时有学生成绩 60，72，80，95，一个老师要查下成绩大于 72 的所有同学的信息，采用的语句是` select * from student where grade > 72 for update`，这个时候 InnoDB 锁住的不仅是 80，95，而是所有在 72-80，80-95，以及 95 以上的所有记录。为什么会 这样呢？实际上是因为如果不锁住这些行，那么如果另一个事务在此时插入了一条分数大于 72 的记录，那会导致第一次的事务两次查询的结果不一样，出现了幻读。所以为了在满足事务隔离级别的情况下需要锁住所有满足条件的行。
5. **Next-Key Locks**，NK 是一种记录锁和间隙锁的组合锁。是 3 和 4 的组合形式，既锁住行也锁住间隙。并且采用的左开右闭的原则。**InnoDB 对于查询都是采用这种锁的**。

---

### 事务

InnoDB 与 MyISAM 另一个最大的区别就是 InnoDB 支持事务，而 MyISAM 不支持事务。

#### 四大特性ACID

1. A: 原子性。原子性表示事务的操作是不可分割的，事务内的一系列操作全部成功事务才成功，任何一个操作失败事务都是失败，必须回滚。
2. C: 一致性。一致性表示事务在操作前和操作后数据都是处于一致的状态。只表示在事务的操作前后的数据一致，但是并不是代表是正确。
3. I：隔离性。事务的并发操作是完全隔离的，不同的事务之间不会相互有影响。事务的隔离级别有四种
4. D：持久性。事务操作结束过后，对数据的更改是可以持久化的，不管事务的操作是成功还是失败，事务日志都能保证事务的持久性。

#### 问题

1. **脏读：指在事务 A 处理过程里读取到了事务 B 未提交的事务中的数据**。比如在转账的例子中：小 A 开了一个事务给小 B 转了1000 块，还没提交事务的时候就跟小 B 说，钱已经到账了。这个时候小 B 去看了一下余额，发现果真到账了（然后就开开心心刷抖音去了），这个时候小 A 回滚了事务，把 1000 块又搞回去了。小 B 刷完抖音再去看下余额，发现钱又不见了。
2. **不可重复读：指在一个事务执行的过程中多次查询某一数据的时候结果不一致的现象，由于在执行的过程中被另一个事务修改了这个数据并提交了事务**。比如：事务 A 第一次读小明的年龄是 18 岁，此时事务 B 将小明的年龄改成了 20 并提交了，这个时候事务 A 再次读取小明的年龄发现是 20，这就是同一条数据不可重复读。
3. **幻读：幻读通常指的是对一批数据的操作完成后，有其他事务又插入了满足条件的数据导致的现象**。比如：事务 A 将数据库性别为男的状态都改成1 表示有钱人，这个时候事务 B 又插入了一条状态为 0 没钱人的记录，这个时候，用户再查看刚刚修改的数据时就会发现还有一行没有修改，这就出现了幻读。幻读往往针对 insert 操作，脏读和不可重复读针对 select 操作。

#### 隔离级别

1. Read uncommitted (读未提交)：表示一个事务内可以读取到另一个事务未提交的数据内容，会出现脏读，不可重复度，幻读。
2. Read committed (读已提交)：表示一个事务内可以读取另一个事务已经提交的内容，会出现不可重复读和幻读。
3. Repeatable read (可重复读)：表示一个事务内两次相同条件读的内容一致，但是会出现幻读。`MySQL默认`
4. Serializable (串行化)：没有问题，但是效率低下。

---

### 执行计划

#### select type - 查询类型

表示一行查询的类型，simple 为简单查询，union 为联合查询等。

#### type - 优化级别

- **const** 确定只有一行匹配，级别最高
- **eq_ref** 最多只返回一条符合条件的记录，通过使用两个表中有关联字段时提现
- **ref** 通过普通索引查询匹配很多行时的类型
- **fulltext** 全文索引
- **ref_or_null** 跟 ref 类似，不过多一个列不能为 null 的条件
- **index_merge** 使用了联合索引
- **range** 范围扫描
- **index** 类似全表扫描，只扫描表的时候按照索引次序进行，而不是行
- **ALL** 全表扫描

#### possible_key

MySQL 可能采用的索引，但是并不一定用到。

#### key

MySQL 真实采用的索引。

#### ref

哪个字段或常数与 key 一起被使用。

#### rows

预估扫描的行数，只能参考，不准确。

#### filtered

表示此查询条件所过滤的数据的百分比。

#### extra

额外的信息，包括是否排序，是否采用临时表等。