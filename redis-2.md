# redis配置文件

# redis的发布订阅机制
直接在一个客户端定于某一个通道的信息：
subscrib channel1 
然后再另一个客户端发布消息：publish channel1 goldstine
可以看到其他的订阅该通道的客户端可以收到该消息

# redis6新数据类型
## bitmaps
合理地使用操作位能够有效地提高内存使用率和开发效率
（1）redis提供了bitmaps这个数据类型，实际上他就是一个字符串（k-v）,但是他可以对字符串的位进行操作
（2）bitmaps单独提供了一套命令，所以在redis中使用bitmaps和使用字符串的方法不太相同，可以把bitmaps想象成一个以位为单位的数组，数组的每一个单位只能存储0和1,数组的下标在bitmaps中叫做偏移量
常用命令：
（1）setbit
setbit key offset value 设置bitmaps中某个偏移量的值0或1
offset偏移量从0开始
setbit users:20210101 12 1
setbit users:20210101 2 1
【注】很多应用的用户id以一个指定数字（例如10000）开头，直接将用户id和bitmaps的偏移量对应，势必会造成一定的浪费，通常的做法是每次setbit操作时将用户id减去指定数字

在第一次初始化bitmaps是，如果偏移量非常大，那么整个初始化过程执行会比较慢，可能会造成redis阻塞

（2）getbit
getbit key offset 获取bitmaps中某一个偏移量的值

getbit 
获取键的第offset位的值（从0开始算）

（3）bitcount
统计字符串被设置为1的bit数  可以设置start也可以设置end  -1表示最后一位

bitcount users:20210101 1 3   统计第一个字节和第二个字节和第三个字节中1的个数  总共24个bit中1的个数  从第一个字节到第三个字节，[]
bitcount users:20210101 0 -2   统计下标0到下标为倒数第2的字节中1的个数
（4）bitop
bitop and(or/not/xor)<destkey> k1 k2
复合操作，可以做多个bitmaps的and(交集)，or（并集），not(非)，xor(异或)操作并将结果保存在destkey中

比如有2个bitmaps：分别表示两天对某一个网站的访问情况
bitop and unique:users:and:20201104_03 unique:users:20201103 unique:users:20201104  
将unique:users:20201103和unique:users:20201104求交集，将结果存入unique:users:and:20201104_03  其实就是按位与

## HyperLogLog数据类型   主要应用于基数问题
在redis里面。每个HyperLogLog键只需要花费12KB内存，就可以计算接近2^64个不同元素的基数
基数：就是不重复元素的个数
（1）pfadd
pfadd key element 
添加指定元素到HyperLogLog
如果估计得基数变化，表示添加成功，返回1.否则返回0

（2）pfcount key
统计基数
（3）pfmerge  
pfmerge destkey sourcekey 
将一个或多个HLL合并以后的结果存储在另一个HLL中，比如每月的活跃用户可以使用每天的活跃用户进行计算和并得到

## Geospatial
表示经度和纬度进行查询
（1）geoadd
geoadd key longitude latitude member 
添加地理位置
geoadd china:city 121.47 31.23 shanghai
（2）geopos key 名称
geopos china:city shanghai

(3)geodist china:city beijing shanghai km
获取两个地方的直线距离

(4)georadius
以给定的经纬度为中心，找出某一半径内的元素
georadius china 110 30 1000 km 
找出以经纬度为110 30的1000km范围以内的城市

