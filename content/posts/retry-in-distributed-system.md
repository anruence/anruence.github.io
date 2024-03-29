---
title: 分布式系统中一种无限重试的机制
date: 2020-03-13 17:50:05
tags: 
- mq
categories:
- mq
---

# 前言：为什么需要重试

## 重试的场景

什么场景需要重试？就是我们认为的暂时的失败，比如说，系统繁忙，网络失败，请求被限流，插入数据时版本失败或者是被调用端返回的可重试的错误码，这个时候可以进行重试。而有些失败是不需要重试的，例如用户权限错误、内容被反垃圾系统拦截、非法数据等等，这些失败，一般是重试多少次都是不可能成功的。

## 重试的策略

重试需要注意哪些事情呢？是否只需要简单的重复调用几次就行？最近我就遇到这样一个场景。有一个业务，并发量非常的大，特别在每天早上8，9点的时候，很多用户都会使用这个功能，这个公司会盘路去写一些数据，有一次底层的数据库系统压力非常大，所以有些请求就会写失败，然后立马重试，最后造成雪崩！这是重试设计中一个比较大的误区，如果是因为资源不够，那么立马不停的重试非常容易压垮系统。所以，在重试设计中，我们有限地进行重试，重试不应该是无限次数的，其次，便是要间歇地进行重试，即每次失败都需要休息一会，并且每次休息的时间逐渐加长。例如有个系统开始限流保护到熔断了，那么连续重试多少次也没用，还不如休息一段时间，这种做法在TCP的拥塞控制中也被应用到。

## 重试设计的要点

我觉得写代码，只要每次远程调用的时候，多思考以下几个点即可：
- 是否需要重试，哪些错误需要重试，哪些错误可以直接返回。这个也跟业务相关，有时候虽然可以后台重试，但是还是会返回失败让用户在操作一次，这样可以保证代码的简洁。
- 重试的次数与频率，哪些失败可以立马重试的（更新数据库版本失败），哪些失败需要异步去重试。
- 重试的时候，任务是否是幂等的。这是非常关键的一点，比如说退款任务，有可能只是网络繁忙，实际上结算系统已经把钱退还给用户。这也是我们设计接口中重要的一点，尽量去保证消息的幂等性。


# 重试的实现

## 实时重试

- spring-retry
自定义超时异常，重试策略，50ms重试，依次递增
```java
    @Retryable(value = {RetryException.class}, backoff = @Backoff(delay = 50, multiplier = 1))
```
spring相关组件完整提供了这种能力，资料很多，就不赘述了

## 异步重试

异步重试的实现方法也多种多样，这里简单介绍一种思路，基于队列的一种重试机制

> 1. 开始流程处理，当捕获到可重试异常时将参数信息发送到一个队列
> 2. 队列消费者异步消费消息进行重试补偿，当重试失败次数超过某个阈值时，将消息转移到队列对应的死信队列中
> 3. 死信队列在消费逻辑中同时可以增加一些额外的操作，限流，白名单功能