
关于"幻读"现象的一些说法好像并不统一，我也说一下我对幻读现象的理解。

> 幻读是指当事务不是独立执行时发生的一种现象，例如第一个事务对一个表中的数据进行了修改，比如这种修改涉及到表中的“全部数据行”。同时，第二个事务也修改这个表中的数据，这种修改是向表中插入“一行新数据”。那么，以后就会发生操作第一个事务的用
    户发现表中还存在没有修改的数据行，就好象发生了幻觉一样.一般解决幻读的方法是增加范围锁RangeS，锁定检索范围为只读，这样就避免了幻读。

上面的引用内容来自百度百科。

MySQL多版本并发控制实现了MVCC机制，

在RR隔离级别下，**幻读现象出现的本质是因为没法锁住不存在的行。** 
大家知道在数据库标准的RR隔离级别下，是不可以避免幻读的，
而MySQL的InnoDB引擎通过next-key锁（行锁+Gap锁）来避免幻读（其实是在特定场景下能避免幻读）。
这也是导致说法不统一的一个重要原因。
另一个原因是在一个事务内的读是要分为**快照读**和**当前读**的，
**如果在快照读操作（select）后面跟着当前读操作（select for update），那就是无法避免幻读的，是因为第二次当前读之前这时候next-key锁还没来得及发挥作用。**
也就是说这时候的"读"其实已经不仅仅是"读"了。


分别举个例子：


|     T1     |     T2    |
|      :-        |     :-      |
|      `start transaction;`        |     `start transaction;`      |
|      `select * from users where id = 1;`        |           |
|             |     `insert into users(id, name) values (1, 'skrT2');`      |
|             |     `commit;`      |
|      `insert into users(id, name) values (1, 'skrT1');`     |           |
|      `commit;`     |           |

这种情况下，"skrT1"值会插入失败。但是这在T1事务内好像是自洽的，因为T2的事务操作结果此时对T1是不可见的。



1、T1：select * from users where id = 1;2、T2：insert into `users`(`id`, `name`) values (1, 'big cat');3、T1：insert into `users`(`id`, `name`) values (1, 'big cat');


我理解其实


