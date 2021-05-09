# redis中的事务
redis的事务分为两个阶段（1）组队阶段（2）执行阶段
multi组队阶段，exec执行阶段，discard撤销组队
** 如果在组队阶段就出现了错误，那么所有的命令都执行失败 **
** 如果在组队阶段没有出现错误，那么执行阶段有错误的执行失败，没有错误的执行成功 **
# 事务的冲突问题
悲观锁：认为数据总是会被修改，所以每次操作数据都加上锁
乐观锁：认为数据不会被修改，每次更新的时候判断在此期间有没有更新数据，乐观锁适合多读的应用类型，这样可以提高吞吐量，redis就是利用这种check-and-set机制实现事务的。

watch key []
在执行multi之前，先执行watch key1 key2可以监视一个或多个key，如果事务在执行之前这个key被其他命令所改动，那么事务将被打断

unwatch key  取消对key的监视

## redis事务三特性
（1）单独的隔离操作
事务中的所有命令都会被序列化，按顺序地执行，事务在执行的过程中，不会被其他客户端发送来的命令请求打断
（2）没有隔离级别的概念
队列中的命令没有提交之前都不会实际被执行，因为事务提交前任何指令都不会被实际执行
（3）不保证原子性
事务中如果有一条命令执行失败，其后的命令仍然会被执行，没有回滚

## 秒杀

输入用户的uuid和商品的prodid
(1)首先判断uuid和prodid是否为空，如果为空，直接返回false
（2）如果uuid或者prodid存在不为空，则连接redis   new Jedis("192.168.56.10",6379);
(3)拼接存在redis中的key   uuid 的String key   和prodid 的key
（4）获得库存商品数量，如果库存为null，说明秒杀还没有开始
（5）通过uuid判断redis中是否有该用户，如果有该用户，则说明该用户已经秒杀成功，不能再进行秒杀，返回false
（6）通过prodid获得redis中的秒杀商品数量   jedis.get("produid key");  如果秒杀商品数量<1  则说明秒杀已经结束,返回false
（7）秒杀成功，直接将用户的uuid添加进入redis，将redis中的库存减一  decr（）

## 秒杀并发模拟

使用工具ab模拟测试
centos6默认安装
centos7需要手动安装

联网yum install httpd-tools
无网络
（1）进入 cd /run/media/root/CentOS 7 x86_64/Packages(路径跟centos6不同)
（2）顺序安装：
apr-1.4.8-3.e17.x86_64.rpm
apr-util-1.5.2-6.e17.x86_64.rpm
httpd-tools-2.4.6-67.e17.centos.x86_64.rpm

测试及结果
直接在linux上发送并发请求：  ab -n 2000 -c 300 -p ~/postfile -T application/x-www-form-urlencoded http://192.168....(项目的请求路径)
-n表示请求数，-c表示并发请求数， -p表示请求类型  -T
postfile需要在当前目录下编写文件postfile prodid=0101&请求参数

通过浏览器测试

## 秒杀系统的并发解决方式   可能会出现并发问题，库存为负数，秒杀结束以后还是显示秒杀成功（超卖问题），还可能会出现连接超时问题
（1）连接超时问题可以通过连接池解决
jedis连接池
JedisPool jedisPoolInstance=JedisPoolUtil.getJedisPoolInstance();
Jedis jedis=jedisPoolInstance.getResource();
(2)通过乐观锁可以解决并发超卖问题，每次对数据操作都判断版本号是否一致
对库存数量进行监视：jedis.watch(kcKey);
然后将秒杀过程（7）放入到事务中去
//使用事务
Transaction multi=jedis.multi();
//组队操作
multi.decr(kcKey);
multi.sadd(userKey,uid);
//执行阶段
List<Object> results=multi.exec();
if(results==null || results.size()==0){
  System.out.println("秒杀失败了.....");
  jedis.close();
  return false;
}  

** 秒杀系统中的事务和锁机制 **

## 通过lua脚本解决库存遗留问题

# redis持久化操作
RDB：在指定的时间间隔内将内存中的数据集快照写入磁盘

dump.rdb
在redis.conf中配置文件名称，默认为dump.rdb
dbfilename dump.rdb

vi 打开行号 ::set nu 421行redis.conf可以看到rdb文件    以及rdb文件的保存目录，可以更改，默认情况下是当前目录下，就是和redis.conf的同一目录下
关于rdb文件的保存备份时间在redis.conf文件中存在配置      370行左右
save 3600 1   3600秒之后，至少1个key改变，则备份   1小时
save 300  100   5min 之后，如果100个key改变，则备份
save 60   1000    1min之后，如果1000个key改变，则备份
如果越短的时间间隔内，改变的key的数量越多，则备份的时间间隔越短

修改redis.conf的备份时间间隔之后，需要将redis重启一下
直接kill进程    ps -ef | grep redis   查看对应的进程号
kill -g 对应的进程号
然后重新启动redis   /usr/local/bin/redis-server /etc/redis.conf   通过/etc目录下的redis.conf文件后台启动redis
然后模拟快速变化redis中的数据
set k1 v1
set k2 v2
...
set k20 v20

可以看到redis.conf同级目录下的dump.rdb文件大小已经变大，说明进行了持节化操作
** 如果设置为save 30 10 表示30秒内如果变化10个key，则进行持久化，但是如果改变了12个key，则该次持久化操作只会持久化10个元素 **

+ 手动持久化还是自动持久化的设置
+ save:save手动保存
+ bgsave:redis会在后台异步进行快照操作，快照同时还可以响应客户端请求

可以通过lastsave命令获取最后一次成功执行的快照时间
** 关于redis.conf文件中rdb持久化的配置**
## rdb备份是如何执行的
redis会创建一个fork子进程来进行持久化，会先将数据写入到一个临时文件中去，待持久化过程都结束了，在用这个临时文件替换上次的持久化好的文件，整个过程，主进程是不进行任何io操作的，这就确保了极高的性能，如果需要进行大规模的数据恢复，且对于数据的恢复的完整性不是非常敏感，那么rdb方式比aof方式更加高效。
rdb的缺点是最后一次持久化的数据可能丢失

fork的作用是复制一个与当前进程一样的进程，新的进程的所有的数据（变量，环境变量，程序计数器等）数值都和原进程一样，但是是一个全新的进程，并作为原进程的子进程

## rdb的恢复
直接关闭redis
把备份的文件拷贝到工作目录下cp dump2.rdb dump.rdb
启动redis，通过配置文件启动redis   ，备份文件的数据会直接自动加载
/usr/local/bin/redis-server /etc/redis.conf   将dump.rdb文件放到redis.conf同级目录下，其实是redis.conf通过配置的rdb文件目录找到该备份文件的
然后会自动通过rdb文件恢复数据到内存中

# aof备份
以日志的形式来记录每一个写操作（增量保存），将redis执行过的所有写指令记录下来（读操作不记录），只许追加文件但不可以改写文件，redis启动之初会读取该文件重新构建数据，换言之，redis重启的活就根据日志文件的内容将写指令从前到后执行一次以完成数据的恢复工作

aof持久化流程

rdb默认是开启的，aof默认不开启

aof文件的保存路径，同rdb的路径一致

# 如果aof和rdb同时开启，系统默认取aof的数据（数据不会存在丢失）

# 直接通过docker拉去的redis进行没有配置文件，需要自己下载配置文件，
如果遇到aof文件损坏没通过/usr/local/bin/redis-check-aof --fix appendonly.aof进行恢复

aof同步频率设置
appendfsync always 始终同步，每次redis的写入都会立刻计入日志，性能较差单数据完整性较好
appendfsync everysec  每秒同步，每秒记录日志一次，如果宕机，本秒的数据可能丢失
appendfsync no，redis不主动进行同步，把同步时机交给操作系统


rewrite压缩
只关心最终的结果，不关心执行的过程
set k1 v1
set k2 v2
最终等价于：set k1 v1 k2 v2   重写压缩

重写需要有一个阈值，只有达到阈值才进行重写
重写过程也是fork一个子进程，进行读写复制，
重写的触发条件：大于64M的100% ----》就是大于128M就开始重写，重写到一个临时文件，然后将临时文件替换aof文件
 

