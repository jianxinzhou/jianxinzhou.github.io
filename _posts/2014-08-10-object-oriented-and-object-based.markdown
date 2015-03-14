---
layout: post
title:  "面向对象编程与基于对象编程"
date:   2014-08-10 12:18:34
categories: cpp
---




##一个简单的定时器

首先我用C编写一个简单的定时器，该定时器使用timerfd系列的函数，定时器的触发不再依靠信号，而是fd可读，所以可以把它交给poll去触发。
    
这里是源代码：

{% highlight c++ %}
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <sys/timerfd.h>
#include <poll.h>
#define ERR_EXIT(m) \
    do { \
        perror(m);\
        exit(EXIT_FAILURE);\
    }while(0)

int main(int argc, const char *argv[])
{
    //创建定时器的fd
    int timerfd = timerfd_create(CLOCK_REALTIME, 0);
    if(timerfd == -1)
        ERR_EXIT("timerfd_create");

    //开启定时器，并设置时间
    struct itimerspec howlong;
    memset(&howlong, 0, sizeof howlong);
    howlong.it_value.tv_sec = 5;  //初始时间
    howlong.it_interval.tv_sec = 1; //间隔时间
    if(timerfd_settime(timerfd, 0, &howlong, NULL) == -1)
        ERR_EXIT("timerfd_settime");

    struct pollfd event[1];
    event[0].fd = timerfd;
    event[0].events = POLLIN;
    char buf[1024];

    while(1)
    {
        int ret = poll(event, 1, 10000);
        if(ret == -1)
            ERR_EXIT("poll");
        else if(ret == 0)
            printf("timeout\n");
        else
        {
            //这里必须read，否则poll被不停的触发
            if(read(timerfd, buf, sizeof buf) == -1)
                ERR_EXIT("read");
            printf("foobar .....\n");
        }
    }

    close(timerfd);
    return 0;
}
{% endhighlight %}
    
这个定时器的逻辑非常简单，我们设置的间隔时间为1s，所以每隔1s，对应的timerfd就变为可读，此时poll返回，执行第44行，用户的操作。
    
##面向对象的解决方案

观察上面的代码，可以发现一个事实，除了极少数代码，如定时器的初始时间/间隔时间和用户自定义的逻辑外，其他的地方基本都是固定的。于是我们用class对其进行封装。
    
定时器的时间可以做成单独的函数，去设置数据成员，定时器的操作呢？我们自然想到采用虚函数，而且是纯虚函数。
    
于是我写出以下的class：

{% highlight c++ %}
class Timer
{
public:
    // ...
    void setTimer(int val, int interval); //设置时间
    virtual void timeout() = 0;   //超时执行的操作
    
    // 以下简略
};
{% endhighlight %}


这个类的编写没有问题，那么如何使用呢？ 答案很简单，稍微学过面向对象的读者就知道，我们需要写一个类继承Timer，然后实现其中的timeout纯虚函数即可。

如下：
{% highlight c++ %}
class MyTimer : public Timer
{
public:
    void timeout()
    {
        printf("foobar......\n");
    }
    // 其他省略
};
{% endhighlight %}

这种把模型封装成基类，然后用户去继承，去实现纯虚函数的做法，叫做面向对象的解决方案。

##基于对象的解决方案

基于对象，我们采用的是一对神器function/bind，这原本是Boost库的组件，于2005年置入C++的TR1，现已经成为C++11的内置标准。

**本文不是function/bind的初级教程**，这里假定读者已经掌握了其基本使用。

之前的Timer，可以这样封装：
{% highlight c++ %}
class Timer
{
public:
    typedef std::function<void ()> TimerCallback;
    // ...
    void setTimer(int val, int interval); //设置时间
    void setTimerCallback(const TimerCallback &cb); //设置回调函数
    // ...
    
private:
    TimerCallback timerCallback_; //回调函数
};
{% endhighlight %}

现在的使用就非常简单了，用户只需要自己编写一个回调函数，然后注册给定时器即可。

{% highlight c++ %}
void foobar()   //这是回调函数
{
    printf("foobar.....\n"); 
}

int main()
{
    Tiemr t;
    t.setTimer(4, 1);
    t.setTimerCallback(&foobar);  //注册foobar为回调函数
    
    t.startTimer();
    
    return 0;
}
{% endhighlight %}

如果你操作的数据比较复杂，当然可以封装成类，**但是此时用的是组合，而不是继承**。

{% highlight c++ %}
class MyTimer
{
public:
    // ....
    void foobar()
    {
        printf("foobar...\n");
    }
    void setCallback()
    {
        timer_.setTimerCallback(
            std::bind(&MyTimer::foobar, this));  //这里展示了bind的用法
    }

    //其他接口，例如启动定时器等
private:
    Timer timer_;
    //Other Data
}

{% endhighlight %}

可以看看这个实现，Timer和Thread的实现参见：[Timer](https://github.com/guochy2012/EchoLib/blob/master/src/Timer.h)、[Thread](https://github.com/guochy2012/EchoLib/blob/master/src/Thread.h)

{% highlight c++ %}
class FooThread
{
    public:
        FooThread();
        void print();
        void startTimerThread();
    private:
        Thread thread_;
        Timer timer_;
};

FooThread::FooThread()
{
}

void FooThread::print()
{
    cout << "Hello World" << endl;
}

void FooThread::startTimerThread()
{
    timer_.setTimer(3, 1);
    timer_.setTimeCallback(bind(&FooThread::print, this));
    thread_.setCallback(bind(&Timer::runTimer, &timer_));
    thread_.start();
    thread_.join();
}



int main(int argc, const char *argv[])
{
    FooThread foo;
    foo.startTimerThread();
    return 0;
}
{% endhighlight %}

##基于对象方便在哪里？

我们先提出一个问题：我们在main里面执行了定时器，那么main就无法从事其他工作了，毕竟只有一个线程。

于是我们打算将Timer封装在一个线程中，称为TimerThread，TimerThread启动后便新建线程执行定时器，而不占用当前线程。

如果是采用面向对象，那么Thread也是采用由用户去实现纯虚函数run的方法实现的。
例如：
{% highlight c++ %}
class Thread
{
public:
    virtual void run() = 0;  // 用户继承去实现这个函数
};
{% endhighlight %}

现在我们去编写TimerThread类，如何具体实现呢？我们要实现Thread里面的run函数，还要保留Timer的功能，更重要的是，**线程启动后需要启动定时器，所以二者之间是一种调用关系**。

这里我决定采用多重继承。

问题又来了，Thread、Timer我们都采用public继承？我们知道public继承是一个"is-a"的关系，**TimerThread我更倾向于看做是一个Thread类，而对于Timer类，我打算采用实现继承而不是接口继承**。

于是，我们采用public继承Thread，private继承Timer。如下：

{% highlight c++ %}
class TimerThread ：public Thread
                    private Timer
{
public:
    void run()  //线程的run方法
    {
        Timer::startTimer();  //在线程里面开启定时器
    }
    
private:
    // ....
};
{% endhighlight %}

这个类依然是抽象类，因为Timer的纯虚函数还没有实现，用户去继承它，实现定时器的timeout函数，就可以使用了。

采用基于对象的解决方案会简单很多，我们把Thread和Timer组合起来即可。  

这里注意，**我们让用户自定义回调函数，传给Timer，然后把Timer的startTimer作为回调函数，传给Thread**。

{% highlight c++ %}
class TimerThread : private NonCopyable
{
public:
    typedef std::function<void()> TimerCallback;
    // Construct 
    void setTimerCallback(const TimerCallback &cb);
    
    // Other
private:
    Timer timer_;
    Thread thread_;
};
{% endhighlight %}

我们看一下里面回调函数的绑定顺序：
{% highlight c++ %}
void TimerThread::setTimerCallback(const TimerCallback &cb)
{
    timer_.setTimerCallback(cb);  //设置Timer的回调
    thread_.setCallback(bind(&Timer::runTimer, &timer_)); //设置线程的回调
}
{% endhighlight %}

这里有一份完整的基于对象实现代码：[TimerThread](https://github.com/guochy2012/EchoLib/blob/master/src/TimerThread.h)

这就是基于对象的解决方案。

##基于对象和面向对象的对比

首先第一点，面向对象是基于继承和多态的，这里的继承是public继承。而基于对象主要使用了function/bind实现事件的回调。

其次，二者的**本质区别**在于基于对象把OOP中的虚函数改为了由function实现的回调函数。

第三，OOP采用虚函数，所以用户必须去继承并实现它，而基于对象则不然，用户写全局函数/类的成员函数皆可，更甚者，**我们有bind，可以自由的修改函数的签名信息，主要是参数列表**。

最后，我们借助function/bind，实现了C#中的**委托**（delegate）机制。


##参考资料

 - [function/bind的救赎（上）](http://blog.csdn.net/myan/article/details/5928531)  by 孟岩
 - [以boost::function和boost:bind取代虚函数 ](http://blog.csdn.net/solstice/article/details/3066268) by 陈硕
