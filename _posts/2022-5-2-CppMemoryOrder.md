---
layout: post
title:  "C++11 Memory Model and Atomic Type"
date:   2022-5-2 18:05:55 +0800
image:  order.jpg
tags:   [Note]
---

写多线程的代码，无锁编程(lock-free programming)是个坎。大概一年前，我一直在看线程池有关的东西，都是用锁、信号量这些在做同步。当时听说过无锁编程，不过看各种博客也没搞明白是什么。后来看了本书，叫《C++ Concurrency in action》，有关的部分讲的比较清楚，就记下来一篇博客，整理一下。

这篇博客主要记C++的内存模型和原子类型，也有一些无锁数据结构的描述。

### C++11 Memory Model

C++11标准中，在语言层面定义了内存模型。内存模型有两个方面，1）结构，即数据是如何在内存中排布的；2）并发，即多线程读写同一块内存时顺序是如何确定的。

内存模型的结构方面很好理解，举个结构体bit field的例子，

{% highlight cpp %}struct S {
  char a;
  int b : 5;
  int c : 11,
      d : 8,
  struct {
    int ee : 8;
  } e;
} obj;{% endhighlight %}

一个结构体S占据5个字节，a占据第1个字节，b占据第2个字节的前5个比特，c占据第2个字节的后3个比特以及第3个字节，d占据第4个字节，ee占据第5个字节。这就是S这个数据类型的内存模型结构方面。

内存模型的并发方面涉及到多线程。两个线程读取同一个内存地址时，不会出现问题；但是当两个线程同时向一个内存地址写入数据的时候，就会出现冲突；同样的，一个线程读取，另一线程写入同一内存地址时也会导致冲突。为了避免这种冲突，需要指定两个线程操作这块内存的顺序。

可以通过两种方式指定这种操作顺序。一是通过锁，例如C++11标准库中提供的mutex，加锁保证了在某一时刻，只有一个线程可以操作被保护的内存地址；二是通过原子操作，来保证操作同一个内存地址时，多个线程间的同步。

这里主要关注**使用原子操作来指定多线程的同步顺序**。

### Atomic Type

原子类型用来保证操作的原子性。原子操作是指不可分割的操作，即不可能观察到该操作的中间状态。举个例子，如果读取一个对象A的操作是原子的，而且所有修改对象A的操作也是原子的，那么，读取对象A的结果，只可能是对象A的初始值，或者其他线程对对象A的修改值。相反，对于非原子性的操作，读取对象A的结果，可能既不是初始值，也不是任何一个修改值，而是处于某种中间状态。

C++11中，在\<atomic\>头文件里提供了atomic类，所有对atomic类的操作都是原子的。

atomic是一个模版类，可以生成各种原子类型，例如atomic\<int\>，atomic\<bool\>。在原子类型的实现中，同步操作可以依靠锁来实现，也可以依靠原子指令(atomic instructions)来实现，依靠原子指令实现的原子类型，被称为无锁的。

原子指令从指令的层面保证操作的原子性，例如`fetch_and_add`指令原子性地加载一块内存地址到寄存器，然后做加法，不会出现只加载到寄存器，没有做加法的情况。由于指令的操作数存在大小限制，所以原子指令不能保证复杂数据的操作原子性。

在标准库中，atomic类型有一个`is_lock_free()`的成员函数，用来表述对应的原子类型是用原子指令实现的，还是用锁来实现的。这个函数的返回值与模版参数有关，也与编译器对atomic类的实现有关。

atomic类型支持`load()`，`store()`，`exchange()`，`compare_exchange_weak()`和`compare_exchange_strong()`操作。每一个对于原子类型的操作都有一个可选的内存序(memory order)的参数，用来指定内存模型的并发方面。原子操作分为三种：

- 写入(store)，对应的内存序可以是`memory_order_relaxed`，`memory_order_release`，`memory_order_seq_cst`。

- 读取(load)，对应的内存序可以是`memory_order_relaxed`，`memory_order_consume`，`memory_order_acquire`，`memory_order_seq_cst`。

- 读写(read-modify-write)，对应的内存序可以是`memory_order_relaxed`，`memory_order_consume`，`memory_order_acquire`，`memory_order_release`，`memory_order_acq_rel`，`memory_order_seq_cst`。

如果没有指定的参数，所有的操作默认使用`memory_order_seq_cst`内存序。

### Memory Order

在介绍内存序之前，首先给出两种关系：

1. happen-before: 

如果动作A发生时，动作B产生的效果已经可以被动作A观测到，则称动作B发生在动作A之前，也就是存在happen before关系。在同一个线程执行的源代码中，可以理解为，前面的语句happen-before后面的语句。

2. synchronized-with:

只有通过原子类型，才可以获得synchronized-with关系。如果一个线程A写入了一个数据，然后线程B读取到线程A写入的数据，这里线程A的写入操作和线程B的读取操作就是synchronized-with关系。


接下来详细描述各个内存序。

给个例子，

{% highlight cpp %}#include <vector>
#include <atomic>
#include <iostream>

std::vector<int> data;
std::atomic<bool> data_ready(false);

void reader_thread() {
  while(!data_ready.load()) {
    std::this_thread::sleep(std::milliseconds(1));
  }
  std::cout << "The answer = " << data[0] << std::endl;
}

void writer_thread() {
  data.push_back(42);
  data_ready = true;
}
{% endhighlight %}

reader_thread函数和writer_thread函数分别运行在两个线程中。

简单分析一下。在writer_thread中，写入data变量happen-before写入data_ready变量；在reader_thread中，读取data_ready变量happen-before读取data变量；同时，由于reader_thread中读取到writer_thread中写入的true之后，循环才会被跳过，所以，写入data_ready变量的操作synchronized-with读取data_ready变量。事实上，synchronized-with关系引入了跨线程的happen-before关系，所以可以得到，写入data变量happen-before读取data变量。

用形式化一点的语言表述为：

1. store data happen-before store data_ready

2. load data_ready happen-before load data

3. store data_ready synchronized-with load data_ready

用以上三个条件，可以得到：

store data happen-before load data

这种表述会在接下来讲内存序的部分中经常用到。

#### SEQUENTIALLY CONSISTENT ORDERING

原子类型默认的内存序。这种内存序在所有的线程中建立统一的修改顺序，也就是，不论在哪一个线程中，观测到的原子数据的修改顺序都是一样的。

可能这个表述让人迷惑，来看一段代码：

{% highlight cpp %}#include <atomic>
#include <thread>
#include <assert.h>
std::atomic<bool> x,y;
std::atomic<int> z;

void write_x()
{
    x.store(true,std::memory_order_seq_cst);
}

void write_y()
{
    y.store(true,std::memory_order_seq_cst);
}

void read_x_then_y()
{
    while(!x.load(std::memory_order_seq_cst));
    if(y.load(std::memory_order_seq_cst))
        ++z;
}

void read_y_then_x()
{
    while(!y.load(std::memory_order_seq_cst));
    if(x.load(std::memory_order_seq_cst))
        ++z;
}

int main() {
    x=false;
    y=false;
    z=0;
    std::thread a(write_x);
    std::thread b(write_y);
    std::thread c(read_x_then_y);
    std::thread d(read_y_then_x);
    a.join();
    b.join();
    c.join();
    d.join();
    assert(z.load()!=0);
}
{% endhighlight %}

这段代码中，assert永远不会终止程序，即z.load() != 0是永远成立的。如上所说，因为所有的原子操作都指定为memory_order_seq_cst，所以在四个线程中，看到的x，y的修改顺序都是一样的。这里只有两种情况，1）write_x happen-before write_y，那么对于read_y_then_x来说，既然y已经被写入了，那么x也必定被写入，即thread d会执行z++，2）write_y happen-before write_x，对于read_x_then_y，x已经被写入，那么y也必定被写入，即thread c会执行z++。当然这里的thread c和thread d可能都执行z++。无论哪种情况，z.load() != 0都是成立的。这个例子可以看出，使用memory_order_seq_cst这一内存序，会保证所有线程见到的修改顺序都是一样的。

memory_order_seq_cst是一个强有力的保证内存顺序的参数，与此同时，会带来一些不必要的性能损耗。保证所有线程见到的修改顺序一样，需要使用一些机制，来同步在各个处理器中执行的线程以及对应处理器上的缓存。可以通过放松限制，来获取更高的性能。

如何放松限制？请看接下来的内存序类型。

#### RELAXED ORDERING

以relaxed ordering为内存序的原子操作，不构成synchronized-with关系。在同一个线程中执行的原子操作仍然保持着happen-before的关系，但是跨线程的原子操作不构成任何相对序列关系。相对于sequencially consistent内存序，使用relaxed ordering可能使不同的线程看到不同的变量修改顺序。

看代码，

{% highlight cpp %}#include <atomic>
#include <thread>
#include <assert.h>
std::atomic<bool> x,y;
std::atomic<int> z;
void write_x_then_y()
{
    x.store(true,std::memory_order_relaxed);
    y.store(true,std::memory_order_relaxed);
}

void read_y_then_x()
{
    while(!y.load(std::memory_order_relaxed));
    if(x.load(std::memory_order_relaxed))
      ++z;
}

int main() {
    x=false;
    y=false;
    z=0;
    std::thread a(write_x_then_y);
    std::thread b(read_y_then_x);
    a.join();
    b.join();
    assert(z.load()!=0);
}
{% endhighlight %}

在如上代码中，assert可能会终止程序，也就是z.load() == 0可能成立。

由于原子操作都是指定memory_order_relaxed内存序，所以跨线程没有变量修改顺序限制关系。在线程a中，store x happen-before store y，并不意味着在线程b里store x happen-before store y。也就是，在线程a中的变量修改顺序，与在线程b中见到的不一定一致。说个简单的可能，在线程a执行的处理器中，x先被写入L1缓存，然后y再被写入L1缓存，在写回内存时，却是y先被写回。这导致在线程b执行的处理器中，加载到被线程a写入的y，以及没有被写入的x。这种情况就会导致z++没有执行。

简单说，由于relaxed ordering没有引入同步，所以在执行上没有很多限制，这会导致不同线程看到变量的修改顺序不一致，同时性能会好一些。

讲完了约束最强的内存序和约束最弱的内存序，接下来讲约束介于两者之间的内存序。

#### ACQUIRE-RELEASE ORDERING

Acquire-Release内存序引入了synchronized-with关系，但不是全局的同步。在这个内存序模型下，原子读取是acquire操作(memory_order_acquire)，原子写入是release操作(memory_order_release)，原子读写操作可以是acquire、release或者acq_rel。acquire的读操作和release的写操作之间存在同步，也就是在这个内存序下，同步关系是成对的。

> A release operation synchronizes-with an acquire operation that reads the value written.

看代码，

{% highlight cpp %}#include <atomic>
#include <thread>
#include <assert.h>
std::atomic<bool> x,y;
std::atomic<int> z;
void write_x_then_y()
{
    x.store(true,std::memory_order_relaxed);
    y.store(true,std::memory_order_release);
}
void read_y_then_x()
{
    while(!y.load(std::memory_order_acquire));
    if(x.load(std::memory_order_relaxed))
        ++z;
}

int main() {
    x=false;
    y=false;
    z=0;
    std::thread a(write_x_then_y);
    std::thread b(read_y_then_x);
    a.join();
    b.join();
    assert(z.load()!=0);
}
{% endhighlight %}

上述代码中，assert永远都不会终止程序，也就是z.load() != 0永远成立。

1. store x happen-before store y

2. store y synchronized-with load y

3. load y happen-before load x 

可以推导出，

store x happen-before load x

所以，++z的操作一定会被执行。这里的synchronized-with关系是由acquire-release对引入的。同时注意，这里load y时使用了while循环，保证了只有在线程b读取到了线程a中的写入时才会退出循环。也正如上文所述，只有acquire的读操作读取到了release的写操作写入的数据，才会形成同步关系。

#### CONSUME

这里不多讲，consume内存序用在读操作上，与release操作匹配使用可以形成依赖同步关系。这里的依赖同步关系比acquire-release形成的同步关系要弱一些，只有与consume读或release写产生依赖的数据才会形成同步关系，无依赖的数据则没有同步关系。

这里不讲内存栅栏(memory fence)，可以找找别的资料看，也是一种做多线程之间同步的工具。

至此，C++11的内存模型已经讲完了。

### 无锁编程

其实理解了上面讲的，无锁编程就很好理解了，就是不用锁来写多线程的代码。一般是利用atomic类型以及内存栅栏来做同步。无锁编程是一项非常耗脑力的工作，很容易出问题，一般不建议非专家的人来写。

这里贴一段代码，简单示范一个无锁的数据结构，

{% highlight cpp %}template<typename T>
class lock_free_stack
{
  private:
    struct node {
        T data;
        node* next;
        node(T const& data_):
            data(data_) {} 
    };

    std::atomic<node*> head;
public:
    void push(T const& data) {
        node* const new_node=new node(data);
        new_node->next=head.load();
        while(!head.compare_exchange_weak(new_node->next,new_node));
    }

    void pop(T& result)
    {
        node* old_head=head.load();
        while(!head.compare_exchange_weak(old_head,old_head->next));
        result=old_head->data;
    }
};
{% endhighlight %}

不做解释，大概看看就可以了。这段代码其实会存在内存泄漏的问题，如果要处理这个内存泄漏会需要用一些其他的算法来处理，最后写出来的无锁栈的代码非常冗长。

> Lots of the complexity in lock-free code comes from managing memory.

一般不写，能看懂就行。

这篇博客到此为止。

### Reference

1. [cppreference: Memory model](https://en.cppreference.com/w/cpp/language/memory_model)

2. [stackoverflow: C++11 introduced a standardized memory model](https://stackoverflow.com/questions/6319146/c11-introduced-a-standardized-memory-model-what-does-it-mean-and-how-is-it-g)

3. [c++ struct bit field](https://en.cppreference.com/w/cpp/language/bit_field)

4. [sequentially consistent ordering](https://en.cppreference.com/w/cpp/atomic/memory_order#Sequentially-consistent_ordering)