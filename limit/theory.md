

# 功能与限制

## 实现原理

如图所示， 一个读写分离中间件由多个高性能 Proxy 节点和 UCloud 分布式负载均衡产品 ULB 构成。

![image](/images/rwproxy_arch.jpg)

客户可以对读请求的分发方式，进行自定义配置（配置方法详见下文），Proxy
节点根据客户配置分发读请求。

读写分离中间件对业务请求的处理方式非常简单， 有三个基本原则：

1\. 从业务请求中，识别Select SQL， 只有Select SQL才考虑做读写分离；

2\. 如果该Select SQL处于一个事务当中， 则将该Select SQL发往主节点，如果该Select SQL不处于事务当中，
则根据读请求分发策略， 将该Select SQL发往主节点或者从节点。

3\. 对于一些必须要广播的语句， 如Use database、Set
Session变量等语句，由中间件进行广播，如果广播未全部成功，则中断客户端连接，以此严格保证各节点的数据一致性。

以及针对一些特殊情况的修正：

1\. 涉及到锁的Select语句， 如Select For Update 、 Select Lock 等， 将被分发到主节点。

2\. 将Set语句中的变量， 分为三种类型： Session、Global、User。 set Session、Set
User变量语句将被广播；考虑到节点间数据一致性问题， Set Global只会分发到主节点。 后续含全局变量的
Select 语句， 也只会发送到主节点。

## 功能限制

### 1.MySQL协议限制

1.1 不支持SSL加密

1.2 暂不支持压缩协议

1.3 暂不支持XA事物

### 2.SQL限制

2.1 支持savepoint语句（该语句将被分发到主节点）， 但暂不支持 rollback to savepoint

2.2 暂不支持XA事务命令

2.3 Lock Tables/Unlock Tables 将被分发到主节点，而Proxy层不会有任何Lock状态。因此， Lock
Tables产生的锁不会影响到从节点。

2.4 存储过程，以及存储过程后的Select语句， 一律分发到主节点。如：

```
call udb\_test('000001',@pp,@qq); select @pp,@qq; select \* from t1;
```

上述两条 Select 语句，都将被分发到主节点。

2.5 show processlists、Show master/slave status、kill query、COM_PROCESS_INFO、COM_STATISTICS命令，目前只会转发到主节点，针对中间件和数据库系统管理场景的，更丰富的系统管理命令正在开发中。

2.6 暂不支持COM_TABLE_DUMP和COM_CHANGE_USER协议。

2.7 如果执行了[Multi-Statement](https://dev.mysql.com/doc/c-api/8.0/en/c-api-multiple-queries.html) 语句，
当前连接的后续请求会全部路由到主节点，需断开当前连接并重新连接才能恢复读写分离。

### 3.对Set语句的特别说明

3.1 Set Session、Set User变量语句， 将被广播到主节点和从节点，如果广播失败，Proxy将断开和客户端连接，
从而撤销广播失败导致的数据不一致； 考虑到节点间的数据一致性， Set global变量语句只会被分发到主节点。
后续含全局变量的 Select 语句， 也只会发送到主节点。

3.2 不允许在一条Set语句中，同时出现Global变量和Session、User变量。

3.3 不支持 set sql_mode 中的值为表达式(e.g set sql_mode=(select replace(@@sql_mode,'ONLY_FULL_GROUP_BY,','')))

3.4 暂时不支持 set sql_mode='XXX',sql_safe_updates='XXX',.... 这种set多个变量的语法。

3.5 暂时不支持mysql8.0 caching_sha2_password的加密算法，若客户端采用caching_sha2_password加密算法，代理会协调客户端和服务端使用mysql_native_password方式

### 4.不推荐使用读写分离的场景

a. 业务的SQL均为事务SQL（所有SQL都包含在事务中）， 由于事务只能被路由到主节点，故该场景下UDB读写分离无法起到分离读请求的作用

b. 业务使用了大量存储过程。 由于存储过程只能被路由到主节点，故该场景下UDB读写分离无法起到分离读请求的作用

