layout: post
date: 2015-09-15 11:36
tags: 
- CPP
- Multithreading
title: 多线程内存模型和C++1x Atomic
---

本文主要总结了《[为什么程序员需要关心顺序一致性（Sequential Consistency）而不是Cache一致性（Cache Coherence？）](http://www.parallellabs.com/2010/03/06/why-should-programmer-care-about-sequential-consistency-rather-than-cache-coherence/)》，《[浅析C++多线程内存模型](http://www.parallellabs.com/2011/08/27/c-plus-plus-memory-model/)》，《[Memory model synchronization modes](http://gcc.gnu.org/wiki/Atomic/GCCMM/AtomicSync)》，《[std::memory_order](http://en.cppreference.com/w/cpp/atomic/memory_order)》
# Sequential Consistency
## 什么是`Sequential Consistency`
`Sequential Consistency`的作者Lamport给的严格定义是：
> … the result of any execution is the same as if the operations of all the processors were executed in some sequential order, and the operations of each individual processor appear in this sequence in the order specified by its program.

`Sequential Consistency`其实就是规定了两件事情：
1. 每个线程内部的指令都是按照程序规定的顺序（program order）执行的（单个线程的视角）。
2. 线程执行的交错顺序可以是任意的，但是所有线程所看见的整个程序的总体执行顺序都是一样的（整个程序的视角）。

假设我们有两个线程（线程1和线程2）分别运行在两个CPU上，有两个初始值为0的全局共享变量x和y，两个线程分别执行下面两条指令：

初始条件：`x = y = 0;`

线程1   | 线程2 
:-----: |:---:
x = 1  | y = 1
r1 = y | r2 = x
____________|____________

因为多线程程序是交错执行的，所以程序可能有如下几种执行顺序：

Execution 1 | Execution 2 | Execution 3
:----------:|:-----------:|:----------:
x = 1;      |y = 1;       |x = 1;
r1 = y;     |r2 = x;      |y = 1;
y = 1;      |x = 1;       |r1 = y;
r2 = x;     |r1 = y;      |r2 = x;
结果:r1==0 and r2 == 1 | 结果: r1 == 1 and r2 == 0 | 结果: r1 == 1 and r2 == 1
__________________________|__________________________|__________________________

上面三种情况并没包括所有可能的执行顺序，但是它们已经包括所有可能出现的结果了，所以我们只举上面三个例子。我们注意到这个程序只可能出现上面三种结果，但是不可能出现`r1==0 and r2==0`的情况。

其实说简单点，SC就是我们最容易理解的那个多线程程序执行顺序的模型。
## 为什么要关心`Sequential Consistency`
现在的CPU和编译器会对代码做各种各样的优化，有时候它们可能会为了优化性能而把程序员在写程序时规定的代码执行顺序(program order)打乱。
例如编译器可能会做如下优化，即把线程1的两条语序调换执行顺序：

初始条件：`x = y = 0;`

线程 1   | 线程 2
:------:|:-----:
r1 = y; | y=1;
x = 1;  | r2 = x;
___________|___________

那么这个时候程序如果按如下顺序执行就可能就会出现`r1==r2==0`这样程序员认为”不正确“的结果：

| Execution 4 |
|:-----------:|
| r1 = y;     |
| y = 1;      |
| r2 = x;     |
| x = 1;      |
|_____________|
为什么编译器会做这样的优化呢？因为读一个在内存中而不是在cache中的共享变量需要很多周期，所以编译器就”自作聪明“的让读操作先执行，从而隐藏掉一些指令执行的latency，提高程序的性能。实际上这种类似的技术是在单核时代非常普遍的优化方法，但是在进入多核时代后编译器没跟上发展，导致了对多线程程序进行了违反SC的错误优化。

我们发现为了保证多线程的正确性，我们希望程序能按照SC模型执行；但是SC的对性能的损失太大了，CPU硬件和编译器为了提高性能就必须要做优化啊！为了既保证正确性又保证性能，在经过十几年的研究后一个新的新的模型出炉了：sequential consistency for data race free programs。简单地说这个模型的原理就是对没有data race的程序可以保证它是遵循SC的，这个模型在多线程程序的正确性和性能间找到了一个平衡点。

说简单点，就是由程序员用同步原语（例如锁或者atomic的同步变量）来保证你程序是没有data race的，这样CPU和编译器就会保证你程序是按你所想的那样执行的（即SC），是正确的。换句话说，程序员只需要恰当地使用具有`acquire`和`release`语义的同步原语标记那些真正需要同步的变量和操作，就等于告诉CPU和编译器你们不要对这些标记出来的操作和变量做违反SC的优化，而其它未被标记的地方你们可以随便优化，这样既保证了正确性又保证了CPU和编译器可以做尽可能多的性能优化。

从根源上来讲，在串行时代，编译器和CPU对代码所进行的乱序执行的优化对程序员都是封装好了的，无痛的，所以程序员不需要关心这些代码在执行时被乱序成什么样子，因为这些都被编译器和CPU封装起来了，你不用担心内部细节，它最终表现出来的行为就是按你想要的那种方式执行的。但是进入多核时代，程序员、编译器、CPU三者之间未能达成一致，所以CPU、编译器就会时不时地给你捣蛋，故作聪明的做一些优化，让你的程序不会按照你想要的方式执行，是错误的。
# C++1x中引入的`atomic`类型
C++1x除了提供传统的锁、条件变量等同步机制之外，还引入了新的`atomic`类型。相对于传统的`mutex`锁来说，`atomic`类型更底层，具备更好的性能，因此能用于实现诸如Lock Free等高性能并行算法。

对常见的数据类型，C++1x都提供了与之相对应的`atomic`类型。以`bool`类型举例，与之相对应的`atomic_bool`类型具备两个新属性：原子性与顺序性。顾名思义，原子性的意思是说`atomic_bool`的操作都是不可分割的，原子的；而顺序性则指定了对该变量的操作何时对其他线程可见。

在C++1x中，为了满足对性能的追求，`atomic`类型提供了三种顺序属性：`sequential consistency ordering`（即顺序一致性），`acquire release ordering`以及`relaxed ordering`。因为`sequential consistency`是最易理解的模型，所以默认情况下所有`atomic`类型的操作都会使`sequential consistency`顺序。当然，顺序一致性的性能相对来说比较差，所以程序员还可以使用对顺序性要求稍弱一些的`acquire release ordering`与最弱的`relaxed ordering`。

在下面这个例子中，`atomic_bool`类型的变量`data_ready`就被用来实现两个线程间的同步操作。需要注意的是，对`data_ready`的写操作仍然可以通过直接使用赋值操作符（即“`=`”）来进行，但是对其的读操作就必须调用`load()`函数来进行。在默认的情况下，所有`atomic`类型变量的顺序性都是顺序一致性（即sequential consistency）。在这个例子中，因为`data_ready`的顺序性被规定为顺序一致性，所以线程1中对`data_ready`的写操作会与线程2中对`data_ready`的读操作构建起`synchronize-with`的同步关系，即#2->#3。又因为`writer_thread()`中的代码顺序规定了#1在#2之前发生，即#1->#2；而且`reader_thread`中的代码顺序规定了#3->#4，所以就有了#1->#2->#3->#4这样的顺序关系，从而可以保证在#4中读取`data`的值时，#1已经执行完毕，即#4一定能读到#1写入的值（10）。

```c++
#include <atomic>
#include <vector>
#include <iostream>
 
std::vector<int> data;
std::atomic_bool data_ready(false);
 
// 线程1
void writer_thread()
{
    data.push_back(10); // #1：对data的写操作
    data_ready = true; // #2：对data_ready的写操作
}
 
// 线程2
void reader_thread()
{
    while(!data_ready.load()) // #3：对data_ready的读操作
    {
        std::this_thread::sleep(std::milliseconds(10));
    }
    std::cout << ”data is ” << data[0] << ”\n”; // #4：对data的读操作
}
```

相信很多朋友会纳闷，这样的执行顺序不是显然的么？其实不然。如果我们把`data_ready`的顺序性制定为`relaxed ordering`的话，编译器和CPU就可以自由地做违反顺序一致性的乱序优化，从而导致#1不一定在#2之前被执行，最终导致#4中读到的`data`的值不为10。

简单的来说，在`atomic`类型提供的三种顺序属性中，`acquire release ordering`对顺序性的约束程度介于`sequential consistency`（顺序一致性）和`relaxed ordering`之间，因为它不要求全局一致性，但是具有`synchronized with`的关系。`Relaxed ordering`最弱，因为它对顺序性不做任何要求。由此可见，除非非常必要，我们一般不建议使用`relaxed ordering`，因为这不能保证任何顺序性。
对于`atomic`的`memory ordering`理解起来相对复杂一些，可以参考如下两个链接：
《[Memory model synchronization modes](http://gcc.gnu.org/wiki/Atomic/GCCMM/AtomicSync)》
《[std::memory_order](http://en.cppreference.com/w/cpp/atomic/memory_order)》
简单（简陋）的总结一下：
- `sequential consistency`保证了所有原子操作在所有线程之间是顺序一致的。可以把一个`sequential consistency`的原子操作看做一个`fence`，在做乱序优化的时候，前后的指令不可以跨越`fence`。同时，保证不同现成可以看到这些原子操作是一致的顺序。
- `acquire release ordering`可以看做是单向的`fence`，在做乱序优化的时候，标记`acquire`的原子操作之后的指令不可以被移动到`acquire`操作之前；`release`原子操作之前的指令不可以被移动到`release`操作之后。需要注意的是，不同线程看到的原子操作之间的顺序可能是不一致的（可能是由于某些Processor的cache还没写入内存等原因）。
- `relaxed ordering`仅仅保证操作的原子性，通常只用在单纯需要原子性的计数变量。
- 还有一种`acquire ordering`的变化`consume ordering`，仅限制依赖于`consume ordering`原子操作取到的变量的相关指令不能移动到`consume ordering`操作之前，对其他指令没有限制。