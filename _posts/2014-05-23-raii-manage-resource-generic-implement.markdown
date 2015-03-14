---
layout: post
title:  "用RAII技术管理资源及其泛型实现"
date:   2014-05-23 16:18:34
categories: cpp
---




##前言
RAII的含义是“资源获取即初始化”。

##一段看似安全的代码
首先看一段代码：

{% highlight c++ %}
try{

	int *p = new int[100];

	// ... do something

	delete[] p;
}catch(exception &e){
	// .....
}
{% endhighlight %}
这段代码中，我们先进行了动态内存分配，使用完释放，看起来很完美，但是这段程序是否真的保证不会发生内存泄漏？

考虑这样一种情形，程序在使用这段内存的过程中throw一个异常，于是程序转向catch块，然后XXX。 这段内存被释放了吗？ 显然没有。 那么这段程序应该从哪里改进呢？



##对象的生命期
考虑一个问题：C++中对象的声明期是怎样的？在C++中，对象创建的方式有两种，一种是栈上，一种是堆上创建。

{% highlight c++ %}
{
	Animal a;	// stack
	Animal *pa = new Animal();	// heap
}
{% endhighlight %}

上面的代码中，第一个对象创建在栈上，更明确的说法是它是一个`局部变量`，这意味着它的生命期起源于被创建的这行语句，终结与所在作用域的末尾，也就是这里的右花括号（}）。
而第二个对象呢？它是采用所谓的动态内存分配生成的，需要程序员手工去释放，当调用delete的时候才销毁，但调用delete的时机不是固定的。
也就是说，`栈对象的生命期是明确`的，而堆对象的生命期由于取决于调用delete的时机，因而是不明确的。

看到这里，之前那段有可能内存泄漏的代码如何去改进呢？

答案就是用`栈对象明确的生命期去管理资源`。

##用对象的生命期管理资源

试想一下，如果我们把之前程序中，对内存的分配写在构造函数中，把释放资源写在析构函数中，而栈对象的生命期是明确的，当该管理资源的对象过期时，连同它管理的资源一起释放，岂不是非常智能化？

我们尝试着写出下列代码：
{% highlight c++ %}
class ScopePtr{
public:
    ScopePtr(int *p):_p(p){

    }
    ~ScopePtr(){
    	delete[] _p;
	}
private:
	int *_p;
};
{% endhighlight %}
我们把之前的代码做如下的改进：
{% highlight c++ %}
try{

	ScopePtr scope(new int[100]);

	// ... do something

}catch(exception &e){
	// .....
}
{% endhighlight %}
再来分析一下这段代码：
    如果正常执行，那么当执行完try块时，scope对象过期，执行析构函数，同时释放了那段数组。如果使用的过程中发生了异常，那么当程序进入catch块时，同样会销毁try内的局部变量。
    无论是哪种情况，内存总是会被释放。
    如果这里不是int，而是其他复杂的类型，使用这个封装的ScopePtr是不是不太方便？显然不会，我们去重载成员操作符就可以了，使它表现的像个指针，这就是一个`最简单的智能指针`的产生。
    问题得到了完美的解决！
    
##资源获取即初始化

我们上面解决问题的办法就是RAII技术，RAII的含义是“资源获取即初始化”，这个概念有两个要点：

-   获得资源后立即放进管理对象
-   管理对象运用析构函数确保资源被释放


看另外一个例子：我们在访问一些临界区资源的时候通常需要加锁，所以产生了下面的代码：

{% highlight c++ %}
{
	mutex.lock();
	//do sth..

	mutex.unlock();
}
{% endhighlight %}
这种方式是很容易出现问题的，例如程序中间遇见错误情况需要退出这个函数，此时很容易忘记解锁：
{% highlight c++ %}
{
	mutex.lock();
	//do sth..
	if(...){
		return false // forgot to unlock
	}

	// ...
	mutex.unlock();
}
{% endhighlight %}

此时如果再次进行Lock操作，就造成了死锁。
解决这个问题的办法仍然很简单，我们去写一个类：
{% highlight c++ %}
class MutexLockGuard{
public:
	MutexLockGuard(MutexLock mutex):_mutex(mutex){
		_mutex.lock();
	}
	~MutexLockGuard(){
		_mutex.unlock();
	}
private:
	MutexLock &_mutex;
};
{% endhighlight %}
这样刚才那段代码就可以修改成：
{% highlight c++ %}
{
	MutexLockGuard guard(lock);
	//do sth..
	if(...){
		return false
	}

	// ...
}
{% endhighlight %}

这样，一旦离开这段代码，程序立刻自动解锁。
不过为了防止错误使用这个类，例如：
{% highlight c++ %}
   MutexLockGuard(lock);
{% endhighlight %}
可以定义一个宏：
{% highlight c++ %}
#define MutexLockGuard(m) "ERR MutexLockGuard"
{% endhighlight %}
这样我们在错误使用的时候，编译期间就能发现错误。

##一种泛型解决方案

[刘未鹏](http://weibo.com/pongba)在他的[《C++11（及现代C++风格）和快速迭代式开发》](http://mindhacks.cn/2012/08/27/modern-cpp-practices/)中提出了一种`泛型`实现，利用了C++11的function和Lambda匿名函数，如下：

{% highlight c++ %}
class ScopeGuard
{
public:
    explicit ScopeGuard(std::function<void()> onExitScope)
        : onExitScope_(onExitScope), dismissed_(false)
    { }

    ~ScopeGuard()
    {
        if(!dismissed_)
        {
            onExitScope_();
        }
    }

    void Dismiss()
    {
        dismissed_ = true;
    }

private:
    std::function<void()> onExitScope_;
    bool dismissed_;

private: // noncopyable
    ScopeGuard(ScopeGuard const&);
    ScopeGuard& operator=(ScopeGuard const&);
};
{% endhighlight %}
使用方式也很简单：
{% highlight c++ %}
HANDLE h = CreateFile(...);
ScopeGuard onExit([&] { CloseHandle(h); });
{% endhighlight %}
其实就是将该资源释放的函数代码段注册到Scope类，其中原理不再赘述。


##与其他语言的对比

RAII是C++独有的编程手段。通过RAII技术我们能够做到资源不需要使用时立即释放，这是其他GC语言所不具备的。
以Java为例，Java具有完善的GC（Garbage    Collection，垃圾回收）机制，但是存在如下的缺点：


-   GC只能回收内存，而对于打开的文件、数据库连接等仍然需要手工关闭。
-   GC因为进程优先级等原因，回收效率底下，详情可以参考孟岩的[《垃圾收集机制(Garbage Collection)批判》](http://blog.csdn.net/myan/article/details/1906)

##conclusion
RAII技术是现代C++编程技术中及其重要的一部分，甚至有人称其为“C++编程中最重要的编程技法”，可见其重要性。通过RAII，我们完全可以实现资源的自动化管理，写出永不内存泄漏的程序。

参考资料

-   《C++ Primer》
-   《Effective C++》
-   《Linux多线程服务器端编程》



