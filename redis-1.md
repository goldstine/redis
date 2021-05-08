# redis
## nosql数据库
nosql数据库不支持ACID，但是支持事务
nosql更适合高性能的数据存储
## redis的安装
```
首先将redis的安装包上传到centos上的/opt，然后通过安装gcc的环境，因为需要make编译将redis编译成.c的项目
然后将.tar.gz文件解压 tar -zxvf redis...tar.gz
然后进入该文件目录 cd redis 执行make编译
然后通过make install安装，默认会安进/usr/local/bin目录下
进入/usr/local/bin,可以看到一下目录,比如redis-cli
```
## 启动redis
（1）前台启动：直接执行redis-server
(2)后台启动：在解压后的redis中有一个redis-conf文件，修改其中的配置
 然后通过该配置文件进行启动

## redis与memcache之间的区别
（1）redis不是使用的多线程，而是使用的单线程+IO多路复用技术来实现多线程的效果 ，而memcache使用的是多线程+锁的机制
（2）memcache只将数据存储在内存中，不能够对数据进行持久化操作，而redis可以对数据进行持久化操作
（3）memcache只支持单一的数据类型，而redis支持更多的数据类型
单线程+IO多路复用技术
## redis的key键操作
连接redis
/usr/local/bin/redis-cli
查看所有的key
+ keys *
+ exists key 判断某一个key是否存在
+ type key查看key是什么类型
+ del key 直接删除指定的key
+ unlink key根据value选择非阻塞删除：仅将keys从keyspace元数据中删除，真正的删除会在后续的异步操作中完成
+ expire key 10 :给指定的key设置过期时间
+ ttl key:查看还有多少秒过期-1表示永不过期，-2表示已经过期

select命令切换数据库
dbsize:查看当前数据库的key的数量
flushdb：清空当前库
flushall通杀全部库，清空所有库
## redis的基本数据类型
### String
+ String
String是redis最基本的类型，
String类型是二进制安全的，意味着redis的String可以包含任何数据，比如jpg图片或者序列化的对象
String类型是redis最基本的数据类型，一个redis中字符串value最多可以是512M
```
常用命令：
（1）set key value  如果对相同的key进行set就会对value进行覆盖
(2)get key
(3)append key value 将指定的value追加到原值的末尾
（4）strlen key获得值的长度
(5)setnx key value：-----》只有当key不存在时，设置key的值
(6)incr key  将key中存储的数字值加1   ；只能对数字值操作，如果为空，新增的值为1
（7）decr key  将key中存储的数字值减1 ；只能对数字值操作，如果为空，新增的值为-1
（8）incrby/decrby key 步长：将key中存储的数字值增减，自定义步长
incr key是对数值进行原子操作，所谓原子操作是指不会被线程调度机制打断的操作，这种操作一旦开始，就一直运行到结束，中间不会出现线程的切换
** 在单线程中，能够在单条指令中完成的操作都可以认为是原子操作，因为中断只能发生于指令之间 **
** 在多线程中，不能被其他进程（线程）打断的操作就叫原子操作 **
redis单命令的原子性主要得益于redis的单线程
（9）mset key1 value1 key2 value2 key3 value3...  ：一次可以设置多个k-v
(10) mget key1 key2 key3...：一次可以获得多个k对应的v
（11）msetnx key1 value1 key2 value2 key3 value3...: 如果之前已经设置的值，那么当前设置的值不会生效，**为了保证原子性，如果有一个已经存在，则设置失败，那么msetnx就失败**
（12）getrange key 起始位置 结束位置    ：获得值得范围类似java中的substring,前包，后包
（13）setrange key 起始位置 value   ： 用value覆盖原来的value，从起始位置开始
（14）setex key 过期时间 value：设置键值的同时设置过期时间，单位秒
（15）getset key value：以新换旧，设置了新值的同时获得旧值，这里获得是在控制台上打印出原先获得的值，然后用新值替换设置
** 由于redis的单线程，所以上面这些操作都是原子性操作 **

** redis的底层结构类似于arrayList的动态冗余内存与分配方式 ** 扩容的方式，当字符串长度小于1M时，每次扩容为原来的2倍；如果超过1M，扩容时只会多扩1M的空间，需要注意的是字符串最大长度是512M
redis的数据类型，指的是value的数据类型，不是指key的数据类型
```
### 列表List
单键多值，redis列表是简单的字符串列表，按照插入顺序排序，可以添加一个元素到列表的头部或者尾部
它的底层实际上是一个双向链表，对两端的操作性能很高，通过索引下标的操作中间的节点性能会较差
常用操作：
```
lpush/rpush key value1 value2 value3 ... 从左边或右边插入一个或多个值
lpop/rpop key 从左边或右边吐出一个值 值在键在，值光键亡
rpoplpush key1 key2从key1列表右边吐出一个值，插入到key2列表左边
lrange key start stop 按照索引下标获得元素（从左到右）

lrange mylist 0 -1  0表示左边第一个，-1表示右边第一个（0-1表示获取所有）
lindex key value按照索引下标获得元素（从左到右）
llen key 获得列表的长度
linsert key before value newvalue在value的后面插入newvalue插入值
lrem key n value从左边删除n个value（从左到右）
lset key index value将列表key下标为index的值替换成value

List的数据结构为快速链表quickList
首先在列表元素较少的情况下会使用一块连续的内存存储，这个结构是zipList，也即是压缩列表，他将所有的元素紧挨着一起存储，分配的是一块连续的内存
当数据量比较多的时候次啊会改成快速链表
普通的双向链表的指针存储空间浪费较大，所以redis将链表和ziplist结合起来组成了quicklist，也就是将多个ziplist使用双向指针串起来使用，这样既满足了快速插入删除的性能，也不会出现太大的空间冗余
```
## 集合Set
set是可以自动重排的，不存在重复的数据，redis的set是String类型的无序集合，他底层是一个value为null的hash表，所以添加，删除，查找复杂度都是O（1）
```
（1）sadd key value1 value2...
将一个或多个member元素加入到集合key中，已经存在的member元素将被忽略
（2）smembers key取出该集合的所有值
（3）sismember key value判断集合key是否含有该value值，有1，没有0
（4）scard key 返回该集合元素的个数
（5）srem key value1 value2... 删除集合中的某个元素
（6）spop key随机从该集合中吐一个值 ，该值会从集合中删除
（7）srandmember key n随即从该集合中取n个值，不会从集合中删除
集合与集合之间的操作，多个集合操作
（8）smove source desstination value把集合中一个值从集合移动到另一个集合 ，原来集合中的元素会被删除
（9）sinter key1 key2 返回两个集合的交集元素
（10）sunion key1 key2 返回两个集合的并集元素
（11）sdiff key1 key2返回两个元素的差集元素（key1中的，不包括key2中的）
set结构是字典，java中hashset的内部实现使用的是hashmap，只不过所有的value都指向同一个对象
redis的set结构也是一样的，它的内部也使用hash结构，所有的value都指向同意内部值
```
## 哈希hash
redis hash是一个string类型的field和value的映射表，实际上value是一个引射表，就是一个表（对象）一张表对应一个bean对象
```
（1）hset key field value给key集合中的field键赋值value
（2）hget key field 从key集合中的field去除value
（3）hmset key1 field1 value1 field2 value2...批量设置hash的值
（4）hexists key1 field 查看hash表中key，给定域field是否存在   存在则返回1，不存在则返回-1
(5)hkeys key列出该hash集合中所有的field
（6）hvals key列出该hash集合所有的value
（7）hincrby key field increment 为哈希表key中的域field的值加上增量1 -1       比如：hincrby key field 2  将field加上2
（8）hsetnx key field value 将哈希表key中的域field的值设置为value，当且仅当域field不存在

Hash类型对应的数据结构有两种，ziplist(压缩列表)，hashtable(哈希表)，当field-value长度较短且个数较少的时候，使用·ziplist，否则hashtable
```
## 有序集合Zset（sorted set）
redis的有序集合zset与普通的集合set非常相似，是一个没有重复元素的字符串集合
不同的是有序集合的每一个成员都关联一个评分（score），这个评分被用来按照从最低分到最高分的方式排序集合中的成员，集合的成员视为一个，但是评分是可以重复的
所以可以根据评分获得一个范围的元素
```
（1）zadd key socer1 value1 score2 value2...将一个或多个member元素及其score值加入到有序集合中
(2)zrange key start stop [withscores]   返回有序集合key中，下标在start stop之间的元素
带withscores，可以让分数一起和值返回到结果集
（3）zrangebyscore key min max [withscores] [limit offset count] 返回有序集合key中，所有score值介于min max之间（包括等于min或max）的成员
zrangebyscore key 300 500 withscores   起初评分在300 到500之间的项       此时默认是从大到小进行排序
（4）zrevrangebyscore key min max [withscores] [limit offset count] 同上，改为从大到小排序
(5)zincrby key increment value  为元素的score加上增量
(6)zrem key value 删除该集合下，指定值的元素               这里是按照值进行删除操作，而不是按照score删除，score更像是key
(7)zcount key min max 统计该集合，分数区间内的元素个数   zcount topn 200 300 统计200到300之间的值的个数
(8)zrank key value返回该值在集合中的排名 ，从0开始

底层还使用了跳表进行查找，可以更快地找到所需要的元素
```

