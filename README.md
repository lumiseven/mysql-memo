# mysql 备忘录

* 监控sql执行情况的手段
* 利用执行计划explain优化sql
* 怎么判定sql需要优化
* 如何建表以及如何优化表结构
* 如何优化表结构
* mysql索引为什么用B+树而不用红黑树？
* 索引分类、索引优化原则、索引覆盖、回表、最左匹配
* MTS主从并发复制原理及实操指南
* 搭建 mysql慢查询 错误日志监控平台

---

## 监控sql执行情况的手段

要想进阶针对mysql学习乃至掌握mysql调优的基本技能，监控mysql的执行情况必不可少。就像我们的代码，如果不能debug，想要进行调优排错，难度将会大大增加。

1. (**deprecated**) show profile
    1. 什么是show profile
        show profile是mysql提供的用来分析当前会话中sql语句执行的资源消耗情况，利用它我们可以用来分析sql的性能，作为调优的测量工具
        show profile默认是关闭的，可以通过set profiling=1;指令来开启，但是需要注意的是每次开启只是生效在当前会话中，想要永久生效的话需要修改mysql配置文件
    2. 如何使用show profile
        1. 首先开启show profile，mysql中执行指令
            ```sql
            set profiling=1;
            ```
        2. 运行一段测试sql
            ```sql
            select * from order_info limit 100;
            ```
        3. 执行 `show profiles`; 查询最近执行的sql的情况
            ![1](./img/1.png)
            如上图所示，我们可以看到，我们刚刚执行的这条sql的执行时间为0.001425s，queryId为15
        4. 那么我们还可以通过这个queryId进阶监控这个sql的其他资源消耗情况，比如查询其CPU的消耗情况
            ```sql
            show profile CPU for query 15;
            ```
            ![2](./img/2.png)
            其中 `status` 表示的是sql指定的各个阶段的状态， `duration` 表示的是各个状态的耗时， `cpu_user` 表示当前用户占用的cpu， `cpu_system` 表示系统占用的cpu

            结果分析：通过上述结果可知，我们执行的sql的大部分时间消耗在启动上，其次消耗在打开table，真正花在执行上的时间只有0.001s

            除了上述演示的监控cpu的资源消耗，show profile还提供了如下的监控类型: 
            |监控类型|语句| 
            |-|-| 
            |显示所有性能信息|all| 
            |显示块IO开销|block io| 
            |显示上下文开销|context switches| 
            |显示用户cpu时间、系统cpu时间|cpu| 
            |显示发送和接收的消息数量|ipc| 
            |显示页错误数量|page faults| 
            |显示源码中的函数名称与位置|source| 
            |显示swap的次数|swaps|

            指令格式: 
            ```sql
            show profile [type] [for query query_id]
            ```

            另外一个常用的指令是show profile;，这个是用于查询最近一个profiling信息

            **注意**: `show profile` 在 `mysql5.7` 中就已经显示过时了，虽然仍然可用，但mysql推荐更还用的 `performance_schema` 语句来监控sql执行情况
2. performance_schema
    1. 什么是performance_schema
        performance_schema实际上是一个数据库，我们可以通过数据库查询指令 `show databases;` 或者像navicat这样的数据库管理软件查看到该数据库。

        ![3](./img/3.png)

        performance_schema是用于监控mysql在一个较低级别的运行过程中的资源消耗、资源等待的情况。它提供了一系列的表格，这些表格中存储了关于数据库运行期间的性能相关的数据，如磁盘、IO、锁、CPU等相关信息。

        performance_schema比show profile更加详尽，5.6版本后默认是开启的，可以在mysql配置文件中看到配置项

        ```txt
        [mysqld]
        ...
        performance_schema=ON
        ...
        ```
    2. performance_schema表分类 (mysql-8.0.29 110张表)
        |分类|表名| 
        |-|-| 
        |语句事件记录表|show tables like '%statement%';| 
        |当前语句事件表|events_statements_current| 
        |历史语句事件表|events_statements_history| 
        |长语句历史事件表|events_statements_history_long| 
        |摘要表|summary| 
        |等待事件记录表，与语句事件类型的相关记录表类似|show tables like '%wait%';| 
        |阶段事件记录表，记录语句执行的阶段事件的表|show tables like '%stage%';| 
        |事务事件记录表，记录事务相关的事件的表|show tables like '%transaction%';|
        |监控文件系统层调用的表|show tables like '%file%';|
        |监控内存使用的表|show tables like '%memory%';|
        |动态对performance_schema进行配置的配置表|show tables like '%setup%';|
    3. 如何使用performance_schema
        * instruments和consumers
            * instruments：生产者，用于采集mysql中各种各样的操作产生的事件信息，对应setup_instruments配置表中的配置项，也可以称为监控采集配置项
            * consumers：消费者，对应的消费者表用于存储来自instruments采集的数据，对应setup_consumers配置表中的配置项，可以称为消费存储配置项
        
        虽然performance_schema默认是开启的，但是数据库刚启动时并非所有的采集项都打开了，也就是说，默认不会采集所有的事件。

        有可能你需要检测的事件并没有打开，那么就需要我们手动将对应项打开。

        比如打开等待时间的采集器开关，需要修改setup_instruments表中对应的采集器配置项

        ```sql
        update setup_instruments set ENABLED='YES',TIMED='YES' where name like 'wait%'; 
        ```

        打开等待事件的保存表配置开关，需要修改setup_consumers配置表中对应的配置项

        ```sql
        update setup_consumers set ENABLE='YES' where name like '%wait%';
        ```

        **常用查询**
        ```sql
        --1、哪类的SQL执行最多？
        SELECT DIGEST_TEXT,COUNT_STAR,FIRST_SEEN,LAST_SEEN FROM events_statements_summary_by_digest ORDER BY COUNT_STAR DESC
        --2、哪类SQL的平均响应时间最多？
        SELECT DIGEST_TEXT,AVG_TIMER_WAIT FROM events_statements_summary_by_digest ORDER BY AVG_TIMER_WAIT DESC
        --3、哪类SQL排序记录数最多？
        SELECT DIGEST_TEXT,SUM_SORT_ROWS FROM events_statements_summary_by_digest ORDER BY SUM_SORT_ROWS DESC
        --4、哪类SQL扫描记录数最多？
        SELECT DIGEST_TEXT,SUM_ROWS_EXAMINED FROM events_statements_summary_by_digest ORDER BY SUM_ROWS_EXAMINED DESC
        --5、哪类SQL使用临时表最多？
        SELECT DIGEST_TEXT,SUM_CREATED_TMP_TABLES,SUM_CREATED_TMP_DISK_TABLES FROM events_statements_summary_by_digest ORDER BY SUM_CREATED_TMP_TABLES DESC
        --6、哪类SQL返回结果集最多？
        SELECT DIGEST_TEXT,SUM_ROWS_SENT FROM events_statements_summary_by_digest ORDER BY SUM_ROWS_SENT DESC
        --7、哪个表物理IO最多？
        SELECT file_name,event_name,SUM_NUMBER_OF_BYTES_READ,SUM_NUMBER_OF_BYTES_WRITE FROM file_summary_by_instance ORDER BY SUM_NUMBER_OF_BYTES_READ + SUM_NUMBER_OF_BYTES_WRITE DESC
        --8、哪个表逻辑IO最多？
        SELECT object_name,COUNT_READ,COUNT_WRITE,COUNT_FETCH,SUM_TIMER_WAIT FROM table_io_waits_summary_by_table ORDER BY sum_timer_wait DESC
        --9、哪个索引访问最多？
        SELECT OBJECT_NAME,INDEX_NAME,COUNT_FETCH,COUNT_INSERT,COUNT_UPDATE,COUNT_DELETE FROM table_io_waits_summary_by_index_usage ORDER BY SUM_TIMER_WAIT DESC
        --10、哪个索引从来没有用过？
        SELECT OBJECT_SCHEMA,OBJECT_NAME,INDEX_NAME FROM table_io_waits_summary_by_index_usage WHERE INDEX_NAME IS NOT NULL AND COUNT_STAR = 0 AND OBJECT_SCHEMA not in ('mysql','test') ORDER BY OBJECT_SCHEMA,OBJECT_NAME;
        --11、哪个等待事件消耗时间最多？
        SELECT EVENT_NAME,COUNT_STAR,SUM_TIMER_WAIT,AVG_TIMER_WAIT FROM events_waits_summary_global_by_event_name WHERE event_name != 'idle' ORDER BY SUM_TIMER_WAIT DESC
        --12-1、剖析某条SQL的执行情况，包括statement信息，stege信息，wait信息
        SELECT EVENT_ID,sql_text FROM events_statements_history WHERE sql_text LIKE '%count(*)%';
        --12-2、查看每个阶段的时间消耗
        SELECT event_id,EVENT_NAME,SOURCE,TIMER_END - TIMER_START FROM events_stages_history_long WHERE NESTING_EVENT_ID = 1553;
        --12-3、查看每个阶段的锁等待情况
        SELECT event_id,event_name,source,timer_wait,object_name,index_name,operation,nesting_event_id FROM events_waits_history_long WHERE nesting_event_id = 1553;
        ```
    
3. 其他常用监控指令
    1. show processlist 监控连接线程数
        show processlist指令可以查询到mysql当前的连接线程，以此来监控mysql是否有大量线程数连接，从而进行排查
        
        ```sql
        show processlist
        ```

        执行结果

        ![4](./img/4.png)

    2. last_query_cost 监控数据页
        mysql中是以数据页为单位来存储数据的，last_query_cost指令用于查询最近一次查询需要查找多少个数据页。查找的数据页越多，IO越高，性能越差

        ```sql
        show status like 'last_query_cost';
        ```
