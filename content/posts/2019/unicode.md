---
title: unicode到底是个什么鬼
date: 2019-01-24 16:01:06
tags: ["python"]
menu: "posts" 
---
&nbsp;&nbsp;&nbsp;&nbsp;编码问题可以说是python中非常让人头疼的一个问题了，看到知乎上一个哥们说：一旦你走上了编程之路，如果你不把编码问题搞清楚，那么它就像幽灵一般纠缠你整个职业生涯，各种灵异事件会接踵而来。只有充分发挥程序员死磕到底的精神你才有可能彻底摆脱编码问题带来的烦恼。     
&nbsp;&nbsp;&nbsp;&nbsp;实在不能更同意了。前几天看到一篇文章写的挺不错<a>https://zhuanlan.zhihu.com/p/40834093</a>。网上也有很多类似的文章，一般都会说到python3的str的编码是unicode。同时在另一本知名的书上《effective python》有这样一段：
  >>>  把Unicode字符表示为二进制数据（也就是原始8位值）有许多的办法。最常见的编码方式就是UTF-8。但是大家要记住，python3的str实例和python2的unicode实例都没有和特定的二进制编码形式相关联。要把Unicode字符转换成二进制数据，就必须使用encode方法。想要把二进制数据转换成Unicode字符，则必须使用decode方法。  

&nbsp;&nbsp;&nbsp;&nbsp;看到这里我就很分裂了，那python3的str到底是什么编码格式呢？如果没有和特定的二进制编码形式相关联，那str在内存中怎么表示呢？unicode到底是个什么鬼？

### Unicode
&nbsp;&nbsp;&nbsp;&nbsp;说到这里，就得去研究下unicode。  
&nbsp;&nbsp;&nbsp;&nbsp;刚开始没有的unicode的时候，英语国家使用ascii，中国使用gbk，日本使用shift_jis……大家都使用自己的编码协议，大家交流起来就特别麻烦，一不小心就乱码了。假设没有一门可以包容所有语言的编码，同时有n个适用于各国自己语言的编码，那大家需要交流的话就要n*(n-1)种映射协议来转换编码，让大家都能交流，这无疑对互联网来说是场灾难。所以unicode出现了，它对世界上大部分的文字系统进行了整理、编码，使得计算机可以用更为简单的方式来呈现和处理文字。  
&nbsp;&nbsp;&nbsp;&nbsp;大概来说，Unicode编码系统可分为编码方式和实现方式两个层次。重点来了，Unicode的编码方式和实现方式是不同的，编码方式是指的字符集，也就是指定了每个值和字符的映射关系，比如我可以自己制定一个很简单的字符集类似于:```{1: '你', 2: '好'，……，260: '呢'}```，这里指定的是值和字母的映射关系，1代表的就是'你'，2代表的是'好'，260代表的是'呢'。根据编码方式我们可以有不同的实现方式。比如如果是用定长编码来实现，要存储260这个中文字符，我们至少需要两个字节，所以我们可以写死用两个字节来存储，尽管对于前面256个字符来说，第一个字节都是0。如果使用变长编码来实现，可以把前面128个汉字用一个字节表示，留出首位当作标识位，其他的汉字用两个字节表示。还有比如两个字节是大端序还是小端序会影响到第一个字节是高位还是第二个字节是高位等等，这些都是不同的实现方式。  
&nbsp;&nbsp;&nbsp;&nbsp;所以Unicode我们通常说的是指的其编码方式，就是映射关系。Unicode的实现方式称为Unicode转换格式（Unicode Transformation Format，简称为UTF）。我们用的比较多的有UTF-8，UTF-16，UTF-32等。所以我们说python3中的str的编码方式是unicode这个说法就有问题了，unicode只是一个字符集，没有约定实现，具体的实现还要看是采用UTF-8还是UTF-16还是别的方式，那到底python中采取的是unicode的哪种实现方式呢？


### 真相大白
&nbsp;&nbsp;&nbsp;&nbsp;然而这个问题，在中文网络上几乎没找到相关的答案，看了好多的博客，都只是说python3的str编码是unicode，这句话本身就有问题。最后还是在stackoverflow上找到了答案: [What is internal representation of string in Python 3.x](https://stackoverflow.com/questions/1838170/what-is-internal-representation-of-string-in-python-3-x) 以及 [How is unicode represented internally in Python?](https://stackoverflow.com/questions/26079392/how-is-unicode-represented-internally-in-python)。  
&nbsp;&nbsp;&nbsp;&nbsp;总结一下，就是python在3.3之前，包括python2，具体的内部unicode采用哪种方式实现其实是编译时的选项。取决于系统的本机字节顺序以及是否选择了UCS2或UCS4。UCS4的话就是采用UTF-32来实现，UCS2的话就采用UTF-16来实现，本机字节顺序决定了是UTF-16 BE还是UTF-16 LE。也就是一旦在编译的时候设置好了，之后python里面unicode的实现都是这种方式。python3.3之后，采用了一种更为灵活的方式来表示，字符串可能是ascii, latin-1, utf-8, utf-16, utf-32等编码形式中的一种，就是说不再是写死一种方式了（这个地方还没完全弄明白，先留个坑），这个主要是pep 393中约定的：[PEP 393 -- Flexible String Representation](https://www.python.org/dev/peps/pep-0393/) 

### 参考链接
* <a>https://www.cnblogs.com/malecrab/p/5300503.html</a>
* <a>https://zh.wikipedia.org/wiki/Unicode</a>  
* <a>https://zh.wikipedia.org/wiki/通用字符集#Unicode和ISO_10646的关系</a>


  
  