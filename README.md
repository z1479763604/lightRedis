# lightRedis
千行左右的简版Redis

根据[《redis设计与实现》](http://redisbook.com/)和[redis开源项目](https://github.com/antirez/redis)简单实现了c++版的redis数据库单机部分:内部存储结构,支持三种数据结构*string*,*list*,*array*,数据表rehash,支持10种命令,aof文件持久化.

## detail


### 数据结构

虽然是C++项目,但大部分是按照C的思路(受源码和《设计》启发)+stl+简单的封装,项目中数据结构学习了redis本体的字典作为数据库支撑,键值类型支持多种数据类型(虽然键大都默认为字符串类型),用标准库的string代替了redis自己封装实现的SDS类型,复写了list类型,，
其作用和方法与标准库list并没有太大区别,array则是直接用数组类型支撑.

>主要文件:
>* dict.h:存储数据的主要数据结构,要求监控rehash状态,存储数据,数据的大部分增删改查操作也集中于此
>
>   --__数据的删除操作可能相当危险,redis为实现支持多种数据类型,将字典内部键值指针都声明为void * 指针,为此redis内部将键值都包装为一个Redisobject对象,对象除了包含数据的void * 指针外还有该数据的类型和编码,以便删除的时候能够正确释放内存__
>
>   --__rehash是数据库的一种动态增长机制,类似于标准库的vector动态增长,区别是字典内部已经准备好一个空表来帮助重新分配内存,而不是临时分配;
字典内部标记了数据数量和数据表大小,数据表是数组+链表的形式,实际上允许数据量大于数据表大小,但规定负载因子大于一的时候,向空表分配比当前表大小*2大的2^n大小的空间
可见数据容量按指数增长但是考虑了初始数据库大小,分配之后将数据移动至新表并且在此过程中的增删改查都要考虑两个表的状况(根据字典标记的rehash状态),转移结束后舍弃旧表并重新准备空表__
>
>* list.h:与标准库list容器功能相似,主要结合字典结构对增删等操作稍作修改.

### 命令解析

redis支持相当丰富的命令,lightRedis中却只有三种数据类型的增操作,通用的删改查,以及3个对数据库的操作;该板块没有过于研究源码,我自己的是通过
converter类来解析客户端发来的命令,将命令分解成指令-键-值的形式交予controller类,获得数据后对数据库进行相应的操作,再将返回值交予converter进行转换,其结果传递给reactor

>主要文件:
>* converter.h:将用户指令转换为数据/将返回数据转换为字符串传递给消息模块
>* controller.h:获取数据并进行数据库操作/返回需要的数据给转换器

### server构成

redis使用单线程+IO复用来处理并发请求(在其集群中可能有更高明的手段(因为并未研究= =)),也就是reactor模型,他以事件驱动组件为核心(linux下为epoll),内部分离并
处理事件,在该数据库的业务逻辑中,reactor的职责是监听新用户连接并监听新用户的可读状态,用户发来命令后reactor利用命令解析模块操纵数据库,并获得用户应该得到的返回值,将其缓存并修改监听用户的可写状态,
缓存数据是保证多个用户之间不会乱传数据,监听用户到可写时将相应的缓存写入
> server开启->reactor监听连接->用户连接并键入命令->reactor监听用户可读状态/可读便处理命令并缓存返回值->reactor转为监听可写状态/可写便写入相应缓存->reactor转为监听可读状态/等待用户键入命令

### 文件持久化

redis有两种文件持久化方式,RDB和AOF;RDB是遍历数据库并写入到文件中,执行恢复的时候读取文件;AOF则是直接记录用户的命令,执行恢复时再次执行命令以复原数据库;
AOF实现简单所以lightRedis做了AOF持久化.恢复过程可能时间较长,要做出相应的措施保护当前用户的数据库访问.

> 两种方式对比:lightRedis的AOF是一种不负责的持久化,单纯的记录命令可能徒增处理机负担,例如用户执行了增加操作不久后又删除了相应的键,在文件恢复的时候就是浪费系统资源
;应该想办法分析并减轻AOF恢复的不必要负担.
>
> RDB可以避免这个问题,但新的问题是RDB开始时会不停操作数据库并占用磁盘IO,如何才能保证现有用户的访问不受影响? lightRedis的aof备份是在reactor当中的,每当有命令被正确执行便写入一次aof文件,
> 相对来说操作比较分散,不会占用大量资源,aof恢复也基本处于server开机的状态,不会过于影响用户.
