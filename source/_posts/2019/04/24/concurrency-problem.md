---
title: 记一次并发问题的排查
date: 2019-04-24 21:31:41
tags: python
---

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
&nbsp;&nbsp;&nbsp;&nbsp; 怎么解决？并发啊，简单，加锁就行！也没多想，迷迷糊糊的在判断上加个锁：
```python

def create_binding(sku_code, settlement_sn)：
    if not Binding.objects.filter(sku_code=sku_code, settlement_sn=settlement_sn, enable=True).select_for_update).exist():
        raise OperationError('binding_created')
    .....

```
&nbsp;&nbsp;&nbsp;&nbsp; 查询上加上select_for_update()，阻塞住并发的读，完美！发到测试环境，一看，并不行...还是会生成重，再回头看才明白这个锁有问题。首先，django的`select_for_update`加上exist()其实是不会生成'select ... for update'语句的，然后就算生成了，但是因为没有这样的数据，所以没有row可以被锁，这个锁其实没有任何作用。    
&nbsp;&nbsp;&nbsp;&nbsp; 那怎么解决呢，分布式锁肯定是可以的，给判断的代码块加上锁，这样创建binding的请求就变成串行的了，如果对并发没要求，比如我们这种场景为了防止重点，可以使用这种办法。