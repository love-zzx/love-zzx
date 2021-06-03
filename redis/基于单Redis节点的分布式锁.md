# 基于单Redis节点的分布式锁

## Redis 实现锁

首先，Redis客户端为了**获取锁**，向Redis节点发送如下命令：

```redis
> set lock:codehole my_random_value ex 5 nx
OK
```

Redis 2.8 版本中作者加入了 set 指令的扩展参数，使得 setnx 和 expire 指令可以一起执行

在上面的`SET`命令中：

- `my_random_value`是由客户端生成的一个随机字符串，它要保证在足够长的一段时间内在所有客户端的所有获取锁的请求中都是唯一的。
- `NX`表示只有当`lock:codehole`对应的key值不存在的时候才能`SET`成功。这保证了只有第一个请求的客户端才能获得锁，而其它客户端在锁被释放之前都无法获得锁。
- `ex 5`表示这个锁有一个5秒的自动过期时间。当然，这里5秒只是一个例子，客户端可以选择合适的过期时间。

最后，当客户端完成了对共享资源的操作之后，执行下面的Redis Lua脚本来**释放锁**：

```lua
if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1])
else
    return 0
end
```

这段Lua脚本在执行的时候要把前面的`my_random_value`作为`ARGV[1]`的值传进去，把`lock:codehole`作为`KEYS[1]`的值传进去。

设置一个随机字符串`my_random_value`是很有必要的，它保证了一个客户端释放的锁必须是自己持有的那个锁。假如获取锁时`SET`的不是一个随机字符串，而是一个固定值，那么可能会发生下面的执行序列：

1. **客户端1获取锁成功。**
2. **客户端1在某个操作上阻塞了很长时间。**
3. **过期时间到了，锁自动释放了。**
4. **客户端2获取到了对应同一个资源的锁。**
5. **客户端1从阻塞中恢复过来，释放掉了客户端2持有的锁。**

之后，客户端2在访问共享资源的时候，就没有锁为它提供保护了。

第四个问题，释放锁的操作必须使用Lua脚本来实现。释放锁其实包含三步操作：’GET’、判断和’DEL’，用Lua脚本来实现能保证这三步的原子性。否则，如果把这三步操作放到客户端逻辑中去执行的话，就有可能发生与前面第三个问题类似的执行序列：

1. **客户端1获取锁成功。**
2. **客户端1访问共享资源。**
3. **客户端1为了释放锁，先执行’GET’操作获取随机字符串的值。**
4. **客户端1判断随机字符串的值，与预期的值相等。**
5. **客户端1由于某个原因阻塞住了很长时间。**
6. **过期时间到了，锁自动释放了。**
7. **客户端2获取到了对应同一个资源的锁。**
8. **客户端1从阻塞中恢复过来，执行`DEL`操纵，释放掉了客户端2持有的锁。**

假如Redis节点宕机了，那么所有客户端就都无法获得锁了，服务变得不可用。为了提高可用性，我们可以给这个Redis节点挂一个Slave，当Master节点不可用的时候，系统自动切到Slave上（故障转移）。但由于Redis的主从复制（replication）是异步的，这可能导致在failover过程中丧失锁的安全性。考虑下面的执行序列：

1. **客户端1从Master获取了锁。**
2. **Master宕机了，存储锁的key还没有来得及同步到Slave上。**
3. **Slave升级为Master。**
4. **客户端2从新的Master获取到了对应同一个资源的锁。**

于是，客户端1和客户端2同时持有了同一个资源的锁。锁的安全性被打破。针对这个问题，antirez设计了Redlock算法

## 分布式锁Redlock

分布式锁的算法Redlock，它基于N个完全独立的Redis节点（通常情况下N可以设置成5）

运行Redlock算法的客户端依次执行下面各个步骤，来完成**获取锁**的操作：

1. 获取当前时间（毫秒数）。
2. 按顺序依次向N个Redis节点执行**获取锁**的操作。这个获取操作跟前面基于单Redis节点的**获取锁**的过程相同，包含随机字符串`my_random_value`，也包含过期时间(比如`PX 30000`，即锁的有效时间)。为了保证在某个Redis节点不可用的时候算法能够继续运行，这个**获取锁**的操作还有一个超时时间(time out)，它要远小于锁的有效时间（几十毫秒量级）。客户端在向某个Redis节点获取锁失败以后，应该立即尝试下一个Redis节点。这里的失败，应该包含任何类型的失败，比如该Redis节点不可用，或者该Redis节点上的锁已经被其它客户端持有（注：Redlock原文中这里只提到了Redis节点不可用的情况，但也应该包含其它的失败情况）。
3. 计算整个获取锁的过程总共消耗了多长时间，计算方法是用当前时间减去第1步记录的时间。如果客户端从大多数Redis节点（>= N/2+1）成功获取到了锁，并且获取锁总共消耗的时间没有超过锁的有效时间(lock validity time)，那么这时客户端才认为最终获取锁成功；否则，认为最终获取锁失败。
4. 如果最终获取锁成功了，那么这个锁的有效时间应该重新计算，它等于最初的锁的有效时间减去第3步计算出来的获取锁消耗的时间。
5. 如果最终获取锁失败了（可能由于获取到锁的Redis节点个数少于N/2+1，或者整个获取锁的过程消耗的时间超过了锁的最初有效时间），那么客户端应该立即向所有Redis节点发起**释放锁**的操作（即前面介绍的Redis Lua脚本）。

当然，上面描述的只是**获取锁**的过程，而**释放锁**的过程比较简单：客户端向所有Redis节点发起**释放锁**的操作，不管这些节点当时在获取锁的时候成功与否。

由于N个Redis节点中的大多数能正常工作就能保证Redlock正常工作，因此理论上它的可用性更高。我们前面讨论的单Redis节点的分布式锁在failover的时候锁失效的问题，在Redlock中不存在了，但如果有节点发生崩溃重启，还是会对锁的安全性有影响的。具体的影响程度跟Redis对数据的持久化程度有关。

假设一共有5个Redis节点：A, B, C, D, E。设想发生了如下的事件序列：

1. 客户端1成功锁住了A, B, C，**获取锁**成功（但D和E没有锁住）。
2. 节点C崩溃重启了，但客户端1在C上加的锁没有持久化下来，丢失了。
3. 节点C重启后，客户端2锁住了C, D, E，**获取锁**成功。

这样，客户端1和客户端2同时获得了锁（针对同一资源）。

锁的用途分为两种：

- 为了效率(efficiency)，协调各个客户端避免做重复的工作。即使锁偶尔失效了，只是可能把某些操作多做一遍而已，不会产生其它的不良后果。比如重复发送了一封同样的email。
- 为了正确性(correctness)。在任何情况下都不允许锁失效的情况发生，因为一旦发生，就可能意味着数据不一致(inconsistency)，数据丢失，文件损坏，或者其它严重的问题。

结论：

- 如果是为了效率(efficiency)而使用分布式锁，允许锁的偶尔失效，那么使用单Redis节点的锁方案就足够了，简单而且效率高。Redlock则是个过重的实现(heavyweight)。
- 如果是为了正确性(correctness)在很严肃的场合使用分布式锁，那么不要使用Redlock。它不是建立在异步模型上的一个足够强的算法，它对于系统模型的假设中包含很多危险的成分(对于timing)。而且，它没有一个机制能够提供fencing token。那应该使用什么技术呢？Martin认为，应该考虑类似Zookeeper的方案，或者支持事务的数据库。



## 客户端加锁失败处理

客户端在处理请求时加锁没加成功怎么办。一般有 3 种策略来处理加锁失败：

1. 直接抛出异常，通知用户稍后重试；
2. sleep 一会再重试；
3. 将请求转移至延时队列，过一会再试；

**直接抛出特定类型的异常**

这种方式比较适合由用户直接发起的请求，用户看到错误对话框后，会先阅读对话框的内容，再点击重试，这样就可以起到人工延时的效果。如果考虑到用户体验，可以由前端的代码替代用户自己来进行延时重试控制。它本质上是对当前请求的放弃，由用户决定是否重新发起新的请求。

**sleep**

sleep 会阻塞当前的消息处理线程，会导致队列的后续消息处理出现延迟。如果碰撞的比较频繁或者队列里消息比较多，sleep 可能并不合适。如果因为个别死锁的 key 导致加锁不成功，线程会彻底堵死，导致后续消息永远得不到及时处理。

**延时队列**

这种方式比较适合异步消息处理，将当前冲突的请求扔到另一个队列延后处理以避开冲突。



## Redis 分布式锁在 ERP 应用

ERP 中有很多需要防并发的业务场景，比如审单和分仓同时进行时，有可能会出现仓库不一致的情况，这时候就需要加锁，执行完后再释放，具体如下：

```php
// Process_RequestCounter 类
// ===================================== 利用 Redis 实现分布锁 START ====================================
// 场景： 串行执行的业务
public function distributedLock($key = '', $ttl = 60, $random = null, $tipMsg = '')
{
    $redis = $this->getRedis();
    $tipMsg = $tipMsg != '' ? $tipMsg : '请求过于频繁,相同业务单号' . $ttl . '秒内仅允许请求1次!';

    // 尝试加锁, 加随机数为了标识当前实例
    $random = $random ? $random : mt_rand(0, 9999999);

    // NX 表示只有当 $key 对应的 key 值不存在的时候才能 SET 成功
    $optional = array(
        'NX',
        'EX' => $ttl,
    );
    $rs = $redis->set($key, $random, $optional); 
    if (!$rs) {
        $tipMsg = $tipMsg != '' ? $tipMsg : '请求过于频繁,相同业务单号' . $ttl . '秒内仅允许请求1次!';
        throw new Exception($tipMsg, 500002);
        //return false;
    }

    return $random;
}
public function distributedUnlock($key = '', $random = 0)
{
    $redis = $this->getRedis();
    // 释放锁, lua 脚本，如果键值的值等于传入的值，才执行删除key操作(保证原子操作)
    $lua = <<<EOF
            if redis.call("get", KEYS[1]) == ARGV[1] then
                return redis.call("del", KEYS[1])
            else
                return 0
            end
EOF;
    $args = array(
        $key,
        $random,
    );
    $rs = $redis->eval($lua, $args, 1); // 1 表示定义一个 KEYS， ARGV[1] 为 $random
    // lua 脚本返回值为1说明当前线程加锁被正常手动释放了
    if ($rs == 1) {
        $ret = true;
    } else {
        $ret = 1;
    }

    return $ret;
}
// ===================================== 利用 Redis 实现分布锁 END ====================================
```



分仓，尝试加锁：

```php
// 加锁，防止多次操作审单以及分仓同步进行
if ($redisObj) {
    try {
        $orderListLockKey = Service_OrderForWarehouseProcessNew::getOrderListLockKey($order_sn, 'audit');
        $orderListLock[$orderListLockKey] = $prcObj->distributedLock($orderListLockKey,300);
    } catch (Exception $e) {
        //失败
        $failArr[] = array(
            'ask' => 0,
            'message' => $order_sn . ': ' . $e->getMessage(),
            'order_id' => $order['order_id'],
            'ref_id' => $order['refrence_no_platform']
        );
        continue;
    }
}

// 业务逻辑.....


// 释放队列锁
try {
    if ($orderListLock) {
        foreach ($orderListLock as $key => $val) {
            $prcObj->distributedUnlock($key, $val);
        }
        unset($orderListLock);
    }
} catch (Exception $e) {
}
```



审单，尝试加锁：

```php
// 加锁，防止多次操作审单以及分仓同步进行
if ($redisObj) {
    try {
        $orderListLockKey = self::getOrderListLockKey($refNo, 'audit');
        $orderListLock[$orderListLockKey] = $prcObj->distributedLock($orderListLockKey,500);
    } catch (Exception $e) {
        throw new Exception($refNo . ': ' . $e->getMessage(), 51898);
    }
}
```

