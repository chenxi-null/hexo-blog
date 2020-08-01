title: 大搜车异步任务队列中间件的建设实践
author: chenxi
tags:
  - distributed
categories: 
  - distributed
date: 2019-08-20 20:40:00
---

本文源自于一次部门内部的技术分享，首发于部门的[微信公众号](https://mp.weixin.qq.com/s/DvuRdY74C0xO_fLVF7lONg)，后被 [InfoQ](https://www.infoq.cn/article/uMQb2CFDgRFcDUz9OFD1) 转载。

---

## 前言
异步任务处理在系统设计中是一个十分常见的场景, 例如将耗时任务异步化, 流量削峰, 接口的延时回调等等。

对于异步任务, 最最常见的一种实现方式就是 JDK 的线程池了, 使用线程池可以满足大多数异步化的场景, 但它还是有一些局限性:

- 一方面它是基于内存的, 一旦机器宕机异步任务就丢失了, 可靠性不足;

- 另一方面它是单机处理任务, 不适合分布式场景, 当单机处理任务的能力不足时，无法水平拓展

而对于分布式异步任务的处理需求, 我们通常使用 MQ 来实现分布式异步处理: 生产者将异步任务以消息的形式丢入消息队列, 消费者异步地从消费队列里获取消息进行任务处理。

![](https://static001.infoq.cn/resource/image/a5/c8/a56c48176d25c1340c465c54c527bac8.png)


这也是大多数 Java 程序员的选择, 但是使用 MQ 实现异步队列也存在一些不足之处:

- 基于 MQ 开发异步任务还是有些复杂的, 开发人员需要写不少业务逻辑之外的代码, 理想的状态是, 开发人员可以专注于任务处理本身的业务逻辑, 将维护异步化操作这类的系统控制交给中间件来完成；

- 功能局限性：比如基于 MQ 无法实现任意大小的延时时间、自定义重试策略、复杂的任务调度场景——如依赖任务等；

- 缺少一个可视化的页面查看任务执行情况, 包括有哪些任务正在执行, 正在哪台机器上执行, 已成功 / 失败的任务等等

放眼其他语言的异步任务队列, 我们知道 Python 有 `Celery`, Ruby 有 `Sidekiq`, 使用起来非常轻量级，只需要额外做一些简单的配置, 就能像调用本地方法一样调用异步任务的执行逻辑。

所以, 为了解决基于 MQ 开发异步任务过于复杂的问题, 让 Java 程序员也能使用轻量级的异步任务队列, Optimus-AsyncTask 便应运而生了。

Optimus-AsyncTask 是我们公司的中间件团队研发的一款分布式异步任务中间件, 追求轻量级和高性能, 特点是配置简单, 功能丰富, 编程模型简单, 可以像调用本地方法一样调用异步任务, 让开发同学可以专注于业务逻辑的开发, 提高开发效率。

在介绍 Optimus-AsyncTask 之前, 我们先探讨一下分布式任务队列需要具备的功能点, 以及实现一个分布式任务队列需要注意的一些地方:

## 如何设计一个分布式异步任务队列
- 支持高并发
能够支持至少万级的 QPS

- 任务持久化
常见的数据存储中间件, 如数据库, Redis, 或者 MQ

- 任务状态流转
在一个任务的生命周期中可能存在多个状态, 这些状态包括: 准备执行, 正在执行, 执行结束 (成功), 执行失败之后进入待重试状态, 执行失败后进入死亡状态等等.
在状态流转的过程中, 需要保证数据的一致性

- 可靠性保障
保证任务不丢失, 任务不被重复执行

- 支持水平拓展
可以动态地添加执行器

- 合理的负载策略
可以合理地分配执行资源, 保证各个执行器对任务进行有效的负载

- 支持重试机制
用户可以配置重试次数, 在任务执行失败或超时的时候进行自动重试

- 幂等性支持
保证幂等性是分布式场景中的一个重要课题, 可以在异步任务中间件里提供幂等性支持:
用户提交任务时提供一个业务 ID, 业务 ID 相同的任务只会执行一次

- 友好的 API
对用户屏蔽异步任务的底层实现细节, 像调用本地方法一样调用异步方法

- 提供控制台查看任务状态

## 核心实现
在抛出以上几个问题之后, 下面我们开始介绍 Optimus-AsyncTask 的具体实现:

### 任务生命周期
任务执行超时 / 失败之后会进入“重试”队列等待下一次执行, 超时 / 失败次数超过用户设置的最大重试次数之后任务就会变成“死亡”状态, 结束生命周期

![alt](https://static001.infoq.cn/resource/image/e0/5d/e00b6715e2cf58cf1b4df48601b76e5d.png)

### 架构图
![](https://static001.infoq.cn/resource/image/a2/4d/a2cea599dc1b6eb8eb6fde7a82cead4d.png)

一个集群 (每个应用实例的代码都是相同的) 配置一个 Redis, 一个机器提交任务, 任意一台机器执行。

#### 任务提交:
SDK 向 Redis 提交任务, 普通异步任务的初始状态为 ready, 延时任务为 delayed。

#### 任务状态变更:
SDK 不断轮询 Redis, 修改任务的状态:

delayed -> ready (延时任务到达执行时间)

retry -> ready (重试任务到达执行时间)

ready -> working (开始执行任务)

working -> retry/dead (任务执行超时进入重试或者死亡状态)

#### 任务执行:
SDK 将任务的状态从 ready 修改为 working 后, 从 Redis 中获取该任务信息, 在本地执行任务。

#### 执行结果上报:
任务执行结束后, SDK 向 Redis 上报执行结果, 修改任务的状态:

working -> success (执行成功)

working -> retry/dead (执行失败)

SDK 中专门开辟了一个线程池用来执行异步任务, 执行线程池的线程数量是开放给用户配置的, SDK 只有在本地存在空闲线程时才去获取任务, 由此可以保证对执行资源的有效分配。


### 任务存储
首先, 为了保障高性能, 我们选择 Redis 作为任务存储层。

#### Redis vs MySQL

我们观察到市场上的一些开源 Java 异步任务中间件 (如 jmppok/AsyncTask, bojiw/asyncmd) 都是基于 MySQL 的, 选择 MySQL 的好处是, 基于关系型数据库我们可以很方便地实现任务查询和任务状态流转:

```SQL
SELECT * FROM task WHERE status = 'ready';
UPDATE task SET status = 'working' WHERE id = 1;
```
像这样通过简单的 SQL 就可以实现, 开发成本较低。

但如果使用 Redis 的话, 因为 Redis 的数据结构较为简单, 不适合 MySQL 这种条件查询和数据嵌套 (为了让一个 Task 元素包含多个属性字段, 在 Redis 里只能用 JSON 字符串来表示), 为了实现相同的语义, 需要设计更复杂的数据结构 (下文会具体介绍), 开发成本更高。

但即便如此, 我们依然选择 Redis 而不是 MySQL, 主要还是考虑到 MySQL 对高并发的支持不是很好, 在我们公司的 C 端场景下异步任务的提交频率会很高, 所以高并发场景是我们必须要考虑的。

此外, 选择 MySQL 的话, 还需要创建额外的表, 对业务数据库有一定的侵入性。

#### Redis vs MQ

我们没有使用 MQ 作为任务存储层是因为 MQ 的运维流程比较复杂, 需要事先申请 Topic, 消费组等信息, 没有 Redis 那么轻量。

并且，基于 MQ 实现的任务队列具有一定的功能局限性，例如无法实现任意大小的延时时间、自定义重试策略、复杂的任务调度场景——如依赖任务等。


### Redis 数据结构设计
一种任务状态对应一个 Redis 集合, 任务数据以 JSON 字符串的形式存储。
```java
[zset] delayed (延时任务), retry (待重试任务). zset 的 score 是预期执行时间, value 是任务数据
	
[hash] working (正在执行中的任务). hash 的 field 代表任务 ID, value 表示任务数据 (json 字符串的形式))

[list] ready (就绪的任务)
[list] success:{date} (处理完成的任务，每天生成一个 list, value 是任务信息，默认过期时间 14 天)
[list] dead:{date} (死亡任务: value 是任务信息，默认过期时间 14 天)
```

### 原子性保障
每次切换任务状态, 至少需要三个步骤:

1. 从原来的集合里删除元素 ;

2. 对于取出的任务数据, 修改 JSON 的部分 value, 也就是任务的属性, 如 ‘executionTime’, ‘finishTime’ 等 ;

3. 将元素加入到新的集合里。

为了保证操作的原子性, 需要使用 lua 脚本来变更任务状态, 例如从 ready 切换到 working 状态, 对应的 lua 伪代码如下:
```lua  
-- ready(list) to working(hash)
local src = KEYS[1]
local dest = KEYS[2]
local currTime = ARGV[1]
local clientId = ARGV[2]

-- get json and delete from 'ready'
local srcJson = redis.call('LPOP', src)
if (srcJson == false) then return end

-- update 'executionTime' and 'clientId' fields of json
local destTask = {}
local srcTask = cjson.decode(srcJson)
for k, v in pairs(srcTask) do
    destTask[k] = v
end
destTask['executionTime'] = currTime
destTask['clientId'] = clientId
local destJson = cjson.encode(destTask)

-- add into 'working'
redis.call('HSET', dest, destTask['taskId'], destJson)

return destJson
```

借助 Redis 的原子性特性, 对于一个任务, 可以保证只有一台机器能够成功获取到该任务, 从而保证该任务不会在多台机器上重复执行。

---

### 高可用保障
#### 轮询时间间隔
为了防止机器数量增加导致访问 Redis 过于频繁, 所以根据机器的数量来动态调整轮询的间隔时间, 并且加入随机因子让每次间隔时间都是随机变化的，保证各台机器可以错开访问，防止类似雪崩之类的系统故障。

设 avgInterval 为 15 秒, random 是随机变量, 取值范围为 [0, 1], size 是客户端机器数
每台机器实际的轮询时间间隔为：avgInterval * (1/2 + random) * size

在概率统计的意义上, 对整个机器集群来讲，整体的平均轮询间隔时间为 15 秒。

#### 任务重试的等待时间
任务失败后需要等一段时间再重试执行, 这里借鉴了 Sidekiq 的实现方式, 等待时间的计算方式如下:

```java
// retryCount 是重试次数, random 是随机变量, 取值范围为 [0, 1]
retryCount ^ 4 + 15 + random * 30 * (retryCount + 1)
```
等待时间以指数级的形式增长, 之所以加入了随机变量是为了防止这样一种情况:
多个异步任务的执行时间相隔很近, 访问某一个资源, 由于并发访问的原因, 资源提供方的负载过高, 导致这些异步任务都执行失败, 如果任务重试的等待时间里没有加入随机因子的话, 这几个任务又会在相近的时间内一起执行, 并发访问资源, 由于同样的原因导致任务执行失败, 如此反复。
加入随机因子可以错开这些任务的执行时间, 避免这种情况的出现。

#### 状态上报失败后的重试补偿
任务执行结束后需要上报成功 / 重试 / 死亡的状态, 但由于网络原因可能导致访问 Redis 失败, 当出现上报失败时, SDK 会将此次上报放到一个内存队列里, 通过后台的定时任务, 不断地重试上报过程。

---

### 编程模型

#### 开发一个异步任务:
```java
@OpAsyncTask(timeout = 600, maxRetry = 3)
public class DemoAsyncTaskHandler implements BaseAsyncTaskHandler<CustomPOJO> {

    // 可以在 TaskHandler 里注入其他 bean
    @Autowired
    private DemoDao demoDao;

    @Override
    public void performAsync(CustomPOJO params) {
        // 任务的执行逻辑
    }

    /**
     * 业务方需要实现这个回调方法，根据自己传给 perform 方法的 params 参数，返回一个 bizId,
     * 以实现任务去重: 对于一个 taskHandler, 多次提交 bizId 相同的任务, 框架只会执行一次。
     *
     * @param params 调用方传给 perform 方法的参数
     */
    @Override
    public String getBizId(CustomPOJO params) {
        // 基于 params 参数生成 bizId
    }
}
```

#### 提交异步任务:
就像调用一个普通的本地方法 (基于 Spring 动态代理实现)
```java
@Service
public class DemoService {

    // TaskHandler 就是一个 Spring Bean, 你可以像注入其他 Bean 一样注入 TaskHandler
    @Autowired
    private DemoAsyncTaskHandler asyncTaskHandler;

    /**
     * 提交普通异步任务
     */
    public void invokeAsync() {
        CustomPOJO param = new CustomPOJO("a", "b");
        asyncTaskHandler.performAsync(param);
    }
}
```

### Web 控制台
查看任务的执行状态

![](https://static001.infoq.cn/resource/image/48/e3/4822519f57b4d0aae34fd501d5fd29e3.jpg)


---

## 进阶功能

### 任务依赖
除了上文提到的普通异步任务和延时任务, 我们还提供了一种任务依赖的功能, 使用场景如下:

例如 A 任务和 B 任务都是外部系统触发的, 但是 B 任务需要在 A 任务执行结束后才能执行成功, A, B 触发时机的先后次序是不一定的, B 的触发时机可能会先于 A 的, 这个时候 B 任务没法立即执行, 需要等待 A 的完成。

这些任务执行次序的逻辑控制由业务方实现的话会比较麻烦, 可以交给 OpAsyncTask 来完成:
把 A 和 B 声明为依赖任务, 且 B 依赖于 A, 在被外部系统触发时, 分别提交 A 任务和 B 任务, A 和 B 的执行次序交给 OpAsyncTask 来协调, 业务方只要专注于任务本身的处理逻辑就可以了。

### 事务支持
当在 Spring 的事务内调用 TaskHandler 的 perform* 方法 (既提交任务) 时,
如果执行出现异常导致事务回滚操作, 那么异步任务提交也会回滚, 既任务不会被提交 ;
如果执行正常, 那么事务正常提交后, 异步任务就会被正常提交。

**实现原理:** 在 TaskHandler 的 perform* 方法被调用时, OpAsyncTask 会判断当前线程是否处于 Spring 事务中, 如果不是的话就直接提交异步任务, 如果是的话会“延迟提交异步任务” —— 事务成功提交之后再去提交异步任务。
```java
// delay the submitting if it's in a transaction
if (TransactionSynchronizationManager.isActualTransactionActive()) {
    log.info("It's in a active transaction, delay the task submitting");
    TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronizationAdapter() {
        @Override
        public void afterCommit() {
            log.info("Submitting async task after committing transaction");
            submitAsyncTask();
        }
    });
} else {
    log.info("It's not in a active transaction, submit task directly");
    submitAsyncTask();
}
```

**注意事项:** 本质上把异步任务的提交行为推迟到事务成功提交之后执行, 所以异步任务的提交这一行为和事务中的行为 (如 DB 操作) 不是原子性的 (要么都成功执行, 要么都不执行), 最终提交异步任务时有可能因为网络原因导致异步任务提交失败, 这时候事务已经提交了, 其他事务行为已经生效了, 例如 DB 操作都已生效, 可异步任务没有成功提交, 所以无法严格保证其同时成功执行:
事务行为不成功的话, 异步任务提交一定不成功 ; 但事务行为成功的话, 有一定概率出现异步任务提交不成功的情况。