---
title: namedtuple第一个参数有什么用？
date: 2018-12-19 19:36:33
tags:
---

&nbsp;&nbsp;&nbsp;&nbsp;namedtuple是python中一个简单实用的数据结构，但是很不爽的是每次使用的时候都需要给第一个参数指定一个字符串，而这个字符串在实际使用中似乎并没有什么作用。比如

```
>>> from collections import namedtuple
>>> Student = namedtuple('StudentTuple', ['sex', 'name', 'age'])
>>> s = Student('male', 'callmedad', 22)
>>> s.age
22
```
&nbsp;&nbsp;&nbsp;&nbsp;我操作的对象都是Student，而不是StudentTuple，那我为什么还要传一个'StudentTuple'，这个字符串似乎对我没有任何帮助。stackoverflow同样有人提了这个问题：[传送门1](https://stackoverflow.com/questions/30526115/whats-the-first-argument-of-namedtuple-used-for)，[传送门2](https://stackoverflow.com/questions/30535678/why-do-you-have-to-provide-the-typename-as-the-first-parameter-when-creating-a-n). 看了下回答，然后又看了下namedtuple的源码，总算明白了。
&nbsp;&nbsp;&nbsp;&nbsp;接着上面的代码，我们可以看下Student是个啥。  

```
>>> import inspect
>>> inspect.isclass(Student)
True
>>> Student
<class '__main__.StudentTuple'>
>>> Student
<class '__main__.StudentTuple'>
>>> s.__class__
<class '__main__.StudentTuple'>
>>> type(namedtuple)
<class 'function'>
```
&nbsp;&nbsp;&nbsp;&nbsp;我们大致知道这几点：1. Student是个类，并且只是一个类的引用，查看s.\_\_class\_\_知道，这个类定义的名字是StudentTuple，就是我们传的第一个参数。2. namedtuple是个方法，但是返回的是一个新的类，也就是说这是一个类的工厂方法。  
&nbsp;&nbsp;&nbsp;&nbsp;既然是一个生成类的工厂方法，那么肯定就需要知道类的名字，所以这就是第一个参数的意义。然后，这一切是怎么实现的呢？  
&nbsp;&nbsp;&nbsp;&nbsp;看下namedtuple的源码，其实非常简单粗暴

```
def namedtuple(typename, field_names, *, verbose=False, rename=False, module=None):
	...
	
	class_definition = _class_template.format(
        typename = typename,  # 第一个参数
        field_names = tuple(field_names),  # 各种检验后的属性列表
        num_fields = len(field_names), 
        arg_list = repr(tuple(field_names)).replace("'", "")[1:-1],
        repr_fmt = ', '.join(_repr_template.format(name=name)
                             for name in field_names),
        field_defs = '\n'.join(_field_template.format(index=index, name=name)
                               for index, name in enumerate(field_names))
    )
   namespace = dict(__name__='namedtuple_%s' % typename)
   exec(class_definition, namespace)
   result = namespace[typename]
   result._source = class_definition
   ...
```
&nbsp;&nbsp;&nbsp;&nbsp;核心代码就这几行，主要思路就是填充一个字符串的类的模版，然后动态执行生成一个类。 \_class\_template大致长这个样(简化后)：

```
_class_template = """\
from builtins import property as _property, tuple as _tuple
from operator import itemgetter as _itemgetter
from collections import OrderedDict

class {typename}(tuple):
    '{typename}({arg_list})'

    __slots__ = ()

    _fields = {field_names!r}

    def __new__(_cls, {arg_list}):
        'Create new instance of {typename}({arg_list})'
        return _tuple.__new__(_cls, ({arg_list}))

       def __repr__(self):
        'Return a nicely formatted representation string'
        return self.__class__.__name__ + '({repr_fmt})' % self

    ....
   
{field_defs}
"""
```

&nbsp;&nbsp;&nbsp;&nbsp;有人说这种方式很"dirty"，因为是用eval来动态执行的，但是这种方法确实是很简单高效，除了这种方式，其实也可以通过type和元类来动态生成一个类，有空再补上


