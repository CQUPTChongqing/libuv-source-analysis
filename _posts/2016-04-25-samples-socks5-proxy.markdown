---
layout: post
title:  "samples:socks5-proxy"
date:   2016-04-25 18:14:10 +0800
categories: pages update
---

# socks5-proxy features

1. A socks proxy service via protocol Socks5.

# test

1. use Firefox as the client:

![]({{ "/jpg/socks5_settings.jpg" | prepend: site.baseurl }})

# socks5-proxy state

![]({{ "/jpg/socks5-proxy_state.jpg" | prepend: site.baseurl }})


# data structure

```c
struct client_ctx;

typedef struct {
  const char *bind_host;
  unsigned short bind_port;
  unsigned int idle_timeout;
} server_config;

typedef struct {
  unsigned int idle_timeout;  /* Connection idle timeout in ms. */
  uv_tcp_t tcp_handle;
  uv_loop_t *loop;
} server_ctx;

typedef struct {
  unsigned char rdstate;
  unsigned char wrstate;
  unsigned int idle_timeout;
  struct client_ctx *client;  /* Backlink to owning client context. */
  ssize_t result;
  union {
    uv_handle_t handle;
    uv_stream_t stream;
    uv_tcp_t tcp;
    uv_udp_t udp;
  } handle;
  uv_timer_t timer_handle;  /* For detecting timeouts. */
  uv_write_t write_req;
  /* We only need one of these at a time so make them share memory. */
  union {
    uv_getaddrinfo_t addrinfo_req;
    uv_connect_t connect_req;
    uv_req_t req;
    struct sockaddr_in6 addr6;
    struct sockaddr_in addr4;
    struct sockaddr addr;
    char buf[2048];  /* Scratch space. Used to read data into. */
  } t;
} conn;

typedef struct client_ctx {
  unsigned int state;
  server_ctx *sx;  /* Backlink to owning server context. */
  s5_ctx parser;  /* The SOCKS protocol parser. */
  conn incoming;  /* Connection with the SOCKS client. */
  conn outgoing;  /* Connection with upstream. */
} client_ctx;
```



# socks proxy

What is socks proxy?  
Socks proxy is versatile proxy for all usage while the http proxy can only be used for surfing. You can use socks proxy to send email, transfer file, chat online, play game as well as surf website. Here is an article about socks proxy and http proxy.

How to use the socks proxy?  
Many programs (such as Firefox, IE, Skype, mIRC) support the socks proxy option. When you enable socks proxy in the program, the proxy will fetch the traffic data for the program. The server will regard the IP of socks proxy as your IP so it cannot trace your real IP. We recommend using Socks Proxy Checker to check socks proxies.

What is the version of socks proxy?  
There are two versions of socks proxy: version 4 and 5, as known as Socks4 and Socks5. Socks5 supports UDP (Skype needs it), name resolution and authentication while Socks4 not. However in most cases either Socks4 or Socks5 is OK.

What is the proxy anonymity?  
All the socks proxies are highly anonymous. The server does not know you are using a proxy. Http proxies are different. They can be anonymous or transparent.

How to check the proxy speed?  
You can use our free software [Socks Proxy Checker](http://www.socksproxychecker.com/) to test the proxy speed. We don't show the speed in the proxy list. It's because one proxy may have different speed for different users. For example, a proxy which is fast for USA users may be slow for European users. So check it by yourself.

What is HTTPS / SSL proxy?  
Secure websites whose url starts with https:// instead of http:// (ex. https://www.paypal.com) use the encrypted (SSL/HTTPS) connections between its web server and the visitors. All socks proxies support SSL/HTTPS.



