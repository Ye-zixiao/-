



|                         类/命令空间                          |                             函数                             |                            返回值                            |
| :----------------------------------------------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
|                  `std::this_thread`命名空间                  |     `sleep_for(duration)`<br />`sleep_until(time_point)`     |                              无                              |
| `std::condition_variable`或<br />`std::confition_variable_any` | `wait_for(lock,duration)`<br />`wait_until(lock,time_point)` | `std::cv_status::time_out`<br />或`std::cv_status::no_timeout` |
|                                                              | `wait_for(lock,duration,predicate)`<br />`wait_util(lock,time_point,predicate)` |                  `bool`——唤醒时谓词的返回值                  |
|    `std::timed_mutex`或<br />`std::recursive_timed_mutex`    |  `try_lock_for(duration)`<br />`try_lock_until(time_point)`  |                      `bool`——是否获得锁                      |
|              `std::unique_lock<TimedLockable>`               | `unique_lock(lockable,duration)`<br />`unique_lock(lockable,time_point)` |                              无                              |
|                                                              |  `try_lock_for(duration)`<br />`try_lock_until(time_point)`  |                          是否获得锁                          |
| `std::future<ValueType>`或<br />`std::shared_future<ValueType>` |      `wait_for(duration)`<br />`wait_until(time_point)`      | `std::future_status::timeout`<br />`std::future_status::ready`<br />`std::future_status::dferred` |





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

