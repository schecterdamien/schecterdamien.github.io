---
title: 记一次并发问题的排查
date: 2019-04-24 21:31:41
tags: ["python"]
menu: "main"
---
### 起源 ###
&nbsp;&nbsp;&nbsp;&nbsp; 事情的起源是这样的，Binding是一张多对多的表，主要有sku_code，settlement_sn，enable这几个字段，但是逻辑上sku_code，settlemen_sn，enable=True的在表中应该是唯一的。因为enable=False可能有多条，所以不能在数据库上加联合唯一的索引，但是代码里面有判断。然后有一天测试发现出现两条记录的sku_code，settlemen_sn相同且enable都为True，然后看代码，创建Binding的代码大致是这样的

```python

def create_binding(sku_code, settlement_sn)：
    if not Binding.objects.filter(sku_code=sku_code, settlement_sn=settlement_sn, enable=True).exist():
        raise OperationError('binding_created')
    # 一系列有些耗时的校验
    do_some_check()
    return Binding.objects.create(settlement_sn=settlement_sn, sku_code=sku_code, enable=True)
```

&nbsp;&nbsp;&nbsp;&nbsp; 先判断这样的Binding是否存在，存在的话就直接抛异常，不存在的话再进行其他的校验，最后生成对应的Binding。一般情况下没问题，但是如果并发请求的sku_code和settlement_sn都相同，接到请求的几个worker同时从数据库查询发现目标binding不存在，做出可以创建的判断（因为下面存在一些耗时的校验让这种情况的概率变得很高），然后就创建了sku_code和settlement_sn都相同的记录。
&nbsp;&nbsp;&nbsp;&nbsp; 因为是公司内部使用的财务规则系统，也就几个人在用，所以做的时候没怎么考虑并发问题。最后发现其实是前端没有做按钮的防重点，用户连续在按钮上点了两下，造成了这个问题。
&nbsp;&nbsp;&nbsp;&nbsp; 我们用的是pg（等下会说为什么强调是pg），这里其实是发生了`幻读`。那改事务隔离级别可以解决吗，`可重复读`和`串行化`不是能解决`幻读`吗。。。。。肯定不行，一言不合就改数据库事务隔离级别肯定是不行的，而且改了也没法解决这种`幻读`问题(后来才知道其实`串行化`可以，但需要加上失败重试的逻辑)。那加锁吧！当时也没多想，迷迷糊糊的在判断上加个锁：
```python

def create_binding(sku_code, settlement_sn)：
    if not Binding.objects.filter(sku_code=sku_code, settlement_sn=settlement_sn, enable=True).select_for_update).exist():
        raise OperationError('binding_created')
    .....

```
&nbsp;&nbsp;&nbsp;&nbsp; 查询上加上select_for_update()，阻塞住并发的读，完美！发到测试环境，一看，并不行...还是会生成重，再回头看才明白这个锁有问题。首先，django的`select_for_update`加上exist()其实是不会生成'select ... for update'语句的，然后就算生成了，但是因为没有这样的数据，所以没有row可以被锁，这个锁其实没有任何作用。  

### 解决 ###  
&nbsp;&nbsp;&nbsp;&nbsp; 那怎么解决呢，分布式锁肯定是可以的，给判断的代码块加上锁，这样创建binding的请求就变成串行的了，如果对并发没要求，比如我们这种场景为了防止重点，可以使用这种办法。代码块就变成这样了：
```python
def create_binding(sku_code, settlement_sn)：
    # 获取分布式锁
    get_lock()

    if not Binding.objects.filter(sku_code=sku_code, settlement_sn=settlement_sn, enable=True).exist():
        raise OperationError('binding_created')
    # 一系列有些耗时的校验
    do_some_check()
    result = Binding.objects.create(settlement_sn=settlement_sn, sku_code=sku_code, enable=True)

    # 释放分布式锁
    release_lock()
    return result
```
&nbsp;&nbsp;&nbsp;&nbsp; 除此之外有没有别的办法呢，其实我们就是想锁住一个“不存在”的一行，google了一下，搜到一篇文章：[https://rosscoded.com/blog/2018/05/02/locking-phantom-postgresql/](https://rosscoded.com/blog/2018/05/02/locking-phantom-postgresql/)，这里面在pg的数据库级别给出了解决办法。首先可以使用`ON CONFLICT`语句，其次还可以使用`advisory locks`，劝告锁，这是pg独有的一种锁，思路挺有意思，不过这是库级别的锁，用多了也不好。但是这些都得在代码里面写sql，最后我们还是选择了基于redis的简单的分布式锁来解决。  
&nbsp;&nbsp;&nbsp;&nbsp; 回到之前说的，为啥强调是pg呢，因为后来试了一下，在mysql里面，其实是可以给不存在的数据加锁的，mysql里面这个叫`gap lock`，间隙锁，专门用来解决`幻读`问题。比如直接`select * from binding where sku_code = "123" for update`其实会同时加`record Lock`和`gap lock`，把已经存在的`sku_code=123`的行给锁住，同时还会锁住不存在的这些目标行，就是这时其他事务其实是没法写`sku_code=123`的数据的。mysql只有在`可重复读`以上的隔离级别才会自动加`gap lock`。  
&nbsp;&nbsp;&nbsp;&nbsp; 之后有空再详细写写上面的几种锁。

### 参考链接 ###
+ [https://rosscoded.com/blog/2018/05/02/locking-phantom-postgresql/](https://rosscoded.com/blog/2018/05/02/locking-phantom-postgresql/)
+ [https://ningyu1.github.io/site/post/50-mysql-gap-lock/](https://ningyu1.github.io/site/post/50-mysql-gap-lock/)
