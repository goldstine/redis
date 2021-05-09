# redis集群
无中心化集群的搭建
每一台主节点都存储所有数据的1/N

配置方式：
（1）首先删除原先的所有的rdb文件
（2）修改配置文件redis6379.conf
include /myredis/redis.conf
pidfile "/var/run/redis_6379.pid"
port 6379
dbfilename "dump6379.rdb"
cluster-enabled yes
cluster-config-file nodes-6379.conf
cluster-node-timeout 15000


redis cluster配置修改

cluster-enabled yes打开集群模式  
cluster-config-file nodes-6379.conf 设置节点配置文件名
cluster-node-timeout 15000 设定节点失联时间，超过该时间（毫秒），集群自动进行主从切换

（3）然后根据6379的配置文件复制出多个配置文件 redis6380.conf  redis6381.conf   redis6389.conf   redis6390.conf   redis6391.conf
同一目录（/myredis）下还有 redis.conf  sentinel.conf文件
将所有配置文件修改
（4）启动所有的配置文件对应的redis服务
redis-server redis6379.conf
......


(5)需要进入redis安装目录的src环境下进行，六个redis服务器的合体操作，   /opt/redis-6.2.1/src   是redis解压文件夹下的src目录下执行命令，如下：（redis6版本一下的需要安装ruby环境，6以上的已经集成了该环境）
然后执行redis-cli --cluster create --cluster-replicas 1 192.168......进行主从服务器分配

（6）连接的方式要改为集群方式进行连接
redis-cli -c -p 6379

通过cluster nodes可以查看集群信息

在往主服务器中写入数据的时候，是通过计算hashcode  将需要存放的位置进行映射到一个slot进行存放，其中slot已经与现在主服务器中进行分配好了
（决定将数据存放到哪一台主服务器上的方式）

（7）集群的故障恢复
如果一台主服务器宕机，则对应的从服务器成为新的主服务器，如果原来的主服务器重启，则作为新的主服务器节点的从服务器
如果对应的主服务器和从服务器都宕机了，则对应的插槽不能够提供服务，但是集群中的其它插槽还是可以正常提供服务

# java通过jedis操作redis无中心化集群

//随便一台主节点都可以作为集群的入口
HostAndPort hostAndPort=new HostAndPort("192.168.56.10",6379);
JedisCluster jedisCluster=new JedisCluster(hostAndPort);
//进行操作
jedisCluster.set("b1","value1");
String value=jedisCluster.get("b1");

System.out.println(value);
jedisCluster.close();

## redis集群提供的好处
（1）实现扩容
（2）分摊压力
（3）无中心化配置相对简单
缺点：
多键的操作是不被支持的
多键的redis事务是不被支持的，lua脚本不被支持


# 缓存问题
（1）缓存穿透
解决方式：
（1）对空值进行缓存，如果一个查询返回的数据为空（不管数据是否存在），我们仍然把这个空结果null进行缓存，设置空结果的过期时间很短，最长不超过5min
（2）设置可访问的白名单：使用bitmaps类型定义一个可以访问的名单，名单id作为bitmaps的偏移量，每次访问和bitmaps里面的d进行比较，如果访问id不在bitmaps里面，进行拦截。不允许访问
（3）采用布隆过滤器    其实就是对bitmaps进行优化
（4）进行实时监控   ：当发现redis的命中率开始急速降低，需要排查访问对象和访问的数据，和运维人员配合，可以设置黑名单限制服务

（2）缓存击穿
解决方式：
（1）预先设置热门数据：在redis高峰访问之前，把一些热门数据提前存入到redis里面，加大这些热门key的时长
（2）实时调整：现场监控哪些数据热门，实时调整key的过期时长
（3）使用锁

（3）缓存雪崩
就是在极短的一段时间，大量的缓存数据集中过期
解决方式：
（1）构建多级缓存架构：nginx缓存+redis缓存+其他缓存（ehcache等）
（2）使用锁或队列
用加锁或者队列的方式保证不会有大量的线程对数据库一次性进行读写，从而避免失效时大量的并发请求落到底层存储系统上，不适用于高并发情况
（3）设置过期标志更新缓存
记录缓存数据是否过期，如果过期会触发通知另外的线程在后台去更新实际key的缓存

（4）将缓存失效时间分散开
可以在原有的失效时间上增加一个随机值，避免引发集体失效事件

# 分布式锁
（1）使用setnx上锁，通过del释放锁
（2）锁一直没有释放，可以通过设置key过期时间，自动释放
setnx users 10
expire users 10
(3)但是上面（2）中存在的问题是，如果在上锁的过程之中出现异常，后面没有设置过期时间
解决方式是将上锁和设置过期时间变为原子操作
在上锁的时候并且设置锁的过期时间： set users 10 nx ex 12

java   springboot中实现redis分布式锁
//获取锁，setnx
Boolean lock=redisTemplate.opsForValue().setIfAbsent("lock","111");
//获取锁成功，查询num的值
if(lock){
  Object value=redisTemplate.opsForValue().get("num");
  //判断num为空return
  if（StringUtils.isEmpty(value)){
    return;
  }
  //有值就转成int
  int num=Integer.parseInt(value+"");
  //把redis的num+1
  redisTemplate.opsForValue().set("num",++num);
  //释放锁
  redisTemplate.delete("lock");
}else{
  
}


释放锁的问题   锁误删的问题
（1）uuid表示不同的操作
set lock uuid nx ex 10
(2)释放锁的时候，首先判断当前uuid和要释放锁的uuid是否一样，如果一样，则可以释放，否则不能释放

通过uuid保证只能够释放自己的锁，而不能够释放别人的锁


lua保证原子性操作
即使加上uuid也不能够保证不误删别人的锁
比如当a在比较uuid时，确认是自己的锁，但是如果此时所自动过期，然后a在删除锁的时候，就有可能将b的锁删除
原因是没有保证原子性





