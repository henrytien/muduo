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

