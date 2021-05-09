# redis主从复制
redis主从复制的优点：
（1）主从复制可以实现读写分离，主redis实现写操作，从redis实现读操作
（2）可以实现快速的容灾恢复

只能是实现一主多从

实现方式：
（1）首先创建/myredis文件夹
（2）复制redis.conf文件到文件夹/myredis文件夹下
（3）配置一主两从，配置三个配置文件
redis6379.conf
redis6380.conf
redis6381.conf

这三个配置文件和redis.conf在同一个目录下，就是在/myredis文件夹下
(4)在三个配置文件中写入内容
include /myredis/redis.conf
pidfile /var/run/redis_6379.pid
port 6379
rdbfilename dump6379.rdb

include /myredis/redis.conf
pidfile /var/run/redis_6380.pid
port 6380
rdbfilename dump6380.rdb

include /myredis/redis.conf
pidfile /var/run/redis_6381.pid
port 6381
rdbfilename dump6381.rdb

**将redis.conf中的aof配置关闭，appendonly no**
**直接通过配置文件就可以启动多个redis服务器**

直接执行如下三条命令启动三台redis服务器：实际上是启动三个redis服务
redis-server redis6379.conf
redis-server redis6380.conf
redis-server redis6381.conf

通过ps -ef | grep redis
查看三个redis进程pid

启动三个redis服务以后，可以通过info replication打印主从复制的相关信息

**可以开多个xshell窗口连接对应的redis-cli**
cd /myredis
redis-cli -p 6379
redis-cli -p 6380
redis-cli -p 6381

分别在三个客户端输入127.0.0.1:6379> info replication
可以发现默认情况下每一台服务器都是master主节点

**将从服务器加入关联主服务器**
分别在从服务器上执行：比如，在6380/6381从服务器上，执行   slaveof 127.0.0.1:6379  在客户端执行

（二）测试
（1）在主机上写，在从机上可以读取数据
在从机上写数据报错

（2）主机挂掉以后，重启就行，一切如初

（3）从机重启需要重新设置，即执行slaveof 127.0.0.1:6379

可以将配置加入文件中，永久生效

（三）一主二仆
**启动redis  redis-server redis6380.conf**
**启动客户端 redis-cli -p 6380**
**从服务器重新加入关联主服务器  slaveof 127.0.0.1:6379**
问题一：
  对于新加入的redis从服务器，是从加入点还是用头将主服务器中的数据进行复制同步？
**实际情况是，新加入的节点重头将主服务器中的数据进行复制同步** 

### 主从复制的原理
（1）首先，当从服务器关联上主服务器之后，会向主服务器发送数据同步的消息，  从服务器只会在刚加入时主动请求与主服务器进行数据同步   **全量复制**
（2）主服务器在收到从服务器的同步请求消息后，会将直接的数据持久化到rdb文件中，然后将持久化的rdb文件交给从服务器
（3）从服务器会根据rdb文件进行数据恢复同步
（4）后面如果主服务器有新的数据写入，则主服务器会主动和从服务器进行数据同步                                     **增量复制**


（四）薪火相传

就是主服务器6379将数据同步给一台从服务器6381，而另一台从服务器6380执行（slaveof 127.0.0.1:6381）此时主服务下只有一台从服务器6381，而6381从服务器下关联了一台从服务器6380，
6381从服务器负责6380从服务器的数据同步，
如果此时在（6379-->6381-->6380），如果主服务器6379宕机了，那么此时服务器主从结构不会发生变化，6379重启之后还是主服务器
（五）反客为主     当master宕机之后，后面的slave可以立刻升为master，其后面的slave不用做任何修改
但是，如果需要让从服务器在主服务器宕机之后成为主服务器，可以通过执行：在6381的客户端上执行：slaveof no one，6381从服务器就可以成为主服务器

（六）哨兵模式（sentinel）    反客为主的自动版，能够后台监控主机是否故障，如果故障了根据投票数自动将从服务器转为主服务器
如果需要在master当即之后，从服务器**自动**成为master服务器
实现方式：
（1）自定义的/myredis目录下新建sentinel.conf文件，**名字绝不能错**
（2）配置哨兵，填写内容
sentinel monitor mymaster 127.0.0.1 6379 1     该配置文件中只写一句就可以了
其中mymaster为监控对象起的服务器名称，1为至少有多少个哨兵同意迁移的数量

（3）启动哨兵
/usr/local/bin/redis-snetinel /myredis/sentinel.conf
启动这个哨兵，哨兵的默认端口是：26379

当主服务器宕机之后，哨兵监控到主服务器故障，选择一个从服务器成为新的主服务器，如果后面原来的主服务器重启，也只能是作为新的从服务器

新皇登基，旧皇俯首
**缺点就是存在主从数据同步的延迟**

从服务中选取新的主服务器的条件：
（1）选择优先级靠前的
（2）选择偏移量最大的
（3）选择runid最小的从服务
优先级在配置文件中redis.conf默认为：slave-priority 100,值越小优先级越高     在redis6中可能时replica-priority 100
偏移量是指获得原主机数据最全的
每一个redis实例启动后都会随机生成一个40位的runid   随机选择

新主登基（哨兵通过上面的三种选择条件选择一个新的主master）--->群仆俯首（挑选出新的master之后，sentinel向原来的所有从服务器发送slaveof新主服从命令，复制新的master，就是将原来的从服务器关联到新的master）
--->旧主俯首（当下线的主服务器重新上线时，sentinel会向其发送slaveof命令原来的master成为从服务器服从新的master）

**在java中实现主从复制的方式jedis**




