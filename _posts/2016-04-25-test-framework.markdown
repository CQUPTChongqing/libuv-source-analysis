---
layout: post
title:  "test framework"
date:   2016-04-25 18:14:10 +0800
categories: pages update
---

# data structure 

```c
/*
 * Struct to store both tests and to define helper processes for tasks.
 */
typedef struct {
  char *task_name;
  char *process_name;
  int (*main)(void);
  int is_helper;
  int show_output;

  /*
   * The time in milliseconds after which a single test or benchmark times out.
   */
  int timeout;
} task_entry_t, bench_entry_t;


task_entry_t TASKS[] = {
    { platform_output, platform_output, run_test_platform_output, 0, 1, 5000 },
    { close_order, close_order, run_test_close_order, 0, 0, 5000 },
    ... ...
    { pipe_ping_pong, pipe_ping_pong, run_test_pipe_ping_pong, 0, 0, 5000 },
    { pipe_ping_pong, pipe_echo_server, run_helper_pipe_echo_server, 1, 0, 5000 },
    ... ...
    { tcp_ping_pong, tcp_ping_pong, run_test_tcp_ping_pong, 0, 0, 5000 },
    { tcp_ping_pong, tcp4_echo_server, run_helper_tcp4_echo_server, 1, 0, 5000 },
    ... ...
    { udp_open, udp_open, run_test_udp_open, 0, 0, 5000 },
    { udp_open, udp4_echo_server, run_helper_udp4_echo_server, 1, 0, 5000 },
    ... ...
    { 0, 0, 0, 0, 0, 0 }
};
```

# program entrance 1

[test/run-tests.c:main()](https://github.com/libuv/libuv/blob/v1.x/test/run-tests.c#L49)

```c
/* Actual tests and helpers are defined in test-list.h */
#include "test-list.h"  // define TASKS

... ...

int main(int argc, char **argv) {
  if (platform_init(argc, argv))
    return EXIT_FAILURE;

  argv = uv_setup_args(argc, argv);

  switch (argc) {
  case 1: return run_tests(0);
  case 2: return maybe_run_test(argc, argv);
  case 3: return run_test_part(argv[1], argv[2]); // argv[1]:task->task_name, argv[2]:task->process_name.
  default:
    fprintf(stderr, "Too many arguments.\n");
    fflush(stderr);
    return EXIT_FAILURE;
  }

  return EXIT_SUCCESS;
}
```

# program entrance 2

[test/run-benchmarks.c:main()](https://github.com/libuv/libuv/blob/v1.x/test/run-benchmarks.c#L35)

```c
/* Actual benchmarks and helpers are defined in benchmark-list.h */
#include "benchmark-list.h"  // define TASKS

... ...

int main(int argc, char **argv) {
  if (platform_init(argc, argv))
    return EXIT_FAILURE;

  switch (argc) {
  case 1: return run_tests(1);
  case 2: return maybe_run_test(argc, argv);
  case 3: return run_test_part(argv[1], argv[2]);
  default:
    fprintf(stderr, "Too many arguments.\n");
    fflush(stderr);
    return EXIT_FAILURE;
  }

  return EXIT_SUCCESS;
}
```



# test samples

* [echo-server.c](https://github.com/libuv/libuv/blob/v1.x/test/echo-server.c)
* [test-ping-pong.c](https://github.com/libuv/libuv/blob/v1.x/test/test-ping-pong.c)

run test samples:

* ./libuv-test pipe_ping_pong pipe_echo_server   // run the pipe echo server.
* ./libuv-test pipe_ping_pong pipe_ping_pong     // run the pipe ping pong client, ping 1000 times.

notes:

1. [test/echo-server.c:after_read()](https://github.com/libuv/libuv/blob/v1.x/test/echo-server.c#L81)

```c
static void after_read(uv_stream_t* handle,
                       ssize_t nread,
                       const uv_buf_t* buf) {
  int i;
  write_req_t *wr;
  uv_shutdown_t* sreq;

  if (nread < 0) {
    /* Error or EOF */
    ASSERT(nread == UV_EOF);

    free(buf->base);
    sreq = malloc(sizeof* sreq);
    ASSERT(0 == uv_shutdown(sreq, handle, after_shutdown));
    return;
  }

  ... ...
}
```

**ASSERT(nread == UV_EOF);** nread is the error no after calling recv(), here is just in order to achieve the test automation. But in fact, it will be equal to a lot of error values.




