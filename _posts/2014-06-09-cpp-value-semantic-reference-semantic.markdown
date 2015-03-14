---
layout: post
title:  "C++对象的值语义和引用语义"
date:   2014-06-09 21:18:34
categories: cpp
---



本文谈谈我对C++对象语义的理解。

##定义
值语义和引用语义的定义是这样的：复制后与以前的对象**无关**的对象叫做值语义，无法复制或者复制后与原来的对象存在关联的对象称为引用语义。

那么该如何理解上面的定义呢？ 我的看法是：**对象看起来表现的像一个值，就叫做值语义**。

##Java
我们先从Java中的相关语法看起：

{% highlight java %}
    int a = 8;
    int b = a;
{% endhighlight %}

这里的a b 都是int，属于java中原生数据类型，他们的特点是复制后二者再无关联。它们表现出来是两个value，所以int类型为值语义（**value semantic**）。
与他们形成鲜明对比的是用户自定义的class：

{% highlight java %}
    Employee a1 = new Employee("zhangsan", 23);
    Employee a2 = a1;
    a2.setAge(30);
    System.out.println(a1.getAge());
{% endhighlight %}

这段代码输出显然是30，第二行语句，导致a和b指向了同一个实际的对象。在这里，java的对象表现的行为**不是一种value的行为，而是一种reference**，所以这里称java中的对象为引用语义（reference semantic）。

##C++
回到C++中，再看这个问题，以C++中一个普通的类为例：
{% highlight c++ %}
class ItemBook{
private:
	string isbn_;
	double price_;
	int amount_;
};

{% endhighlight %}

如果我们写出以下的代码：
{% highlight c++ %}
ItemBook a1;
// set member of a1;
ItemBook a2 = a1;

{% endhighlight %}
那么a1和a2复制后没有任何关联，他们的表现和java中的原生数据类型一样，属于value语义。事实上，C++内置的标准库类型例如string、vector、list等都是值语义，**凡是可以放入容器的元素都必须具备value语义**，即必须具备拷贝的能力，放入标准容器的元素和之前的元素没有任何的关联。

那么C++中的引用语义看起来有些奇怪，可以这样理解：正是因为类不可复制，所以我们可以采用智能指针，来把它们转化为reference语义。
典型的例子是TcpConnection，它显然是不可复制的（因为牵扯到系统的资源），那么我们在程序中该如何把它作为参数传递呢？我们定义下面的类型：
{% highlight c++ %}
typedef boost::shared_ptr<TcpConnection> TcpConnectionPtr;
{% endhighlight %}

这样我们在使用TcpConnection的场合一概采用TcpConnectionPtr，它**表现的像一个引用**，这就实现了引用语义。
对于那些对象复制后存在关联的对象，很多时候我们干脆禁用掉copy能力。

##Java和C++的本质区别
我们可以得出这样一个结论，**C++和java的本质区别在于语义不同**！

##总结
在设计C++的类时，`严格的区分开对象是否可复制`，也就是**严格区分value语义和reference语义**，这样可以避免很多潜在的BUG。

###本文系原创，作者本人保留所有权利。







