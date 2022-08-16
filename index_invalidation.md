# 索引失效

## 本文主要内容
* 唯一索引字段包含 `null`
* 逻辑删除表中增加唯一约束
* 重复历史数据如何加唯一索引
* 给大字段加唯一索引
* 批量插入数据

## 唯一索引字段包含 `null`
如果唯一索引的字段中，出现了null值，则唯一性约束不会生效。

## 逻辑删除表中增加唯一约束
我们都知道唯一索引非常简单好用，但有时候，在表中它并不好加。
通常情况下，要删除表的某条记录的话，如果用 `delete` 语句操作的话。

例如
```sql
delete from product where id=1;
```

这种delete操作是物理删除，即该记录被删除之后，后续通过sql语句基本查不出来。（不过通过其他技术手段可以找回，那是后话了）

还有另外一种是逻辑删除，主要是通过update语句操作的。

例如
```sql
update product set delete_status=1 where id=1;
```

逻辑删除需要在表中额外增加一个删除状态字段，用于记录数据是否被删除。在所有的业务查询的地方，都需要过滤掉已经删除的数据。

通过这种方式删除数据之后，数据仍然还在表中，只是从逻辑上过滤了删除状态的数据而已。

其实对于这种逻辑删除的表，是没法加唯一索引的。

为什么呢？

假设之前给用户表中的 name 和 module 加了唯一索引，如果用户把某条记录删除了，delete_status设置成1了。后来，该用户发现不对，又重新添加了一模一样的商品。

由于唯一索引的存在，该用户第二次添加商品会失败，即使该商品已经被删除了，也没法再添加了。

这个问题显然有点严重。

有人可能会说：把name、model和delete_status三个字段同时做成唯一索引不就行了？

答：这样做确实可以解决用户逻辑删除了某个商品，后来又重新添加相同的商品时，添加不了的问题。但如果第二次添加的商品，又被删除了。该用户第三次添加相同的商品，不也出现问题了？

由此可见，如果表中有逻辑删除功能，是不方便创建唯一索引的。

但如果真的想给包含逻辑删除的表，增加唯一索引，该怎么办呢？

1. 删除状态+1

通过前面知道，如果表中有逻辑删除功能，是不方便创建唯一索引的。

其根本原因是，记录被删除之后，delete_status会被设置成1，默认是0。相同的记录第二次删除的时候，delete_status被设置成1，但由于创建了唯一索引（把name、model和delete_status三个字段同时做成唯一索引），数据库中已存在delete_status为1的记录，所以这次会操作失败。

我们为啥不换一种思考：不要纠结于delete_status为1，表示删除，当delete_status为1、2、3等等，只要大于1都表示删除。

这样的话，每次删除都获取那条相同记录的最大删除状态，然后加1。

这样数据操作过程变成：

添加记录a，delete_status=0。
删除记录a，delete_status=1。
添加记录a，delete_status=0。
删除记录a，delete_status=2。
添加记录a，delete_status=0。
删除记录a，delete_status=3。
由于记录a，每次删除时，delete_status都不一样，所以可以保证唯一性。

该方案的优点是：不用调整字段，非常简单和直接。

缺点是：可能需要修改sql逻辑，特别是有些查询sql语句，有些使用delete_status=1判断删除状态的，需要改成delete_status>=1; 并且在删除时需要查询所有相同名称的记录之前的status是多少。

2. 增加时间戳字段

导致逻辑删除表，不好加唯一索引最根本的地方在逻辑删除那里。

我们为什么不加个字段，专门处理逻辑删除的功能呢？

答：可以增加时间戳字段。

把name、model、delete_status和timeStamp，四个字段同时做成唯一索引

在添加数据时，timeStamp字段写入默认值1。

然后一旦有逻辑删除操作，则自动往该字段写入时间戳。

这样即使是同一条记录，逻辑删除多次，每次生成的时间戳也不一样，也能保证数据的唯一性。

时间戳一般精确到秒。

除非在那种极限并发的场景下，对同一条记录，两次不同的逻辑删除操作，产生了相同的时间戳。

这时可以将时间戳精确到毫秒。

该方案的优点是：可以在不改变已有代码逻辑的基础上，通过增加新字段实现了数据的唯一性。

缺点是：在极限的情况下，可能还是会产生重复数据。

3. 增加id字段

其实，增加时间戳字段基本可以解决问题。但在在极限的情况下，可能还是会产生重复数据。

有没有办法解决这个问题呢？

答：增加主键字段：delete_id。

该方案的思路跟增加时间戳字段一致，即在添加数据时给delete_id设置默认值1，然后在逻辑删除时，给delete_id赋值成当前记录的主键id。

把name、model、delete_status和delete_id，四个字段同时做成唯一索引。

这可能是最优方案，无需修改已有删除逻辑，也能保证数据的唯一性。

## 重复历史数据如何加唯一索引？

前面聊过如果表中有逻辑删除功能，不太好加唯一索引，但通过文中介绍的三种方案，可以顺利的加上唯一索引。

但来自灵魂的一问：如果某张表中，已存在历史重复数据，该如何加索引呢？

最简单的做法是，增加一张 `防重表` ，然后把数据初始化进去。

## 给大字段加唯一索引

有时候，我们需要给几个字段同时加一个唯一索引，比如给name、model、delete_status和delete_id等。

但如果model字段很大，这样就会导致该唯一索引，可能会占用较多存储空间。

我们都知道唯一索引，也会走索引。

如果在索引的各个节点中存大数据，检索效率会非常低。

由此，有必要对唯一索引长度做限制。

目前mysql innodb存储引擎中索引允许的最大长度是3072 bytes，其中unqiue key最大长度是1000 bytes。

如果字段太大了，超过了1000 bytes，显然是没法加唯一索引的。

1. 增加hash字段

我们可以增加一个hash字段，取大字段的hash值，生成一个较短的新值。该值可以通过一些hash算法生成，固定长度16位或者32位等。

我们只需要给name、hash、delete_status和delete_id字段，增加唯一索引。

这样就能避免唯一索引太长的问题。

但它也会带来一个新问题：

一般hash算法会产生hash冲突，即两个不同的值，通过hash算法生成值相同。

当然如果还有其他字段可以区分，比如：name，并且业务上允许这种重复的数据，不写入数据库，该方案也是可行的。

2. 不加唯一索引

如果实在不好加唯一索引，就不加唯一索引，通过其他技术手段保证唯一性。

如果新增数据的入口比较少，比如只有job，或者数据导入，可以单线程顺序执行，这样就能保证表中的数据不重复。

如果新增数据的入口比较多，最终都发mq消息，在mq消费者中单线程处理。

3. redis分布式锁

由于字段太大了，在mysql中不好加唯一索引，为什么不用redis分布式锁呢？

但如果直接加给name、model、delete_status和delete_id字段，加redis分布式锁，显然没啥意义，效率也不会高。

我们可以结合5.1章节，用name、model、delete_status和delete_id字段，生成一个hash值，然后给这个新值加锁。

即使遇到hash冲突也没关系，在并发的情况下，毕竟是小概率事件。

## 批量插入数据

假如通过查询操作之后，发现有一个集合：list的数据，需要批量插入数据库。

针对这种批量操作，如果此时使用mysql的唯一索引，直接批量insert即可，一条sql语句就能搞定。

数据库会自动判断，如果存在重复的数据，会报错。如果不存在重复数据，才允许插入数据。