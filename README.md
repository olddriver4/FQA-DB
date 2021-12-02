### Mysql  
##### Mysql 那些因素会导致慢查询？（主要还是 精细化监控！）  
> 参考：https://www.infoq.cn/article/xw54gb1oe6fpxiggch19
1.	硬件层面（基本可以忽略）  
2.	数据库层面  
（1） 没有索引，索引不正确，全表扫描  
（2） 数据巨大，select * 或者 count * ，没有 limit 等等奇葩sql语句  
（3） 隐式转换  
> db不能利用合适的索引导致查询变慢： 简单说,就是在IN的入口有一个判断, 如果in中的字段类型不兼容, 则认为不可使用索引.   

（4） 执行计划错误  
>在检查某业务数据库的 slowlog 时发现一个慢查询,查询时间 1.57s ,检查表结构 where 条件字段存在正确的组合索引,正确的情况下优化器应该选择组合索引,而非为啥会导致慢查询呢  

（5） Metadata lock 锁等待：  
>比如 ddl 开始时，针对同一个表的长查询还没结束，后续的写操作都会被堵住导致 thread running 飙高

（6） 并发更新同一行  
>高并发场景下，数据库并发执行 update，更新同一行的动作会被其他已经持有锁的会话堵住，并且需要要进行判断会不会由于自己的加入导致死锁，这个时间复杂度 O(n)，如果有 1000 个请求，每个线程都要检测自己和其他 999 个线程是否死锁。如果其他线程都没有持有其他锁，约比较 50w 次(计算方式 999+998+…+1)。这个种锁等待和检查死锁冲突带来巨大的时间成本。对于 OLTP 业务高并发大流量访问的情况下，锁等待会直接导致 thread running 飙高，所有的请求会被阻塞并等待 innodb 引擎层处理，于是 sql 会变慢。  

>【优化建议】：将数据库的刷盘策略从同步改为异步也即批量执行刷盘，提高IO利用效率。关闭死锁检测，减少单行整体服务时间。 相关参数: 使用 mysqlslap 自行压测（需根据业务情况选择）  
sync_binlog=0  
innodb_flush_log_at_trx_commit=0  
innodb_deadlock_detect=OFF  

（7） innodb刷脏页  
>InnoDB 引擎采用 Write Ahead Log(WAL)策略，即事务提交时，先写日志(redo log)，再写磁盘。 为了提高 IO 效率,在写日志的时候会先写 buffer，然后集中 flush buffer pool 到磁盘。 这个过程 我们称之为 刷脏页 （8.0以前使用同步刷脏页）  

（8） History list length 过高， UNDO没有被purge  
>【优化建议1】开始比较早的事务，如果还没提交，就需要通过 UNDO 去构建对应版本历史时，保证数据库的可重复读（跟 MVCC 有关）  
 【优化建议2】可能是有历史事务出现异常没有提交，也有可能是一致性快照的备份。可以通过 information_schema.innodb_trx 表去确认对应的事务信息。经过查询，的确发现一个事务执行了4个小时左右，没有提交，且不是备份用户。手动将该线程执行 kill 操作，慢查消失  

##### Mysql 主从延迟问题  
>【参考】https://mp.weixin.qq.com/s/oj-DzpR-hZRMMziq2_0rYg  
【问题】主从延迟过高，尤其在大并发批量插入数据时  
5.6：MTS即使是Database，基于库级并行复制    
5.7：MTS机制是COMMIT_ORDER，基于组提交的并行复制
优化建议：    
5.7.22+：MTS机制是Writeset/ Writeset_session，基于Writeset的并行复制    

##### Mysql insert into select 引发锁表（insert into t values(-1,-1,-1);）
>【参考】https://cloud.tencent.com/developer/article/1744166  
【问题】在执行语句的时候，MySQL是逐行加锁的（扫描一个锁一个），直至锁住所有符合条件的数据，执行完毕才释放锁。所以当业务在进行的时候，切忌使用这种方法
解决：加条件，强制走索引，不要全表扫描，比如：INSERT INTO Table2 SELECT * FROM Table1 FORCE INDEX (create_time) WHERE update_time <= '2020-03-08 00:00:00';  加上limit限制  

### Redis
##### Redis内存溢出问题
>【问题】因为开发使用的Redis 没有设置ttl过期，而这块设置的maxmemory内存限制触发的定期删除策略无用  
【优化】主动删除策略配置的是存在ttl过期缓存使用少的删除， 因此 联系开发让其配置ttl过期策略，主动策略配置为allkeys-lru

##### Redis CPU百分百问题
>【问题】redis 开启aof 每秒持久化数据落盘本地 并开启了重写功能，在高并发的写入数据时，由于redis默认CPU单进程，导致cpu百分百，aof从内存落盘到本地跟不上实际 写入时间，redis无法访问  
【优化】redis 6.0后支持开启多线程， infura数据可以容忍数据短暂丢失，关闭aof持久化  
“实际集群环境中，redis备份rdb aof策略需要单独一台与业务无关的从服务器进行备份操作”

##### Redis大Key导致耗时过长问题
>【问题】redis 高并发压测场景时，研发写入redis批次为每秒2000个， 一个批次大概耗时51s-5s，出现写入过慢现象，比MySQL还慢；  
【思路】去grafana看监控，cpu利用率不高，内存利用率不高，硬盘iops不高， 怀疑写入阻塞问题，利用rdb工具查看redis大于1key的key数量，大key范围是10-80key有1千个左右，然后在使用redis-benchmark工具压测模拟现在大key 10-80key 2000批次分为 3份，1key、10key、60key 测试高场景会出现一个批次共几秒的现象，这就解释的通了，拿这测试的证据和测试的方法并且 查出线上大key的结果给研发后 ，哑口无言，确定问题原因后，建议个别大key存在mysql，有的key进行拆分后，在测试发现速度立马提升！

### Postgresql
##### postgresql  cpu使用率高排查
>【参考】https://help.aliyun.com/knowledge_detail/43562.html  
【建议】提前做好慢SQL监控！（死锁、语句查询过长等） 

##### postgresql 主从同步异常，主从角色切换，主库回退变为从库
>【参考】https://developer.aliyun.com/article/746272  
【问题】客户内网环境中断引起  
【场景】1、PG物理流复制的从库，当激活后，可以开启读写，使用pg_rewind可以将从库回退为只读从库的角色。而不需要重建整个从库。  
2、当异步主从发生角色切换后，主库的wal目录中可能还有没完全同步到从库的内容，因此老的主库无法直接切换为新主库的从库。使用pg_rewind可以修复老的主库，使之成为新主库的只读从库。而不需要重建整个从库。  
3、如果没有pg_rewind，遇到以上情况，需要完全重建从库。或者你可以使用存储层快照，回退回脑裂以前的状态。又或者可以使用文件系统快照，回退回脑裂以前的状态。   
【解决】使用pg_rewind修复，从库变为主库，主库变为从库。 没有pg_rewind只能重建从库！


### PgPool
##### pgpool 重启节点后，节点状态为down
>【参考】https://segmentfault.com/a/1190000020850773  
【问题】pgpool作为postgresql的中间件，当集群内存在至少两个节点时，就会进行选举，如果此时第三个节点还没起来，当选举完成后，pgpool不会将没有参加选举的节点自动加入集群，需要手工attach进集群，或者同时重启pgpool进行重启选举，即pgpool本身不具有重启后能自动加入集群并恢复的机制。  
【解决】1. 手动attach  2. 重启pgpool 触发重新选举  
注意：每次重新选举需要删除 pgpool_status 文件，不然无效

##### pgpool 重启主节点后，数据库进入只读模式
>【参考】https://segmentfault.com/a/1190000020850773   
【问题】该问题是个小概率偶现问题，即master节点断电后，新的master被选举出，新的master会将本地配置文件修改为master对应的，然后还在成为新master的过程中，这时候通过数据库VIP读取的master信息仍为旧master，这就使得本地数据库failover脚本认为新master出现了不一致，于是将之前postgresql修改为master的一系列配置文件又改回了standby对应的配置文件，其中primary info仍指向为旧master。这就导致没有新的master产生，旧的master一直为down的状态。而没有master节点， 数据库则会进入只读模式。  
【解决】修改failover脚本代码逻辑，当本地配置文件与数据库角色状态不一致时，不会第一时间去修改本地recovery文件。之前再加一层判断：如果master节点postgresql服务还能正常访问，再去修改recovery文件。  

##### pgpool 客户端阻塞（经常出现短暂卡顿）
>【问题】最近遇到一个PgPool连接阻塞问题，PgPool刚开启是能成功连接的，过段时间就连接不上了。查看PgPool日志，启动成功，连接数据库节点成功，健康检查成功。然后怀疑是并发数过多导致阻塞。  
一开始，更改了pgpool.conf的max_pool,num_init_children参数然后重启，结果仍然阻塞。查资料可知：
num_init_children：pgPool允许的最大并发数，默认32。
max_pool：连接池的数量，默认4。  
pgpool需要的数据库连接数=num_init_childrenmax_pool；  
后检查Postgresql数据库的postgresql.conf文件的max_connections=100，superuser_reserved_connections=3。  
【优化】pgpool的连接参数应当满足如下公式：  
num_init_childrenmax_pool<max_connections-superuser_reserved_connections  
当需要pgpool支持更多的并发时，需要更改num_init_children参数，同时要检查下num_init_children*max_pool是否超过了max_connections-superuser_reserved_connections，如果超过了，可将max_connections改的更大。
