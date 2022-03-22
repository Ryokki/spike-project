## 场景

• 具体场景有哪些？• 实际需求有什么？• 详细流程怎么样？![image-20220319134528055](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20220319134528055.png)

### 秒杀流程

![image-20220319134636663](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20220319134636663.png)

### 解决的问题

![image-20220319135309295](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20220319135309295.png)

### 需求拆解

![image-20220319135424263](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20220319135424263.png)

## 服务

单体架构 or 微服务？

### 单体架构

![image-20220319135642735](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20220319135642735.png)

单体架构就是，所有的模块暴露在同个服务里面

* 不能单独对一个性能瓶颈的模块进行扩展(如.加服务器)
* 数据库是给所有模块一起用的，崩溃时所有服务崩溃

### 微服务架构

![image-20220319135650609](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20220319135650609.png)

微服务架构就是解决单体架构的各个问题，主要是为了解耦



## 存储

数据如何存储和访问

### 数据库设计

![image-20220319140153269](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20220319140153269.png)

1. `commodity_info`表:商品信息
2. `seckill_info`表:秒杀活动表,记录了一个秒杀活动
3. `stock_info`表:库存信息表
4. `order_info`表:订单信息表

![image-20220319140513000](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20220319140513000.png)

做一个秒杀活动：

1. 商家需要在秒杀活动表中插入数据，表示需要进行怎样的秒杀。
2. 用户需要查询秒杀活动表来知道秒杀活动信息，秒杀的时候还会去查库存信息，插入一个订单信息，如果购买的话再更新库存信息

### 秒杀操作

![image-20220319140857530](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20220319140857530.png)

这是最简单的方法。就是去查询库存余量，如果有的话去扣减库存。

问题：并发时会超卖，原因是当剩下最后几个库存的时候，很多个并发查询都发现有库存，都去扣减，扣减的时候可能就没有库存了，导致超卖

解决方法：利用数据库提供的事务

![image-20220319141310921](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20220319141310921.png)

这里我们查询库存余量时通过加上`FOR UPDATE`来加上行锁，这样别人不能修改这个数据了。然后我们去扣减库存

```sql
update stock_info set stock = stock - 1 where commodity_id = 180 and seckill_id FOR UPDATE
```

这种方法会很耗时，一般不会用这种方法



更好的方法：使用update自带的行锁

![image-20220319141755668](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20220319141755668.png)

在update中去检查stock>0，用行锁保证正确性，这样就解决了超卖

其他问题:高QPS下会让数据库崩溃

解决办法：库存预热

![image-20220319141958932](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20220319141958932.png)

mysql单点支撑1000qps，redis能10w，所以我们将活动的秒杀和库存信息放到redis，来减轻mysql的压力

![image-20220319142155439](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20220319142155439.png)

![image-20220319142324635](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20220319142324635.png)

一般在活动开始前将数据库的数据存到redis中，来进行预热

![image-20220319144109539](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20220319144109539.png)

通过redis来限流，当库存为0时就直接在redis层就拦住了，不为0时在redis中让库存-1，并去数据库中存。

![image-20220319142749964](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20220319142749964.png)



