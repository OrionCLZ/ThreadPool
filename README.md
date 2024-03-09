# ThreadPool
C++11变参模板编程实现线程池
---
title: 项目总结-基于C++11可变参模板实现的线程池
tags:

---

# 线程池项目

**环境：**vs2019开发、C++17标准；centos7编译so动态库



## 一.项目介绍

作为五大池之一 (内存池、连接池、线程池、进程池、协程池)，线程池的应用非常广泛，不管是客户端程序，还是后台服务程序，都是提高业务处理能力的必备模块。有很多开源的线程池实现，虽然各自接口使用上稍有区别，但是其核心实现原理都是基本相同的。



## 二.技术背景

+ 熟练基于C++ 11标准的面向对象编程

  组合和继承、继承多态、STL容器、智能指针、函数对象、绑定器、可变参模板编程等。

+ 熟悉C++11多线程编程

​        thread、mutex、atomic、condition_variable、unique_lock等。

+ C++17和C++20标准的内容

​       C++17的any类型和C++20的信号量semaphore, 项目上的代码实现。

+ 熟悉多线程理论

​       多线程基本知识、线程互斥、线程同步、原子操作、CAS等。



## 三.并发和并行

+ CPU单核
+ CPU多核、多CPU

### 并发

单核上，多个线程占用不同的CPU时间片，物理上还是串行执行的，但是由于每个线程占用的CPU时间片非常短（比如10ms），看起来就像是多个线程都在共同执行一样，这样的场景称作并发
（concurrent）。

### 并行

在多核或者多CPU上，多个线程是在真正的同时执行，这样的场景称作并行（parallel）。



## 四.多线程的优势

多线程程序一定就好吗？不一定，要看具体的应用场景：

程序是IO密集型？  程序里面指令的执行，涉及一些IO操作，比如设备、文件、网络操作（等待客户端的连接 IO操作是可以把程序阻塞住的，再分配给这样程序CPU时间片，CPU相当于空闲下来了）

程序是CPU密集型？  程序里面的指令都是做计算用的

### IO密集型

无论是CPU单核、CPU多核、多CPU，都是比较适合多线程程序的

### CPU密集型

+ CPU单核 （不适合设计成多线程程序）

  多线程存在上下文切换，是额外的花销，线程越多上下文切换所花费的额外时间也越多，倒不如一个线程一直进行计算。

+ CPU多核、多CPU

  多个线程可以并行执行，对CPU利用率好。

  

## 五.线程池

### 线程的消耗

为了完成任务，创建很多的线程可以吗？线程真的是越多越好？

+ 线程的创建和销毁都是非常"重"的操作
+ 线程栈本身占用大量内存
+ 线程的上下文切换要占用大量时间
+ 大量线程同时唤醒会使系统经常出现锯齿状负载或者瞬间负载量很大导致宕机

### 线程池的优势

​       操作系统上创建线程和销毁线程都是很"重"的操作，耗时耗性能都比较多，那么在服务执行的过程中， 如果业务量比较大，实时的去创建线程、执行业务、业务完成后销毁线程，那么会导致系统的实时性能 降低，业务的处理能力也会降低。 

​       线程池的优势就是（每个池都有自己的优势），在服务进程启动之初，就事先创建好线程池里面的线 程，当业务流量到来时需要分配线程，直接从线程池中获取一个空闲线程执行task任务即可，task执行 完成后，也不用释放线程，而是把线程归还到线程池中继续给后续的task提供服务。

#### fixed模式线程池

线程池里面的线程个数是固定不变的，一般是ThreadPool创建时根据当前机器的CPU核心数量进行指定。

#### cached模式线程池

线程池里面的线程个数是可动态增长的，根据任务的数量动态的增加线程的数量，但是会设置一个线程 数量的阈值（线程过多的坏处上面已经讲过了），任务处理完成，如果动态增长的线程空闲了60s还没 有处理其它任务，那么关闭线程，保持池中最初数量的线程即可。

### 线程同步

#### 1.线程互斥

能不能在多线程环境下执行？？？ 看这段代码是否存在竟态条件 ->称作临界区代码段 ===》保证它的原子操作。

如果在多线程环境是不存在竟态条件的==》 可重入的

​                                       存在竟态条件  ==》不可重入的！

竟态条件：代码片段在多线程环境下执行，随着线程的调度顺序不同，而得到不同的运行结果！

+ 1.1互斥锁mutex

+ 1.2atomic原子类型  CAS操作（无锁机制）无锁队列、无锁链表、无锁数组

##### 1.1互斥锁mutex

在C++中，`std::mutex`是用于多线程同步的一个关键工具，它代表了一种互斥量（Mutex），用来保护共享资源免受多个线程同时访问的影响。当你有多条线程可能同时访问同一段临界区代码或同一个全局变量时，可以使用`std::mutex`来确保在同一时刻只有一个线程能够进入临界区并对资源进行操作。

以下是`std::mutex`的一些基本用法：

######  1.1.1 定义和初始化mute:

```cpp
#include <mutex>
std::mutex mtx; // 默认构造一个互斥锁，初始状态为未锁定
// 或者显式初始化
std::mutex mtx2(std::defer_lock); // 初始化为延迟锁定状态
```

###### 1.1.2 加锁和解锁

```cpp
void someFunction() {
    std::lock_guard<std::mutex> lock(mtx); // 使用RAII方式自动锁定和解锁
    // 进入临界区，此处的代码仅在一个线程中执行
    // ...
} // 当离开这个作用域时，lock_guard会自动释放锁

// 或者手动管理锁
void anotherFunction() {
    mtx.lock(); // 手动锁定
    try {
        // 进入临界区
        // ...
    } catch (...) {
        // 异常处理，记得解锁
    } finally {
        mtx.unlock(); // 手动解锁
    }
}
```

###### 1.1.3 使用`std::unique_lock`进行更灵活的锁定

```cpp
void flexibleLocking() {
    std::unique_lock<std::mutex> lock(mtx);
    if (someCondition()) {
        lock.unlock(); // 解锁以便其他线程可以进入
        doSomethingElse();
        lock.lock(); // 再次锁定
    }
    // 继续在临界区工作
}
```

###### 1.1.4 阻塞和超时锁定

```cpp
bool timedLockAttempt() {
    std::chrono::milliseconds timeout(1000); // 设置超时时间为1秒
    std::unique_lock<std::mutex> lock(mtx, std::try_to_lock); // 尝试非阻塞获取锁
    if (!lock.owns_lock()) { // 没有获取到锁
        if (!mtx.try_lock_for(timeout)) { // 阻塞尝试获取锁，最多等待指定时间
            return false; // 获取锁失败，超时
        }
    }
    // 已经获得锁，执行临界区代码
    // ...
    return true;
}
```

###### 1.1.5. 注意事项

- `std::mutex`不支持线程间的递归锁定，试图在已经持有锁的线程中再次锁定同一个互斥量会导致死锁。如果需要递归锁定，应使用`std::recursive_mutex`。
- 在多线程程序中，必须小心处理锁定和解锁操作，确保每次锁定都有对应的解锁，否则可能出现死锁或资源泄露。
- 使用`std::lock_guard`或`std::unique_lock`这样的RAII（Resource Acquisition Is Initialization）机制可以简化代码并确保在抛出异常时也能正确解锁。

###### 1.1.6. 示例代码片段

```cpp
#include <iostream>
#include <thread>
#include <mutex>
std::mutex mtx;
int count = 0;

void incrementCounter() {
    std::lock_guard<std::mutex> guard(mtx);
    count++;
    std::cout << "Thread " << std::this_thread::get_id() << " incremented count to " << count << std::endl;
}

int main() {
    std::vector<std::thread> threads;
    const int NUM_THREADS = 10;

    for (int i = 0; i < NUM_THREADS; ++i) {
        threads.emplace_back(incrementCounter);
    }

    // 等待所有线程完成
    for (auto& t : threads) {
        t.join();
    }
    std::cout << "Final count value: " << count << std::endl;
    return 0;
}
```

上述代码创建了10个线程，每个线程都会锁定`mtx`然后递增全局变量`count`。由于`std::lock_guard`的存在，可以确保在任何时候只有一个线程在修改`count`，从而避免了竞态条件。

**lock_guard和unique_lock的用法**

`std::unique_lock` 和 `std::lock_guard` 都是用来管理互斥量（mutex）的对象，它们都是C++标准库中的类模板，用于自动管理互斥锁的生命周期，以防止在程序中忘记解锁互斥量造成死锁。下面分别举例说明它们的作用和使用。

>**std::lock_guard**
>
>`std::lock_guard` 是一种非常简单的 RAII（Resource Acquisition Is Initialization）工具，它在构造时自动锁定互斥量，在析构时自动解锁互斥量。这种特性确保了只要 lock_guard 对象存在，互斥量就会保持锁定状态，而且在任何情况下（包括异常抛出）都能正确解锁互斥量。
>
>```cpp
>#include <mutex>
>#include <iostream>
>std::mutex mtx;
>void printWithLock()
>{
>std::lock_guard<std::mutex> lock(mtx); // 构造时自动锁定互斥量
>std::cout << "Critical section accessed by thread " << std::this_thread::get_id() << std::endl; // 临界区代码
>} // lock_guard析构时自动解锁互斥量
>
>int main()
>{
>std::thread t1(printWithLock);
>std::thread t2(printWithLock);
>
>t1.join();
>t2.join();
>
>return 0;
>}
>```
>
>`printWithLock` 函数内的代码段是临界区，两个线程通过创建 `std::lock_guard` 对象来保证同一时间内只有一个线程能进入并执行这段代码。
>
>**std::unique_lock**
>
>`std::unique_lock` 提供了更多的灵活性，它可以**手动锁定和解锁互斥量，可以选择是否在构造时立即锁定，也可以在后续的代码中决定何时锁定和解锁。**此外，它还提供了 `try_lock` 和 `try_lock_for` 等方法，可以尝试锁定而不阻塞线程，以及尝试在一定时间内锁定互斥量。
>
>```cpp
>#include <mutex>
>#include <iostream>
>#include <thread>
>#include <chrono>
>
>std::mutex mtx;
>
>void worker(bool &shouldContinue)
>{
>std::unique_lock<std::mutex> lock(mtx, std::defer_lock); // 不立即锁定，而是延迟锁定
>
>while (shouldContinue)
>{
>   if (lock.try_lock()) // 尝试非阻塞地锁定互斥量
>   {
>       std::cout << "Thread " << std::this_thread::get_id() << " has the lock.\n";
>       std::this_thread::sleep_for(std::chrono::seconds(1)); // 模拟工作
>       lock.unlock(); // 手动解锁互斥量
>   }
>   else
>   {
>       std::this_thread::yield(); // 如果没能获取到锁，则让出CPU
>   }
>}
>}
>
>int main()
>{
>bool shouldContinue = true;
>std::thread t1(worker, std::ref(shouldContinue));
>std::thread t2(worker, std::ref(shouldContinue));
>
>std::this_thread::sleep_for(std::chrono::seconds(5)); // 主线程等待一段时间
>shouldContinue = false; // 改变条件，使工作线程退出循环
>
>t1.join();
>t2.join();
>
>return 0;
>}
>```
>
>`worker` 函数使用 `std::unique_lock` 并设置为延迟锁定。在循环中尝试获取锁，如果获取成功则执行任务并手动解锁，否则让出处理器时间片。当外部条件变化时，线程停止尝试获取锁并退出循环。

#### 2.线程通信

+ 条件变量 condition_variable

+ 信号量 semaphore（C++20）

  可以使用C++11条件变量实现信号量

##### 2.1 条件变量

​	C++中的条件变量是一种线程同步机制，它主要用于**解决线程间通信**的问题，特别是在一个多线程环境里，**当一个线程需要等待某个特定条件满足后再继续执行时**，条件变量就派上了用场。条件变量不能单独使用，它总是配合互斥量（通常是`std::mutex`）一起使用，以确保线程安全性和同步的有效性。

>条件变量的核心概念和功能如下：
>
>1. **等待条件**：
>
>  - 线程通过调用条件变量的`wait()`方法来挂起自己，直到满足特定的条件。在调用`wait()`之前，线程必须已获得互斥锁，这样就能保证在检查和等待条件的过程中，不会发生数据竞争。
>  - `wait()`会自动释放互斥锁，让其他线程有机会修改共享数据，进而可能满足等待的条件。
>
>2. **触发条件**：
>
>  - 另一个线程在满足条件后，可以调用条件变量的`notify_one()`或`notify_all()`方法来唤醒一个或所有正在等待该条件变量的线程。
>  - 唤醒并不立即传递控制权给等待的线程，而是让等待的线程回到`wait()`函数内部，这时它会重新获取互斥锁并检查条件是否仍然满足。如果满足，则继续执行；如果不满足，可能会再次进入等待状态。
>
>3. **限时等待**：
>
>  - C++11标准还提供了`wait_for()`和`wait_until()`方法，允许线程在等待条件满足的同时设定一个超时时间，超过这个时间后，即使条件没有满足也会返回，避免无限期地等待。
>
>4. **使用模式**：
>
>  - 使用条件变量的典型模式包括：
>    - 线程首先锁定互斥锁。
>    - 检查条件是否满足，如果不满足，则调用`wait()`函数释放互斥锁并等待。
>    - 当其他线程改变了条件并调用了通知函数后，等待的线程被唤醒，重新获取互斥锁并检查条件，如果条件满足则继续执行，否则可能再次进入等待。
>
>5. **成员函数**：
>
>  - ```
>    std::condition_variable
>    ```
>
>    类的主要成员函数包括：
>
>    - `wait()`：挂起当前线程并释放互斥锁。
>    - `wait(lock)`：接受一个`std::unique_lock<std::mutex>`作为参数，同样挂起线程并释放锁。
>    - `wait(lock, predicate)`：除了等待外，还会在唤醒后检查一个谓词（一个返回布尔值的函数对象），只有当谓词返回`true`时，线程才会继续执行。
>    - `notify_one()`：唤醒一个正在等待的线程。
>    - `notify_all()`：唤醒所有正在等待的线程。
>    - 通过以上机制，条件变量使得线程能够在复杂同步场景中更加高效地协作，避免了无效的轮询和不必要的阻塞。

##### 2.2 信号量

​	在C++中，信号量是一种线程同步机制，它**用来控制有限资源的访问或限制同时执行特定任务的线程数量**。信号量维护一个整数值，该值可以增加（信号/发布）或减少（等待/获取），并且可以用来阻塞线程直到特定条件达成。

C++标准库直到C++20才正式包含了信号量的原生支持，通过`std::semaphore`类实现。在此之前，开发者需要依赖第三方库或操作系统API来实现信号量功能。C++20中引入的`std::semaphore`有两种形式：计数信号量（`std::counting_semaphore`）和二进制信号量（`std::binary_semaphore`）。

>**计数信号量（std::counting_semaphore）**
>
>- 计数信号量有一个非负整数计数器，它表示可用资源的数量。
>- 初始化时指定一个初始计数值。
>- `acquire()`（相当于P操作）：当信号量的计数值大于0时，调用此函数会使计数值减1，并允许线程继续执行；若计数值为0，则线程会被阻塞，直到其他线程调用`release()`函数增加了计数值。
>- `release()`（相当于V操作）：增加信号量的计数值，如果至少有一个线程正阻塞在这个信号量上，那么将唤醒一个等待的线程。
>
>**二进制信号量（std::binary_semaphore）**
>
>- 二进制信号量类似于互斥量，但它没有所有权的概念，只有两种状态：有信号（计数值为1）和无信号（计数值为0）。
>- 同样具有`acquire()`和`release()`操作，但计数值只能是0或1。
>
>**示例用法（C++20之后）：**
>
>```cpp
>#include <semaphore>
>
>std::counting_semaphore<5> semaphore; // 初始化一个最多允许5个线程同时访问的信号量
>
>void threadFunction() {
>   semaphore.acquire(); // 请求资源，如果资源不足则阻塞
>   // ... 进行临界区操作，访问共享资源 ...
>   semaphore.release(); // 释放资源，允许其他线程进入
>}
>
>int main() {
>   std::vector<std::jthread> threads;
>   for (int i = 0; i < 10; ++i) {
>       threads.emplace_back(threadFunction);
>   }
>
>   for (auto& t : threads) {
>       t.join();
>   }
>
>   return 0;
>}
>在C++11及以前版本中，由于标准库并未提供信号量，开发者通常借助于条件变量、互斥量或者其他第三方库模拟信号量的行为。而在C++20以后，可以直接使用内置的信号量类实现更为简洁和直观的同步控制。
>```



## 六.项目设计

![ ](../img/QQ截图20240220172157.png)

### 1.threadpool.h接口

>```cpp
>//一.线程池类型
>class ThreadPool
>{
>public:
>private:
>};
>//二.线程类型
>class Thread
>{
>public:
>private:  
>};
>
>//三.线程池支持的模式
>//支持两种线程模式
>enum class PoolMode   
>{
>	MODE_FIXED, //固定数量的线程
>	MODE_CACHED,//线程数量可动态增长
>};
>
>//四.任务抽象基类
>// 用户可以自定义任意任务类型，从task继承，重写run方法，实现自定义任务处理
>class Task
>{
>public:
>private:
>};
>```

### 2.ThreadPool类设计

>```cpp
>class ThreadPool
>{
>public:
>	//线程池构造
>	ThreadPool( );
>	//线程池析构
>	~ThreadPool( );
>
>	//2.  设置线城池的工作模式
>	void setMode(PoolMode mode);
>
>	//3.   设置task任务队列上限的阈值
>	void setTaskQueMaxThreshold(int threshold);
>
>	//设置线程池catched模式下线程阈值
>	void setThreadSizeThreshHold(int threshold);
>
>	//4.   给线程池提交任务
>	Result submitTask(std::shared_ptr<Task>sp);//用户可能会传入生命周期比较短的任务,使用智能指针封装任务
>
>	//1.  开启线程池
>	void start(int initThreadSize = 4);
>
>	ThreadPool(const ThreadPool&) = delete;//禁止对象拷贝构造功能
>	ThreadPool& operator=(const ThreadPool&) = delete;//禁止对象赋值操作符
>
>
>private:
>	//定义线程函数
>	void threadFunc(int threadid);
>
>	//检查pool的运行状态
>	bool checkRunningState()const;
>
>private: //linux下 -std=c++11
>	//1.std::vector<std::unique_ptr<Thread>>threads_;//线程列表
>	std::unordered_map<int, std::unique_ptr<Thread>>threads_;  //线程列表
>
>	size_t initThreadSize_;//2.初始的线程数量    size_t无符号整型
>	std::atomic_int curThreadSize_;//4.记录当前线程池里面线程的总数量 不是线程安全的,使用互斥锁太重,故使用原子类型
>	int threadSizeThreshHold_;//线程数量上限阈值
>	std::atomic_int idleThreadSize_;//记录空闲线程的数量
>
>	std::queue<std::shared_ptr<Task>>taskQue_;//3.任务队列
>	std::atomic_int taskSize_;//5.任务数量， 原子操作保证内部任务调度与外部任务增加
>	int taskQueMaxThreshold_;//6.任务队列数量上限阈值
>
>	std::mutex taskQueMtx_;//7.保证任务队列的线程安全
>	std::condition_variable notFull_;//8.表示任务队列不满
>	std::condition_variable notEmpty_;//9.表示任务队列不空
>	std::condition_variable exitCond_;//等待线程资源全部回收
>
>	PoolMode poolMode_;//当前线程池的工作模式
>	std::atomic_bool isPoolrunning_;//表示当前线程池的启动状态  多个线程中使用可能会发生线程安全问题
>
>};
>```

### 3.ThreadPool方法接口实现(线程池方法实现)

#### 3.1线程池方法实现

>```cpp
>#include "threadpool.h"
>#include<functional>
>#include<thread>
>#include<iostream>
>const int TASK_MAX_THRESHOLD = 1024;
>const int THREAD_MAX_THRESHOLD = 10;
>const int THREAD_MAX_IDLE_TIME = 60; // 单位s
>
>//一、线程池方法实现
>//1. 线程池构造
>ThreadPool::ThreadPool()
>: initThreadSize_(0)  
>, taskSize_(0)
>,idleThreadSize_(0)
>,curThreadSize_(0)
>, taskQueMaxThreshold_(TASK_MAX_THRESHOLD)  //（项目中除了0 1不能出现其他数字，其他数字用有意义的名字表示 ）
>,threadSizeThreshHold_(THREAD_MAX_THRESHOLD)
>,poolMode_(PoolMode::MODE_FIXED)
>,isPoolrunning_(false)
>{ }
>
>//2. 线程池析构     C++中只要出现了构造一定要有析构
>ThreadPool::~ThreadPool()
>{ 
>isPoolrunning_ = false;
>notEmpty_.notify_all();
>//等待线程池里面所有的线程返回  有两种状态： 阻塞 & 正在执行任务中
>std::unique_lock<std::mutex>lock(taskQueMtx_);
>exitCond_.wait(lock, [&]()->bool {return threads_.size() == 0; });
>}
>
>//3. 设置线城池的工作模式
>void ThreadPool::setMode(PoolMode mode)
>{
>if (checkRunningState())
>{
>   return;
>}
>poolMode_ = mode;
>}
>
>//4. 设置task任务队列上线的阈值
>void ThreadPool::setTaskQueMaxThreshold(int threshold)
>{
>if (checkRunningState())
>{
>   return;
>}
>taskQueMaxThreshold_ = threshold;
>}
>
>//设置线程cached模式下线程阈值
>void ThreadPool::setThreadSizeThreshHold(int threshold)
>{
>if (checkRunningState())
>{
>   return;
>}
>if (poolMode_ == PoolMode::MODE_CACHED)
>{
>   threadSizeThreshHold_ = threshold;
>}
>
>}
>```

#### 3.2 ThreadPool::start()的实现

线程池ThreadPool::start()开启线程池函数的实现

>```cpp
>//6. 开启线程池
>void ThreadPool::start(int initThreadSize)
>{
>//设置线程池的运行状态
>isPoolrunning_ = true;
>
>//1.记录初始线程个数
>initThreadSize_ = initThreadSize;
>curThreadSize_ = initThreadSize;
>
>//2.创建线程对象
>for (int i = 0; i < initThreadSize_; i++)
>{
>   //创建thread线程对象的时候，把线程函数给到thread线程对象
>   std::unique_ptr<Thread> ptr = std::make_unique<Thread>(std::bind(&ThreadPool::threadFunc, this,              std::placeholders::_1));
>
>   //threads_.emplace_back(new Thread(std::bind(&ThreadPool::threadFunc,this)));
>   //threads_.emplace_back(std::move(ptr));//原本（ptr）但是unique_ptr禁止了拷贝构造，实参ptr转为形参ptr底层会拷贝一份，
>                                           //所以使用move移动语义做资源转移
>   //threads_.emplace_back(ptr)报错原因,实参到形参,threads_尝试拷贝构造一份ptr并插入到线程池中,unique_ptr禁用了拷贝构造和赋值构造
>
>   int threadId = ptr->getId();
>   threads_.emplace(threadId, std::move(ptr));
>}
>
>//启动所有线程 std::vector<Thread*>threads_;
>for (int i = 0; i < initThreadSize_; i++)
>{
>   threads_[i]->start();  //需要去执行一个线程函数  线程函数放在线程类里面无法访问线程池的私有变量，
>                          //1.可以将线程函数定义为全局函数，2.将线程函数定义在线程池类中。
>   idleThreadSize_++;     //记录初始空闲线程的数量
>}
>}
>```

线程池工作需要调用线程池start函数, 线程工作需要调用线程start函数, 线程的start函数需要在线程池中定义, 因为线程池的参数都在线程池类的private作用域中.

线程调用线程池中的线程启动函数start(), 需要在线程创建时候将线程池中的start()函数动态绑定到线程中, 方法是使用function函数和绑定器.

##### **function函数作用?**

在C++中，`std::function` 是C++标准库中的一部分，它是一个泛型函数包装器类模板，主要作用如下：

1. **类型擦除**：`std::function` 可以容纳任何形式的可调用对象，如全局函数、成员函数、Lambda 表达式、仿函数（functor）等，并将其转换成统一的类型。这意味着你可以定义一个`std::function`变量，它能够存储不同类型的可调用对象，提供了类型安全的函数指针类似的功能。
2. **存储和延迟调用**：`std::function` 对象可以存储任意可调用实体，允许你在稍后合适的时间点调用它，这对于实现回调函数、事件处理器等非常有用。
3. **多态性**：由于其能够存储多种类型的可调用对象，所以它可以支持运行时多态，即在运行时确定调用哪个具体的函数。
4. **方便传参和容器化**：`std::function` 对象可以作为函数参数传递，也可以放入容器（如 `std::vector<std::function<...>>`）中，使得程序设计更加灵活和模块化。

>```cpp
>#include <iostream>
>#include <functional>
>
>// 声明一个普通的函数
>void simpleFunction(int x) {
>std::cout << "Called simpleFunction with " << x << std::endl;
>}
>
>// 定义一个类，包含一个成员函数
>class MyClass {
>public:
>void memberFunction(int y) {
>   std::cout << "Called memberFunction with " << y << std::endl;
>}
>};
>
>int main() {
>// 创建一个std::function对象，能接受一个int参数并返回无值
>std::function<void(int)> func;
>
>// 将自由函数赋值给std::function对象
>func = simpleFunction;
>func(10); // 输出 "Called simpleFunction with 10"
>
>// 使用lambda表达式
>func = [](int z) { std::cout << "Called lambda with " << z << std::endl; };
>func(20); // 输出 "Called lambda with 20"
>
>// 将成员函数转换为可调用对象，并通过std::bind绑定this指针
>MyClass obj;
>std::function<void()> boundMemberFunc =       std::bind(&MyClass::memberFunction, &obj, 30);
>boundMemberFunc(); // 输出 "Called memberFunction with 30"
>
>return 0;
>}
>```

上述代码展示了如何使用 `std::function` 存储不同类型和来源的可调用对象，并在需要的时候调用它们。`std::function` 具有延迟计算的特点，非常适合在事件处理、回调机制、策略模式等场景下使用。

>```cpp
>#include <iostream>
>#include <functional>
>
>void printString(const std::string& s) {
>std::cout << "Print: " << s << std::endl;
>}
>
>int main() {
>std::function<void(const std::string&)> printer; // 定义一个可调用对象包装器
>
>// 将全局函数赋值给std::function对象
>printer = printString;
>printer("Hello, World!"); // 调用printString函数
>
>// 使用Lambda表达式
>printer = [](const std::string& s) { std::cout << "Lambda says: " << s << std::endl; };
>printer("Hello from Lambda!");
>
>return 0;
>}
>```
>
>在这个示例中，`std::function<void(const std::string&)>` 可以存储任何接受一个`const std::string&`参数且没有返回值的可调用对象。通过改变赋给它的可调用实体，我们可以动态地改变行为。

##### **智能指针**

C++11标准引入了三种智能指针来协助管理和自动释放动态分配的对象，分别是`std::unique_ptr`、`std::shared_ptr`和`std::weak_ptr`。这些智能指针通过RAII（Resource Acquisition Is Initialization）原则确保了资源在适当的时间得到释放，从而有效防止内存泄漏等问题。

1. **std::unique_ptr**

   - `std::unique_ptr` 是独占所有权的智能指针，同一时刻只有一个`unique_ptr`指向某个动态分配的对象。当`unique_ptr`超出其作用域或被重新赋值时，它会自动释放所指向的对象，确保了内存的有效回收。

   - `unique_ptr` 不支持拷贝构造和赋值操作，只能移动（move），这就意味着所有权不能被复制，只能转移。

   - 示例：

     ```cpp
     std::unique_ptr<MyClass> uptr(new MyClass());
     ```

2. **std::shared_ptr**

   - `std::shared_ptr` 实现了共享所有权，多个`shared_ptr`可以同时指向同一个动态分配的对象。对象的生命周期由所有指向它的`shared_ptr`共同维护，只要至少有一个`shared_ptr`存在，对象就不会被释放。

   - `shared_ptr` 内部使用引用计数机制，每当创建一个新的`shared_ptr`引用同一对象时，引用计数加1；当`shared_ptr`被销毁或赋值给另一个对象时，引用计数减1。当引用计数降为0时，对象会被自动释放。

   - `shared_ptr` 支持拷贝构造、赋值和移动操作。

   - 示例：

     ```cpp
     std::shared_ptr<MyClass> sptr1(new MyClass());
     std::shared_ptr<MyClass> sptr2 = sptr1; // 共享同一对象
     ```

3. **std::weak_ptr**

   - `std::weak_ptr` 是一种不增加引用计数的智能指针，它用来观察由`shared_ptr`管理的对象，但并不参与对象的生命周期管理。`weak_ptr`不能单独使用，必须先通过调用`lock()`方法获取一个临时的`shared_ptr`来访问对象，如果对象已经不存在（所有`shared_ptr`都不再引用它），`lock()`会返回一个空的`shared_ptr`。

   - `weak_ptr` 主要用于解决循环引用问题，防止对象在不再需要时仍然因为循环引用而无法释放。

   - 示例：

     ```cpp
     std::shared_ptr<MyClass> sptr(new MyClass());
     std::weak_ptr<MyClass> wptr(sptr);
     if (std::shared_ptr<MyClass> locked = wptr.lock()) {
         // 对象依然存在，可以安全访问
     } else {
         // 对象已经被释放
     }
     ```

总的来说，C++11智能指针通过自动化的内存管理，极大地提高了程序的安全性和健壮性。开发人员只需关注业务逻辑，而不用过多关心内存的细节。此外，根据实际需求合理选择适合的智能指针类型，能够有效地降低代码出错的可能性，并提高程序性能。

##### make_shared和shared_ptr

`std::make_shared` 和 `std::shared_ptr` 是 C++11 标准库中智能指针部分的相关工具。它们之间的关系密切，但是有着不同的功能和目的：

1. **std::shared_ptr**：

   - `std::shared_ptr` 是一个智能指针类，它实现了引用计数的共享所有权模型。当你创建一个 `shared_ptr` 对象时，它会自动管理所指向对象的生命周期——当最后一个 `shared_ptr` 不再引用该对象时，对象会被自动删除，避免了内存泄漏的问题。

   - 通常情况下，创建一个 `shared_ptr` 需要显式地调用构造函数，传入指向动态分配对象的原始指针：

     ```cpp
     struct MyObject {};
     std::shared_ptr<MyObject> ptr(new MyObject());
     ```

2. **std::make_shared**：

   - `std::make_shared` 是一个工厂函数，用于高效地创建一个 `std::shared_ptr` 实例以及它所指向的对象。它在一个步骤中完成内存分配和智能指针初始化，相较于直接使用 `new` 和 `shared_ptr` 构造函数，`make_shared` 有一些优势：

     - 内存优化：`make_shared` 只做一次内存分配，将智能指针的控制块（管理引用计数等信息）和对象本身分配在同一块连续内存中，降低了内存碎片和额外开销。
     - 一致性和安全性：在多线程环境下，`make_shared` 创建的 `shared_ptr` 对象在初始化过程中更具原子性，减少了潜在的数据竞争问题。
     - 更简洁的语法：它可以直接用于创建并初始化对象，减少了代码量。

     使用 `std::make_shared` 的示例：

     Cpp

     ```cpp
     std::shared_ptr<MyObject> ptr = std::make_shared<MyObject>();
     ```

总的来说，`std::make_shared` 是为了更安全、高效地创建 `std::shared_ptr` 而提供的便捷工具函数，它强化了 `shared_ptr` 的使用体验，并且在性能和内存管理上做了优化。在实际编码中，除非有特殊理由，通常建议优先使用 `std::make_shared` 来创建 `shared_ptr`。

##### 为什么C++11摒弃了auto_ptr?

C++11标准废弃`std::auto_ptr`主要是由于以下几个原因：

1. **所有权转移**： `std::auto_ptr`的设计初衷是提供一个自动管理资源（特别是动态分配内存）的智能指针，但其拷贝和赋值操作会导致所有权的转移而非共享。这意味着每次拷贝或赋值操作都会使原有`auto_ptr`变得无效（指向NULL），这与大多数程序员对普通指针拷贝操作的预期不符，容易引发错误。
2. **不支持容器**： `std::auto_ptr`不遵守STL容器的元素要求，因为它违反了CopyConstructible和Assignable的要求。由于所有权转移的特性，将`auto_ptr`放入容器（如`std::vector`或`std::map`）会导致容器内部元素的状态发生意外变化，从而引起未定义行为甚至内存泄漏。
3. **不支持数组**： `std::auto_ptr`不能用于管理动态分配的数组，因为其析构函数只会调用单个对象的删除器，而不是适用于数组的删除器。
4. **线程安全性**： `std::auto_ptr`并未提供线程安全的引用计数，这意味着在多线程环境中使用它可能会导致竞态条件和未知的行为。
5. **与C++11新特性的不兼容性**： C++11引入了右值引用和移动语义，这两个新特性使得能够更好地设计和实现智能指针。相比之下，`std::auto_ptr`基于旧有的C++98标准设计，未能充分利用这些新特性。

鉴于上述问题，C++11引入了两个新的智能指针类型来取代`std::auto_ptr`：

- `std::unique_ptr`：它同样具有独占所有权，但其设计符合C++11的标准要求，支持移动语义而不支持拷贝，解决了所有权转移带来的问题，并且可以用于管理动态分配的数组。
- `std::shared_ptr`：它支持共享所有权，通过引用计数来决定何时释放资源，可以用于容器，并且在多线程环境下的使用相对安全。

##### 什么是右值引用?

右值引用是C++11引入的新特性，它允许对即将销毁的临时对象进行高效的资源管理。右值引用记作 `T&&`，其中 `T` 是类型名。

在C++中，值可以分为左值（lvalue）和右值（rvalue）两类。左值是可以持久存在的，有名称的，可以出现在赋值操作符左侧的表达式，比如变量。右值则是临时的、无名的、不能取地址的表达式，比如临时对象、函数返回值、字面量等。右值引用就是为了绑定到右值而设计的引用类型。

右值引用的主要特点和用途包括：

1. **移动语义（Move Semantics）**： 当我们将一个右值引用绑定到一个即将销毁的临时对象上时，可以通过移动构造函数或移动赋值运算符将临时对象的资源“偷走”（move），而非进行代价高昂的复制操作。这样既节省了资源，又提高了效率。
2. **完美转发（Perfect Forwarding）**： 在模板编程中，右值引用可以帮助实现完美的函数参数转发，使得函数能够透明地处理其参数无论是左值还是右值的情况，进而设计出更通用和高效的函数模板。
3. **转移所有权**： 在资源管理领域，如智能指针等类库设计中，右值引用可以用来实现资源所有权的高效转移，避免了不必要的拷贝操作。

```cpp
class MyClass {
public:
    MyClass(MyClass&& other) // 移动构造函数
    : data(std::move(other.data)) // 移动资源
    {
        other.data = nullptr; // 清理原对象的资源
    }

private:
    std::unique_ptr<Data> data; // 假设data是一个独占资源
};

MyClass CreateInstance() {
    return MyClass{/* ... */}; // 返回的是一个临时对象，即右值
}

int main() {
        MyClass obj = CreateInstance(); // 使用移动构造函数，高效地转移资源
}
```

在这个例子中，`CreateInstance()` 函数返回的临时对象就是一个右值，而 `MyClass obj = CreateInstance();` 中的 `=` 运算符通过调用移动构造函数，将临时对象的资源转移给了 `obj`，避免了深拷贝。

##### emplace_back和push_back的区别?

`std::vector`容器（以及其他类似容器如`std::deque`等）中的`push_back`和`emplace_back`函数均用于向容器末尾添加元素，但它们在工作方式和效率上有显著区别：

1. **push_back()**：

   - `push_back`函数接受一个元素的副本作为参数，然后将这个副本添加到容器的末尾。

   - 当传递给`push_back`的是一个**用户自定义类型**（非POD类型）时，如果容器内元素不是小型对象优化（SOO）的候选者，通常会发生两次构造过程：首先调用构造函数创建参数副本，然后在容器内部再次调用构造函数（可能是拷贝构造函数或移动构造函数）将副本添加到容器中。

     ```cpp
     std::vector<std::string> vec;
     vec.push_back("Hello, world!"); // 字符串会被拷贝或移动到vec中
     ```

2. **emplace_back()**：

   - `emplace_back`函数接受的是构造元素所需的一组参数，它会直接在容器内部调用元素类型的构造函数来创建元素，而不是先创建一个副本再插入。

   - 使用`emplace_back`可以避免不必要的拷贝或移动操作，尤其当元素构造成本较高或者不支持移动构造时，这种方法更为高效。

     ```cpp
     struct Person {
         std::string name;
         int age;
         Person(const std::string& n, int a) : name(n), age(a) {}
     };
     
     std::vector<Person> people;
     people.emplace_back("Alice", 30); // 直接在容器内部构造Person对象
     ```

**`emplace_back`相比`push_back`更适合于直接构造元素的情况，可以减少临时对象的创建和销毁，提高代码性能**。尤其是在大型结构体、类对象等非基本类型的情况下，利用`emplace_back`可以更好地利用C++11及后续版本引入的完美转发和移动语义等特性。不过，在实际使用中也需要根据具体情况进行权衡，例如对于内置类型或者小对象，两者的效率差异可能不大。



#### 3.3submitTask()的实现

submitTask()函数作用: 外部用户想使用线程池会通过线程池对象pool调用submitTask()函数向线程池提交任务,如pool.submitTask().

线程池里使用继承和多态思想设计了通用抽象基类Task(), 派生类是用户根据基类Task()实现的具体任务, 从而实现提交不同类型任务功能.

>```cpp
>//5. 给线程池提交任务  用户调用该接口，传入用户对象，生产任务
>Result ThreadPool::submitTask(std::shared_ptr<Task>sp)//智能指针指向从基类task派生来的用户提交的具体task对象
>{
>
>//1.获取锁  用户提交任务生产者，线程执行任务消费者，为保证线程安全，使用互斥锁
>std::unique_lock<std::mutex> lock(taskQueMtx_);//抢到锁就可以向任务队列放任务
>
>//2.线程的通信  等待任务队列有空余  
>// wait:一直等lambda函数条件满足 wait_for：有时间参数，最多等一段时间 wait_until：有时间参数，设置了等待节点
>
>/*
>while (taskQue_.size() == taskQueMaxThreshold_)
>{
>notFull_.wait(lock);
>}
>*/
>
>//用户提交爱任务，最长不能阻塞超过1s，否则判断提交任务失败，返回
>//查看任务队列是否满，如果满了，条件变量阻塞住，同时释放锁，不释放锁的话线程无法消费任务。
>if (!notFull_.wait_for(lock, std::chrono::seconds(1),
>[&]()->bool {return taskQue_.size() < (size_t)taskQueMaxThreshold_; }))
>//“&”的作用，lamda函数访问外部成员变量，需要变量捕获 ，引用捕获
>{
>//表示notfull_等待1s钟，条件仍然为满足
>std::cerr << "task queue is full ,submit task fail." << std::endl;
>return Result(sp,false);//Task  Result
>//return task->getResult(); 不能用这种方式返回，线程执行完task，task对象就被析构掉了
>}
>
>//3.如果有空余，把任务放入任务队列中
>taskQue_.emplace(sp);
>taskSize_++;
>
>//4.因为新放了任务，任务队列肯定不为空了，在notEmpty_上进行通知.赶快分配线程执行任务
>notEmpty_.notify_all(); //通知线程池消费
>
>//cached模式 任务处理比较紧急 场景：小而快的任务 需要根据线程数量和空闲线程的数量，判断是否需要创建新的线程出来？
>      //耗时任务多不适合cached模式，因为耗时的任务会长时间占用一个线程，耗时任务比较多会导致线程池创建线程过多，
>          //创建线程过多会对系统性能影响非常大
>if(poolMode_==PoolMode::MODE_CACHED
>&&taskSize_>idleThreadSize_
>&& curThreadSize_<threadSizeThreshHold_)
>{
>std::cout << ">>>> create new thread...."  << "exit!"
>  << std::endl;
>
>//创建新的线程对象
>std::unique_ptr<Thread> ptr = std::make_unique<Thread>(std::bind(&ThreadPool::threadFunc, this, std::placeholders::_1));
>//threads_.emplace_back(std::move(ptr));
>int threadId = ptr->getId();
>threads_.emplace(threadId, std::move(ptr));
>//启动线程
>threads_[threadId]->start(); 
>//修改线程个数相关的变量
>curThreadSize_++;
>idleThreadSize_++;
>}
>
>
>//返回任务的Result对象
>//return task->getResult();
>return Result(sp);
>}
>```

#### 3.4 Any类型 线程池返回任意类型结果

有些任务需要线程执行结束后返回给用户结果, 用户提交的任务不一样相应的会有不同的返回值

如何设计返回值类型? 用模板T? 不能使用模板

>```cpp
>//任务抽象基类
>class Task
>{
>public:
>//用户可以自定义任意任务类型,从Task继承, 重写run方法,实现自定义任务处理
>virtual T run()=0;
>//代码从上向下编译,看到虚函数,会给Task类产生虚函数表,将虚函数的地址记录到虚函数表里,T模板类型,并没有实例化,找不到函数地址.virtual虚函数不能和T函数模板放一起用
>}
>
>```

**如何构建一个Any类型?**

+ 任意的其他类型 **template**
+ 能让一个类型 指向 其他任意的类型呢?  **基类类型指向派生类类型**

![](./img/QQ截图20240306223539.png)

Any => Base*------>  Derive: public Base

1.Any类型的成员是基类指针Base* 因为只有基类指针可以指向不同的派生类对象.

2.让基类指针指向从基类继承来的派生类对象Derive.

3.data直接返回为Any不行,Any是基类指针, 只有将data作为派生类Derive的成员变量.

4.data是什么类型?不知道,因为用户会有不同的任务,需要template模板定义data的类型.

>```cpp
>//run()函数返回值如何实现? 可以表示任意的类型
>//Java Python 都有Object类型 是所有其他类类型的基类
>//C++17 Any类型
>Any run() 
>{
>std::cout << "tid: " << std::this_thread::get_id() << "begin!"<<std::endl;
>//this_thread::sleep_for(chrono::seconds(2));
>ULong sum = 0;
>for (ULong i = begin_; i <= end_; i++)
>{
>sum += i;
>}
>std::cout << "tid: " << std::this_thread::get_id() <<"end!" << std::endl;
>return sum;
>}
>
>```

**Any类型的实现**



>```cpp
>//Any类型:可以接收任意数据的类型
>class Any
>{
>public:
>Any()=default;//默认构造
>~Any()=default;//默认析构
>
>//成员变量data_已经禁止了左值引用的拷贝构造和赋值,当前对象肯定也不能左值引用拷贝构造和赋值
>Any(const Any&)=delete;
>Any& operator=(const Any&)=delete;
>Any(Any&&)=default;//默认右值引用的成员变量拷贝构造
>Any& operator=(Any&&)=default;//默认右值引用的成员变量赋值
>
>//这个构造函数可以让Any接受任意其他类型的数据
>template<typename T>
>Any(T data):base_(std::make_unique<Derive<T>>(data))
>{}//模板的代码，为什么声明不写在头文件，定义写在源文件，而是定义声明都写在头文件中
>	//只能写在头文件中，编译阶段才能实例化，实例化后才能产生真真正正的函数。
>
>//这个方法能把Any对象里面存储的data数据提取出来
>template<typename T>
>T cast_()
>{
>//怎么从base_找到他所指向的Derive对象,从它里面提出data成员变量
>//基类指针=>转成派生类指针  RTTI类型识别 C++四种类型强转
>Derive<T>* pd=dynamic_cast<Derive<T>*>(base_.get());
>if (pd == nullptr)
>		{
>			throw "type is unmatch!";
>		}
>		return pd->data_; 
>}
>
>private:
>//基类类型
>class Base
>{
>public:
>virtual ~Base()=default;
>};
>
>//派生类类型
>template<typename T>
>class Derive:public Base
>{
>public:
>Derive(T data):data(data_)
>{}
>T data_;//保存了任意的其他类型
>}
>
>private:
>//定义一个基类的指针Base *  用于指向派生类对象
>std::unique_ptr<Base>base_;
>//unique_ptr将左值引用的拷贝构造和赋值禁用了,公布了右值引用的拷贝构造和赋值
>}
>```

**Derive<T>* pd=dynamic_cast<Derive<T>*>(base_.get());解释**

C++中智能指针`std::unique_ptr<Base>`类型的对象`base_`进行动态类型转换，并试图将其转换为指向派生类`Derive<T>`类型的智能指针。

`std::unique_ptr<Base>`是一个持有`Base`类类型或其派生类对象的智能指针。`base_.get()`返回的是智能指针内部存储的原始指针。

`dynamic_cast<Derive<T>*>(base_.get())` 这一行代码的含义是：

1. `dynamic_cast`: 这是C++中的动态类型转换操作符，它用于安全地在类层次结构中进行向下转型。如果`base_`所指向的对象实际上是一个`Derive<T>`类型的实例，或者`Derive<T>`是`Base`的派生类，那么这个转换就会成功，并返回一个指向`Derive<T>`对象的指针。
2. `<Derive<T>*>`: 指定转换的目标类型，这里是将`Base`类型的指针转换为`Derive<T>`类型的指针。
3. `base_.get()`: 获取`std::unique_ptr<Base>`所拥有的原始`Base`类型的指针，以便进行类型转换。注意，`get()`方法返回的是裸指针，转换的结果仍需存储回一个智能指针中以保持资源的自动管理。

转换后的结果`Derive<T>* pd`是一个指向`Derive<T>`对象的指针，若转换失败（即`base_`所指向的对象并非`Derive<T>`或其派生类实例），`pd`将为`nullptr`。

但是要注意，如果要将转换后的指针存储回一个智能指针，应该创建一个对应的`std::unique_ptr<Derive<T>>`来接管该指针的生命周期管理，例如：

```cpp
if (Derive<T>* pd = dynamic_cast<Derive<T>*>(base_.get())) {
    std::unique_ptr<Derive<T>> derivedPd(pd); // 正确地接管转换后指针的生命周期管理
    // 现在可以安全地使用derivedPd
} else {
    // 转换失败，base_所指向的对象不是一个Derive<T>实例
}
```


>```cpp
>//问题二：如何设计Result机制呢
>		Result res1 = pool.submitTask(std::make_shared<MyTask>(1, 100000000));
>		res.get();//如果线程还没执行完,用户调用get()函数返回,应该阻塞;
>```

res.get()返回Any类型, 那么如何从Any类型提取用户需要的类型呢?

>```cpp
>int sum = res.get().cast_<int>();//get返回一个Any类型,怎么转成具体的类型呢?
>```

##### C++11四种强制类型转换?

在C++11中，为了增强类型转换的安全性和表达意图的清晰度，引入了四种强制类型转换操作符，分别是`static_cast`、`const_cast`、`dynamic_cast`和`reinterpret_cast`。每种转换都有其特定的应用场景和限制。

1. **static_cast**：

   - 用途：主要用于静态类型转换，包括底层类型之间的转换（如int转double）、指针或引用类型之间的转换（如父类指针转子类指针，前提是存在继承关系并且向下转型是安全的），以及枚举类型与整数类型的相互转换。

     ```cpp
     double d = 3.14;
     int i = static_cast<int>(d); // 将double转换为int，可能会丢失精度
     Base* basePtr = new Derived();
     Derived* derivedPtr = static_cast<Derived*>(basePtr); // 向下转型，仅当basePtr实际指向Derived对象时才安全
     ```

2. **const_cast**：

   - 用途：唯一用于修改表达式中常量属性的转换，它可以去除指针或引用的const、volatile限定符，但并不能改变对象的实际内容。

     ```cpp
     void foo(const int* cptr) {
         // ...
     }
     int val = 10;
     foo(const_cast<int*>(&val)); // 将非const指针转换为const指针，但不能通过此指针修改值
     ```

3. **dynamic_cast**：

   - 用途：用于运行时类型识别和安全的向下转型，适用于类层次结构中的指针和引用。如果试图将一个父类指针转换为子类指针，但实际指向的对象并非子类实例，则转换结果为nullptr（对于指针）或抛出std::bad_cast异常（对于引用）。

     ```cpp
     class Base {};
     class Derived : public Base {};
     
     Base* basePtr = new Derived();
     Derived* derivedPtr = dynamic_cast<Derived*>(basePtr); // 安全向下转型，这里derivedPtr将是有效的
     Base* anotherBasePtr = new Base();
     Derived* failedCast = dynamic_cast<Derived*>(anotherBasePtr); // 这里failedCast将为nullptr
     ```

4. **reinterpret_cast**：

   - 用途：用于低级别的比特级转换，它允许几乎任何指针类型间的转换，以及整数类型和足够大的指针类型之间的转换。这种转换通常用于操纵底层二进制表示，是危险的操作，如果没有充分的理由，一般应避免使用。

     ```cpp
     int i = 12345;
     char* cp = reinterpret_cast<char*>(&i); // 将int指针转换为char指针，操作底层内存
     long* lp = reinterpret_cast<long*>(cp); // 风险操作，将char*转换回long*
     4// 注意：这种转换依赖于底层平台的内存对齐和字节序，通常不适合跨类型数据交换
     
         //C++11的四种强制类型转换各有侧重，开发者应谨慎使用，尽量在确保类型转换安全和有意义的前提下进行转换，避免引入难以察觉的bug和未定义行为。
     ```

#### 3.5 Result及semaphore的实现

应用背景: 

>```cpp
>//问题二：如何设计Result机制呢
>Result res1 = pool.submitTask(std::make_shared<MyTask>(1, 100000000));
>res.get();//如果线程还没执行完,用户调用get()函数返回,应该阻塞;
>```
>
>Result用于接收线程执行完任务的结果, 如果线程还未执行完, 用户调用get()方法应该阻塞, 等任务结束, 通过信号量通知,get()继续执行并返回结果.
>
>信号通信使用到信号量semaphore, 在C++20已经有了定义, 现在自己实现semaphore.

![](./img/QQ截图20240307095157.png)

**重点: 线程的同步**

**线程互斥:** mutex 基于CAS的原子类型atomic 例如data++,data--

mutex: 互斥锁是一个比较重的锁会改变线程的状态

CAS:相当于给总线加锁,以原子操作做了一个寄存器,跟内存的交换,并不会改变线程的状态

在C++中，基于CAS（Compare and Swap）的原子类型主要体现在`std::atomic`模板类中。`std::atomic`是C++11及更高版本引入的标准库的一部分，主要用于在多线程环境下提供原子操作的支持，尤其是针对那些需要无锁同步的场景。

`std::atomic`通过底层硬件指令（如Intel的`cmpxchg`指令或AMD的相应指令）实现了CAS操作，允许程序员在不使用互斥锁的情况下安全地修改共享数据。当一个线程尝试修改`std::atomic`类型的变量时，它会先比较当前值是否与期望值相匹配，如果匹配，则将变量设置为新的值。这一过程是原子的，意味着不会有其他线程可以在比较和交换这两个步骤之间干扰。

CAS在C++中的典型用处包括但不限于：

1. **无锁数据结构**: 可以构建无锁栈、队列或其他数据结构，其中元素的添加、删除或更新操作通过CAS来实现，减少因锁定而导致的线程阻塞和上下文切换。
2. **计数器**: 使用`std::atomic<int>`作为线程安全的计数器，例如统计事件次数、引用计数等，多个线程可以同时递增或递减计数器而无需担心数据竞争。
3. **线程间同步**: 可以实现高效的自旋锁和其他轻量级同步原语，例如原子地改变某个标志位来协调线程间的活动。
4. **避免死锁**: 由于不需要传统的互斥锁，因此可以避免死锁的发生，尤其是在复杂的多线程系统中。

**线程通信:** 条件变量(condition_variable)+信号量(semaphore)



##### 3.5.1实现一个sempaphore类

>```cpp
>calss Semaphore{
>public:
>Semaphore(int limit=0)
>   :resLimit_(limit)
>   {}
>~Semaphore()=default;
>
>//获取一个信号量资源  //操作系统的pv操作
>void wait()
>{
>   std::unique_lock<std::mutex>lock(mtx_);
>   //等待信号量资源没有资源的话,阻塞当前线程
>   cond_wait(lock,[&]()->bool {return resLimit_>0; });
>   resLimit--;
>}
>
>//增加一个信号量资源
>void post()
>{
>   std::unique_lock<std::mutex>lock(mtx_);
>   resLimit_++;
>   cond_.notify_all();
>}
>
>private:
>int resLimit_;
>std::mutux mtx_;
>std::conditon_variable cond_;
>}
>```

##### 3.5.2实现Result

实现接收提交到线程池的task任务执行完成后返回的返回值类型Result;

用户提交任务的线程和任务执行的线程并不是一个线程 , 需要进行线程通信机制

**Result类设计**

>```cpp
>//Task类型的前置声明
>class Task;
>
>//实现接收提交到线程池的task任务执行完成后返回的返回值类型Result;
>class Result{    
>public:
>
>Result(std::shared_ptr<Task>task,bool isValid=true);
>~Reslut()=default;     //智能指针管理,析构函数不必做事,设置为默认;
>// 问题一： setVal方法，获取任务执行完的返回值的
>	void setVal(Any any);
>	//问题二：  get方法  用户调用这个方法获取task的返回值
>	Any get();
>private:
>
>Any any_; //存储任务的返回值
>Semaphore sem_; //线程通信信号量
>std::shared_ptr<Task>task_; //指向对应获取返回值的任务对象,因为task执行完成后会被析构,要想拿到task的返回值
>                         //需要使用智能指针将task绑定到Result对象,防止task提前析构.
>std::atomic_bool isValid_; //返回值是否有效  如果用户提交任务失败,返回值肯定无效
>}
>//4.任务抽象基类
>class Task
>{
>public:
>	Task();
>	~Task() = default;
>	void exec();
>	void setResult(Result* res);
>	// 用户可以自定义任意任务类型，从task继承，重写run方法，实现自定义任务处理
>	virtual Any run() = 0;
>
>private:
>	Result * result_;  //Result对象的生命周期强于Task  不能使用智能指针会与Task发生交叉引用
>};
>```

**Result方法的实现**

>```cpp
>//////////////////////   Task方法实现
>Task::Task()
>:result_(nullptr){ }
>
>void Task::exec()
>{
>if (result_ != nullptr)
>{
>   result_->setVal(run());//这里发生多态调用；
>}
>
>}
>
>void Task::setResult(Result* res)
>{
>result_ = res;
>}
>
>Result::Result(std::shared_ptr<Task>task,bool isValid)
>:isValid_(isvalid)
>,task_(task)    //task_是强智能指针,task_的引用计数不为零task_不会被析构
>{}
>
>/////////////////////   Result方法的实现
>Result::Result(std::shared_ptr<Task>task, bool isValid )
>:isValid_(isValid)
>,task_(task)
>{
>task_->setResult(this);
>}
>Any Result::get()  //用户调用的
>{
>if (!isValid_)
>{
>return "";
>}
>sem_.wait(); //task任务如果没有执行完，这里会阻塞用户的线程
>return std::move(any_);  //Any类中的any_是unique_ptr类型的 右值禁止拷贝，使用move();
>}
>
>void Result::setVal(Any any)
>{
>//存储task的返回值
>this->any_ = std::move(any);  //any禁用左值的拷贝构造和赋值,因为消耗很大,使用move做资源转移
>sem_.post();//已经获取了任务的返回值，增加信号量资源
>}
>
>```
>
>

#### 3.6 Cached模式设计实现

1.用户在使用线程池时候可以自己设计线程模式 setMode()方法

2.需要根据任务数量和空闲线程数量,判断是否需要创建新的线程出来?

3.catched模式下,有可能已经创建了很多的线程,但是空闲时间超过60s, 应该把多余的线程结束回收掉

>```cpp
>           //cached模式 任务处理比较紧急 场景：小而快的任务 需要根据线程数量和空闲线程的数量，判断是否需要创建新的线程出来？
>           //耗时任务多不适合cached模式，因为耗时的任务会长时间占用一个线程，耗时任务比较多会导致线程池创建线程过多，
>           //创建线程过多会对系统性能影响非常大
>if(poolMode_==PoolMode::MODE_CACHED
>   &&taskSize_>idleThreadSize_
>   && curThreadSize_<threadSizeThreshHold_)
>{
>   std::cout << ">>>> create new thread...."  << "exit!"
>       << std::endl;
>
>   //创建新的线程对象
>   std::unique_ptr<Thread> ptr = std::make_unique<Thread>(std::bind(&ThreadPool::threadFunc, this, std::placeholders::_1));
>   //threads_.emplace_back(std::move(ptr));
>   int threadId = ptr->getId();
>   threads_.emplace(threadId, std::move(ptr));
>   //启动线程
>   threads_[threadId]->start(); 
>   //修改线程个数相关的变量
>   curThreadSize_++;
>   idleThreadSize_++;
>
>}
>
>
>```
>
>```cpp
>//cached模式下，有可能已经创建了很多的线程，但是空闲时间超过60s，应该把多余的线程回收掉
>       //结束回收掉？(超过initThreadSize_数量的线程要进行回收)
>       //当前时间-上一次线程执行的时间 > 60s
>if (poolMode_ == PoolMode::MODE_CACHED)
>       {
>           //每一秒钟返回一次    怎么区分：超时返回？ 还是有任务待执行返回
>           while (taskQue_.size() == 0)
>           {
>               //条件变量，超时返回了
>               if (std::cv_status::timeout ==
>                   notEmpty_.wait_for(lock, std::chrono::seconds(1)))
>               {
>                   auto now = std::chrono::high_resolution_clock().now();
>                   auto dur = std::chrono::duration_cast<std::chrono::seconds>(now - lastTime);
>                   if (dur.count() >= THREAD_MAX_IDLE_TIME
>                       && curThreadSize_>initThreadSize_)
>                   {
>                       //开始回收当前线程
>                       //记录线程数量的相关变量的值修改
>                       //把线程对象从线程列表容器中删除   没有办法 threadFunc   匹配哪个thread对象
>                       //threadid => thread对象 => 删除
>                       threads_.erase(threadid);  //std::this_thread::getid()
>                       curThreadSize_--;
>                       idleThreadSize_--;
>
>                       std::cout << "threadid:" << std::this_thread::get_id()<<"exit!"<<std::endl;
>                       return;
>                   }
>               }
>           }
>       }
>```

#### 3.7 线程池资源回收

ThreadPool对象析构以后,怎么样把线程池相关线程资源全部回收?

>```cpp
>//2. 线程池析构     C++中只要出现了构造一定要有析构
>ThreadPool::~ThreadPool()
>{ 
>isPoolrunning_ = false;
>//通知线程从等待状态到阻塞状态,然后才能抢锁
>notEmpty_.notify_all();
>//等待线程池里面所有的线程返回  有两种状态： 阻塞 & 正在执行任务中
>std::unique_lock<std::mutex>lock(taskQueMtx_);
>//继续析构的前提是线程容器里的线程对象清空了threads_.size() == 0,如果还有线程对象说明有线程未被回收,线程池先不析构
>exitCond_.wait(lock, [&]()->bool {return threads_.size() == 0; });
>}
>
>if (!isPoolRunning_)
>{
>	threads_.erase(threadid); // std::this_thread::getid()
>	std::cout << "threadid:" << std::this_thread::get_id() << " exit!"
>	<< std::endl;
>	exitCond_.notify_all();
>	return; // 结束线程函数，就是结束当前线程了!
>}
>```

### 4.Thread类设计

>```cpp
>//线程类型
>class Thread
>{
>public:
>	//线程函数对象类型
>	using ThreadFunc = std::function<void(int)>;//ThreadFunc 是 std::function 模板的实例化
>   //自C++11开始，using还可以用于定义类型别名，这类似于typedef但语法更为清晰。
>	//其模板参数是 void()，表示接受零个参数、返回 void 的函数类型
>
>	//线程构造
>	Thread(ThreadFunc func);
>	//线程析构
>	~Thread();
>
>	//启动线程
>	void start();
>
>	//获取线程id
>	int getId()const;
>private:
>	ThreadFunc func_;// 函数对象 线程函数类型
>	static int generateId_; 
>	int threadId_;   //保存线程Id
>};
>
>/*
>example:
>ThreadPool pool;
>pool.start(4);
>
>class MyTask : public Task
>{
>	public:
>	    void run(){ //线程代码.... }
>}
>
>pool.submitTask(std::make_shared<MyTask>());
>
>*/
>```

### 5.Thread方法接口实现(线程方法实现)

thread线程的方法实现

>```cpp
>#include "threadpool.h"
>#include<functional>
>#include<thread>
>#include<iostream>
>//二、线程方法实现
>//1.线程构造
>Thread::Thread(ThreadFunc func)
>:func_(func)
>,threadId_(generateId_++)
>{}
>//2.线程析构
>Thread::~Thread()
>{}
>
>//1.启动线程
>void Thread::start()
>{
>//创建一个线程来执行一个线程函数
>std::thread t(func_,threadId_);  //C++11来说 线程对象t 和线程函数func_
>t.detach();//设置分离线程  linux中的 pthread_detach  pthread_t设置成分离线程,作用分离线程对象和线程函数func_
>          //线程对象t出了作用域会释放，但是线程函数func_不能释放，所以设置分离线程。
>}
>
>//获取线程Id
>int  Thread::getId()const
>{
>return threadId_;
>}
>
>//////////////////////   Task方法实现
>Task::Task()
>:result_(nullptr){ }
>
>void Task::exec()
>{
>if (result_ != nullptr)
>{
>   result_->setVal(run());//这里发生多态调用；
>}
>
>}
>
>
>void Task::setResult(Result* res)
>{
>result_ = res;
>}
>```

在C++（特别是C++11及以后版本）中，**分离线程**的主要作用是用来管理线程生命周期和资源回收的方式。当一个线程被创建并启动后，默认情况下它是非分离（joinable）状态，这意味着线程执行完毕后，如果不通过某种方式（通常是调用`std::thread::join()`）获取其退出状态并等待它终止，那么线程资源不会立即释放，主线程或其他相关线程需要负责清理这些资源。

分离（detached）线程则是另一种线程生命周期管理模式，它允许线程在执行完毕后自行释放与其关联的所有资源，不需要其他线程显式地等待或尝试加入（join）它。一旦线程被分离，即使主线程或其他任何线程在此之前结束，操作系统也会自动回收分离线程占用的系统资源，包括栈空间和其他内部资源。

分离线程的优点在于：

1. **资源自动回收**：避免了因忘记或无法正确地调用`join()`而导致的资源泄露问题，尤其是对于那些执行完成后不需要返回结果的后台任务或守护线程特别有用。
2. **并发效率**：由于线程结束时无需等待其他线程来回收资源，程序可以更快地响应和释放系统资源。
3. **无须同步**：分离线程执行完毕后，不需要主线程或其他线程对其进行同步操作，减少了同步复杂度和潜在的死锁风险。

在C++11中，使用`std::thread::detach()`方法可以使一个线程变为分离状态。一旦线程被分离，就无法再对该线程调用`join()`方法，因为此时线程已不再是joinable状态。



在C++中，一个线程对象（`std::thread`）的作用周期涵盖了从创建到销毁的过程，具体分为以下几个阶段：

1. **创建**： 当通过`std::thread`构造函数创建一个线程对象时，实际上创建了一个新的线程，该线程将执行传递给构造函数的可调用对象（如函数、函数对象、Lambda 表达式等）。

   ```cpp
   std::thread myThread(func, arg1, arg2); // 创建并启动一个线程
   ```

2. **运行**： 新创建的线程在调度器安排下开始执行其指定的任务。这个阶段的持续时间取决于线程要执行的代码逻辑。

3. **等待/同步**： 如果线程是joinable（非分离）状态，主线程或其他线程可以选择通过调用`std::thread::join()`方法等待该线程结束。在join操作完成之前，线程对象及其关联的资源（如线程堆栈）不会被销毁。

   Cpp

   ```cpp
   myThread.join(); // 等待myThread执行完毕
   ```

4. **分离**： 若选择对线程对象调用`std::thread::detach()`方法，则该线程将变为分离状态。分离后的线程在其执行完毕后，系统会自动回收其资源，无需调用`join()`。

   Cpp

   ```cpp
   myThread.detach(); // 分离线程，线程结束后资源会被自动回收
   ```

5. **销毁**： 当线程对象离开其作用范围，或手动调用析构函数时，如果线程仍然是joinable状态且尚未被`join()`或`detach()`，则程序的行为未定义（可能导致资源泄露或异常）。正确的做法是在线程结束前确保其已经被`join()`或`detach()`。

总结来说，线程对象的作用周期应当包括创建、运行、等待/同步（join或detach）以及最终的销毁。在实际编程中，应确保妥善管理线程的生命周期，防止出现资源泄露或其他未定义行为。

#### 5.1threadfunc()线程函数实现

>```cpp
>//定义线程函数   线程池的所有线程从任务队列里面消费任务
>void ThreadPool::threadFunc(int threadid)   //线程函数返回，相应的线程也就结束了
>{
>auto lastTime = std::chrono::high_resolution_clock().now();
>/*std::cout << "begin threadFunc tid: " << std::this_thread::get_id() << std::endl;
>std::cout << "end threadFunc tid: " <<std::this_thread::get_id() << std::endl;*/
>for (;;)
>{
>   std::shared_ptr<Task>task;
>   {
>       //先获取锁
>       std::unique_lock<std::mutex>lock(taskQueMtx_);
>
>
>       std::cout << "tid: " <<    std::this_thread::get_id() << "尝试获取任务..." << std::endl;
>
>       //cached模式下，有可能已经创建了很多的线程，但是空闲时间超过60s，应该把多余的线程回收掉
>       //结束回收掉？(超过initThreadSize_数量的线程要进行回收)
>       //当前时间-上一次线程执行的时间 > 60s
>       if (poolMode_ == PoolMode::MODE_CACHED)
>       {
>           //每一秒钟返回一次    怎么区分：超时返回？ 还是有任务待执行返回
>           while (taskQue_.size() == 0)
>           {
>               //条件变量，超时返回了
>               if (std::cv_status::timeout ==
>                   notEmpty_.wait_for(lock, std::chrono::seconds(1)))
>               {
>                   auto now = std::chrono::high_resolution_clock().now();
>                   auto dur = std::chrono::duration_cast<std::chrono::seconds>(now - lastTime);
>                   if (dur.count() >= THREAD_MAX_IDLE_TIME
>                       && curThreadSize_>initThreadSize_)
>                   {
>                       //开始回收当前线程
>                       //记录线程数量的相关变量的值修改
>                       //把线程对象从线程列表容器中删除   没有办法 threadFunc   匹配哪个thread对象
>                       //threadid => thread对象 => 删除
>                       threads_.erase(threadid);  //std::this_thread::getid()
>                       curThreadSize_--;
>                       idleThreadSize_--;
>
>                       std::cout << "threadid:" << std::this_thread::get_id()<<"exit!"<<std::endl;
>                       return;
>                   }
>               }
>           }
>       }
>       else
>       {
>           //等待notEmpty条件
>           notEmpty_.wait(lock, [&]()->bool {return taskQue_.size() > 0; });
>       }
>
>
>       //等待notEmpty条件
>       notEmpty_.wait(lock, [&]()->bool {return taskQue_.size() > 0; });
>       idleThreadSize_--;
>
>       std::cout << "tid: " << std::this_thread::get_id() << "获取任务成功..." << std::endl;
>
>       //从任务队列中取出一个任务出来
>       auto task = taskQue_.front();
>       taskQue_.pop();
>       taskSize_--; 
>
>       //如果依然有剩余任务，继续通知其他的线程执行任务
>       if (taskQue_.size() > 0)
>       {
>           notEmpty_.notify_all();
>       }
>
>       //取出一个任务，进行通知,通知继续生产任务
>       notFull_.notify_all();
>
>   }//就应该把锁释放掉  别的线程也能去取任务去执行，或者用户能够取到锁提交任务
>   //作用域的原因，局部对象出作用域自动析构，把锁释放掉。
>
>
>   //从当前线程负责执行这个任务
>   if (task != nullptr)
>   {
>       //task->run(); //基类指针指向那个派生类对象，就会调用派生类的同名覆盖方法；
>                 //1. 执行任务；2. 把任务的返回值setVal方法给到Result
>       task->exec();
>   }
>
>   idleThreadSize_++;
>   lastTime = std::chrono::high_resolution_clock().now();// 更新线程执行完任务的时间
>
>   }    
>}
>```

### 6.linux编译线程池动态库

>```cpp
>//生成动态库
>g++ -fPIC -shared threadpool.cpp -o libtdpool.so -std=c++17
>```
>
>linux从 /usr/lib  /usr/local/lib  找静态库  .a  动态库  .so
>
>从  /usr/include  /usr/local/include 找 *.h 
>
>将threadpool.h 放入 /usr/local/include/, 生成的动态库libtdpool.so 放到/usr/local/lib/, 删除threadpool.cpp源文件
>
>```cpp
>g++ 线程池项目测试.cpp -std=c++17 -ltdpool -lpthread
>```

程序编译时候会从 /usr/lib  /usr/local/lib查询动态库, 程序运行阶段 会从/etc/ld.so.conf查找 :解决方法在/etc/ld.so.conf.d/ 添加一个myconf配置文件,加入路径

>```cpp
>ldconfig  //将myconf文件添加的路径刷新到 /etc/ld.so.cache
>```
>
>

### 7.线程池使用说明

用户使用说明

>```cpp
>example:
>ThreadPool pool;  //定义线程池
>pool.start(4);    //启动四个线程
>
>class MyTask : public Task  
>{
>	public:
>	    void run(){ //线程代码.... } //重写run方法
>}
>
>pool.submitTask(std::make_shared<MyTask>()); //用户提交任务
>//make_shared<MyTask>()创建对象的好处是,同时创建对象内存和对象引用计数的内存,以免产生对象分配成功,却无法释放的现象
>
>```



### 8.优化线程池代码-基于可变参模板编程

#### 8.1packaged_task和future机制

##### packaged_task和future机制 ?

在C++11及后续标准中，`std::future` 和 `std::packaged_task` 是两个用于处理异步计算结果的核心组件，它们都在 `<future>` 标准库中定义。

**std::packaged_task** `std::packaged_task` 是一个类模板，它能够封装任何可调用对象（包括函数、lambda 表达式、`std::bind` 结果等），并将其实现为可以异步执行的任务。当 `std::packaged_task` 实例化时，它会与一个 `std::future` 关联，这个 `future` 对象随后可以用来查询封装任务的结果。

使用 `std::packaged_task` 的典型步骤如下：

1. 创建一个 `std::packaged_task` 实例，传入想要异步执行的可调用对象。
2. 通过调用 `packaged_task::get_future()` 获取与该任务关联的 `std::future` 对象，这个 future 将用于检索任务完成后产生的结果。
3. 将 `std::packaged_task` 传递给某个线程进行执行，通常是通过 `std::thread` 或者 `std::async`，或者在事件循环中调度执行。
4. 当任务完成后，`packaged_task` 会将其计算结果放入与其关联的 `std::future` 中。
5. 在主线程或者其他需要结果的线程中，通过调用前面得到的 `future` 对象的 `get()` 方法来等待并获取结果。

**std::future** `std::future` 是一个表示异步计算结果的容器对象。它可以容纳由另一个线程执行的函数的结果或异常。一旦异步操作完成，`std::future` 对象可以通过调用其 `get()` 方法来获取结果或者重新抛出捕获的异常。`std::future` 有以下关键特性：

- **异步等待结果**：`get()` 方法可以阻塞当前线程直到异步操作完成并获取结果。
- **共享状态**：同一份计算结果可以被多个 `std::future` 共享，每个 `std::future` 都指向同一个异步计算的结果。
- **单次获取**：每个 `std::future` 对象上的 `get()` 只能调用一次，再次调用会导致未定义行为（通常表现为抛出异常）。

`std::future` 是一个接口，用于从异步操作接收结果，而 `std::packaged_task` 则是一个工具，它提供了一种创建和管理异步任务的方式，并且自动生成了与任务结果对应的 `std::future` 对象。两者结合使用，极大地简化了C++中异步编程模型的设计和实现。

#### 8.2优化思路

如何能让线程池提交任务更加方便

1. pool.submitTask(sum1, 10, 20);
   pool.submitTask(sum2, 1 ,2, 3);
   submitTask:可变参模板编程

2. 之前造了一个Result Any Semaphore的类型，代码挺多
   C++11 线程库   thread   packaged_task(function函数对象)  async 
   使用future来代替Result节省线程池代码

去除第一版代码中的Any Semaphore Result task类,使用可变参模板编程, 让submitTask可以接受任意函数和数量的参数

>```cpp
>// 给线程池提交任务
>	// 使用可变参模板编程，让submitTask可以接收任意任务函数和任意数量的参数
>	// pool.submitTask(sum1, 10, 20);   
>	// 返回值future<返回值类型> 使用decltype进行类型推导
>	template<typename Func, typename... Args>
>	auto submitTask(Func&& func, Args&&... args) -> std::future<decltype(func(args...))>
>	{
>		// 打包任务，放入任务队列里面
>		using RType = decltype(func(args...));
>		auto task = std::make_shared<std::packaged_task<RType()>>(
>			std::bind(std::forward<Func>(func), std::forward<Args>(args)...));
>		std::future<RType> result = task->get_future();
>
>		// 获取锁
>		std::unique_lock<std::mutex> lock(taskQueMtx_);
>		// 用户提交任务，最长不能阻塞超过1s，否则判断提交任务失败，返回
>		if (!notFull_.wait_for(lock, std::chrono::seconds(1),
>			[&]()->bool { return taskQue_.size() < (size_t)taskQueMaxThreshHold_; }))
>		{
>			// 表示notFull_等待1s种，条件依然没有满足
>			std::cerr << "task queue is full, submit task fail." << std::endl;
>			auto task = std::make_shared<std::packaged_task<RType()>>(
>				[]()->RType { return RType(); });
>			(*task)();
>			return task->get_future();
>		}
>```
>
>
>
>



#### 8.3ThreadPool设计实现

>```cpp
>#ifndef THREADPOOL_H
>#define THREADPOOL_H
>
>#include <iostream>
>#include <vector>
>#include <queue>
>#include <memory>
>#include <atomic>
>#include <mutex>
>#include <condition_variable>
>#include <functional>
>#include <unordered_map>
>#include <thread>
>#include <future>
>
>const int TASK_MAX_THRESHHOLD = 2; // INT32_MAX;
>const int THREAD_MAX_THRESHHOLD = 1024;
>const int THREAD_MAX_IDLE_TIME = 60; // 单位：秒
>
>
>// 线程池支持的模式
>enum class PoolMode
>{
>	MODE_FIXED,  // 固定数量的线程
>	MODE_CACHED, // 线程数量可动态增长
>};
>
>// 线程类型
>class Thread
>{
>public:
>	// 线程函数对象类型
>	using ThreadFunc = std::function<void(int)>;
>
>	// 线程构造
>	Thread(ThreadFunc func)
>		: func_(func)
>		, threadId_(generateId_++)
>	{}
>	// 线程析构
>	~Thread() = default;
>
>	// 启动线程
>	void start()
>	{
>		// 创建一个线程来执行一个线程函数 pthread_create
>		std::thread t(func_, threadId_);  // C++11来说 线程对象t  和线程函数func_
>		t.detach(); // 设置分离线程   pthread_detach  pthread_t设置成分离线程
>	}
>
>	// 获取线程id
>	int getId()const
>	{
>		return threadId_;
>	}
>private:
>	ThreadFunc func_;
>	static int generateId_;
>	int threadId_;  // 保存线程id
>};
>
>int Thread::generateId_ = 0;
>
>// 线程池类型
>class ThreadPool
>{
>public:
>	// 线程池构造
>	ThreadPool()
>		: initThreadSize_(0)
>		, taskSize_(0)
>		, idleThreadSize_(0)
>		, curThreadSize_(0)
>		, taskQueMaxThreshHold_(TASK_MAX_THRESHHOLD)
>		, threadSizeThreshHold_(THREAD_MAX_THRESHHOLD)
>		, poolMode_(PoolMode::MODE_FIXED)
>		, isPoolRunning_(false)
>	{}
>
>	// 线程池析构
>	~ThreadPool()
>	{
>		isPoolRunning_ = false;
>
>		// 等待线程池里面所有的线程返回  有两种状态：阻塞 & 正在执行任务中
>		std::unique_lock<std::mutex> lock(taskQueMtx_);
>		notEmpty_.notify_all();
>		exitCond_.wait(lock, [&]()->bool {return threads_.size() == 0; });
>	}
>
>	// 设置线程池的工作模式
>	void setMode(PoolMode mode)
>	{
>		if (checkRunningState())
>			return;
>		poolMode_ = mode;
>	}
>
>	// 设置task任务队列上线阈值
>	void setTaskQueMaxThreshHold(int threshhold)
>	{
>		if (checkRunningState())
>			return;
>		taskQueMaxThreshHold_ = threshhold;
>	}
>
>	// 设置线程池cached模式下线程阈值
>	void setThreadSizeThreshHold(int threshhold)
>	{
>		if (checkRunningState())
>			return;
>		if (poolMode_ == PoolMode::MODE_CACHED)
>		{
>			threadSizeThreshHold_ = threshhold;
>		}
>	}
>
>	// 给线程池提交任务
>	// 使用可变参模板编程，让submitTask可以接收任意任务函数和任意数量的参数
>	// pool.submitTask(sum1, 10, 20);   
>	// 返回值future<>
>	template<typename Func, typename... Args>
>	auto submitTask(Func&& func, Args&&... args) -> std::future<decltype(func(args...))>
>	{
>		// 打包任务，放入任务队列里面
>		using RType = decltype(func(args...));
>		auto task = std::make_shared<std::packaged_task<RType()>>(
>			std::bind(std::forward<Func>(func), std::forward<Args>(args)...));
>		std::future<RType> result = task->get_future();
>
>		// 获取锁
>		std::unique_lock<std::mutex> lock(taskQueMtx_);
>		// 用户提交任务，最长不能阻塞超过1s，否则判断提交任务失败，返回
>		if (!notFull_.wait_for(lock, std::chrono::seconds(1),
>			[&]()->bool { return taskQue_.size() < (size_t)taskQueMaxThreshHold_; }))
>		{
>			// 表示notFull_等待1s种，条件依然没有满足
>			std::cerr << "task queue is full, submit task fail." << std::endl;
>			auto task = std::make_shared<std::packaged_task<RType()>>(
>				[]()->RType { return RType(); });
>			(*task)();
>			return task->get_future();
>		}
>
>		// 如果有空余，把任务放入任务队列中
>		// taskQue_.emplace(sp);  
>		// using Task = std::function<void()>;
>		taskQue_.emplace([task]() {(*task)();});
>		taskSize_++;
>
>		// 因为新放了任务，任务队列肯定不空了，在notEmpty_上进行通知，赶快分配线程执行任务
>		notEmpty_.notify_all();
>
>		// cached模式 任务处理比较紧急 场景：小而快的任务 需要根据任务数量和空闲线程的数量，判断是否需要创建新的线程出来
>		if (poolMode_ == PoolMode::MODE_CACHED
>			&& taskSize_ > idleThreadSize_
>			&& curThreadSize_ < threadSizeThreshHold_)
>		{
>			std::cout << ">>> create new thread..." << std::endl;
>
>			// 创建新的线程对象
>			auto ptr = std::make_unique<Thread>(std::bind(&ThreadPool::threadFunc, this, std::placeholders::_1));
>			int threadId = ptr->getId();
>			threads_.emplace(threadId, std::move(ptr));
>			// 启动线程
>			threads_[threadId]->start();
>			// 修改线程个数相关的变量
>			curThreadSize_++;
>			idleThreadSize_++;
>		}
>
>		// 返回任务的Result对象
>		return result;
>	}
>
>	// 开启线程池
>	void start(int initThreadSize = std::thread::hardware_concurrency())
>	{
>		// 设置线程池的运行状态
>		isPoolRunning_ = true;
>
>		// 记录初始线程个数
>		initThreadSize_ = initThreadSize;
>		curThreadSize_ = initThreadSize;
>
>		// 创建线程对象
>		for (int i = 0; i < initThreadSize_; i++)
>		{
>			// 创建thread线程对象的时候，把线程函数给到thread线程对象
>			auto ptr = std::make_unique<Thread>(std::bind(&ThreadPool::threadFunc, this, std::placeholders::_1));
>			int threadId = ptr->getId();
>			threads_.emplace(threadId, std::move(ptr));
>			// threads_.emplace_back(std::move(ptr));
>		}
>
>		// 启动所有线程  std::vector<Thread*> threads_;
>		for (int i = 0; i < initThreadSize_; i++)
>		{
>			threads_[i]->start(); // 需要去执行一个线程函数
>			idleThreadSize_++;    // 记录初始空闲线程的数量
>		}
>	}
>
>	ThreadPool(const ThreadPool&) = delete;
>	ThreadPool& operator=(const ThreadPool&) = delete;
>
>private:
>	// 定义线程函数
>	void threadFunc(int threadid)
>	{
>		auto lastTime = std::chrono::high_resolution_clock().now();
>
>		// 所有任务必须执行完成，线程池才可以回收所有线程资源
>		for (;;)
>		{
>			Task task;
>			{
>				// 先获取锁
>				std::unique_lock<std::mutex> lock(taskQueMtx_);
>
>				std::cout << "tid:" << std::this_thread::get_id()
>					<< "尝试获取任务..." << std::endl;
>
>				// cached模式下，有可能已经创建了很多的线程，但是空闲时间超过60s，应该把多余的线程
>				// 结束回收掉（超过initThreadSize_数量的线程要进行回收）
>				// 当前时间 - 上一次线程执行的时间 > 60s
>
>				// 每一秒中返回一次   怎么区分：超时返回？还是有任务待执行返回
>				// 锁 + 双重判断
>				while (taskQue_.size() == 0)
>				{
>					// 线程池要结束，回收线程资源
>					if (!isPoolRunning_)
>					{
>						threads_.erase(threadid); // std::this_thread::getid()
>						std::cout << "threadid:" << std::this_thread::get_id() << " exit!"
>							<< std::endl;
>						exitCond_.notify_all();
>						return; // 线程函数结束，线程结束
>					}
>
>					if (poolMode_ == PoolMode::MODE_CACHED)
>					{
>						// 条件变量，超时返回了
>						if (std::cv_status::timeout ==
>							notEmpty_.wait_for(lock, std::chrono::seconds(1)))
>						{
>							auto now = std::chrono::high_resolution_clock().now();
>							auto dur = std::chrono::duration_cast<std::chrono::seconds>(now - lastTime);
>							if (dur.count() >= THREAD_MAX_IDLE_TIME
>								&& curThreadSize_ > initThreadSize_)
>							{
>								// 开始回收当前线程
>								// 记录线程数量的相关变量的值修改
>								// 把线程对象从线程列表容器中删除   没有办法 threadFunc《=》thread对象
>								// threadid => thread对象 => 删除
>								threads_.erase(threadid); // std::this_thread::getid()
>								curThreadSize_--;
>								idleThreadSize_--;
>
>								std::cout << "threadid:" << std::this_thread::get_id() << " exit!"
>									<< std::endl;
>								return;
>							}
>						}
>					}
>					else
>					{
>						// 等待notEmpty条件
>						notEmpty_.wait(lock);
>					}
>				}
>
>				idleThreadSize_--;
>
>				std::cout << "tid:" << std::this_thread::get_id()
>					<< "获取任务成功..." << std::endl;
>
>				// 从任务队列种取一个任务出来
>				task = taskQue_.front();
>				taskQue_.pop();
>				taskSize_--;
>
>				// 如果依然有剩余任务，继续通知其它得线程执行任务
>				if (taskQue_.size() > 0)
>				{
>					notEmpty_.notify_all();
>				}
>
>				// 取出一个任务，进行通知，通知可以继续提交生产任务
>				notFull_.notify_all();
>			} // 就应该把锁释放掉
>
>			// 当前线程负责执行这个任务
>			if (task != nullptr)
>			{
>				task(); // 执行function<void()> 
>			}
>
>			idleThreadSize_++;
>			lastTime = std::chrono::high_resolution_clock().now(); // 更新线程执行完任务的时间
>		}
>	}
>
>	// 检查pool的运行状态
>	bool checkRunningState() const
>	{
>		return isPoolRunning_;
>	}
>
>private:
>	std::unordered_map<int, std::unique_ptr<Thread>> threads_; // 线程列表
>
>	int initThreadSize_;  // 初始的线程数量
>	int threadSizeThreshHold_; // 线程数量上限阈值
>	std::atomic_int curThreadSize_;	// 记录当前线程池里面线程的总数量
>	std::atomic_int idleThreadSize_; // 记录空闲线程的数量
>
>	// Task任务 =》 函数对象
>	using Task = std::function<void()>;
>	std::queue<Task> taskQue_; // 任务队列
>	std::atomic_int taskSize_; // 任务的数量
>	int taskQueMaxThreshHold_;  // 任务队列数量上限阈值
>
>	std::mutex taskQueMtx_; // 保证任务队列的线程安全
>	std::condition_variable notFull_; // 表示任务队列不满
>	std::condition_variable notEmpty_; // 表示任务队列不空
>	std::condition_variable exitCond_; // 等到线程资源全部回收
>
>	PoolMode poolMode_; // 当前线程池的工作模式
>	std::atomic_bool isPoolRunning_; // 表示当前线程池的启动状态
>};
>
>#endif
>
>```



#### 8.4ThreadPool实现

>```cpp
>// 线程池项目-最终版.cpp : 此文件包含 "main" 函数。程序执行将在此处开始并结束。
>//
>
>#include <iostream>
>#include <functional>
>#include <thread>
>#include <future>
>#include <chrono>
>using namespace std;
>
>#include "threadpool.h"
>
>
>
>
>
>int sum1(int a, int b)
>{
>this_thread::sleep_for(chrono::seconds(2));
>// 比较耗时
>return a + b;
>}
>int sum2(int a, int b, int c)
>{
>this_thread::sleep_for(chrono::seconds(2));
>return a + b + c;
>}
>// io线程 
>void io_thread(int listenfd)
>{
>
>}
>// worker线程
>void worker_thread(int clientfd)
>{
>
>}
>int main()
>{
>ThreadPool pool;
>// pool.setMode(PoolMode::MODE_CACHED);
>pool.start(2);
>
>future<int> r1 = pool.submitTask(sum1, 1, 2);
>future<int> r2 = pool.submitTask(sum2, 1, 2, 3);
>future<int> r3 = pool.submitTask([](int b, int e)->int {
>   int sum = 0;
>   for (int i = b; i <= e; i++)
>       sum += i;
>   return sum;
>   }, 1, 100);
>future<int> r4 = pool.submitTask([](int b, int e)->int {
>   int sum = 0;
>   for (int i = b; i <= e; i++)
>       sum += i;
>   return sum;
>   }, 1, 100);
>future<int> r5 = pool.submitTask([](int b, int e)->int {
>   int sum = 0;
>   for (int i = b; i <= e; i++)
>       sum += i;
>   return sum;
>   }, 1, 100);
>//future<int> r4 = pool.submitTask(sum1, 1, 2);
>
>cout << r1.get() << endl;
>cout << r2.get() << endl;
>cout << r3.get() << endl;
>cout << r4.get() << endl;
>cout << r5.get() << endl;
>
>//packaged_task<int(int, int)> task(sum1);
>//// future <=> Result
>//future<int> res = task.get_future();
>//// task(10, 20);
>//thread t(std::move(task), 10, 20);
>//t.detach();
>
>//cout << res.get() << endl;
>
>/*thread t1(sum1, 10, 20);
>thread t2(sum2, 1, 2, 3);
>
>t1.join();
>t2.join();*/
>}
>```
