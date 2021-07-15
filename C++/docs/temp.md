|                         类/命令空间                          |                             函数                             |                            返回值                            |
| :----------------------------------------------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
|                  `std::this_thread`命名空间                  |     `sleep_for(duration)`<br />`sleep_until(time_point)`     |                              无                              |
| `std::condition_variable`或<br />`std::confition_variable_any` | `wait_for(lock,duration)`<br />`wait_until(lock,time_point)` | `std::cv_status::time_out`<br />或`std::cv_status::no_timeout` |
|                                                              | `wait_for(lock,duration,predicate)`<br />`wait_util(lock,time_point,predicate)` |                  `bool`——唤醒时谓词的返回值                  |
|    `std::timed_mutex`或<br />`std::recursive_timed_mutex`    |  `try_lock_for(duration)`<br />`try_lock_until(time_point)`  |                      `bool`——是否获得锁                      |
|              `std::unique_lock<TimedLockable>`               | `unique_lock(lockable,duration)`<br />`unique_lock(lockable,time_point)` |                              无                              |
|                                                              |  `try_lock_for(duration)`<br />`try_lock_until(time_point)`  |                          是否获得锁                          |
| `std::future<ValueType>`或<br />`std::shared_future<ValueType>` |      `wait_for(duration)`<br />`wait_until(time_point)`      | `std::future_status::timeout`<br />`std::future_status::ready`<br />`std::future_status::dferred` |



这个阅读笔记必须围绕如下几个问题：

- 什么是内存模型？什么是共享内存模型？什么是消息传递模型？[《C++0x漫谈》系列之：多线程内存模型](https://blog.csdn.net/pongba/article/details/1659952)
- 为什么我们需要内存模型？[《C++0x漫谈》系列之：多线程内存模型](https://blog.csdn.net/pongba/article/details/1659952)
- 为什么说C++11之前的内存模型是单线程的，无法保证多线程程序的正确执行？[《C++0x漫谈》系列之：多线程内存模型](https://blog.csdn.net/pongba/article/details/1659952)
- 编译器和CPU对我们的程序做了什么？①编译器按照单线程内存模型对程序进行优化；②CPU使用了[乱序执行技术](https://en.wikipedia.org/wiki/Out-of-order_execution)，这么做的动机非常自然，CPU要尽量塞满每个cycle，在单位时间内运行尽量多的指令。[百度brpc文档-atomic_instructions](https://github.com/apache/incubator-brpc/blob/master/docs/cn/atomic_instructions.md)
- 如果我们显式的组织编译器的优化是否真的能够保证我们的多线程程序达到我们想要的目的？可能不行，一方面这个不行指的是即使阻止编译器对多线程程序的不良优化，也未必能保证CPU在执行我们的多线程程序时能够正确的按照我们想要的顺序执行程序并达到想要的结果（违反SC原则）；其次如果我们禁止了编译器对程序的优化，那么所有的操作严格按照代码顺序执行，所有的操作都触发*[cache coherence](http://en.wikipedia.org/wiki/Cache_coherence)*操作以确保它们的副作用在跨线程间的*visibility*顺序（也就说单单依靠硬件提供的缓存一致性所带来的性能损耗过大）。[《C++0x漫谈》系列之：多线程内存模型](https://blog.csdn.net/pongba/article/details/1659952)
- 我为什么会觉得程序执行的可见性在多个不同的线程之家是非常重要的？甚至可以说它是引出顺序一致性问题的关键！因为各个处于不同CPU核上的线程必须知道其他线程在某个关键共享变量上的执行顺序。[《C++0x漫谈》系列之：多线程内存模型](https://blog.csdn.net/pongba/article/details/1659952)
- 顺序一致性是保证我们多线程程序的正确执行的核心！[为什么程序员需要关心顺序一致性（Sequential Consistency）而不是Cache一致性（Cache Coherence？）](http://www.parallellabs.com/2010/03/06/why-should-programmer-care-about-sequential-consistency-rather-than-cache-coherence/)
- 究竟如何才能允许用户编写正确的多线程代码呢？[《C++0x漫谈》系列之：多线程内存模型](https://blog.csdn.net/pongba/article/details/1659952)
- 怎么理解缓存一致性问题和顺序一致性问题？[为什么程序员需要关心顺序一致性（Sequential Consistency）而不是Cache一致性（Cache Coherence？）](http://www.parallellabs.com/2010/03/06/why-should-programmer-care-about-sequential-consistency-rather-than-cache-coherence/)
- C++11做出了哪些改进？[官方文档memory_order](https://zh.cppreference.com/w/cpp/atomic/memory_order)
- *Acquire*语意是说所有下方的操作都不能往其上方移动，*Release*语意则相反。[百度brpc文档-atomic_instructions](https://github.com/apache/incubator-brpc/blob/master/docs/cn/atomic_instructions.md)
- “与之同步“关系和顺序一致性有着何种关联？”与之同步“关系在C++内存模型中有着何种地位？
- C++ memory consistency model 标准从语义上讲定义的是**原子变量操作之间内存可见性的顺序,** 并没有保证单个原子变量操作的内存**立即可见性.**[C++ memory consistency model 本质的理解](https://zhuanlan.zhihu.com/p/158932307)
- 顺序----执行顺序对于内存一致性模型的重要性，正是即使是原子操作的执行顺序也能够在不同的线程中有着不同的可见性？
- 释放语义和内存获取语义只保障在两个线程之间有效，使得载入操作所在的线程能够见到另一个线程写入该原子变量之前的所有内存访问操作，但这并不保证其他的线程的可见性？
- wait-free和lock-free算法的含义（它们都是用来描述一个并发算法，不是用来描述它有没有用到锁，当然前提肯定是使用了无锁操作，即原子操作）[对wait-free和lock-free的理解](https://zhuanlan.zhihu.com/p/342921323)



百度brpc的文档对获取acquire和释放release语义的描述其实非常好，我该怎么重新组织这部分的语言来描述C++11的内存模型呢？

A happens-before B表示的是程序正如我们所愿正确顺序执行的语义，而不是代表在实际多线程程序中执行时操作A真的在操作B之前完成！
