基于Redis的分布式锁
---
应用场景：比如扣库存的时候，多个买家同时下单，假如库存为100，扣库存的时候，先查询库存余额，如果库存大于当前要扣除的数量
则扣除成功，库存还有最后10件的时候，有两个买家都下单了9件的话，两个买家同时获取库存的余额，都是得到10，分别执行减库存操作
都下单成功，最后库存却显示 -8件，这是怎么回事？因为查询库存和扣库存的代码可以多个线程同时执行了，所以才会出现这种库存为负数
的情况。那有什么办法解决这个问题呢？如果是单应用部署，直接通过synchronized关键字修改方法，就能解决，但是如果是分布式的部署
该方法就不能解决这个问题啦，此时就引出了一个分布式锁的概念。

分布式锁的实现方式：
- 基于数据库乐观锁来实现
- 基于Redis来实现
- 基于ZooKeeper来实现

我们这使用的是Redis来实现的。

思路：Redis中有个函数`setnx`,通过redis.setnx(key,value)调用该函数的时候，如果该key已经存在redis中，
则该命令会执行失败，返回结果0，如果不存在，则会将该key=value保存到redis中，并返回1，我们正好通过这一特性来
实现分布式锁，线程在执行某段代之前去获取锁，如果获取失败表示，该代码已经有别的线程正在执行，当拿到锁的线程在执行完
代码之后，判断当前锁是否属于自己，如果是则删掉redis中的值，把锁让出来，让其他线程能够拿到锁。


#### 加锁
`spring-boot-data-redis`已经帮我们封装好了`setnx` 方法，还能设置过期时间，这就帮我们解决了锁超时的问题，超时自动删除
数据，释放锁。
```java
 boolean flag = stringRedisTemplate.opsForValue().setIfAbsent(key,value, TIME_OUT, TimeUnit.SECONDS);
```
如果获取锁成功则返回true，如果获取失败则返回false。

#### 释放锁

原本释放锁只需要将该key从redis中删除掉就行了。可能你在执行删除命令之前，该key已经自动到期了，并且别的线程已经拿到了
该锁，如果此时去删除该key，那么又会有新的线程拿到该锁，就会造成多线程共享一把锁的情况。为了避免这个情况我们必须判断当前
锁是否属于该线程。



释放锁前提：只有自己的锁才能释放，不属于自己的锁是不能释放的。

那如何判断该锁是否属于当前线程呢？

我们在保存key的时候保存了value,我们只要查询该key的值是不是当前线程设置的值就行，如果两值相等，表示该锁属于当前线程
，可以删除。这里涉及到两个命令，一个查询操作和删除操作，为了避免删除别人的锁，我们必须保证该两条命令同时执行，中间不能穿插
任何命令。正好redis所支持的`lua`脚本符合原子性，通过lua脚本确保这两条命令之间没有别的命令。

```
if redis.call('get', KEYS[1]) == KEYS[2] then return redis.call('del', KEYS[1]) else return 0 end
```
这才是一个正确的释放锁的过程。

源码地址：https://github.com/niezhiliang/distributed-redis-lock