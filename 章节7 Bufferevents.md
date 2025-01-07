### Bufferevents：概念与基础

通常，一个应用程序除了响应事件外，还希望执行一定量的数据缓冲。当我们想要写入数据时，通常的模式如下：

1. 决定要将一些数据写入连接，将数据放入缓冲区。
2. 等待连接变为可写状态。
3. 尽可能多地写入数据。
4. 记住我们写入的数量，如果还有更多数据需要写入，则再次等待连接变为可写状态。

这种缓冲IO模式是相当常见的，因此Libevent提供了一种通用机制。**“bufferevent”**包含一个底层传输（如套接字）、一个读取缓冲区和一个写入缓冲区。与常规事件不同，bufferevent在读取或写入足够数据时调用用户提供的回调函数。

#### Bufferevent的类型

存在多种类型的bufferevent，它们都共享一个公共接口。根据目前的信息，以下类型的bufferevent可用：

1. **基于套接字的bufferevent**  
   一个从底层流套接字发送和接收数据的bufferevent，使用`event_*`接口作为其后端。

2. **异步IO bufferevent**  
   一个使用Windows IOCP接口从底层流套接字发送和接收数据的bufferevent。（仅限Windows；实验性）

3. **过滤bufferevent**  
   一个在将传入和传出的数据传递给底层bufferevent对象之前处理这些数据的bufferevent，例如，用于压缩或转换数据。

4. **配对bufferevent**  
   两个bufferevent相互传输数据。

> **注意**  
> 从Libevent 2.0.2-alpha开始，这些bufferevents接口在所有bufferevent类型之间并非完全正交。换句话说，并不是所有接口都能在所有bufferevent类型上工作。Libevent的开发人员打算在未来的版本中纠正这一点。

> **另外注意**  
> 当前，bufferevents仅适用于面向流的协议，如TCP。未来可能会支持面向数据报的协议，如UDP。

本节中的所有函数和类型在`event2/bufferevent.h`中声明。与evbuffers相关的函数特别声明在`event2/buffer.h`中；请参见下一章以获取有关这些内容的信息。

#### Bufferevents和evbuffers

每个bufferevent都有一个输入缓冲区和一个输出缓冲区。这些缓冲区的类型是“struct evbuffer”。当你有数据要写入bufferevent时，你将其添加到输出缓冲区；当bufferevent有数据供你读取时，你从输入缓冲区中提取它。

`evbuffer`接口支持许多操作；我们将在后面的部分中讨论它们。

#### 回调和水位线

每个bufferevent都有两个与数据相关的回调：读取回调和写入回调。默认情况下，当从底层传输中读取到任何数据时，读取回调被调用，而当输出缓冲区中的数据被排空到底层传输时，写入回调被调用。你可以通过调整bufferevent的读取和写入“水位线”来覆盖这些函数的行为。

每个bufferevent都有四个水位线：

1. **读取低水位线**  
   每当读取操作后的输入缓冲区水平高于或等于此水平时，将调用bufferevent的读取回调。默认值为0，因此每次读取都会导致读取回调被调用。

2. **读取高水位线**  
   如果bufferevent的输入缓冲区达到此水平，则bufferevent停止读取，直到从输入缓冲区中排出足够的数据以降低到此水平以下。默认值为无限制，因此我们永远不会因输入缓冲区的大小而停止读取。

3. **写入低水位线**  
   每当写入操作使我们达到此水平或以下时，将调用写入回调。默认值为0，因此只有在输出缓冲区被清空时才会调用写入回调。

4. **写入高水位线**  
   bufferevent不会直接使用此水位线，但当bufferevent作为另一个bufferevent的底层传输时，此水位线可能具有特殊含义。有关过滤bufferevent的说明如下。

bufferevent还有一个“错误”或“事件”回调，用于告知应用程序非数据导向事件，例如连接关闭或发生错误。定义了以下事件标志：

- **BEV_EVENT_READING**  
  在bufferevent上读取操作时发生了事件。请参见其他标志以了解发生了什么事件。

- **BEV_EVENT_WRITING**  
  在bufferevent上写入操作时发生了事件。请参见其他标志以了解发生了什么事件。

- **BEV_EVENT_ERROR**  
  在bufferevent操作期间发生错误。有关错误的详细信息，请调用`EVUTIL_SOCKET_ERROR()`。

- **BEV_EVENT_TIMEOUT**  
  bufferevent上的超时已过期。

- **BEV_EVENT_EOF**  
  我们在bufferevent上收到了文件结束指示。

- **BEV_EVENT_CONNECTED**  
  我们在bufferevent上完成了请求的连接。

（上述事件名称是在Libevent 2.0.2-alpha中引入的。）

#### 延迟回调

默认情况下，bufferevent的回调在相应条件发生时立即执行（evbuffer回调也是如此；稍后会讨论）。这种即时调用可能会在依赖关系复杂时导致问题。例如，假设有一个回调在evbuffer A变为空时将数据移动到A中，另一个回调在evbuffer A变满时处理数据。由于这些调用都发生在栈上，如果依赖关系足够复杂，你可能会面临栈溢出风险。

为了解决这个问题，你可以告诉bufferevent（或evbuffer）其回调应该延迟。当满足延迟回调的条件时，而不是立即调用它，它会作为`event_loop()`调用的一部分被排队，并在常规事件的回调之后被调用。

（延迟回调是在Libevent 2.0.1-alpha中引入的。）

#### Bufferevents的选项标志

在创建bufferevent时，你可以使用一个或多个标志来改变其行为。已识别的标志包括：

- **BEV_OPT_CLOSE_ON_FREE**  
  当bufferevent被释放时，关闭底层传输。这将关闭底层套接字，释放底层bufferevent等。

- **BEV_OPT_THREADSAFE**  
  自动为bufferevent分配锁，以便可以在多个线程中安全使用。

- **BEV_OPT_DEFER_CALLBACKS**  
  设置此标志时，bufferevent延迟所有回调，如上所述。

- **BEV_OPT_UNLOCK_CALLBACKS**  
  默认情况下，当bufferevent被设置为线程安全时，任何用户提供的回调被调用时，bufferevent的锁是被持有的。设置此选项会使Libevent在调用你的回调时释放bufferevent的锁。

（Libevent 2.0.5-beta引入了`BEV_OPT_UNLOCK_CALLBACKS`。上述其他选项在Libevent 2.0.1-alpha中是新的。）

#### 使用基于套接字的bufferevents

最简单的bufferevent是基于套接字的类型。基于套接字的bufferevent使用Libevent的底层事件机制来检测底层网络套接字何时准备好进行读/写操作，并使用底层网络调用（如`readv`、`writev`、`WSASend`或`WSARecv`）来传输和接收数据。
### 创建基于套接字的 bufferevent

您可以使用 `bufferevent_socket_new()` 创建一个基于套接字的 bufferevent：

#### 接口
```c
struct bufferevent *bufferevent_socket_new(
    struct event_base *base,
    evutil_socket_t fd,
    enum bufferevent_options options);
```
- **base**: 一个 `event_base`。
- **fd**: 套接字的可选文件描述符。您可以将 `fd` 设置为 -1，以便稍后设置文件描述符。
- **options**: 一个 bufferevent 选项的位掩码（例如，`BEV_OPT_CLOSE_ON_FREE` 等）。

**提示**：确保您提供给 `bufferevent_socket_new` 的套接字处于非阻塞模式。Libevent 提供了便利方法 `evutil_make_socket_nonblocking`。

该函数在成功时返回一个 `bufferevent`，失败时返回 `NULL`。

`bufferevent_socket_new()` 函数是在 Libevent 2.0.1-alpha 中引入的。

### 在基于套接字的 bufferevent 上发起连接

如果 bufferevent 的套接字尚未连接，您可以发起新的连接。

#### 接口
```c
int bufferevent_socket_connect(struct bufferevent *bev,
    struct sockaddr *address, int addrlen);
```
- **address** 和 **addrlen** 参数与标准的 `connect()` 调用相同。如果 bufferevent 还没有设置套接字，调用此函数会为其分配一个新的流套接字，并将其设置为非阻塞。
- 如果 bufferevent 已经有套接字，调用 `bufferevent_socket_connect()` 会告诉 Libevent 套接字尚未连接，直到连接操作成功前不应对该套接字进行读写。

在连接完成之前，可以向输出缓冲区添加数据。

该函数返回 0 表示连接成功启动，返回 -1 表示发生错误。

### 示例
```c
#include <event2/event.h>
#include <event2/bufferevent.h>
#include <sys/socket.h>
#include <string.h>

void eventcb(struct bufferevent *bev, short events, void *ptr)
{
    if (events & BEV_EVENT_CONNECTED) {
         /* 我们已连接到 127.0.0.1:8080。通常我们会在这里执行
            一些操作，例如开始读取或写入。 */
    } else if (events & BEV_EVENT_ERROR) {
         /* 连接时发生错误。 */
    }
}

int main_loop(void)
{
    struct event_base *base;
    struct bufferevent *bev;
    struct sockaddr_in sin;

    base = event_base_new();

    memset(&sin, 0, sizeof(sin));
    sin.sin_family = AF_INET;
    sin.sin_addr.s_addr = htonl(0x7f000001); /* 127.0.0.1 */
    sin.sin_port = htons(8080); /* 端口 8080 */

    bev = bufferevent_socket_new(base, -1, BEV_OPT_CLOSE_ON_FREE);

    bufferevent_setcb(bev, NULL, NULL, eventcb, NULL);

    if (bufferevent_socket_connect(bev,
        (struct sockaddr *)&sin, sizeof(sin)) < 0) {
        /* 启动连接时发生错误 */
        bufferevent_free(bev);
        return -1;
    }

    event_base_dispatch(base);
    return 0;
}
```

`bufferevent_base_connect()` 函数是在 Libevent-2.0.2-alpha 中引入的。在此之前，您必须手动调用 `connect()` 来连接套接字，当连接完成时，bufferevent 会报告为写操作。

请注意，只有在使用 `bufferevent_socket_connect()` 启动连接尝试时，您才会收到 `BEV_EVENT_CONNECTED` 事件。如果您自己调用 `connect()`，则连接将作为写操作进行报告。

如果您想自己调用 `connect()`，但在连接成功时仍然接收 `BEV_EVENT_CONNECTED` 事件，请在 `connect()` 返回 -1 且 `errno` 等于 `EAGAIN` 或 `EINPROGRESS` 时调用 `bufferevent_socket_connect(bev, NULL, 0)`。

此函数是在 Libevent 2.0.2-alpha 中引入的。

### 通过主机名发起连接

通常，您希望将解析主机名和连接到主机的操作结合为一个单独的操作。为此提供了一个接口：

#### 接口
```c
int bufferevent_socket_connect_hostname(struct bufferevent *bev,
    struct evdns_base *dns_base, int family, const char *hostname,
    int port);
int bufferevent_socket_get_dns_error(struct bufferevent *bev);
```
此函数解析 DNS 名称 `hostname`，查找类型为 `family` 的地址（允许的家庭类型为 `AF_INET`、`AF_INET6` 和 `AF_UNSPEC`）。如果名称解析失败，它将使用错误事件调用事件回调。如果成功，它将发起连接尝试，就像 `bufferevent_connect` 一样。

`dns_base` 参数是可选的。如果为 `NULL`，则 Libevent 在等待名称查找完成时会阻塞，这通常不是您想要的。如果提供了它，则 Libevent 将使用它异步查找主机名。有关 DNS 的更多信息，请参见 R9 章节。

与 `bufferevent_socket_connect()` 一样，此函数告诉 Libevent bufferevent 上的任何现有套接字尚未连接，直到解析完成并连接操作成功之前不应进行读写。

如果发生错误，可能是 DNS 主机名查找错误。您可以通过调用 `bufferevent_socket_get_dns_error()` 来查看最近的错误。 如果返回的错误代码为 0，则未检测到 DNS 错误。

### 示例：简单的 HTTP v0 客户端
```c
/* 不要实际复制这段代码：它实现 HTTP 客户端的方式很差。
   请查看 evhttp 以获取更好的实现。 */
#include <event2/dns.h>
#include <event2/bufferevent.h>
#include <event2/buffer.h>
#include <event2/util.h>
#include <event2/event.h>

#include <stdio.h>

void readcb(struct bufferevent *bev, void *ptr)
{
    char buf[1024];
    int n;
    struct evbuffer *input = bufferevent_get_input(bev);
    while ((n = evbuffer_remove(input, buf, sizeof(buf))) > 0) {
        fwrite(buf, 1, n, stdout);
    }
}

void eventcb(struct bufferevent *bev, short events, void *ptr)
{
    if (events & BEV_EVENT_CONNECTED) {
         printf("连接成功。\n");
    } else if (events & (BEV_EVENT_ERROR|BEV_EVENT_EOF)) {
         struct event_base *base = ptr;
         if (events & BEV_EVENT_ERROR) {
                 int err = bufferevent_socket_get_dns_error(bev);
                 if (err)
                         printf("DNS 错误: %s\n", evutil_gai_strerror(err));
         }
         printf("关闭\n");
         bufferevent_free(bev);
         event_base_loopexit(base, NULL);
    }
}

int main(int argc, char **argv)
{
    struct event_base *base;
    struct evdns_base *dns_base;
    struct bufferevent *bev;

    if (argc != 3) {
        printf("简单的 HTTP 0.x 客户端\n"
               "语法: %s [hostname] [resource]\n"
               "示例: %s www.google.com /\n",argv[0],argv[0]);
        return 1;
    }

    base = event_base_new();
    dns_base = evdns_base_new(base, 1);

    bev = bufferevent_socket_new(base, -1, BEV_OPT_CLOSE_ON_FREE);
    bufferevent_setcb(bev, readcb, NULL, eventcb, base);
    bufferevent_enable(bev, EV_READ|EV_WRITE);
    evbuffer_add_printf(bufferevent_get_output(bev), "GET %s\r\n", argv[2]);
    bufferevent_socket_connect_hostname(
        bev, dns_base, AF_UNSPEC, argv[1], 80);
    event_base_dispatch(base);
    return 0;
}
```

`bufferevent_socket_connect_hostname()` 函数在 Libevent 2.0.3-alpha 中引入；`bufferevent_socket_get_dns_error()` 在 2.0.5-beta 中引入。

### 通用 bufferevent 操作
本节中的函数适用于多种 bufferevent 实现。

### 释放 `bufferevent`
#### 接口
```c
void bufferevent_free(struct bufferevent *bev);
```
此函数用于释放一个 `bufferevent`。`bufferevent` 是内部引用计数的，因此如果在释放时存在待处理的回调，它不会立即被删除，直到这些回调完成后才会被释放。

`bufferevent_free()` 函数会尽可能尽快地释放 `bufferevent`。如果 `bufferevent` 上有待写入的数据，它可能不会在 `bufferevent` 被释放之前被冲刷（flush）。

如果设置了 `BEV_OPT_CLOSE_ON_FREE` 标志，并且此 `bufferevent` 关联了一个套接字或基础 `bufferevent` 作为其传输层，则在释放 `bufferevent` 时，该传输层将被关闭。

此函数在 Libevent 0.8 中引入。

---

### 操作回调、水位线和启用的操作
#### 接口
```c
typedef void (*bufferevent_data_cb)(struct bufferevent *bev, void *ctx);
typedef void (*bufferevent_event_cb)(struct bufferevent *bev, short events, void *ctx);

void bufferevent_setcb(struct bufferevent *bufev,
    bufferevent_data_cb readcb, bufferevent_data_cb writecb,
    bufferevent_event_cb eventcb, void *cbarg);

void bufferevent_getcb(struct bufferevent *bufev,
    bufferevent_data_cb *readcb_ptr,
    bufferevent_data_cb *writecb_ptr,
    bufferevent_event_cb *eventcb_ptr,
    void **cbarg_ptr);
```
`bufferevent_setcb()` 函数用于更改一个 `bufferevent` 的一个或多个回调。`readcb`、`writecb` 和 `eventcb` 分别在读取到足够数据、写入足够数据或发生事件时被调用。每个回调的第一个参数是发生事件的 `bufferevent`，最后一个参数是用户在 `bufferevent_callcb()` 中提供的 `cbarg` 值，可用于向回调传递数据。事件回调的 `events` 参数是事件标志的位掩码。

你可以通过传递 NULL 来禁用某个回调。请注意，所有回调函数共享一个单一的 `cbarg` 值，因此更改它会影响所有回调。

可以通过 `bufferevent_getcb()` 函数检索当前设置的回调，传递指针以设置 `*readcb_ptr`、`*writecb_ptr`、`*eventcb_ptr` 和 `*cbarg_ptr`，当前的回调参数字段。如果这些指针中的任何一个设置为 NULL，将被忽略。

`bufferevent_setcb()` 函数在 Libevent 1.4.4 中引入，类型名称 `bufferevent_data_cb` 和 `bufferevent_event_cb` 在 Libevent 2.0.2-alpha 中引入，`bufferevent_getcb()` 函数在 2.1.1-alpha 中添加。

#### 接口
```c
void bufferevent_enable(struct bufferevent *bufev, short events);
void bufferevent_disable(struct bufferevent *bufev, short events);
short bufferevent_get_enabled(struct bufferevent *bufev);
```
你可以启用或禁用 `bufferevent` 的 `EV_READ`、`EV_WRITE` 或 `EV_READ | EV_WRITE` 事件。当读取或写入未被启用时，`bufferevent` 不会尝试读取或写入数据。

当输出缓冲区为空时，没有必要禁用写操作：`bufferevent` 会自动停止写入，并在有数据可写时重新启动。同样，当输入缓冲区达到其高水位标记时，没有必要禁用读取操作：`bufferevent` 会自动停止读取，并在有空间可读时重新启动。

默认情况下，新创建的 `bufferevent` 启用写入，但不启用读取。

你可以调用 `bufferevent_get_enabled()` 来查看当前哪些事件在 `bufferevent` 上启用。

这些函数在 Libevent 0.8 中引入，除了 `bufferevent_get_enabled()`，它在 2.0.3-alpha 中引入。

#### 接口
```c
void bufferevent_setwatermark(struct bufferevent *bufev, short events, size_t lowmark, size_t highmark);
```
`bufferevent_setwatermark()` 函数调整单个 `bufferevent` 的读取水位线、写入水位线或两者。如果在 `events` 字段中设置了 `EV_READ`，则调整读取水位线；如果设置了 `EV_WRITE`，则调整写入水位线。

高水位线为 0 等同于“无限制”。

此函数首次在 Libevent 1.4.4 中提供。

---

### 示例代码
```c
#include <event2/event.h>
#include <event2/bufferevent.h>
#include <event2/buffer.h>
#include <event2/util.h>

#include <stdlib.h>
#include <errno.h>
#include <string.h>

struct info {
    const char *name;
    size_t total_drained;
};

void read_callback(struct bufferevent *bev, void *ctx)
{
    struct info *inf = ctx;
    struct evbuffer *input = bufferevent_get_input(bev);
    size_t len = evbuffer_get_length(input);
    if (len) {
        inf->total_drained += len;
        evbuffer_drain(input, len);
        printf("从 %s 中排空了 %lu 字节\n", inf->name, (unsigned long) len);
    }
}

void event_callback(struct bufferevent *bev, short events, void *ctx)
{
    struct info *inf = ctx;
    struct evbuffer *input = bufferevent_get_input(bev);
    int finished = 0;

    if (events & BEV_EVENT_EOF) {
        size_t len = evbuffer_get_length(input);
        printf("从 %s 收到关闭信号。我们排空了 %lu 字节，剩下 %lu 字节。\n", inf->name,
            (unsigned long)inf->total_drained, (unsigned long)len);
        finished = 1;
    }
    if (events & BEV_EVENT_ERROR) {
        printf("从 %s 收到错误: %s\n", inf->name, evutil_socket_error_to_string(EVUTIL_SOCKET_ERROR()));
        finished = 1;
    }
    if (finished) {
        free(ctx);
        bufferevent_free(bev);
    }
}

struct bufferevent *setup_bufferevent(void)
{
    struct bufferevent *b1 = NULL;
    struct info *info1;

    info1 = malloc(sizeof(struct info));
    info1->name = "buffer 1";
    info1->total_drained = 0;

    /* 在此处应设置 bufferevent 并确保它已连接... */

    /* 仅在缓冲区中有至少 128 字节数据时触发读取回调。 */
    bufferevent_setwatermark(b1, EV_READ, 128, 0);

    bufferevent_setcb(b1, read_callback, NULL, event_callback, info1);
    bufferevent_enable(b1, EV_READ); /* 开始读取. */
    return b1;
}
```

### 操作数据
#### 接口
```c
struct evbuffer *bufferevent_get_input(struct bufferevent *bufev);
struct evbuffer *bufferevent_get_output(struct bufferevent *bufev);
```
这两个函数非常强大，基本上返回输入和输出缓冲区。有关可以对 `evbuffer` 类型执行的所有操作的完整信息，请参见下一章。

请注意，应用程序只能从输入缓冲区删除数据，而只能向输出缓冲区添加数据。

如果因为数据不足而写入 `bufferevent` 被停滞（或因为过多数据而读取被停滞），则添加数据到输出缓冲区（或从输入缓冲区移除数据）将自动重新启动写入或读取。

这些函数在 Libevent 2.0.1-alpha 中引入。

### 接口

```c
int bufferevent_write(struct bufferevent *bufev, const void *data, size_t size);
int bufferevent_write_buffer(struct bufferevent *bufev, struct evbuffer *buf);
```

这两个函数将数据添加到 `bufferevent` 的输出缓冲区。调用 `bufferevent_write()` 将从内存 `data` 处添加 `size` 字节到输出缓冲区的末尾。调用 `bufferevent_write_buffer()` 将移除 `buf` 的所有内容并将其放置在输出缓冲区的末尾。如果成功，则两个函数均返回 `0`，如果发生错误，则返回 `-1`。

这两个函数自 Libevent 0.8 版本起就存在。

### 接口

```c
size_t bufferevent_read(struct bufferevent *bufev, void *data, size_t size);
int bufferevent_read_buffer(struct bufferevent *bufev, struct evbuffer *buf);
```

这两个函数从 `bufferevent` 的输入缓冲区移除数据。`bufferevent_read()` 函数从输入缓冲区移除最多 `size` 字节的数据，并将其存储到 `data` 的内存中。它返回实际移除的字节数。`bufferevent_read_buffer()` 函数则会排空输入缓冲区的所有内容并将其放入 `buf`；成功时返回 `0`，失败时返回 `-1`。

注意，在使用 `bufferevent_read()` 时，`data` 内存块必须有足够的空间以容纳 `size` 字节的数据。

`bufferevent_read()` 函数自 Libevent 0.8 版本起就存在；`bufferevent_read_buffer()` 函数是在 Libevent 2.0.1-alpha 版本中引入的。

### 示例代码

```c
#include <event2/bufferevent.h>
#include <event2/buffer.h>
#include <ctype.h>

void read_callback_uppercase(struct bufferevent *bev, void *ctx) {
    /* 此回调函数从 bev 的输入缓冲区每次移除 128 字节数据，转为大写后开始发送回去。
       (注意！实际使用中，你不应使用 toupper 来实现网络协议，除非你知道当前的区域设置是你想使用的。)
    */

    char tmp[128];
    size_t n;
    int i;
    while (1) {
        n = bufferevent_read(bev, tmp, sizeof(tmp));
        if (n <= 0)
            break; /* 没有更多数据。 */
        for (i = 0; i < n; ++i)
            tmp[i] = toupper(tmp[i]);
        bufferevent_write(bev, tmp, n);
    }
}

struct proxy_info {
    struct bufferevent *other_bev;
};

void read_callback_proxy(struct bufferevent *bev, void *ctx) {
    /* 如果你正在实现一个简单的代理，你可以使用这样的函数：它将从一个连接（在 bev 上）
       接收数据并将其写入另一个连接，尽可能少地复制数据。 */
    struct proxy_info *inf = ctx;

    bufferevent_read_buffer(bev, bufferevent_get_output(inf->other_bev));
}

struct count {
    unsigned long last_fib[2];
};

void write_callback_fibonacci(struct bufferevent *bev, void *ctx) {
    /* 这是一个回调，它向 bev 的输出缓冲区添加一些斐波那契数。当我们添加了 1k 的数据后将停止；
       一旦这些数据被排出，我们将添加更多。 */
    struct count *c = ctx;

    struct evbuffer *tmp = evbuffer_new();
    while (evbuffer_get_length(tmp) < 1024) {
        unsigned long next = c->last_fib[0] + c->last_fib[1];
        c->last_fib[0] = c->last_fib[1];
        c->last_fib[1] = next;

        evbuffer_add_printf(tmp, "%lu", next);
    }

    /* 现在我们将 tmp 的所有内容添加到 bev 中。 */
    bufferevent_write_buffer(bev, tmp);

    /* 我们不再需要 tmp。 */
    evbuffer_free(tmp);
}
```

### 读写超时

与其他事件一样，如果在指定的时间内未成功读取或写入任何数据，则可以触发超时。

### 接口

```c
void bufferevent_set_timeouts(struct bufferevent *bufev,
    const struct timeval *timeout_read, const struct timeval *timeout_write);
```

将超时设置为 `NULL` 应该会移除它；然而在 Libevent 2.1.2-alpha 之前，这在所有事件类型中并不适用。（对于旧版本，可以尝试将超时设置为多天的间隔，和/或让你的 `eventcb` 函数在你不想要超时事件时忽略 `BEV_TIMEOUT` 事件。）

当 `bufferevent` 在尝试读取时等待了至少 `timeout_read` 秒，读取超时将触发；当 `bufferevent` 在尝试写入数据时等待了至少 `timeout_write` 秒，写入超时将触发。

注意，超时仅在 `bufferevent` 想要读取或写入时计时。换句话说，当 `bufferevent` 禁用读取，或者输入缓冲区已满（达到其高水位线）时，读取超时不会启用。类似地，当写入被禁用，或者没有数据可以写时，写入超时也不会启用。

当读取或写入超时发生时，相应的读取或写入操作在 `bufferevent` 上将被禁用。然后，事件回调将以 `BEV_EVENT_TIMEOUT|BEV_EVENT_READING` 或 `BEV_EVENT_TIMEOUT|BEV_EVENT_WRITING` 的形式被调用。

此函数自 Libevent 2.0.1-alpha 版本起就存在。在 Libevent 2.0.4-alpha 版本之前，它在 `bufferevent` 类型之间的行为不一致。

### 强制刷新 bufferevent

### 接口

```c
int bufferevent_flush(struct bufferevent *bufev,
    short iotype, enum bufferevent_flush_mode state);
```

刷新 `bufferevent` 会告诉它尽可能强制读取或写入底层传输中的字节，忽略可能会阻止它们被写入的其他限制。其具体功能取决于 `bufferevent` 的类型。

`iotype` 参数应为 `EV_READ`、`EV_WRITE` 或 `EV_READ|EV_WRITE`，以指示要处理读取、写入或两者的字节。`state` 参数可以是 `BEV_NORMAL`、`BEV_FLUSH` 或 `BEV_FINISHED`。`BEV_FINISHED` 表示应告知另一方不再发送数据；`BEV_NORMAL` 和 `BEV_FLUSH` 之间的区别取决于 `bufferevent` 的类型。

`bufferevent_flush()` 函数在失败时返回 `-1`，如果没有数据被刷新则返回 `0`，如果某些数据被刷新则返回 `1`。

截至 Libevent 2.0.5-beta，`bufferevent_flush()` 仅针对某些 `bufferevent` 类型实现。特别是，基于套接字的 `bufferevent` 不支持该功能。

### 特定类型的 bufferevent 函数

这些 bufferevent 函数并不支持所有类型的 bufferevent。

#### 接口

```c
int bufferevent_priority_set(struct bufferevent *bufev, int pri);
int bufferevent_get_priority(struct bufferevent *bufev);
```

该函数调整用于实现 `bufev` 的事件的优先级为 `pri`。有关优先级的更多信息，请参见 `event_priority_set()`。

该函数在成功时返回 `0`，失败时返回 `-1`。它仅适用于基于套接字的 bufferevent。

`bufferevent_priority_set()` 函数是在 Libevent 1.0 中引入的；而 `bufferevent_get_priority()` 直到 Libevent 2.1.2-alpha 版本才出现。

#### 接口

```c
int bufferevent_setfd(struct bufferevent *bufev, evutil_socket_t fd);
evutil_socket_t bufferevent_getfd(struct bufferevent *bufev);
```

这些函数设置或返回 fd 基于事件的文件描述符。只有基于套接字的 bufferevent 支持 `setfd()`。两个函数在失败时均返回 `-1`；`setfd()` 在成功时返回 `0`。

`bufferevent_setfd()` 函数是在 Libevent 1.4.4 中引入的；而 `bufferevent_getfd()` 函数是在 Libevent 2.0.2-alpha 中引入的。

#### 接口

```c
struct event_base *bufferevent_get_base(struct bufferevent *bev);
```

该函数返回一个 bufferevent 的 `event_base`。它是在 2.0.9-rc 版本中引入的。

#### 接口

```c
struct bufferevent *bufferevent_get_underlying(struct bufferevent *bufev);
```

该函数返回一个 bufferevent，如果另一个 bufferevent 使用它作为传输。这种情况的详细信息，请参见有关过滤 bufferevent 的说明。

该函数是在 Libevent 2.0.2-alpha 中引入的。

### 手动锁定和解锁 bufferevent

与 evbuffers 一样，有时你希望确保对 bufferevent 的多个操作都是原子执行的。Libevent 提供了可以手动锁定和解锁 bufferevent 的函数。

#### 接口

```c
void bufferevent_lock(struct bufferevent *bufev);
void bufferevent_unlock(struct bufferevent *bufev);
```

请注意，如果在创建时没有为 bufferevent 指定 `BEV_OPT_THREADSAFE` 线程，或者没有启用 Libevent 的线程支持，则锁定 bufferevent 不会产生任何效果。

使用此函数锁定 bufferevent 时也会锁定其关联的 evbuffers。这些函数是递归的：可以安全地锁定你已经持有锁的 bufferevent。当然，对于每次锁定的 bufferevent，必须调用一次解锁。

这些函数是在 Libevent 2.0.6-rc 中引入的。

### 废弃的 bufferevent 功能

在 Libevent 1.4 和 Libevent 2.0 之间，bufferevent 底层代码经历了重大修订。在旧接口中，通常可以通过访问 `struct bufferevent` 的内部结构来构建，并使用依赖于此访问的宏。

为了使事情变得混乱，旧代码有时使用以“evbuffer”为前缀的名称来描述 bufferevent 的功能。

以下是 Libevent 2.0 之前一些功能的旧名称与新名称的对照：

| 当前名称                      | 旧名称         |
|-------------------------------|----------------|
| bufferevent_data_cb           | evbuffercb     |
| bufferevent_event_cb          | everrorcb      |
| BEV_EVENT_READING             | EVBUFFER_READ   |
| BEV_EVENT_WRITE               | EVBUFFER_WRITE  |
| BEV_EVENT_EOF                 | EVBUFFER_EOF    |
| BEV_EVENT_ERROR               | EVBUFFER_ERROR   |
| BEV_EVENT_TIMEOUT             | EVBUFFER_TIMEOUT |
| bufferevent_get_input(b)      | EVBUFFER_INPUT(b) |
| bufferevent_get_output(b)     | EVBUFFER_OUTPUT(b)|

旧函数定义在 `event.h` 中，而不是在 `event2/bufferevent.h` 中。

如果你仍然需要访问 bufferevent 结构的公共部分的内部信息，可以包含 `event2/bufferevent_struct.h`。我们不推荐这样做：`struct bufferevent` 的内容将在 Libevent 的版本之间发生变化。包含 `event2/bufferevent_compat.h` 可以访问本节中的宏和名称。

在较早的版本中，设置 bufferevent 的接口有所不同：

#### 接口

```c
struct bufferevent *bufferevent_new(evutil_socket_t fd,
    evbuffercb readcb, evbuffercb writecb, everrorcb errorcb, void *cbarg);
int bufferevent_base_set(struct event_base *base, struct bufferevent *bufev);
```

`bufferevent_new()` 函数仅创建一个套接字 bufferevent，并在已废弃的“默认” event_base 上执行。调用 `bufferevent_base_set` 仅调整套接字 bufferevent 的 event_base。

而且，超时不是作为 `struct timeval` 设置的，而是作为秒数设置的：

#### 接口

```c
void bufferevent_settimeout(struct bufferevent *bufev,
    int timeout_read, int timeout_write);
```

最后，请注意，Libevent 2.0 之前的底层 evbuffer 实现效率相当低，以至于将 bufferevent 用于高性能应用程序是值得怀疑的。

