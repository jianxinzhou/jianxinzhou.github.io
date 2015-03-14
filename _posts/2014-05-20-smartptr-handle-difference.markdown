---
layout: post
title:  "Smart Pointer"
date:   2014-05-20 17:18:34
categories: cpp
---

##自动化管理资源

从较浅的层面看，智能指针是利用了一种叫做RAII（资源获取即初始化）的技术对普通的指针进行封装，
这使得智能指针实质是一个对象，行为表现的却像一个指针。  
作用当然很明显，防止忘记调用delete，当然还有另一个作用，就是异常安全。在一段进行了try/catch的代码段里面，即使你写入了delete，也有可能因为发生异常，程序进入catch块，从而忘记释放内存，这些都可以通过智能指针解决。

##编程语义的转换

但是智能指针还有一重更加深刻的含义，就是把 [@陈硕](http://weibo.com/giantchen)所说的value语义转化为reference语义。  
C++和Java有一处最大的区别在于语义不同，在Java里面下列代码：

{% highlight java %}
Animal a = new Animal();
Animal b = a;
{% endhighlight %}

你当然知道，这里其实只生成了一个对象，a和b仅仅是把持对象的引用而已。但在C++中不是这样，
{% highlight c++ %}
    Animal a;
    Animal b;
{% endhighlight %}

这里确实就是生成了两个对象。
在编写OOP程序时，value语义带来太多的困扰，例如TCP连接中我封装一个accept函数接收请求，那么应该是这样的：

{% highlight c++ %}
Socket accept();
{% endhighlight %}
这就带来一个问题，采用对象做返回值，这里面有一个对象的复制的过程，但是Socket因为某些原因，我让他继承了boost::noncopyable，总之就是Socket失去了复制和赋值的能力，那么该怎么办？  
我们首先想到指针，在accept内部new生成一个对象，然后返回指针。但是问题更多，这个对象何时析构？ 过早析构，程序发生错误，不进行析构，又造成了内存泄露。  
这里的解决方案就是智能指针，而且是引用计数型的智能指针。

{% highlight c++ %}
    typedef boost::shared<Socket> SocketPtr;
    SocketPtr accept();
{% endhighlight %}

这样外部就可以用智能指针去接收，那么何时析构？当然是引用计数为0，也就是我不再需要这个Socket的时候析构。  
这样，我们利用了SockerPtr，实现了跟Java类似的Reference语义。

还有一个例子，Java中往容器中放对象，实际放入的是引用，不是真正的对象，而C++在vector中push_back采用的是值拷贝，如果想实现Java中的引用语义，就应该使用智能指针，可以参考《C++标准库程序》（侯捷/孟岩 译）的第五章讲容器的部分，有一节叫做“用Value语义实现Reference语义”


参考资料：陈硕《Linux多线程服务器端编程》


