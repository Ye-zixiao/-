## 七. 并发API

### 条款35：有效考虑基于任务的编程而非基于线程的编程



### 条款36：如果有异步的必要请指定`std::launch::async`



### 条款37：使`std::thread`在所有路径最后都不可结合

每个`std::thread`线程对象都有可能处于两种情况之一：可结合状态（joinable）和分离状态（detachable或被称为不可结合状态）。处于可结合状态的`std::thread`对象必然有一个与之对应的异步执行线程，而处于分离状态的`std::thread`对象与之相反，它不存在与之关联的执行线程。这些不可结合的`std::thread`对象包括：①默认构造的`std::thread`对象；②已被移动的`std::thread`对象；③已被`join`的`std::thread`对象；④已被`detach`的`std::thread`对象（`detach()`成员函数可以断开`std::thread`对象原先关联的执行线程）。

可结合性如此重要的原因在于：当一个可结合`std::thread`对象的析构函数被调用时，其内部的调用的`terminate()`函数会终止整个程序的运行。因此我们**必须保证在当前函数执行完成并准备离开当前作用域之前必须在所有的路径上使得`std::thread`对象都变得不可结合**。

标准委员会之所以对`std::thread`的析构函数做出如此规定是因为**隐式的执行`join()`函数还是隐式的执行`detach()`函数都可能对程序的表现造成一些让人困惑的影响**。前者会使得当前函数在离开作用于之前必须等待其底层的异步执行线程完成，而后者会使得当前的函数离开作用域之后自动销毁的局部变量可能对异步线程的正常执行造成一些影响（因为我们在初始化线程时可能使用了lambda表达式，并且引用了一个局部变量）。如下例所示：

```cpp
void doSomeWorks() {
    std::vector<int> vec;
   	std::thread t([&vec] {
				        ...
    			  });
    // 由于函数中需要使用到线程底层的api，所以我们不得不使用
    // std::thread，而不是基于任务的期物
    auto nh = t.native_handle();
    ...
        
    // 如果thread对象的析构函数隐式的调用detach()，那么该线程
    // 就会导致异步执行的子线程空悬引用一个已被销毁的局部变量
}
```

因此**最佳的实现方式就是使用RAII技术来保证`std::thread`对象的自动析构，并且能够使得我们在初始化RAII对象的时候就指定线程对象的符合预期的析构方式**。如下面的`ThreadRAII`对象的实现所示：

```cpp
class ThreadRAII {
 public:
  enum class DtorAction { kJoin, kDetach };

  ThreadRAII(std::thread &&t, DtorAction da)
      : action_(da), thread_(std::move(t)) {}

  ThreadRAII(ThreadRAII &&) = default;
  ThreadRAII &operator=(ThreadRAII &&) = default;

  ~ThreadRAII() {
    if (thread_.joinable()) {
      if (action_ == DtorAction::kJoin)
        thread_.join();
      else
        thread_.detach();
    }
  }

  std::thread &get() { return thread_; }

 private:
  DtorAction action_;
  std::thread thread_;
};
```



对于这一条款，我们**需要记住**：

- 在所有路径上保证`thread`最终是不可结合的。
- 析构时`join`会导致难以调试的表现异常问题。
- 析构时`detach`会导致难以调试的未定义行为。
- 声明类数据成员时，最后声明`std::thread`对象。



### 条款38：关注不同线程句柄的析构行为

>  这个条款实际上是在关注`std::future`的析构行为。

我们知道线程对象`std::thread`和期物`std::future`都可以视作系统线程的句柄：`std::thread`必然有一个系统线程在底层为其支撑，但期物`std::future`可能有可能没有，只有在使用`std::async()`并指定任务不延迟的情况下期物才会有一个底层系统线程为其支撑。这也导致两者迥异的析构行为：

1. 对于一个线程对象`std::thread`，若可结合`joinable`，则其析构函数会隐式的调用`terminate()`函数来终止程序的运行；否则做一些简单的处理。

2. 对于一个期物`std::future`而言，若其关联的异步任务是延迟的，那么其底层就不会有一个系统线程去做支撑，此时期物的析构实际上不做什么；但若异步任务是非延迟的，那么其底层会有一个系统线程做支撑，那么此时最后一个引用之的期物的析构会做一个隐式的`join()`操作。

上述的说法是从底层支撑线程与否的角度来说明期物的析构行为，但这种说法并不完全准确。实际上**期物`std::future/std::shared_future`的析构函数所表现的行为取决于与期物相关的共享状态。**

![item38_fig2](image/item38_fig2.png)

其中共享状态指的是期物所关联的异步任务存放结果的地方，它独立于与被调用者关联的对象（指期物）和与调用者关联的对象（指承诺、打包任务、线程对象），通常它是一个基于堆的对象。期物和与之相关的异步任务都需要对这个共享状态进行引用，前者从中读取结果，后者将执行的结果放入到其中。因此，准确区分期物析构行为的规则如下：

- **如果异步任务是通过`std::async()`启动的未延迟任务，那么引用共享状态的最后一个期物会对相关的异步线程隐式的调用`join()`，等待线程执行完毕。**

  也就是说这个期物有如下三个特点：①关联到`std::async()`创建出来的共享状态；②任务的启动策略是`std::launch::async`（说明它底层有一个系统线程在支撑）；③该期物是最后一个引用共享状态的期物。

- **如果不是上述的情况，那么期物的析构函数实际上就仅仅销毁自己而已。**

但问题在于C++期物并不支持查看当前期物是否引用了一个`std::async`创建来的共享状态的API，因此我们无法确定一个给定的期物的析构是否会出现异常的行为。一种显式控制这些行为的方法就是使用打包任务+自建异步线程的方式来完成，如下所示：

```cpp
{
    std::packaged_task<void()> pt(aTask);
    auto fut = pt.get_future();
    std::thread t(std::move(pt));
    ...
    t.join();
    // fut的析构函数执行时不会出现隐式调用join阻塞等待的现象
}
```

当有一个关联了`std::packaged_task`创建的共享状态的*future*时，其析构函数中就不会出现一些乱七八糟的异常行为，因为通常我们可以在代码中做终止、结合或分离这些动作，来操作`std::packaged_task`运行所在的那个`std::thread`。



对于这一条款，我们需要记住：

- *future*的正常析构行为就是销毁*future*本身的数据成员。
- <font color=green>**引用了共享状态**</font>——**使用`std::async`启动的未延迟任务建立的那个**——<font color=green>**的最后一个*future*的析构函数会阻塞住，直到任务完成。**</font>



### 条款39：对于一次性事件通信考虑使用`void`的*futures*

