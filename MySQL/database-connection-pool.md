# 数据库连接池
## 关于数据库连接池的安全问题
数据库连接池承担的任务是客户端与数据库之间建立连接的桥梁，为什么需要数据库连接池呢？只有一个目的，就是复用连接资源。较少系统开销，提升系统性能。
### 连接池的线程数设为多少比较合理
首先需要思考一个问题，线程数越多越好吗，物极必反，答案肯定不是越多越好，但是原因呢？学过nginx的都知道，nginx用4个线程就能扛住几万的并发，这就不得不提起计算机的基础知识。计算机的一个核心可以运行几百个线程，当然，这是操作系统时间片轮转机制的体现，一个核心在同一时刻只能运行一个线程，在这个线程运行时间满足后，切换下一个线程，同时需要保存上一个线程运行多信息，以便于下次轮到它时继续执行，这就是上下文切换。由于这个过程时间很短，我们无法察觉，但是这就是一个核心运行多个线程的秘密。所以当你的线程数越多，上下文切换就会越多，系统开销就会增大。

但是，这不足以作为线程数越多就不好的主要原因，系统慢的原因无非就三种：CPU，磁盘，网络，我们先排除磁盘和网络，CPU的影响点就是上面说到点上下文切换，那么磁盘怎么影响呢？磁盘的IO是有时间消耗的，在这一段时间内，线程是阻塞等待获取数据的，这段时间线程可以被切走，所以为了避免线程在很久的等待时间中无事可做，可以适当将线程数调大，这就就能在更多的阻塞时间内让更多的线程做事。当然，这都是基于机械硬盘，如果是固态硬盘，那么寻址，IO花费的时间极少，那么就需要调小线程数，避免线程之间过多的等待。
## 手写一个数据库连接池