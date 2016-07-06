---
layout: post
title:  "asynchronous function"
date:   2016-04-25 18:14:10 +0800
categories: pages update
---

多线程实现异步函数

# 基础结构

## struct uv_req_t

![]({{ "/jpg/class_uv_req_t.jpg" | prepend: site.baseurl }})


## struct uv_getnameinfo_t

![]({{ "/jpg/uv_getnameinfo_t.jpg" | prepend: site.baseurl }})


## struct uv_async_t

![]({{ "/jpg/uv_async_t.jpg" | prepend: site.baseurl }})



## [struct uv_loop_s]({{ site.baseurl }}{% post_url 2016-04-25-tcp-io %}#struct-uvloopt)
```c
  struct uv_async_t;  // See above

  struct uv__async {
    uv__async_cb cb;
    uv__io_t io_watcher;
    int wfd;
  };

  struct uv_loop_s {
    ... ...
    void* wq[2];                    // work queue
    uv_mutex_t wq_mutex;            // work queue mutex
    uv_async_t wq_async;            // work async handle
    ... ...
    void* async_handles[2];         // async handle queue
    struct uv__async async_watcher; // async watcher
    ... ...
  };
```
 
In general, these members are divided into three groups:

1. Work queue _wq_ is to store finishing working works.
2. Async handle _wq_async_ is a handle, it defines in struct uv_loop_t, then its life cycle is the same as the loop. While initial the loop, _wq_async_ is inserted into _async_handles_.
3. Async watcher _async_watcher_ contains a uv__async_cb cb, a uv__io_t io_watcher, a int wfd. io_watcher registers to loop->watchers. At this struct, there are two callback functions. That one does I/O, another do async events. At the second callback, it will call all handles in async handle queue _async_handles_. Notes: In Linux, io_watcher.fd is created by [eventfd()](http://man7.org/linux/man-pages/man2/eventfd.2.html), instead of [pipe()](http://man7.org/linux/man-pages/man2/pipe.2.html) or [pipe2()](http://man7.org/linux/man-pages/man2/pipe2.2.html).


libuv currently uses a global thread pool on which all loops can queue work on. 3 types of operations are currently run on this pool:

1. Filesystem operations, see [asynchronous-fs]({{ site.baseurl }}{% post_url 2016-04-25-asynchronous-fs %})
2. DNS functions ([getaddrinfo](#struct-uv_getnameinfo_t) and getnameinfo)
3. User specified code via uv_queue_work()


libuv 通过多线程的方式来实现异步调用同步函数。  
异步实现getaddrinfo、getnameinfo，获取到结果后，通过线程间发送消息通知网络线程。

流程：（request的传递 + loop的回调）

```c
  uv_getaddrinfo -> uv__work_submit(loop,
                    &req->work_req,
                    uv__getaddrinfo_work,
                    uv__getaddrinfo_done);  -> thread::worker -> uv__getaddrinfo_work -> getaddrinfo -> uv_async_send
                 -> uv__async_io -> uv__async_event -> uv__work_done -> uv__getaddrinfo_done -> typedef void (*uv_getaddrinfo_cb)(uv_getaddrinfo_t* req,
                                    int status,
                                    struct addrinfo* res);

  uv_getnameinfo -> uv__work_submit(loop,
                    &req->work_req,
                    uv__getnameinfo_work,
                    uv__getnameinfo_done); -> thread::worker -> uv__getnameinfo_work -> getnameinfo
```

应用层调用 uv_getaddrinfo、uv_getnameinfo，函数内部会产生一个 work request，这个 request 会被放入到线程间的共享队列里。 

loop的回调：uv__async_io -> uv__async_event -> uv__work_done -> uv__getaddrinfo_done。


线程的创建：程序启动时创建的网络I/O线程，即主线程。在第一次调用 uv__work_submit 时，通过 pthread_once 的方式只一次创建所需要的异步工作线程，默认创建4个异步工作线程。

多个异步工作线程，竞争地从 work request queue 获取 work request，获取成功，调用 request 设置的 work 函数，即实现具体功能的同步函数（getaddrinfo 、getnameinfo），然后发送消息通知网络I/O线程，异步完成了对应函数的调用，返回了结果。

异步工作线程与网络I/O线程间的通讯：

1. 采用共享队列，像上面的 work request queue，类似生产者与消费者模型，网络I/O线程产生request，异步工作线程处理request。这种模型要求消费者线程不停低检查是否有新的request到达，为实现高效，一般采用 [mutex]({{ site.baseurl }}{% post_url 2016-04-25-thread %}#mutex) lock + [condition variable]({{ site.baseurl }}{% post_url 2016-04-25-thread %}#condition-variable) 。 网络I/O线程主动与异步工作线程通讯时采用的是这种方式。
1. 管道 pipe。libuv 的异步工作线程在完成 work 之后，就是通过 pipe 的方式发送消息通知网络I/O线程，异步调用完成了，网络I/O线程可以继续做后面的逻辑。之所以采用这种方式，是因为网络I/O线程主逻辑就是在调用 epoll_wait 等待各种事件，实现 pipe 的通讯，只需要把 pipe 一端 fd 的读事件加入到 epoll 的监控列表。

Linux 线程间 pipe 通讯，两个线程可以都使用同一个 fd ：

```c
 #if defined(__linux__)
   if (fd == -1) {
     static const uint64_t val = 1;
     buf = &val;
     len = sizeof(val);
     fd = wa->io_watcher.fd;  /* eventfd */
   }
 #endif
 
   do
     r = write(fd, buf, len);
   while (r == -1 && errno == EINTR);
 
   if (r == len)
     return;
```

异步工作线程通过 pipe 给网络I/O线程发送消息后，uv__async_io 被调用，是因为在对应的 fd 上注册了这个回调函数：

```c
  uv__io_init(&wa->io_watcher, uv__async_io, pipefd[0]);
  uv__io_start(loop, &wa->io_watcher, UV__POLLIN);
  wa->wfd = pipefd[1];
  wa->cb = cb;
```

work request 的创建和销毁：

1. 创建：uv_getaddrinfo_t getaddrinfo_req;
2. 销毁：uv__getaddrinfo_done -> xxx_cb -> uv_freeaddrinfo(addrs) -> freeaddrinfo(ai)

```c
  uv_getaddrinfo_t is a subclass of uv_req_t.
  typedef struct uv_getaddrinfo_s uv_getaddrinfo_t;
  uv_getaddrinfo_s is a request object for uv_getaddrinfo.
```


* 参考：
1. [offsetof]({{ site.baseurl }}{% post_url 2016-04-25-offsetof %})
2. [QUEUE]({{ site.baseurl }}{% post_url 2016-04-25-queue %})
