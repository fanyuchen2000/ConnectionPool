# ConnectionPool
## 项目背景
在C++ 开发中，我们在和数据库交互通常需要以下几个过程：

-   TCP三次握手
-   mysql_connect 建立连接
-   执行SQL语句
-   mysql_close 释放连接
-   TCP四次挥手

可以看到，我们每次进行**SQL语句**执行的时候是有很多时间消耗在1、2、4、5，造成了大量的时间浪费。
所以，这里我们就需要引入**连接池**，减小2、4的时间消耗，以达到连接复用的目的。

## 关键技术点
MySQL数据库编程，单例模式，queue队列容器，C++11多线程编程，线程互斥，线程同步通信和unique_lock，基于CAS的原子整形，智能指针shared_ptr，lambda表达式，生产者-消费者线程模型。

## 连接池功能点
连接池一般包括了数据库连接所用的ip地址、port端口号、用户名和密码以及其他的性能参数，例如初始连接量，最大连接量，最大空闲时间和连接超时时间等，该项目是基于C++语言实现的连接池，主要也是实现以上几个所有连接池都支持的通用基础功能。

**初始连接量(initSize):** 表示连接池事先会和MySQL Server创建的连接个数，当应用发起数据库访问时，不用再创建和MySQL Server新的连接，直接从连接池中获取一个可用的连接就可以了，使用完成后，并不用释放连接，而是把当前连接归还于连接池中。

**最大连接量(maxSize):** 当并发访问MySQL Server的请求增多时，初始连接量已经不够使用了，此时会根据新的请求数量去创建更多的连接给应用去使用，但是新创建的连接数量上限是maxSize，不能无限制的创建连接，因为每一个连接都会占用一个socket资源，一般连接池和服务器程序是部署在一台主机上的，如果连接池占用过多的socket资源，那么服务器就不能接收太多的客户端请求了。当这些连接使用完成后，再次归还到连接池当中来维护。

**最大空闲时间(maxIdleTime):** 当访问MySQL的并发请求多了以后，连接池里面的连接数量会动态增加，上限
是maxSize个，当这些连接用完再次归还到连接池当中。如果在指定的maxldleTime里面，这些新增加的连接都没有被再次使用过，那么新增加的这些连接资源就要被回收掉，只需要保持初始连接量initSize个连接就可以了。

**连接超时时间(connectionTimeout):** 当MySQL的并发请求量过大，连接池中的连接数量已经到达maxSize了，而此时没有空闲的连接可供使用，那么此时应用从连接池获取连接无法成功，它通过阻塞的方式获取连接的时间如果超过connectionTimeout时间，那么获取连接失败，无法访问数据库。

## MySQL Server参数介绍
```
mysql> show variables like 'max_connections';
```
该命令可以查看MySQL Server所支持的最大连接个数，超过max_connections数量的连接，MySQL Server会直接拒绝，所以在使用连接池增加连接数量的时候，MySQL Server的max_connections参数也要适当的进行调整，以适配连接池的连接上限。

## 功能设计实现
ConnectionPool.cpp 和 ConnectionPool.hpp: 连接池代码实现
Connection.cpp 和 Connection.hpp: 数据库操作代码
main.cpp: 测试代码
mysql.ini: 配置文件

### 连接池主要包含了以下功能点

 1. 连接池只需要一个实例，所以ConnectionPool以单例模式进行设计
 2. 从ConnectionPool中可以获取和MySQL的连接Connection
 3. 空闲连接Connection全部维护在一个线程安全的Connection队列中，使用线程互斥锁保证队列的线程安全
 4. 如果Connection队列为空，还需要再获取连接，此时需要动态创建连接，上限数量是maxSize
 5. 队列中空闲连接时间超过maxldleTime的就要被释放掉，只保留初始的initSize个连接就可以了，这个功能点放在独立的线程中去做
 6. 如果Connection队列为空，而此时连接的数量已达上限maxSize，那么等待connectionTimeout时间如果还获取不到空闲的连接，那么获取连接失败，此处从Connection队列获取空闲连接，可以使用带超时时间的mutex互斥锁来实现连接超时时间
 7. 用户获取的连接用shared＿ptr智能指针来管理，用lambda表达式定制连接释放的功能（不真正释放连接，而是把连接归还到连接池中）
 8. 连接的生产和连接的消费采用生产者—消费者线程模型来设计，使用了线程间的同步通信机制条件变量和互斥锁

## 开发平台
采用vscode连接的远程Linux平台进行开发

## 压力测试
| 数据量 |未使用连接池花费时间|使用连接池花费时间|
|--|--|--|
| 1000 | 单线程：1834ms  四线程：501ms | 单线程：1055ms 四线程：389ms|
| 5000 | 单线程：10050ms 四线程：2114ms | 单线程：5240ms 四线程：2033ms |
|10000 | 单线程：19350ms 四线程：4558m | 单线程：10284ms 四线程：4019ms |

可以看到，在单线程的表现下，连接池的性能尤为优异。但是，在多线程表现的表现并不是特别突出。

经过分析：**多线程涉及到锁所带来的一系列开销，这个开销可能会抵消避免频繁建立连接所省去的开销**。
