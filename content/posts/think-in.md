---
title: 一些想法的落地
date: 2019-02-26 16:03:59
tags:
- think
categories:
- think
---

## 为什么分库分表的数量都采用2^n之类的数字

扩容时只需要迁移一半的数据，计算路由规则时也较为方便。
数据在裂变时一般是 1 -> 2 -> 4 -> 8
2 的 n 次方直接使用位运算操作 如果使用十进制的取模的话，必须先从内存里面读到二进制数的这两个数，再转成十进制数，再运算，性能比较差。
更容易做一些表的迁移，数据只需要动一半，比如  id  =  1，对  8  取余得  1  ，4  取余也是  1，  2  取余也是  1。这样我们不需要迁移行记录，只要迁移表记录，这样迁移成本会小得多。

## 迁移服务的基本思路

消息队列迁移时，可以考虑在消息体重添加新字段。消费者兼容逻辑先上线。
原有字段尽量不改变语义，上线顺序要求兼容逻辑先上线。

## 使用CountDownLatch实现任务线程执行排序

A B C D E，5个任务，ABC同时执行 -> DE同时执行，要求DE，在ABC都执行完成后才执行。
思路：D E持有一个数量为3的CountDownLatch实例，当A B C 任务执行完成时对D E 持有的CountDownLatch实例进行减1。

## 高并发大量级下的秒级超时检测机制

常规思路:启动一个每秒定时任务，定时获取队列数据，一个个扫描过期时间。但是，会导致扫描的对象 过多，浪费 CPU 资源以及增加锁竞争开销。
优化思路:采用分桶策略，将某一秒内的数据按顺序添加到桶中，然后将桶以过期时间作为 key 添加到 map 中。扫描线程中首先获取跟下一次过期时间的时间间隔，若间隔 > 0，则说明还未过期，线程休眠得到的时间间隔，然后再取出过期的桶，对桶中的数据进行一一处理;因为未过期的早已经从桶中移除了， 所以桶中剩下的数据肯定就是已经过期。这个思路，事实上参考了 ZooKeeper 的会话分桶策略，在一些需要管理大量超时会话的场景下，会非常有用。因此，超时检测线程只要在这些指定的时间点上进行检査即可，这样既提髙了会话检査的效率。我们将一个时间段的数据放入一个桶中，然后定时任务扫描这些任务桶，那么过期的数据肯定是同一个桶里面的，就不需要扫描其他任务桶了。

## 业务上实现幂等的方式

幂等机制的核⼼是保证资源唯⼀性，例如客户端重复提交或服务端的多次重试只会产⽣⼀份结果。⽀付场景、退款场景，涉及⾦钱的交易不能出现多次扣款等问题。事实上，查询接⼝⽤于获取资源，因为它只是查询数据⽽不会影响到资源的变化，因此不管调⽤多少次接⼝，资源都不会改变，所以是它是幂等的。⽽新增接⼝是⾮幂等的，因为调⽤接⼝多次，它都将会产⽣资源的变化。因此，我们需要在出现重复提交时进⾏幂等处理。那么，如何保证幂等机制呢？事实上，我们有很多实现⽅案。

## 数据拆分原则

如果在“尽量减少事务边界”与“数据尽可能平均拆分”两个原则间发生了冲突，那么请选择“数据尽可能平 均拆分”作为优先考虑原则，因为事务边界的问题相对来说更好解决，无论是全表扫描或做异构索引复制都 是可以解决的。如果为每一个存在跨 join 或全表扫描的场景都采用数据异构索引的方式，整个数据库出现 大量数据冗余，数据一致性的保障也会带来挑战，同时数据库间的业务逻辑关系也变得非常复杂，因此以 82 法则，在实际中，我们仅针对那些在 80% 情况下访问的那 20% 的场景进行如数据异构索引这样的处 理，达到这类场景的性能最优化，而对其他 80% 偶尔出现跨库 join、全表扫描的场景，采用最为简单直 接的方式往往是最有效的方式。

### ⽅案一：创建唯⼀索引

在数据库中针对我们需要约束的资源字段创建唯⼀索引，可以防⽌插⼊重复的数据。但是，遇到分库分表的情况是，唯⼀索引也就不那么好使了，此时，我们可以先查询⼀次数据库，然后判断是否约束的资源字段存在重复，没有的重复时再进⾏插⼊操作。注意的是，为了避免并发场景，我们可以通过锁机制，例如悲观锁与乐观锁保证数据的唯⼀性。这⾥，分布式锁是⼀种经常使⽤的⽅案，它通常情况下是⼀种悲观锁的实现。但是，很多⼈经常把悲观锁、乐观锁、分布式锁当作幂等机制的解决⽅案，这个是不正确的。除此之外，我们还可以引⼊状态机，通过状态机进⾏状态的约束以及状态跳转，确保同⼀个业务的流程化执⾏，从⽽实现数据幂等。事实上，并不是所有的接⼝都要保证幂等，换句话说，是否需要幂等机制可以通过考量需不需要确保资源唯⼀性，例如⾏为⽇志可以不考虑幂等性。

### 方案二：版本号

接⼝不考虑幂等机制，在业务实现的时候通过业务层来保证，例如允许存在多份数据，但是在业务处理的时候获取最新的版本进⾏处理。

## 引入消息中间件有什么缺点？

- 复杂系统的解耦
- 复杂链路的异步调用
- 瞬时高峰的削峰处理

### 系统可用性降低

### 系统稳定性降低

### 分布式一致性问题

## “并发性，时间和相对性”

“如果两个操作“同时”发生，似乎应该称为并发——但事实上，它们在字面时间上重叠与否并不重要。由于分布式系统中的时钟问题，现实中是很难判断两个事件是否同时发生的，这个问题我们将在第8章中详细讨论。

为了定义并发性，确切的时间并不重要：如果两个操作都意识不到对方的存在，就称这两个操作并发，而不管它们发生的物理时间。人们有时把这个原理和狭义相对论的物理学联系起来【54】，它引入了信息不能比光速更快的思想。因此，如果事件之间的时间短于光通过它们之间的距离，那么发生一定距离的两个事件不可能相互影响。

在计算机系统中，即使光速原则上允许一个操作影响另一个操作，但两个操作也可能是并行的。例如，如果网络缓慢或中断，两个操作间可能会出现一段时间间隔，且仍然是并发的，因为网络问题阻止一个操作意识到另一个操作的存在”

## 参考

- https://juejin.im/post/5c1214a66fb9a049bb7c3410
- https://juejin.im/post/5bf2c6b6e51d456693549af4
- https://juejin.im/post/5bf6b40de51d4536656f1f28 分布式锁高并发优化，锁分段