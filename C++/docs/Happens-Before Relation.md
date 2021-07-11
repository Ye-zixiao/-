# 1. [Happens-Before关系](https://preshing.com/20130702/the-happens-before-relation/)

我可以确信这个关系的名字很容易造成一些误解。我们值得沥青它的概念：Happens-Before关系如上所定义的，它与“A真的发生在B之前”这个概念不相同。需要特别指出的是：

1. A happens-before B 并不意味着“A真的发生于B之前”
2. 而“A真的发生于B之前”也并不意味着A happens-before B

上面这两个表述可能让人觉得诡异，但它们确实是这样的。我将在下面几节中解释之。**记住，*happens-before*是两个操作之间的形式化关系，由一系列语言规范所定义，与时间上的概念相互独立。**这与我们通常所说的（在现实世界中的时间顺序）“A 发生在 B 之前”的概念不同。通过这篇博客，我将小心翼翼地将前一个术语happens-before，以便将其与后者区分开来。

## *happens-before*关系并不意味着“真的发生于前”

在第一个例子中，作者举了编译器重排序指令造成的实际程序执行时A并没有发生在B之前，但这个重排序确确实实在单线程程序上得到了我们编写代码时所期望的结果，那么我们并不认为这个“A没有真正的发生在B之前”违反了“A happens-before B”的关系。

## “真的发生于前”也并不意味着*happens-before*关系

在第二个例子中，作者举了两个不使用任何C++严格的内存序语义的赋值、读取操作的“真的A发生在B之前”说明了A和B并不存在所谓的happens-before关系，从而告诉我们：只有当语言标准所它们存在happens-before关系的时候才真的存在happens-before关系？！

## 总结

虽然happens-before的概念与happening-before的概念有所区别，但实际上我们理解时并不需要做太多可以的记忆，更甚之我们可以将其等同（A happens-before B真的就是指A发生于B之前）。以为这个关系是语言上提出的概念，我们只要认为它确确实实达到了我们所期望的结果就可以了。

我们只要知道：***happens-before*关系是一个线程看到另一个线程所执行的内存读写操作的保证，即线程之间的内存读写可见性保证。**这一关系的存在使得线程1的操作A若*happens-before* 线程2的操作B，那么就说明了操作A之前的内存读写对于线程2而言是可见的（前提是操作A和其之前的内存操作是存在*happens-before*关系，当然这是显而易见的）。



# 2. [Synchronizes-With关系](https://preshing.com/20130823/the-synchronizes-with-relation/)

synchronizes-with关系强调的是变量被修改之后的传播关系（propagate），即如果一个线程修改某变量的之后的结果能被其它线程可见，那么就是满足synchronizes-with关系的。

## 一个写释放操作能与一个读获取操作同步

![img](https://preshing.com/images/two-cones.png)



## *Synchronizes-With*是一个运行时关系

> In the previous example, we saw how the last line of `SendTestMessage` *synchronized-with* the first line of `TryReceiveMessage`. **But don’t fall into the trap of thinking that *synchronizes-with* is a relationship between statements in your source code. It isn’t! It’s a relationship between operations which occur at runtime, based on those statements.**
>
> It all depends on whether the read-acquire sees the value written by the write-release, or not. **That’s what the C++11 standard means when it says that atomic operation B must “take its value” from atomic operation A.**
>
> C++标准指出原子操作B与原子操作A有着“与之同步”关系前提是原子操作B必须从原子操作A写入的变量中获取其值。

![img](https://preshing.com/images/no-cones.png)



## 其他达到*Synchronizes-With*的途径

![img](https://preshing.com/images/org-chart.png)

> 注意这个Mintomic是博客原作者写的一个原子操作库。



# 3. [获取与释放语义](https://preshing.com/20120913/acquire-and-release-semantics/)

> **Acquire semantics** *is a property that can only apply to operations that* **read** *from shared memory, whether they are* [read-modify-write](http://preshing.com/20120612/an-introduction-to-lock-free-programming#atomic-rmw) *operations or plain loads. The operation is then considered a* **read-acquire***. Acquire semantics prevent memory reordering of the read-acquire with any read or write operation that* **follows** *it in program order.*
>
> 获取语义是一种只能应用于共享内存读操作的属性。不管它是读-修改-写还是普通的加载操作，这些操作都被认为是读-获取。获取语义防止了读-获取操作之后的任何读/写操作的重排序（指排到获取操作之前）。

![img](https://preshing.com/images/read-acquire.png)

> **Release semantics** *is a property that can only apply to operations that* **write** *to shared memory, whether they are read-modify-write operations or plain stores. The operation is then considered a* **write-release***. Release semantics prevent memory reordering of the write-release with any read or write operation that* **precedes** *it in program order.*

![img](https://preshing.com/images/write-release.png)

![img](https://preshing.com/images/platform-fences.png)

在这个例子中作者说：

> If we let both threads run and find that `r1 == 1`, that serves as confirmation that the value of `A` assigned in Thread 1 was passed successfully to Thread 2. As such, we are guaranteed that `r2 == 42`. In my previous post, I already [gave a lengthy analogy](http://preshing.com/20120710/memory-barriers-are-like-source-control-operations) for `#LoadLoad` and `#StoreStore` to illustrate how this works, so I won’t rehash that explanation here.
>
> In formal terms, we say that the store to `Ready` ***synchronized-with*** the load. 

因为线程1中的释放 -写操作前的内存读写操作不可能重排序到后面，从而保证了先前的内存读写操作与释放写操作形成了happens-before的关系。而同时获取-读操作之后的内存读写操作不能重排序到它的前面，所以这两者也构成了happens-before关系。如果这两个程序一起运行，并且获取-读操作读到了1，那么我们就可以说这两个运行时的操作存在*synchronized-with*关系，而该关系又可以推出线程间的*happens-before*关系！

## 使用显式平台特定的屏障指令



## 使用C++11中可移植的屏障



## 不使用C++11中的可移植屏障实现释放-获取语义

需要指出的是`std::atomic_thread_fench`的约束性比使用`std::memory_order_release`和`std::memory_order_acquire`更强。

## 使用锁达到的获取-释放语义



# 4. [原子操作中的消费语义](https://preshing.com/20140709/the-purpose-of-memory_order_consume-in-cpp11/)

> Both consume and acquire serve the same purpose: To help pass [non-atomic](http://preshing.com/20130618/atomic-vs-non-atomic-operations) information safely between threads. And just like acquire operations, a consume operation must be combined with a release operation in another thread. The main difference is that there are fewer cases where consume operations are legal. In return for the inconvenience, consume operations are meant to be more efficient on some platforms. I’ll illustrate all of these points using an example.

## 快速回顾获取-释放语义



## 获取语义的代价



## 数据依赖顺序

> Now, I’ve said that PowerPC and ARM are weakly-ordered CPUs, but in fact, there are some cases where they *do* enforce memory ordering at the machine instruction level without the need for explicit memory barrier instructions. Specifically, these processors always preserve memory ordering between data-dependent instructions.

刚进入作者就开始描述了指令之间存在的数据依赖关系对指令重排序的影响，在有些弱内序平台上即使不使用内存屏障指令也可以保证它们之间的内存序不会发生改变（如我们所期望的有序）。

> Because there is a **data dependency** between these two instructions, the loads will be performed in-order.
>
> You may think that’s obvious. After all, how can the second instruction know which address to load from before the first instruction loads `r9`? Obviously, it can’t. Keep in mind, though, that it’s also possible for the load instructions to read from different cache lines. If another CPU core is modifying memory concurrently, and the second instruction’s cache line is not as up-to-date as the first, that would result in memory reordering, too! PowerPC goes the extra mile to avoid that, keeping each cache line fresh enough to ensure data dependency ordering is always preserved.

![img](https://preshing.com/images/data-dependency-1.png)

作者又指出CPU（原文指的是它，我想是CPU吧）是如何知道指令之间存在着数据依赖关系？有人觉得这不是显而易见的吗？因为第二条指令不从第一条指令写入到的寄存器中加载数据还能从哪里加载数据，所以存在着数据依赖关系，但作者指出实际上第二条指令可能在有些情况下从cache行中加载，所以数据依赖关系不能仅仅直观的认为。不过这个数据依赖关系确实是存在的，同时这些PowerPC CPU也会通过一些额外的努力（总是维持cache行中的数据最新）来保证数据依赖次序。

同时作者还指出不仅通过寄存器可以来知晓指令之间的数据依赖关系，而且通过读写同一个内存单元也可以推导出存在的数据依赖关系。如下：

![img](https://preshing.com/images/data-dependency-2.png)

如果多条指令之间存在多个连续的数据依赖关系，那么我们称它们形成了一个数据依赖链。如下：

![img](https://preshing.com/images/data-dependency-3.png)

> **Data dependency ordering guarantees that all memory accesses performed *along a single chain* will be performed in-order.** For example, in the above listing, memory ordering between the first blue load and last blue load will be preserved, and memory ordering between the first green load and last green load will be preserved. **On the other hand, *no* guarantees are made about independent chains!** So, the first blue load could still effectively happen after any of the green loads.
>
> 那么**数据依赖次序会保证所有通过单一（依赖）串链组成的内存访问指令会得到有序的执行**。但作者又指出**具有相互独立的数据依赖关系的指令之间的有序性是不被保证的**（也就是说上面蓝色的具有数据依赖关系的指令之间的有序执行会得到保证，但蓝色的指令s和绿色的指令s之间的执行有序性是不能被保证的有可能是绿色的先执行，蓝色的后执行，也有可能是蓝色的先执行，绿色的后执行，甚至有可能交错执行）。
>
> 同时作者指出大多数的CPU体系架构都会遵守上述的规则，但也有少部分的体系架构并不这样，更不用说是那些强序CPU体系架构，例如x86。



## 数据依赖语义旨在利用这一点

而C++11中的`memory_order_consume`内存序标志正是在利用上述的规则：**数据依赖关系能够保证（在绝大部分的CPU体系架构上）拥有单一数据依赖串联的内存访问指令会得到有序的执行，但这并不保证具有相互独立关系的数据依赖串链的内存访问指令之间的执行有序性。**

> When you use consume semantics, you’re basically trying to make the compiler exploit data dependencies on all those processor families. That’s why, in general, it’s not enough to simply change `memory_order_acquire` to `memory_order_consume`. You must also make sure there are data dependency chains at the C++ source code level.

![img](https://preshing.com/images/source-code-machine-dep-levels.png)

> At the source code level, a dependency chain is a sequence of expressions whose evaluations all *carry-a-dependency* to each another. ***Carries-a-dependency*** is defined in [§1.10.9](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2012/n3337.pdf) of the C++11 standard. **For the most part, it just says that one evaluation *carries-a-dependency* to another if the value of the first is used as an operand of the second.** It’s kind of like the language-level version of a machine-level data dependency. (There is actually a strict set of conditions for what constitutes *carrying-a-dependency* in C++11 and what does not, but I won’t go into the details here.)
>
> 作者指出“**带有依赖于**”关系是C++11标准中所引入的术语，它表示如果第一个表达式的求值结果是第二个表达式的操作数，那么两者就存在一种“带有依赖于”的关系。它可以认为是机器级别的数据依赖关系在高级语言层面的体现（翻译有改动）。这一条件的建立在C++11语言标准上有着严格的规定，但这里并没有指出。

下面的例子展示了C++11释放消费语义体现的数据依赖关系对跨线程间数据读写访问有序的可见性的影响（例子我稍微改变了下）：

```cpp
atomic<int *> g_guard(nullptr);
int g_value = 0;

void write_value() {
  ...
  g_value = 42;
  g_guard.store(&g_value, std::memory_order_release);
}

void read_value() {
  int *p = nullptr, ignore;
  // 使用指针是关键，因为它保证了下面两行代码之间数据依赖关系的成立，
  // 这样就可以保证相关内存访问指令的执行有序性
  while (!(p = g_guard.load(std::memory_order_consume)));
  ignore = *p;
}
```

> Now, this modified example works every bit as reliably as the original example. Once the asynchronous task writes to `Guard`, and the main thread reads it, the C++11 standard guarantees that `p` will equal 42, no matter what platform we run it on. The difference is that, this time, we don’t have a *synchronizes-with* relationship anywhere. What we have this time is called a ***dependency-ordered-before*** relationship.
>
> 作者指出在这两个线程的的释放-消费操作之间并不存在“与之同步”的关系，构成的关系被称为“**依赖先序**”关系~~（我的感觉是C++11设置这样关系是旨在在两个不同的线程之间建立起数据依赖的关系）~~。
>
> **In any *dependency-ordered-before* relationship, there’s a dependency chain starting at the consume operation, and all memory operations performed before the write-release are guaranteed to be visible to that chain.**
>
> （上述这句话非常重要）作者指出：在任一”依赖先序“关系之中，都有一个从消费操作开始的数据依赖串链，对它们而言所有在另一个线程中的释放-写操作之前的内存操作都保证对它们可见（如果没有数据依赖关系，那么这个可见性是无法保证的！）。如下图所示：

![img](https://preshing.com/images/dependency-ordered-guard.png)

当然这种机制对于x86这种强内存体系架构而言是没有什么用的，因为即使不这样做它也能保证相关加载操作后面指令的有序执行。



## 消费语义的价值所在

C++11的消费语义在x86中起始没有什么影响。对于PowerPC、ARMv7这样的处理器的好处在于它可以避免为了实现内存访问指令的有序性而不得不加入像内存屏障这样的指令（而内存屏障指令众所周知，开销是非常大的）。下图展示了消费语义在这些CPU平台上性能的提升：

![img](https://preshing.com/images/consume-timings.png)

若我们所见这种好处主要是带给那些弱内存序但又支持数据依赖关系不会造成内存访问指令重排序的CPU体系架构。

> One real-world example of a codebase that uses this technique – exploiting data dependency ordering to avoid memory barriers – is the Linux kernel. Linux provides an implementation of [read-copy-update (RCU)](http://lwn.net/Articles/262464/), which is suitable for building data structures that are read frequently from multiple threads, but modified infrequently. As of this writing, however, Linux doesn’t actually use C++11 (or C11) consume semantics to eliminate those memory barriers. Instead, it relies on its own API and [conventions](https://www.kernel.org/doc/Documentation/RCU/rcu_dereference.txt). Indeed, RCU served as [motivation for adding consume semantics](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2008/n2664.htm) to C++11 in the first place.
>
> 作者还指出数据依赖语义在Linux上应用所带来的好处，这也是C++11将消费语义引入的一个动机。



## 消费语义缺乏编译器的支持

> I have a confession to make. Those assembly code listings I just showed you for PowerPC and ARMv7? Those were **fabricated**. Sorry, but GCC 4.8.3 and Clang 4.6 don’t actually generate that machine code for consume operations! I know, it’s a little disappointing. But the goal of this post was to show you the *purpose* of `memory_order_consume`. Unfortunately, the reality is that today’s compilers do not yet play along.
>
> 接着，作者指出上面的描述实际上是C++11标准或者相关人员设计消费语义的憧憬，他们在设计初衷非常好，但实际上不得不承认消费语义可能并没有被编译器得到有效的支持（作者撰写的时间为2013年，正是C++11刚发布的头几年，我不知道现在的编译器是否支持了这一语义！）。

最后作者指出两种编译器实现消费语义的策略，一种就是在那些弱序但又遵循数据依赖关系规则的平台上的实现策略——"outputs a machine-level dependency chain for each source-level dependency chain that begins at a consume operation."；另一种是在那些不准许数据依赖关系的平台上的实现策略——重量级策略，即像`memory_order_acquire`内存序那样对待`memory_order_consume`，且完全忽略这些数据依赖关系（而在作者所处的那个年份gcc和clang采用的正是这种重量级的策略）。

