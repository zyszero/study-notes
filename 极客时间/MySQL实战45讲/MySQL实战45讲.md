# MySQL实战45讲

## 01 基础架构：一条SQL查询语句是如何执行的？

1. MySQL的框架有几个组件, 各是什么作用？

   客户端 -》 连接器 -》分析器 -》优化器 -》执行器 -》 存储引擎

   ​							   -》查询缓存

   ![](images/01-MySQL的逻辑架构图.png)

   连接器：负责与客户端建立连接，获取权限、维持和管理连接

   查询缓存：一种key-value缓存，之前执行过的SQL会被缓存在这，不建议使用查询缓存，因为查询缓存失效很频繁，弊大于利，MySQL 8.0开始，已经移除查询缓存。

   分析器：对SQL进行词法分析，识别SQL字符串中特定的字符串，比如select、from等，或者将字符T识别为表T，字符列名，识别为字段列。然后进行语法分析，根据语法规则校验词法分析的结果。

   优化器：在表里面有多个索引的时候，决定使用哪个索引；或者在一个语句有多表关联（join）的时候，决定各个表的连接顺序。

   执行器：执行语句，执行时会判断对被操作的表有无执行查询权限，若无，则返回没有权限的错误，若有，则打开表继续执行。打开表的时候，执行器就会根据表的引擎定义，去使用这个引擎提供的接口。

   存储引擎：数据的存储和提取，提供读写接口。

2. Server层和存储引擎层各是什么作用？

   Server 层包括连接器 、分析器、优化器、执行器等，涵盖 MySQL 的大多数核心服务功能，以及所有的内置函数（如日期、时间、数学和加密函数等），所有跨存储引擎的功能都在这一层实现，比如存储过程、触发器、视图等。

   存储引擎层主要负责数据的存储和提取，其架构模式是插件式的，支持InnoDB、MyISAM、Memory等各种存储引擎。

3. you have an error in your SQL syntax 这个保存是在词法分析里还是在语法分析里报错？

   语法分析

4. 对于表的操作权限验证在哪里进行？

   是在执行器执行阶段进行的

5. 执行器的执行查询语句的流程是什么样的？

   假设语句如下所示，引擎是InnoDB：

   ```mysql
   
   mysql> select * from T where ID=10;
   
   ERROR 1142 (42000): SELECT command denied to user 'b'@'localhost' for table 'T'
   ```

   执行器的执行流程:

   1. 调用InnoDB引擎接口获取表的第一行，判断ID是不是10，如果不是，则跳过，否则，将其存在结果集中。
   2. 调用引擎接口获取下一行，重复相同的逻辑判断，直至取到表的最后一行。
   3. 执行器将上述遍历过程中所有满足条件的行组成的记录集作为结果集返回给客户端。

评论摘抄：

1. 连接器是从权限表里边查询用户权限并保存在一个变量里边以供查询缓存，分析器，执行器在检查权限的时候使用。
2. sql执行过程中可能会有触发器这种在运行时才能确定的过程，分析器工作结束后的precheck是不能对这种运行时涉及到的表进行权限校验的，所以需要在执行器阶段进行权限检查。另外正是因为有precheck这个步骤，才会在报错时报的是用户无权，而不是 k字段不存在（为了不向用户暴露表结构）。
3. 词法分析阶段是从information schema里面获得表的结构信息的。
4. 可以使用连接池的方式，将短连接变为长连接。
5. mysql_reset_connection是mysql为各个编程语言提供的api，不是sql语句。
6. wait_timeout是非交互式连接的空闲超时，interactive_timeout是交互式连接的空闲超时。执行时间不计入空闲时间。这两个超时设置得是否一样要看情况。

## 02 日志系统：一条SQL更新语句是如何执行的？

1. 什么是redo log？作用是什么？
2. 什么是binlog？有什么用？
3. binlog 和 redo log 的区别？
4. redo log 能替换 binlog 吗？



## 03 事务隔离：为什么你改了我还看不见？

1. 什么是事务？

   ACID（Atomicity、Consistency、Isolation、Durability）：原子性、一致性、隔离性、持久性。

2. 事务异常有哪些？

   - 脏读（dirty read）
   - 不可重复读（unrepeatable read）
   - 幻读（phantom read）

3. 事务隔离级别？

   - 读未提交
   - 读已提交
   - 可重复读
   - 串行化

4. 事务隔离级别的本质？

   在实现上，数据库里面会创建一个视图，访问的时候以视图的逻辑结果为准。在“可重复读”隔离级别下，这个视图是在事务启动时创建的，整个事务存在期间都用这个视图。在“读提交”隔离级别下，这个视图是在每个 SQL 语句开始执行的时候创建的。这里需要注意的是，“读未提交”隔离级别下直接返回记录上的最新值，没有视图概念；而“串行化”隔离级别下直接用加锁的方式来避免并行访问。

5. 事务隔离级别的具体实现的？

   以“可重复读”为例，在 MySQL 中，实际上每条记录在更新的时候都会同时记录一条回滚操作。记录上的最新值，通过回滚操作，都可以得到前一个状态的值。不同时刻启动的事务会有不同的read-view，且不同read-view之间相互不干扰。

6.  什么是MVCC（Multiple Version Concurrence Control 多版本并发控制）？

   同一条记录在系统中可以存在多个版本，这就是数据库的多版本并发控制（MVCC）。

7.  回滚日志什么时候删除？

   系统会判断当没有事务需要用到这些回滚日志的时候，回滚日志会被删除。

8. 什么时候回滚日志不需要了？

   当系统里么有比这个回滚日志更早的read-view的时候。

9. 为什么尽量不要使用长事务？

   长事务意味着系统里面会存在很老的事务视图，在这个事务提交之前，回滚记录都要保留，这会导致大量占用存储空间。除此之外，长事务还占用锁资源，可能会拖垮库。

10. 事务启动的方式？

    - 显式启动事务语句，begin或者start transaction,提交commit，回滚rollback；
    - set autocommit=0，该命令会把这个线程的自动提交关掉。这样只要执行一个select语句，事务就启动，并不会自动提交，直到主动执行commit或rollback或断开连接。
    - 建议使用方法一，如果考虑多一次交互问题，可以使用commit work and chain语法。在autocommit=1的情况下用begin显式启动事务，如果执行commit则提交事务。如果执行commit work and chain则提交事务并自动启动下一个事务。

11. 系统里面应该避免长事务，如果你是业务开发负责人同时也是数据库负责人，你会有什么方案来避免出现或者处理这种情况呢？

