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

