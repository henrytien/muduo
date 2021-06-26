## `EventLoop.cc`
1. 整个事件循环是怎么执行的你清楚么？
2. 你认为`bool`是原子性操作么？相比`int`呢？

```cpp
int createEventfd()
{
  int evtfd = ::eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);
  if (evtfd < 0)
  {
    LOG_SYSERR << "Failed in eventfd";
    abort();
  }
  return evtfd;
}
```
For an explanation of the terms used in this section, see attributes(7).

       ┌──────────┬───────────────┬─────────┐
       │Interface │ Attribute     │ Value   │
       ├──────────┼───────────────┼─────────┤
       │eventfd() │ Thread safety │ MT-Safe │
       └──────────┴───────────────┴─────────┘

- event
```c++
//The event argument describes the object linked to the file descriptor fd.  The struct epoll_event is defined as:
typedef union epoll_data {
   void        *ptr;
   int          fd;
   uint32_t     u32;
   uint64_t     u64;
} epoll_data_t;

struct epoll_event {
   uint32_t     events;      /* Epoll events */
   epoll_data_t data;        /* User data variable */
};
```

- 创建epollevent事件
```c++
Timestamp EPollPoller::poll(int timeoutMs, ChannelList* activeChannels)
{
  int numEvents = ::epoll_wait(epollfd_,
                               &*events_.begin(),
                               static_cast<int>(events_.size()),
                               timeoutMs);
  Timestamp now(Timestamp::now());
  if (numEvents > 0)
  {
    LOG_TRACE << numEvents << " events happended";
    fillActiveChannels(numEvents, activeChannels);
    if (implicit_cast<size_t>(numEvents) == events_.size())
    {
      events_.resize(events_.size()*2); // default size is 16
    }
  }
  else if (numEvents == 0)
  {
    LOG_TRACE << " nothing happended";
  }
  else
  {
    LOG_SYSERR << "EPollPoller::poll()";
  }
  return now;
}
```
- epoll_create1()
> epoll_create1()
       If  flags is 0, then, other than the fact that the obsolete size argument is dropped, epoll_create1() is the same as epoll_create().  The following value can be included in flags to obtain different behavior:
>       EPOLL_CLOEXEC
>              Set the close-on-exec (FD_CLOEXEC) flag on the new file descriptor.  See the description of the O_CLOEXEC flag in open(2) for reasons why this may  be useful.

- 如何管理`Channel`的生存周期呢？
  ```cpp
   // unlike in TimerQueue, which is an internal class,
  // we don't expose Channel to client.
  std::unique_ptr<Channel> wakeupChannel_;
  ```
  
- `EventLoop`里`wakeup`函数的实现？
  ```cpp
    void EventLoop::wakeup()
    {
        uint64_t one = 1;
        ssize_t n = sockets::write(wakeupFd_, &one, sizeof one);
        if (n != sizeof one)
        {
            LOG_ERROR << "EventLoop::wakeup() writes " << n << " bytes instead of 8";
        }
    }
  ```
  
- 请问这个函数能够实现同步和异步调用么？
   ```cpp
   void EventLoop::runInLoop(Functor cb)
   {
     if (isInLoopThread())
     {
       cb(); // sync
     }
     else
     {
       queueInLoop(std::move(cb)); // async
     }
   }
   ```
   
- 请问`I/O`线程能负责一部分计算任务么？
  
  ```c++
  void EventLoop::doPendingFunctors()
  {
    std::vector<Functor> functors;
    callingPendingFunctors_ = true;
  
    {
    MutexLockGuard lock(mutex_);
    functors.swap(pendingFunctors_);
    }
  
    for (const Functor& functor : functors)
    {
      functor();
    }
    callingPendingFunctors_ = false;
  }
  ```
  你知道为什么只对`swap`部分加锁么？
  - 减少临界区的长度
  - 不会阻塞其他线程的`queueInLoop`
  - 避免了死锁
  
- 如果直接for循环调用`doPendingFunctors()`,会发生什么？

   死循环的可能。

## `EventLoopThread.cc`

```c++
class EventLoop;

class EventLoopThread : noncopyable
{
 public:
  typedef std::function<void(EventLoop*)> ThreadInitCallback;

  EventLoopThread(const ThreadInitCallback& cb = ThreadInitCallback(),
                  const string& name = string());
  ~EventLoopThread();
  EventLoop* startLoop();			// start an IO thread

 private:
  void threadFunc();

  EventLoop* loop_ GUARDED_BY(mutex_);
  bool exiting_;
  Thread thread_;
  MutexLock mutex_;
  Condition cond_ GUARDED_BY(mutex_);
  ThreadInitCallback callback_;
};
```

## `Socket.cc`

- Nagle

  ```c++
  void Socket::setTcpNoDelay(bool on)
  {
    int optval = on ? 1 : 0;
    ::setsockopt(sockfd_, IPPROTO_TCP, TCP_NODELAY,
                 &optval, static_cast<socklen_t>(sizeof optval));
    // FIXME CHECK
  }
  ```

- KeepAlive

  ```c++
  void Socket::setKeepAlive(bool on)
  {
    int optval = on ? 1 : 0;
    ::setsockopt(sockfd_, SOL_SOCKET, SO_KEEPALIVE,
                 &optval, static_cast<socklen_t>(sizeof optval));
    // FIXME CHECK
  }
  ```

## `Acceptor.cc`

- HandleRead

  ```c++
  void Acceptor::handleRead()
  {
    loop_->assertInLoopThread();
    InetAddress peerAddr;
    //FIXME loop until no more
    int connfd = acceptSocket_.accept(&peerAddr);
    if (connfd >= 0)
    {
      // string hostport = peerAddr.toIpPort();
      // LOG_TRACE << "Accepts of " << hostport;
      if (newConnectionCallback_)
      {
        newConnectionCallback_(connfd, peerAddr);
      }
      else
      {
        sockets::close(connfd);
      }
    }
    else
    {
      LOG_SYSERR << "in Acceptor::handleRead";
      // Read the section named "The special problem of
      // accept()ing when you can't" in libev's doc.
      // By Marc Lehmann, author of libev.
      if (errno == EMFILE)
      {
        ::close(idleFd_);
        idleFd_ = ::accept(acceptSocket_.fd(), NULL, NULL);
        ::close(idleFd_);
        idleFd_ = ::open("/dev/null", O_RDONLY | O_CLOEXEC);
      }
    }
  ```

  Use a idleFd deal with EMFILE problem.

  

