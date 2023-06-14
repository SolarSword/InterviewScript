# Redis的基本数据类型和使用场景
>Redis的数据类型是指键值对的value的类型。
- **String**，最*基本*的key-value结构。key是唯一标识；value是具体的值，可以是字符串，也可以是整数或浮点数: 
  - **缓存对象**
    - 使用String来缓存对象的整个JSON： SET user:1 '{"name":"cxk", "birth":"1998-08-02"}'
    - 将key分离成类似 user:{id}:{attribute}来用作缓存：MSET user:1:name cxk user:1:birth 1998-08-02 user:2:name wyf user:2:birth 1990-11-06
  - **常规计数**
    - 因为Redis处理命令是单线程，所以执行命令的过程是原子的。因此String数据类型适合计数场景，如计算访问次数、点赞、转发、库存数量等。例如：`SET article:readcount:1001 0`, `INCR article:readcount:1001`
  - **分布式锁**
    - SET命令有个NX参数，它可以实现key不存在才插入，可以利用这个特点实现分布式锁：如果key存在，插入成功，表示加锁成功；否则插入失败，标识加锁失败。再加上一个设置过期时间的参数PX以防客户端异常无法释放锁。 `SET lock_key unique_value NX PX 10000`，10s后过期。
    - 解锁的过程就是把`lock_key`键删除。但是要保证解锁的客户端与加锁的客户端是同一个，因此需要先判断锁的`unique_value`与客户端的标识是否一致。（这样的话解锁就会有两个操作，实际使用时需要用Lua脚本来保证解锁的原子性。Lua脚本可以将多个命令当作一个命令在Redis中执行。）
  - **共享Session信息**
    - 分布式的服务可能存在一个问题：同一个用户的session被存放在server1，但是在他第二次访问的时候如果被分配到server2，就需要他重复登录。Redis可以对session信息进行统一的存储和管理，让所有server都去同样一个Redis server获取session信息。

>以上个别应用场景也许和String联系不大，但是String是*基本*的key-value类型，所以将他们理解为Redis的*基本*使用场景都是可以的。
- **List**，列表是简单的字符串列表，按照插入顺序排序，头尾均可。
  - **消息队列**
    - 基本上只要满足`消息保序`、`处理重复的消息`和`保证消息的可靠性`这三个需求就可以实现消息队列。List可以满足。
    - List本身就是FIFO的，满足消息保序。生产值用`LPUSH key value`将消息插入队列，消费者使用`RPOP key`消费消息。(RPUSH+LPOP的组合也可以)
    >不过毕竟Redis不是消息队列，它不会通知消费者有新消息写入。为了防止消费值不停调用`RPOP`而消耗CPU资源，可以用`BRPOP`命令，这是阻塞式读取，在List里没有数据时，自动阻塞，直到有新数据写入。
    - 消费者在处理重复消息时需要 1.每条消息都有一个全局的ID；2.消费者要记录已经处理过的ID。这样，通过比对新消息的ID和处理过的消息的ID，就可以避免处理重复的消息。在Redis的这一层面，只需要在插入数据时再带上生成的全局ID就可以了。如：`LPUSH mq "19980802:cxk:rap"`，其中19980802就是全局ID。
    - 保证消息的可靠性是指，如果消费者在处理消息时宕机，当其重启后还可以再读取到这条消息并处理。Redis是有能够留存消息的POP命令`BRPOPLPUSH`的：它在弹出一个消息时，会再把这个消息插入另一个List（可以视为备份）作留存。
    - 总结一下：
      - 消息保序：LPUSH+RPOP
      - 阻塞读取：BRPOP
      - 重复消息处理：生产者自行实现全局唯一ID
      - 消息可靠性：BRPOPLPUSH
    >List实现的消息队列不支持多个消费者消费同一个消息。
- **Hash**，Hash是一个key-value集合，value的形式如：value=[{field1, value1}, ..., {fieldX, valueX}]。所以它比较适合存储对象。命令：`HSET key field value`，`HGET key field`。多个字段：`HMSET key field value [field value...]`
  - **缓存对象**
    - `HMSET uid:1 name CXK age 25`
    >一般来说String+JSON可以满足对象存储的需求，但是若一个对象里的字段会频繁变化，可以考虑使用Hash。
  - **购物车** （购物车三要素:user_id, item_id, number）
    - 添加商品：`HSET cart:{user_id} {item_id} 1`
    - 添加数量：`HINCRBY cart:{user_id} {item_id} 1` (+1)
    - 商品总数：`HLEN cart:{user_id}`
    - 删除商品：`HDEL cart:{user_id} {item_id}`
    - 所有商品：`HGETALL cart:{user_id}`
    >详细的商品信息的获取肯定要再用item_id去查库或者请求上游来实现。
- **Set**，Set是无序并唯一的键值集合。命令：`SADD key member [member ...]`
  - **点赞**
    - Set可以保证一个用户只能点一个赞。
    - 点赞：`SADD article:1 uid:1`
    - 取消：`SREM article:1 uid:1`
    - 获取所有点赞用户：`SMENBERS article:1`
    - 获取点赞的用户数：`SCARD article:1`
    - 判断是否点赞：`SISMEMBER article:1 uid:1`
  - **共同关注**
    - Set支持集合运算。那么利用集合的交集运算就能得出共同关注的账号ID。`SINTER uid:1 uid:2`，这两个user_id是key。
  - **抽奖**
    - Set的去重可以保证同一个用户不会中奖两次。
    - key为抽奖活动标识，set内的元素是用户ID。`SADD lottery 19980802 19901106 ...`
    - 不允许重复中奖，可以用SPOP命令。抽x人就是`SPOP lottery x`
    - 允许重复中：`SRANDMEMBER lottery x`
- **Zset**，有序集合，比Set类型多了一个排序属性score。命令：`ZADD key score member [[score member] ...]`
  - **排行榜**
    - `ZADD cxk 100 hug_me`
    - `ZADD cxk 99 zhiyinnitaimei`
    - `ZADD cxk 95 qingren`
    - `ZADD wyf 96 dawankuanmian`
    - 获取cxk分数最高的前两部作品`ZREVRANGE cxk 0 2 WITHSCORES`
  - **电话、姓名排序**
- **BitMap**，位图，一串连续的二进制数组，可以通过偏移量定位元素。适合数据量的二值统计场景。命令：`SETBIT key offset value`，value只能是0或1。
  - **签到统计**
    - 即使一年的签到也就只需要365个bit位而已。`SETBIT uid:attendance:100:202306 2 1` 表示user_id是100的用户在2023.6.3签到
  - **判断用户登录状态**
    - 只需要用一个很长的BitMap，每一位表示相应offset对应的user_id的用户的登录状态即可。5000万用户的登录状态也仅需要约6MB的空间。
- **Stream**，专为消息队列设计的数据类型
  - **消息队列**
    - 生产者插入 `XADD mq * name cxk` `mq`是消息队列名，`name`是key，`cxk`是value。成功后会返回全局唯一ID。
    - 消息保序：`XADD/XREAD`
    - 阻塞读取：`XREAD BLOCK`
    - 重复消息处理：`XADD`会生成全局唯一ID
    - 消息可靠性：Stream有内部队列`PENDING List`作留存；消费者用`XACK`确认消息处理完成。
    - 支持消费组

# Redis的Zset类型的底层实现是什么？
Zset有两种实现：
- ziplist：满足 1. value-score键值对数量少于128个 2.每个元素的长度小于64字节 时使用这种实现
- skiplist：不满足以上两个条件时，再使用跳表这种实现，它组合使用了hash和skiplist。1. hash用来存储value到score的映射 2.skiplist按照从小到大的顺序存储分数 3. skiplist每个元素的值都是value-score对。
