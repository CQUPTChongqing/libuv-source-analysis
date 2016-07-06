---
layout: post
title:  "prepare,check and idle"
date:   2016-04-25 18:14:10 +0800
categories: pages update
---

# uv__run_prepare,uv__run_check and uv__run_idle

```c
int uv_run(uv_loop_t* loop, uv_run_mode mode) {
  int timeout;
  int r;
  int ran_pending;

  r = uv__loop_alive(loop);
  if (!r)
    uv__update_time(loop);

  while (r != 0 && loop->stop_flag == 0) {
    uv__update_time(loop);
    uv__run_timers(loop);
    ran_pending = uv__run_pending(loop);
    uv__run_idle(loop);
    uv__run_prepare(loop);

    timeout = 0;
    if ((mode == UV_RUN_ONCE && !ran_pending) || mode == UV_RUN_DEFAULT)
      timeout = uv_backend_timeout(loop);

    uv__io_poll(loop, timeout);
    uv__run_check(loop);
    uv__run_closing_handles(loop);

    if (mode == UV_RUN_ONCE) {
      /* UV_RUN_ONCE implies forward progress: at least one callback must have
       * been invoked when it returns. uv__io_poll() can return without doing
       * I/O (meaning: no callbacks) when its timeout expires - which means we
       * have pending timers that satisfy the forward progress constraint.
       *
       * UV_RUN_NOWAIT makes no guarantees about progress so it's omitted from
       * the check.
       */
      uv__update_time(loop);
      uv__run_timers(loop);
    }

    r = uv__loop_alive(loop);
    if (mode == UV_RUN_ONCE || mode == UV_RUN_NOWAIT)
      break;
  }

  /* The if statement lets gcc compile it to a conditional store. Avoids
   * dirtying a cache line.
   */
  if (loop->stop_flag != 0)
    loop->stop_flag = 0;

  return r;
}
```

Notes: 

1. uv__run_prepare(loop), uv__run_check(loop) and uv__run_idle(loop) run at different points in time. This allows users to add their own needs callback functions.
2. Interface is defined in the file [src/unix/loop-watcher.c](https://github.com/libuv/libuv/blob/v1.x/src/unix/loop-watcher.c).
3. Reference to [Design overview#The I/O loop](http://docs.libuv.org/en/latest/design.html#the-i-o-loop).



# struct uv_prepare_t

```c
int uv_prepare_init(uv_loop_t* loop, uv_prepare_t* handle)
{
  uv__handle_init(loop, (uv_handle_t* )handle, UV_PREPARE);
  handle->prepare_cb = NULL;
  return 0;
}

int uv_prepare_start(uv_prepare_t* handle, uv_prepare_cb cb)
{
  if (uv__is_active(handle))
    return 0;
  if (cb == NULL)
    return -EINVAL;
  QUEUE_INSERT_HEAD(&handle->loop->prepare_handles, &handle->queue);
  handle->prepare_cb = cb;
  uv__handle_start(handle);
  return 0;
}

int uv_prepare_stop(uv_prepare_t* handle)
{
  if (!uv__is_active(handle))
    return 0;
  QUEUE_REMOVE(&handle->queue);
  uv__handle_stop(handle);
  return 0;
}

void uv__run_prepare(uv_loop_t* loop)
{
  uv_prepare_t* h;
  QUEUE queue;
  QUEUE* q;
  QUEUE_MOVE(&loop->prepare_handles, &queue);
  while (!QUEUE_EMPTY(&queue))
  {
    q = QUEUE_HEAD(&queue);
    h = QUEUE_DATA(q, uv_prepare_t, queue);
    QUEUE_REMOVE(q);
    QUEUE_INSERT_TAIL(&loop->prepare_handles, q);
    h->prepare_cb(h);
  }
}

void uv__prepare_close(uv_prepare_t* handle)
{
  uv_prepare_stop(handle);
}
```

# struct uv_check_t

```c
int uv_check_init(uv_loop_t* loop, uv_check_t* handle)
{
  uv__handle_init(loop, (uv_handle_t* )handle, UV_CHECK);
  handle->check_cb = NULL;
  return 0;
}

int uv_check_start(uv_check_t* handle, uv_check_cb cb)
{
  if (uv__is_active(handle))
    return 0;
  if (cb == NULL)
    return -EINVAL;
  QUEUE_INSERT_HEAD(&handle->loop->check_handles, &handle->queue);
  handle->check_cb = cb;
  uv__handle_start(handle);
  return 0;
}

int uv_check_stop(uv_check_t* handle)
{
  if (!uv__is_active(handle))
    return 0;
  QUEUE_REMOVE(&handle->queue);
  uv__handle_stop(handle);
  return 0;
}

void uv__run_check(uv_loop_t* loop)
{
  uv_check_t* h;
  QUEUE queue;
  QUEUE* q;
  QUEUE_MOVE(&loop->check_handles, &queue);
  while (!QUEUE_EMPTY(&queue))
  {
    q = QUEUE_HEAD(&queue);
    h = QUEUE_DATA(q, uv_check_t, queue);
    QUEUE_REMOVE(q);
    QUEUE_INSERT_TAIL(&loop->check_handles, q);
    h->check_cb(h);
  }
}

void uv__check_close(uv_check_t* handle)
{
  uv_check_stop(handle);
}
```

# struct uv_idle_t

```c
int uv_idle_init(uv_loop_t* loop, uv_idle_t* handle)
{
  uv__handle_init(loop, (uv_handle_t* )handle, UV_IDLE);
  handle->idle_cb = NULL;
  return 0;
}

int uv_idle_start(uv_idle_t* handle, uv_idle_cb cb)
{
  if (uv__is_active(handle))
    return 0;
  if (cb == NULL)
    return -EINVAL;
  QUEUE_INSERT_HEAD(&handle->loop->idle_handles, &handle->queue);
  handle->idle_cb = cb;
  uv__handle_start(handle);
  return 0;
}

int uv_idle_stop(uv_idle_t* handle)
{
  if (!uv__is_active(handle))
    return 0;
  QUEUE_REMOVE(&handle->queue);
  uv__handle_stop(handle);
  return 0;
}

void uv__run_idle(uv_loop_t* loop)
{
  uv_idle_t* h;
  QUEUE queue;
  QUEUE* q;
  QUEUE_MOVE(&loop->idle_handles, &queue);
  while (!QUEUE_EMPTY(&queue))
  {
    q = QUEUE_HEAD(&queue);
    h = QUEUE_DATA(q, uv_idle_t, queue);
    QUEUE_REMOVE(q);
    QUEUE_INSERT_TAIL(&loop->idle_handles, q);
    h->idle_cb(h);
  }
}

void uv__idle_close(uv_idle_t* handle)
{
  uv_idle_stop(handle);
}
```


