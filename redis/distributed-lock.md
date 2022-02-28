# Distribute Lock
## OverView
多个机器实例访问共享资源时需要加的锁就是分布式锁，举个例子：一个服务在线上环境有10个实例，一个定时任务会在固定时间去修改数据库中的某个值使其加1，我们的期望是每次定时任务只加1，但是10个实例都运行着服务，那么就有可能竞争这个定时任务，不做措施就可能引起线程安全问题，因为对一个数据库的操作是线程不安全的。分布式锁就是解决这个问题。
## 使用Redis做分布式锁
* redis是单进程单线程模式的数据库
* 提供了像lua脚本的原子操作方法
* 实现起来简单方便
## 使用zk做分布式锁
* 

## redis分布式锁demo
使用分布式锁的前提就是，确定这个非原子的操作在分布式环境下会引起线程安全问题。比如这样一个情况：在玩游戏的时候需要加载很大的图形场景，这个加载的过程如果是异步的，当你点击加载后程序立刻返回，服务器开始加载这个场景，而你看到的页面一定是加载中，但是你可以点击页面上的其他操作，不影响图形界面的加载。但你一定不想等太久，所以程序需要判断超时加载失败的处理，假设我们用一个定时任务每隔5秒访问一次数据库看有没有加载成功，如果超过1分钟就置失败处理。那么这个定时任务会去修改数据库的状态，如果不加锁，一个发现超时并将游戏状态置为成功，另一个线程有可能进来又改成失败了。上代码：
```java
@Component
public class RedisLock {
    private static final String NX = "NX";
    private static final String EX = "EX";
    private static final String OK = "OK";

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    /**
     * 加锁
     */
    public void lock(String key, String value, long expire) {
        String result = set(key, value, expire);
        return OK.equalsIgnoreCase(result);
    }

    public String set(String key, String value, long time) {
        return redisTemplate.execute((RedisCallback<String>) connection -> {
            final Object nativeConnection = connection.getNativeConnection();
            String result = null;
            // jedis
            if (nativeConnection instanceof JedisCommands) {
                result = ((JedisCommands) nativeConnection).set(key, value, NX, EX, seconds);
            }
            return result;
        });
    }

    public void unlock(final String key, final String lockValue) {
        final String value = redisTemplate.opsForValue().get(key);
        if (StringUtils.equals(lockValue, value)) {
            redisTemplate.delete(key);
        }
    }
}
```
```java
public class CheckGameStatusScheduler {

    private static final String GAME_STATUS_CHECK = "ServiceName:GameStatusCheck";

    @Autowired
    private RedisLock redisLock;

    @Scheduled(cron = "0 0 * * * ?")
    public void checkWorldStatus() {
        final String lockValue = UUID.randomUUID().toString();
        try {
            if (redisLock.lock(GAME_STATUS_CHECK, lockValue, TimeUnit.MINUTES.toSeconds(30)))) {
                updateStatus();
            }
        } catch(final Exception e) {
            LOG.print("catch an error");
        } finally {
            redisLock.unlock(GAME_STATUS_CHECK, lockValue);
        }
    }
}
```
## zk分布式锁demo
```java

```
## redis与zookeeper做分布式锁的区别
### redis
如果要求有较好的准确性，建议使用zookeeper做分布式锁，redis做分布式锁存在一些缺陷。目前企业对redis的架构主要以主从架构为主，假设现在锁放在了master，任务正在执行，redis还未将锁同步到slave节点，master突然挂了，这就会产生锁丢失的问题。
### zk
zk做分布式锁的原因是利用了zk创建的临时文件的唯一性和监听机制，比如有三个节点，他们分别创建了三个有序的临时文件，LOCK_00000000、LOCK_00000001、LOCK_00000002，在获取锁时，判断自己是不是文件中最小的，如果是就获取锁执行，执行完成后再finally块中删除这个文件，监听机制感应到后，通知下一个节点可以获取锁，每个节点只需要监听自己前一个节点的文件有没有被删除，如果服务节点过多，监听每个会浪费大量的资源。用临时文件的原因是zk在监听到客户端与服务端断开后，会删除这个临时文件，释放这个锁。