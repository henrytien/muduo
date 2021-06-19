### install boost and setup
[Installing boost libraries for GCC (MinGW) on Windows](https://gist.github.com/sim642/29caef3cc8afaa273ce6)
[在 Visual Studio Code 中构建一个C++开发环境](https://www.jianshu.com/p/e254efbc8345)

### `Timestamp.h`
```cpp
///
/// Gets time difference of two timestamps, result in seconds.
///
/// @param high, low
/// @return (high-low) in seconds
/// @c double has 52-bit precision, enough for one-microsecond
/// resolution for next 100 years.
inline double timeDifference(Timestamp high, Timestamp low)
{
  int64_t diff = high.microSecondsSinceEpoch() - low.microSecondsSinceEpoch();
  return static_cast<double>(diff) / Timestamp::kMicroSecondsPerSecond;
}
```
`boost::less_than_comparable<Timestamp>` 和`boost::equality_comparable<Timestamp>` 这两个类其实是两个是用来比较的， 类只要实现对operator==就会自动实现!=,less_than_comparable类 只要实现operator<，可自动实现>、<=、>=。

- **static成员和函数**
  > 查看这几个static函数的实现发现静态函数是没有对静态成员有写的操作，所以就没有上锁的操作，如果有静态函数的实现有对static成员的写操作，那么就会有性能上的下降，这时就应该考虑设计是否合理。
- **static_assert断言**
    >static_assert函数是boost提供的一个函数，该函数可以实现在编译期间断言的功能，如果在编译期间`static_assert`内语句不为真，那么就不能编译通过。
## `Thread.cc`

```cpp
class ThreadNameInitializer
{
 public:
  ThreadNameInitializer()
  {
    muduo::CurrentThread::t_threadName = "main";
    CurrentThread::tid();
    pthread_atfork(NULL, NULL, &afterFork);
  }
};
```
`pthread_atfor()`在子进程中调用`afterFork`，这里涉及到了在子进程里重新设置主线程，相当于多进程多线程，但是不推荐这样去使用，容易发生死锁。

## `Mutex.h`
```cpp
// Prevent misuse like:
// MutexLockGuard(mutex_);
// A tempory object doesn't hold the lock for long!
#define MutexLockGuard(x) error "Missing guard object name"
```
这里要是不看别人的代码，你又怎会知道呢？

## `Conditon.h`
```cpp
void wait()
  {
    MutexLock::UnassignGuard ug(mutex_);
    MCHECK(pthread_cond_wait(&pcond_, mutex_.getPthreadMutex()));
  }
``` 
Condition类涉及到两个对象，`MutexLock`和`pthread_cond_t`

## `CountDownLatch.h`
```cpp
void CountDownLatch::countDown()
{
  MutexLockGuard lock(mutex_);
  --count_;
  if (count_ == 0)
  {
    condition_.notifyAll();
  }
}
```

## `BlockingQueue.h`
```cpp
 void put(T&& x)
  {
    MutexLockGuard lock(mutex_);
    queue_.push_back(std::move(x));
    notEmpty_.notify();
  }
  ```

  ## `BoundBlockingQueue.h`
  ```cpp
  private:
  mutable MutexLock          mutex_;
  Condition                  notEmpty_ GUARDED_BY(mutex_);
  Condition                  notFull_ GUARDED_BY(mutex_);
  boost::circular_buffer<T>  queue_ GUARDED_BY(mutex_);
  ```
  `circular_buffer<T>` 循环队列，也可以使用数组实现。