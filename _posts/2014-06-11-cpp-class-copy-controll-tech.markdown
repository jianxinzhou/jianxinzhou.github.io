---
layout: post
title:  "浅谈C++类的复制控制技术"
date:   2014-06-11 14:18:34
categories: cpp
---




##前言

笔者第一次读《C++ Primer》（第四版）这本经典著作时，对第十三章“类的复制控制”部分，产生一些疑问，如此经典的书为什么没有详述深拷贝和浅拷贝的问题，当时以为这是此书的一个缺憾，后来我逐渐了解到这本书的第三版曾经花了很多篇幅讲述这个问题，在第四版中删除了这部分内容。后者在后面引入了智能指针，这实际上是一种进步，不再局限于深浅拷贝。

实际上，目前为止就笔者了解的而言，类的复制控制一共有五种方法。

##问题来源
看下面的这个类：
{% highlight c++ %}
class PointPtr{

public:
	// some member function
private:
	Point *ptr_;
};
{% endhighlight %}

这个类持有一个Point类型的指针，Point的定义很简单，如下：
{% highlight c++ %}
class Point{
public:
	// some member function
private:
	int x_;
	int y_;
};

{% endhighlight %}

问题究竟出在哪里？

##浅拷贝

如果我们没有自定义类的复制构造函数和赋值运算符，那么编译器会自动帮我们合成相应的版本，大体功能如下：
{% highlight c++ %}
PointPtr(const PointPtr &other)
				:ptr(other.ptr_){

}

PointPtr &operator=(const PointPtr &other){
	if(this != &other){
		ptr_ = other.ptr_;
	}
	return *this;
}
{% endhighlight %}
试想，如果我们复制一份PointPtr：
{% highlight c++ %}
PointPtr p1;
PointPtr p2 = p1;
{% endhighlight %}
那么p1和p2内部的ptr都指向同一个对象，如果我们通过p1改动对象，显然p2也受到了牵连，这显然不是我们想要的。更何况，这里还有一个更严重的问题：如果我们加入析构函数，里面delete掉ptr，那么p1析构后，里面的Point对象已经释放，此时p2再去析构就发生了问题。

这种系统默认的仅仅拷贝指针值的状况，称为浅拷贝（shallow copy）。

##第一种解决方案：禁止复制
上面浅拷贝的问题出在对象复制和赋值的时候，最简单的解决方案就是禁用掉类的copy能力。具体做法是将类的拷贝构造函数和赋值运算符设为私有，而且只提供声明，不提供实现（这是为了防止friend成员）。
一个更加通用的方法是写一个noncopyable类，如下：
{% highlight c++ %}
class noncopyable {

protected:
	noncopyable() {

	}
	~noncopyable() {

	}

private:
	noncopyable(const noncopyable&);
	noncopyable &operator=(const noncopyable &);
};
{% endhighlight %}

以后凡是继承它的类均失去了copy和assign能力。
实际上，boost::noncopyable就是这样实现的。

这种方式看起来简单粗暴，但是大部分情况下它是有效的。  

**尤其是当我们的类持有系统资源的时候，例如文件描述符、网络套接字、数据库连接，此时禁用掉copy语义，可以帮助我们避免很多潜在的bug。**

##深拷贝
即然有浅拷贝，那么另一种解决方案自然就是深拷贝。
深拷贝的含义是：对于那些内部持有资源的对象，复制时不是简单的copy指针的值，而是复制指针指向的资源。
具体实现如下：
{% highlight c++ %}
PointPtr(const PointPtr &other)
	:ptr_(new Point(*(other.ptr_))){
	
}

PointPtr &operator=(const PointPtr &other){
	
	if(this != &other){
		ptr_ = new Point(*(other.ptr_));
	}
	return *this;
}
{% endhighlight %}
此时我们就可以放心的进行析构：
{% highlight c++ %}
~PointPtr(){
	delete ptr_;
}
{% endhighlight %}

读者应该注意到，这里的PointPtr自身是一种值语义。

还有一个典型的案例就是String的实现，读者可以自行尝试。

##引用计数

前面我们采用深拷贝，解决了问题，但是他有一个很严重的问题，每次都去copyPoint对象，这个带来的开销较大。于是我们采用引用计数。


我们再次回顾下浅拷贝引发的问题：  
1. 资源归属不清，多个PointPtr可能持有同一个Point对象  
2. 析构时可能造成灾难性的后果  

这里我们采用引用计数，实则是默认第一个问题的存在，把第二个问题解决好。
具体将就是我们在Point中（PointPtr中也可以）增加一个use成员，记录下当前总共有多少个PointPtr指向它，这样：  
1. 当进行类的copy或assign时，use加1。  
2. 析构时，仅仅把use减一，仅当use为0时才真正析构Point对象。  

大概实现如下：
{% highlight c++ %}
class Point{
	friend class PointPtr;
public:
	// some member function
private:
	int x_;
	int y_;

	int use_; // use count
};

class PointPtr{

public:
	// some member function
	PointPtr(const PointPtr &other)
					:ptr(other.ptr_){
		++ptr_->use_; // use count + 1
	}
	PointPtr &operator=(const PointPtr &other){
		
		++other.ptr_->use_;   // avoid assign to itself
		if(--ptr_->use_ == 0){
			delete ptr_;
		}
		ptr_ = other.ptr_;
		return *this;
	}

	~PointPtr(){
		if(--ptr_->use_ == 0){
			delete ptr_; // only delete object when use count equals zero
		}
	}
private:
	Point *ptr_;
};

{% endhighlight %}

这里有几个注意点：  
1. 在重载=运算符时，要防止自身赋值问题，否则会先析构实际的对象，造成资源无效。这里的解决方案是先将other指向对象的use加一，这样就防止了自身赋值时析构Point的问题。  
2. 只有当use为0时才真正析构对象。  

这里的PointPtr如果重载了成员操作符，那么它就是一个具有引用计数功能的智能指针。  
相对于深复制，引用计数大大减少了Point对象复制的开销。  

##写时复制

我们简单回顾下前面各种手段的优缺点：  
1. 浅拷贝： 毫无疑问，这是错误的。  
2. 禁止复制：简单粗暴，没有后患．  
3. 深拷贝：　对象之间毫无关联，但是复制成本高  
4. 引用计数：　无对象复制开销，但是对象共享资源  

有没有一种方式，即可以实现值语义（像深拷贝那样），又可以减少对象复制的开销（像引用计数）呢？  
答案就是在引用计数的基础上加入写时复制（Copy On Write）．

具体做法是：我们仍然采用引用计数，但是只允许读，一旦我们尝试改动Ｐｏｉｎｔ对象，那么就自动拷贝一份，此时的ＰｏｉｎｔＰｔｒ就单独持有一份Point对象。

以前的代码不变，只是我们添加几个函数：
{% highlight c++ %}
PointPtr &setX(int x){
	if(ptr_->use_ != 1){
		ptr_ = new Point(*ptr_);
	}
	ptr_->setX(x);
	return *this;
}
{% endhighlight %}
每当我们试图修改Point时，就会自动产生一个新的copy，当然如果是use为1，没有共享的情况，就不必生成新的对象。

这里证明一个事实：**使用写时复制技术，引用计数也可以实现值语义**。

写时复制技术一个最典型的应用就是Linux系统fork进程。在传统的UNIX进程模型中，创建子进程需要完全复制父进程，以确保二者几乎独立（实际就是我们所指的值语义），但是这样做：一是耗费内存空间，二是浪费时间。于是Linux采用COW技术，fork子进程时仅仅复制页表，把复制页表项的时机延迟到Write时。


##总结

以上的各种技术除了浅拷贝，均要考虑实际场景。  
涉及到系统资源的一般采用禁止复制。  
我们自己实现String大部分采用深拷贝，有的实现使用了COW。  
对于那些禁止复制的对象，如果想把他们作为参数传递，引用计数型的智能指针是正确选择（例如boost::shared_ptr<T>）。  
很多句柄类，采用了COW技术减少开销。  

这里讲的五种资源控制的情况，对于理解智能指针和句柄类，尤其是后者，有很大的帮助。句柄类不管多么复杂，采用的实现方式总是上面的其中一种。

参考资料  
 - 《C++ Primer》第四版   
 - 《C++ 沉思录》英文版   
 - 《深入Linux内核架构》 （主要是fork进程部分)   
 - 《Effective C++》  





