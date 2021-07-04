# C++并发编程

## 四. 同步并发操作

### 4.1 条件变量`condition_variable`

等待由另一个线程触发一个事件的最基本机制就是条件变量。**从概念上说，条件变量与某些事件或其他条件相关，并且一个或多个线程可以等待该条件被满足。当某个线程已经确定条件得到满足，它就可以通知一个或多个正在条件变量上等待的线程，以便唤醒它们。**

在C++11上主要有两种条件变量的实现：

- `std::condition_variable`

- `std::condition_variable_any`

其中`std::condition_variable_any`是`std::condition_variable`的泛化，它能在任何满足[*基本可锁定* *(BasicLockable)* ](https://zh.cppreference.com/w/cpp/named_req/BasicLockable)要求的锁上工作。不过，实际中用的最多的还是`std::condition_variable`，它的接口描述如下所示：

![Snipaste_2021-07-03_15-58-20](image/Snipaste_2021-07-03_15-58-20.png)



### 4.2 期物`future`

条件变量非常适合像生产者-消费者模式这样的需要不断循环等待-处理事件的场景，但有些情况下我们实际上并不需要等待多次事件，对于那些一次性事件的等待，使用期物`std::future`（有时也被称为期望、期值）更为合适。

期物允许我们将一个异步的任务交给一个背景线程去处理（也可能就是同步的在当前线程执行），但并不等待它立即执行完毕，而是立即返回继续执行别的事务，只有当调用`get()`或`wait()`接口的时候才会去显式地等待这个任务的完成（就绪）。而**期物本身实际上就是对这个异步任务的一种关联，且当异步任务完成后，期物提供了一种可以访问其返回值的能力。**

在C++11中有两类期物，如下所示：

- 唯一期物`std::future<>`：该期物的实例时仅有的一个指向其关联事件的实例。
- 共享期物`std::shared_future<>`：多个共享期物可以指向同一个事件，当任务完成后，所有期物都变为就绪，并且都可以访问与该事件相关联的数据。

虽然`std::future`被用于线程间通信，但实际上它本身是需要其他的同步机制如`std::mutex`来进行保护，不过共享期物没有这个问题。另外需要注意的是`std::xxx-future<>`这一特例化是用于无关联数据的场合，正常的期物都是有一个与之相关的返回数据。

我们知道期物本身并不是异步任务或者异步执行流，它只是对另一个异步任务的关联抽象，我们可以从异步任务中创建出期物`std::future`。它的创建方式主要有如下三种：

1. 通过`std::async`创建异步后台任务，它会返回一个期物`std::future`；
2. 通过`std::packaged_task<>`将一个期物`std::future`绑定到一个函数或可调用对象；
3. 通过`std::promise`来创建一个承诺，它可以返回一个与之关联的期物。

<img src="image/期物.png" alt="期物" style="zoom:67%;" />



#### 4.2.1 通过`async`创建期物

在不需要立刻得到结果的时候，我们可以使用函数`std::async()`来启动一个异步任务，它会返回一个期物对象，而不是让用户在一个线程上等待，最终期物对象会持有函数的返回值。当需要这个值的时候，只需要在期物对象上调用`get()`成员函数，当前线程就可以阻塞到期物就绪为止然后返回该值。

至于系统是否会为这个异步任务创建一个背景线程取决于传递进去的`std::launch`类型标记：

- 如果传递给`std::async()`一个`std::launch::deferred`，那么系统并不会创建一个背景线程来执行这个异步任务，而是延迟到当前线程调用`get()`的时候同步执行。
- 如果传递给`std::async()`一个`std::launch::async`，那么系统会创建一个背景线程来执行这个异步任务。
- 如果传递给`std::async()`前两者的或（即默认情况），那么有具体实现来选择。

如下展示上述方法的使用：

```cpp
void doAsyncWork() {
  auto fut = std::async(std::launch::async,       // 异步执行一个打印字符串任务
                        [](std::string_view sv) {
                          cout << sv << endl;
                        }, "hello world");
  ...
  fut.wait();
}
```



#### 4.2.2 通过`packaged_task`将期物与可调用对象绑定

`std::packaged_task`将一个期物`std::future`绑定到一个函数或可调用对象上，我们可以通过`get_future()`接口来获取其相关联的期物。当这个打包任务对象被调用时，它就调用相关联的函数或可调用对象，并让期物就绪，同时将返回值作为关联数据存储。显然这种东西可能对于线程池的构建有一定的帮助。

作为示例，我们可以通过`std::packaged_task`+`std::thread`来实现将一个异步任务放到一个背景线程中执行，然后返回一个期物给当前的线程，从而达到类似`std::async()`的效果：

```cpp
template<typename Func, typename... Args>
decltype(auto) doAsyncWork(Func &&f, Args...args) {
  using result_type = std::result_of_t<Func(Args &&...)>;

  // 打包一个异步任务
  std::packaged_task<result_type(Args &&...)> task(std::forward<Func>(f));
  // 获取打包任务相关联的期物
  std::future<result_type> fut(task.get_future());
  // 将异步任务放到一个背景线程中执行
  std::thread backupThread(std::forward<Func>(f), std::forward<Args>(args)...);
  backupThread.detach();
  return fut;
}
```

> 其中上面的`std::result_of_t<>`的作用就是来获取一个可调用对象的返回类型。



#### 4.2.3 通过`promise`来创建期物

`std::promise`之所以会有这个名字，是因为它承诺自己会将一个有效的结果存放到它所指向的共享状态之中。显然，它是`promise-future`交流通道的“推”端，而期物`std::future`是“拉”端，从而构成了线程间通信的一种有效机制。在线程间数据交换的“推”端，承诺`std::promise`可以通过`set_value()`的方式来存放结果；而在“拉”端，`std::future`可以通过`get()`来获取这个结果。如下示例：

```cpp
int main() {
  std::vector<int> ivec{1, 2, 3, 4, 5, 6};
  std::promise<int> accPromise;
  auto fut = accPromise.get_future();

  std::thread t([](auto begin, auto end, std::promise<int> accPro) {
                  auto sum = std::accumulate(begin, end, 0);
                  accPro.set_value(sum);
                },
                ivec.begin(), ivec.end(), std::move(accPromise));

  cout << "accumulate result: " << fut.get() << endl;
  t.join();
  return 0;
}
```



#### 4.2.4 为期物保存异常

如果在期物`std::future`关联的异步任务中发生了异常，那么异常将会自动被存储在期物之中或者通过诸如`set_exception()`这样的接口显式存放。然后期物将会变为就绪状态，并且在调用`get()`的时候重新抛出这个存储的异常。



#### 4.2.5 等待自多个线程

对于唯一期物`std::future`而言，若多个线程同时访问之却不使用任何同步措施，那么就会出现数据竞争和未定义行为。为了解决这个问题，我们可以使用共享期物`std::shared_future`来解决这个问题，它可复制，共同引用相同的数据，且线程安全。其使用方法可以查看[cppreference的示例](https://zh.cppreference.com/w/cpp/thread/shared_future)。



### 4.3 有时间限制的等待

#### 4.3.1 时钟`xxx_clock`



#### 4.3.2 时间段`duration`



#### 4.3.3 时间点`time_point`



#### 4.3.4 线程库中接受超时的函数



## 五. 原子操作和内存模型

### 5.1 原子类型与操作

#### 5.1.1 原子类型`atomic`



#### 5.1.2 原子标志`atomic_flag`





### 5.2 同步操作与强制顺序

#### 5.2.1 初识C++内存模型

相关参考资料：

1. [为什么程序员需要关心顺序一致性（Sequential Consistency）而不是Cache一致性（Cache Coherence？）](http://www.parallellabs.com/2010/03/06/why-should-programmer-care-about-sequential-consistency-rather-than-cache-coherence/)
2. [《C++0x漫谈》系列之：多线程内存模型](https://blog.csdn.net/pongba/article/details/1659952)
3. [如何理解 C++11 的六种 memory order？](https://www.zhihu.com/question/24301047)
4. [The Purpose of memory_order_consume in C++11](https://preshing.com/20140709/the-purpose-of-memory_order_consume-in-cpp11/)
5. [The Synchronizes-With Relation](https://preshing.com/20130823/the-synchronizes-with-relation/)
6. [Acquire and Release Semantics](https://preshing.com/20120913/acquire-and-release-semantics/)
7. [百度brpc文档-atomic_instructions](https://github.com/apache/incubator-brpc/blob/master/docs/cn/atomic_instructions.md)
8. [C++11中的内存模型下篇 - C++11支持的几种内存模型](https://www.codedump.info/post/20191214-cxx11-memory-model-2/#%E4%BF%AE%E6%94%B9%E5%8E%86%E5%8F%B2)
9. [C++11中的内存模型上篇 - 内存模型基础](https://www.codedump.info/post/20191214-cxx11-memory-model-1/)





##### 5.2.1.1 内存模型基础

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
- C++ memory consistency model 标准从语义上讲定义的是**原子变量操作之间内存可见性的顺序,** 并没有保证单个原子变量操作的内存**立即可见性.**[](https://zhuanlan.zhihu.com/p/158932307)
- 顺序----执行顺序对于内存一致性模型的重要性，正是即使是原子操作的执行顺序也能够在不同的线程中有着不同的可见性？
- 释放语义和内存获取语义只保障在两个线程之间有效，使得载入操作所在的线程能够见到另一个线程写入该原子变量之前的所有内存访问操作，但这并不保证其他的线程的可见性？



百度brpc的文档对获取acquire和释放release语义的描述其实非常好，我该怎么重新组织这部分的语言来描述C++11的内存模型呢？





#### 5.2.2 关系术语

##### 5.2.2.1 happens-before



##### 5.2.2.2 synchronizes-with



#### 5.2.3 原子操作的内存顺序



<img src="https://cdn.jsdelivr.net/gh/lichuang/lichuang.github.io/media/imgs/20191214-cxx11-memory-model-2/c++model.png" alt="c++model" style="zoom:67%;" />

##### 5.2.3.1 顺序一致序

```cpp
std::atomic<bool> x, y;
std::atomic<int> z;

void write_x() {
  x.store(true, std::memory_order_seq_cst);
}

void write_y() {
  y.store(true, std::memory_order_seq_cst);
}

void read_x_then_y() {
  while (!x.load(std::memory_order_seq_cst));
  if (y.load(std::memory_order_seq_cst))
    ++z;
}

void read_y_then_x() {
  while (!y.load(std::memory_order_seq_cst));
  if (x.load(std::memory_order_seq_cst))
    ++z;
}

int main() {
  x = y = false;
  z = 0;
  thread t1(write_x);
  thread t2(write_y);
  thread t3(read_x_then_y);
  thread t4(read_y_then_x);
  t1.join();
  t2.join();
  t3.join();
  t4.join();
  cout << z << endl;
  return 0;
}
```





##### 5.2.3.2 松散序

```cpp
std::atomic<bool> x(false), y(false);
std::atomic<int> z = 0;

void write_x_then_y() {
  x.store(true, std::memory_order_relaxed);
  y.store(true, std::memory_order_relaxed);
}

void read_y_then_x() {
  while (!y.load(std::memory_order_relaxed));
  if (x.load(std::memory_order_relaxed))
    ++z;
}

int main() {
  thread t1(write_x_then_y);
  thread t2(read_y_then_x);
  t1.join();
  t2.join();
  cout << z << endl;
  return 0;
}
```



```cpp
atomic<int> x(0), y(0), z(0);
atomic<bool> go(false);

constexpr int g_loopCount = 10;

struct read_values {
  int x, y, z;
};

read_values values1[g_loopCount];
read_values values2[g_loopCount];
read_values values3[g_loopCount];
read_values values4[g_loopCount];
read_values values5[g_loopCount];

void increment(atomic<int> *var_to_inc, read_values *values) {
  while (!go)
    this_thread::yield();
  for (size_t i = 0; i < g_loopCount; ++i) {
    values[i].x = x.load(memory_order_relaxed);
    values[i].y = y.load(memory_order_relaxed);
    values[i].z = z.load(memory_order_relaxed);
    var_to_inc->store(i + 1, std::memory_order_relaxed);
    this_thread::yield();
  }
}

void read_vals(read_values *values) {
  while (!go)
    this_thread::yield();
  for (size_t i = 0; i < g_loopCount; ++i) {
    values[i].x = x.load(memory_order_relaxed);
    values[i].y = y.load(memory_order_relaxed);
    values[i].z = z.load(memory_order_relaxed);
    std::this_thread::yield();
  }
}

void print(read_values *v) {
  for (size_t i = 0; i < g_loopCount; ++i) {
    if (i) cout << ", ";
    cout << "(" << v[i].x << ", " << v[i].y << ", " << v[i].z << ")";
  }
  cout << endl;
}

int main() {
  thread t1(increment, &x, values1);
  thread t2(increment, &y, values2);
  thread t3(increment, &z, values3);
  thread t4(read_vals, values4);
  thread t5(read_vals, values5);

  go = true;

  t5.join();
  t4.join();
  t3.join();
  t2.join();
  t1.join();

  print(values1);
  print(values2);
  print(values3);
  print(values4);
  print(values5);
  return 0;
}
```





##### 5.2.3.3 获取-释放序

```cpp
atomic<bool> x(false), y(false);
atomic<int> z(0);

void write_x_then_y() {
  x.store(true, std::memory_order_relaxed);
  y.store(true, std::memory_order_release);
}

void read_y_then_x() {
  while (!y.load(std::memory_order_acquire));
  if (x.load(std::memory_order_relaxed))
    ++z;
}

int main() {
  thread t1(read_y_then_x);
  thread t2(write_x_then_y);
  t1.join();
  t2.join();
  cout << z << endl;
  return 0;
}
```



```cpp
atomic<int> g_flag = 0;
atomic<int> g_data[5];

void threadFunc1() {
  g_data[0].store(42, std::memory_order_relaxed);
  g_data[1].store(97, std::memory_order_relaxed);
  g_data[2].store(17, std::memory_order_relaxed);
  g_data[3].store(-141, std::memory_order_relaxed);
  g_data[4].store(203, std::memory_order_relaxed);
  g_flag.store(1, std::memory_order_release);
}

void threadFunc2() {
  int expected = 1;
  while (!g_flag.compare_exchange_strong(expected, 2, std::memory_order_acq_rel))
    expected = 1;
}

void threadFunc3() {
  while (!(g_flag.load(std::memory_order_acquire) == 2));
  assert(g_data[0].load(memory_order_relaxed) == 42);
  assert(g_data[1].load(memory_order_relaxed) == 97);
  assert(g_data[2].load(memory_order_relaxed) == 17);
  assert(g_data[3].load(memory_order_relaxed) == -141);
  assert(g_data[4].load(memory_order_relaxed) == 203);
}

int main() {
  thread t1(threadFunc1);
  thread t2(threadFunc2);
  thread t3(threadFunc3);
  t1.join();
  t2.join();
  t3.join();
  return 0;
}
```



##### 5.2.3.4 消耗序

```cpp
struct X {
  string str_;
  int i_;
};

atomic<X *> g_pX(nullptr);
atomic<int> g_a(0);

void createX() {
  X *x = new X{"hello world", 32};
  g_a.store(99, std::memory_order_relaxed);
  g_pX.store(x, std::memory_order_release);
}

void consumeX() {
  X *x = nullptr;
  while (!(x = g_pX.load(std::memory_order_consume)));
  assert(x->str_ == "hello world");
  assert(x->i_ == 32);
  delete x;
}

int main() {
  thread t1(createX);
  thread t2(consumeX);
  t1.join();
  t2.join();
  return 0;
}
```



#### 5.2.4 释放序列

```cpp
std::vector<int> queueData;
std::atomic<int> count;

void populate_queue() {
  constexpr int number_of_items = 100;
  for (int i = 0; i < number_of_items; ++i)
    queueData.push_back(i);
  count.store(number_of_items, std::memory_order_release);
}

void consume_queue_items() {
  while (true) {
    int item_idx;
    if ((item_idx = count.fetch_sub(1, std::memory_order_acquire)) <= 0) {
      this_thread::sleep_for(1ms);
      continue;
    }
    cout << this_thread::get_id() << ": "
         << queueData[item_idx - 1] << endl;
  }
}

int main() {
  thread t1(populate_queue);
  thread t2(consume_queue_items);
  thread t3(consume_queue_items);
  t1.join();
  t2.join();
  t3.join();
  return 0;
}
```



#### 5.2.5 内存屏障

```cpp
bool x = false;
atomic<bool> y(false);
atomic<int> z(0);

void write_x_then_y() {
  x = true;
  std::atomic_thread_fence(std::memory_order_release);
  y.store(true, std::memory_order_release);
}

void read_y_then_x() {
  while (!y.load(std::memory_order_acquire));
  if (x) ++z;
}

int main() {
  thread t1(write_x_then_y);
  thread t2(read_y_then_x);
  t1.join();
  t2.join();
  assert(z != 0);
  return 0;
}
```





参考文献：

1. https://preshing.com/20140709/the-purpose-of-memory_order_consume-in-cpp11/
2. https://preshing.com/20130823/the-synchronizes-with-relation/
