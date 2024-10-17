## Bufferevents: 高级主题

本章介绍了 Libevent 的 bufferevent 实现的一些高级特性，这些特性并非典型使用所必需。如果您刚刚学习如何使用 bufferevents，建议先跳过本章，继续阅读 evbuffer 章节。

### 配对 Bufferevents

有时，您可能有一个网络程序需要与自身通信。例如，您可以编写一个程序，通过某种协议隧道用户连接，同时该程序可能还想通过该协议隧道自身的连接。当然，您可以通过打开一个与自己监听端口的连接，让您的程序使用自己来实现这一点，但这将浪费资源，因为程序要通过网络堆栈与自身通信。

相反，您可以创建一对配对的 bufferevents，使得写入其中一个的所有字节都能在另一个中接收（反之亦然），而无需实际使用平台套接字。

#### 接口

```c
int bufferevent_pair_new(struct event_base *base, int options,
    struct bufferevent *pair[2]);
```

调用 `bufferevent_pair_new()` 会将 `pair[0]` 和 `pair[1]` 设置为一对相互连接的 bufferevents。所有常用选项都受支持，除了 `BEV_OPT_CLOSE_ON_FREE`，该选项无效，`BEV_OPT_DEFER_CALLBACKS` 总是开启的。

**为什么配对 bufferevent 需要以延迟回调的方式运行？** 由于一对中的一个元素的操作通常会调用一个回调，从而更改 bufferevent，这将调用另一个 bufferevent 的回调，依此类推。若不延迟这些回调，这种调用链很可能会导致堆栈溢出、其他连接的饥饿，并要求所有回调可重入。

配对 bufferevents 支持刷新；将模式参数设置为 `BEV_NORMAL` 或 `BEV_FLUSH` 将强制将一对中的一个 bufferevent 的所有相关数据传输到另一个 bufferevent，忽略其他限制水位。将模式设置为 `BEV_FINISHED` 还会在对面 bufferevent 上生成 EOF 事件。

释放配对中的任何成员不会自动释放另一个成员或生成 EOF 事件；它仅使另一个成员变得不再链接。一旦 bufferevent 失去链接，它将不再成功读取或写入数据，也不会生成任何事件。

#### 接口

```c
struct bufferevent *bufferevent_pair_get_partner(struct bufferevent *bev);
```

有时您可能需要根据仅一个成员获取 bufferevent 配对的另一个成员。要实现这一点，可以调用 `bufferevent_pair_get_partner()` 函数。如果 `bev` 是一对中的一个成员，并且另一个成员仍然存在，则它将返回该成员；否则，返回 NULL。

配对 bufferevents 在 Libevent 2.0.1-alpha 中新增；`bufferevent_pair_get_partner()` 函数在 Libevent 2.0.6 中引入。

### 过滤 Bufferevents

有时，您希望转换通过 bufferevent 对象传递的所有数据。这可以用于添加压缩层，或在传输时将协议包装在另一个协议中。

#### 接口

```c
enum bufferevent_filter_result {
    BEV_OK = 0,
    BEV_NEED_MORE = 1,
    BEV_ERROR = 2
};

typedef enum bufferevent_filter_result (*bufferevent_filter_cb)(
    struct evbuffer *source, struct evbuffer *destination, ev_ssize_t dst_limit,
    enum bufferevent_flush_mode mode, void *ctx);

struct bufferevent *bufferevent_filter_new(struct bufferevent *underlying,
        bufferevent_filter_cb input_filter,
        bufferevent_filter_cb output_filter,
        int options,
        void (*free_context)(void *),
        void *ctx);
```

`bufferevent_filter_new()` 函数创建一个新的过滤 bufferevent，包装在现有的“基础” bufferevent 周围。所有通过基础 bufferevent 接收的数据都在到达过滤 bufferevent 之前经过“输入”过滤器转换，所有通过过滤 bufferevent 发送的数据在发送到基础 bufferevent 之前都经过“输出”过滤器转换。

向基础 bufferevent 添加过滤器将替换基础 bufferevent 的回调。您仍然可以向基础 bufferevent 的 evbuffer 添加回调，但如果要使过滤器继续工作，您无法设置 bufferevent 本身的回调。

输入过滤器和输出过滤器函数如下所述。所有常用选项都支持在选项中。如果设置了 `BEV_OPT_CLOSE_ON_FREE`，则释放过滤 bufferevent 也会释放基础 bufferevent。ctx 字段是传递给过滤函数的任意指针；如果提供了 free_context 函数，则在过滤 bufferevent 关闭之前，将对 ctx 调用该函数。

输入过滤器函数将在基础输入缓冲区有新可读数据时调用。输出过滤器函数将在过滤器的输出缓冲区有新可写数据时调用。每个函数接收一对 evbuffers：一个用于从中读取数据的源 evbuffer 和一个用于写入数据的目标 evbuffer。dst_limit 参数描述要添加到目标的字节上限。过滤器函数可以忽略此值，但这样做可能违反高水位或速率限制。如果 dst_limit 为 -1，则没有限制。mode 参数告诉过滤器写入的激进程度。如果是 `BEV_NORMAL`，则应尽可能方便地写入。如果是 `BEV_FLUSH`，则应尽可能多地写入，而 `BEV_FINISHED` 意味着过滤函数应在流的末尾执行任何必要的清理。最后，过滤函数的 ctx 参数是提供给 `bufferevent_filter_new()` 构造函数的 void 指针。

过滤函数必须返回 `BEV_OK`（如果成功写入任何数据到目标缓冲区），`BEV_NEED_MORE`（如果在获取更多输入或使用不同刷新模式之前无法将更多数据写入目标缓冲区），或 `BEV_ERROR`（如果在过滤器上发生不可恢复的错误）。

创建过滤器时启用基础 bufferevent 的读取和写入。您不需要自己管理读取/写入：当过滤器不想读取时，过滤器将为您挂起对基础 bufferevent 的读取。对于 2.0.8-rc 及更高版本，可以独立于过滤器启用/禁用对基础 bufferevent 的读取和写入。如果您这样做，可能会使过滤器无法成功获取所需数据。

您不需要同时指定输入过滤器和输出过滤器：您省略的任何过滤器都被替换为一个无转换数据的过滤器。

### 限制最大单次读写大小

默认情况下，bufferevents 不会在事件循环的每次调用中读取或写入最大可能的字节；这样做可能导致奇怪的不公平行为和资源饥饿。然而，默认值对于所有情况可能并不合理。

#### 接口

```c
int bufferevent_set_max_single_read(struct bufferevent *bev, size_t size);
int bufferevent_set_max_single_write(struct bufferevent *bev, size_t size);

ev_ssize_t bufferevent_get_max_single_read(struct bufferevent *bev);
ev_ssize_t bufferevent_get_max_single_write(struct bufferevent *bev);
```

这两个“设置”函数分别替换当前的读取和写入最大值。如果 size 值为 0 或大于 `EV_SSIZE_MAX`，则它们会将最大值设置为默认值。这些函数在成功时返回 0，在失败时返回 -1。

这两个“获取”函数分别返回当前的每次循环的读取和写入最大值。

这两个函数是在 2.1.1-alpha 中新增的。

### Bufferevents 和速率限制

一些程序希望限制任何单个 bufferevent 或一组 bufferevents 使用的带宽。Libevent 2.0.4-alpha 和 Libevent 2.0.5-alpha 添加了基本功能，可以对单个 bufferevent 设置上限，或将 bufferevents 分配到速率限制组中。

#### 速率限制模型

Libevent 的速率限制使用令牌桶算法来决定一次可以读取或写入多少字节。每个被速率限制的对象在任何给定时间都有一个“读取桶”和一个“写入桶”，它们的大小决定了该对象立即允许读取或写入的字节数。每个桶都有一个补充速率、一个最大突发大小和一个计时单位或“滴答”。每当计时单位过期时，桶将根据补充速率按比例补充——但如果其满容量超过最大突发大小，则会丢失任何多余字节。

因此，补充速率决定了对象发送或接收字节的最大平均速率，而突发大小决定了在单次突发中发送或接收的最大字节数。计时单位决定了流量的平滑程度。

### 设置 `bufferevent` 的速率限制

#### 接口

```c
#define EV_RATE_LIMIT_MAX EV_SSIZE_MAX

struct ev_token_bucket_cfg;
struct ev_token_bucket_cfg *ev_token_bucket_cfg_new(
        size_t read_rate, size_t read_burst,
        size_t write_rate, size_t write_burst,
        const struct timeval *tick_len);
        
void ev_token_bucket_cfg_free(struct ev_token_bucket_cfg *cfg);
int bufferevent_set_rate_limit(struct bufferevent *bev,
    struct ev_token_bucket_cfg *cfg);
```

`ev_token_bucket_cfg` 结构体表示用于限制单个 `bufferevent` 或一组 `bufferevent` 的读取和写入的配置值。要创建一个，调用 `ev_token_bucket_cfg_new` 函数，并提供最大平均读取速率、最大读取突发、最大写入速率、最大写入突发和 tick 的长度。如果 `tick_len` 参数为 NULL，则 tick 的长度默认为一秒。该函数可能在错误时返回 NULL。

请注意，`read_rate` 和 `write_rate` 参数以每个 tick 的字节为单位进行缩放。也就是说，如果 tick 为十分之一秒，而 `read_rate` 为 300，则最大平均读取速率为每秒 3000 字节。超过 `EV_RATE_LIMIT_MAX` 的速率和突发值不受支持。

要限制 `bufferevent` 的传输速率，调用 `bufferevent_set_rate_limit()` 并传入 `ev_token_bucket_cfg`。该函数成功时返回 0，失败时返回 -1。可以将任何数量的 `bufferevent` 设置为相同的 `ev_token_bucket_cfg`。要移除 `bufferevent` 的速率限制，调用 `bufferevent_set_rate_limit()`，将 `cfg` 参数传递为 NULL。

要释放 `ev_token_bucket_cfg`，请调用 `ev_token_bucket_cfg_free()`。请注意，在没有 `bufferevents` 使用 `ev_token_bucket_cfg` 之前，当前不安全进行此操作。

---

### 为一组 `bufferevent` 设置速率限制

如果希望限制其总带宽使用，可以将 `bufferevent` 分配给速率限制组。

#### 接口

```c
struct bufferevent_rate_limit_group;

struct bufferevent_rate_limit_group *bufferevent_rate_limit_group_new(
        struct event_base *base,
        const struct ev_token_bucket_cfg *cfg);
        
int bufferevent_rate_limit_group_set_cfg(
        struct bufferevent_rate_limit_group *group,
        const struct ev_token_bucket_cfg *cfg);
        
void bufferevent_rate_limit_group_free(struct bufferevent_rate_limit_group *);
        
int bufferevent_add_to_rate_limit_group(struct bufferevent *bev,
    struct bufferevent_rate_limit_group *g);
    
int bufferevent_remove_from_rate_limit_group(struct bufferevent *bev);
```

要构造一个速率限制组，调用 `bufferevent_rate_limit_group()`，并传入一个 `event_base` 和初始的 `ev_token_bucket_cfg`。可以使用 `bufferevent_add_to_rate_limit_group()` 和 `bufferevent_remove_from_rate_limit_group()` 将 `bufferevent` 添加到组中；这些函数成功时返回 0，错误时返回 -1。

一个 `bufferevent` 最多只能是一个速率限制组的成员。一个 `bufferevent` 可以同时拥有个体速率限制（通过 `bufferevent_set_rate_limit()` 设置）和组速率限制。当同时设置两个限制时，适用于每个 `bufferevent` 的较低限制将生效。

可以通过调用 `bufferevent_rate_limit_group_set_cfg()` 更改现有组的速率限制。该函数成功时返回 0，失败时返回 -1。`bufferevent_rate_limit_group_free()` 函数释放速率限制组并移除其所有成员。

从版本 2.0 开始，Libevent 的组速率限制尝试在总体上保持公平，但在非常短的时间尺度上可能会不公平。如果你对调度公平性有强烈关注，请帮助为未来的版本提供补丁。

---

### 检查当前的速率限制值

有时你的代码可能想要检查适用于给定 `bufferevent` 或组的当前速率限制。Libevent 提供了一些函数来执行此操作。

#### 接口

```c
ev_ssize_t bufferevent_get_read_limit(struct bufferevent *bev);
ev_ssize_t bufferevent_get_write_limit(struct bufferevent *bev);
ev_ssize_t bufferevent_rate_limit_group_get_read_limit(
        struct bufferevent_rate_limit_group *);
ev_ssize_t bufferevent_rate_limit_group_get_write_limit(
        struct bufferevent_rate_limit_group *);
```

上述函数返回 `bufferevent` 或组的读取或写入令牌桶的当前大小（以字节为单位）。请注意，如果 `bufferevent` 被强制超出其分配的值，这些值可能为负（刷新 `bufferevent` 可以做到这一点）。

#### 接口

```c
ev_ssize_t bufferevent_get_max_to_read(struct bufferevent *bev);
ev_ssize_t bufferevent_get_max_to_write(struct bufferevent *bev);
```

这些函数返回 `bufferevent` 当前愿意读取或写入的字节数，考虑到适用于 `bufferevent` 的速率限制、其速率限制组（如果有的话）以及 Libevent 整体施加的最大读取/写入限制值。

---

### 手动调整速率限制

对于具有非常复杂需求的程序，您可能希望调整令牌桶的当前值。例如，如果您的程序以某种方式生成流量，而不是通过 `bufferevent`。

#### 接口

```c
int bufferevent_decrement_read_limit(struct bufferevent *bev, ev_ssize_t decr);
int bufferevent_decrement_write_limit(struct bufferevent *bev, ev_ssize_t decr);
int bufferevent_rate_limit_group_decrement_read(
        struct bufferevent_rate_limit_group *grp, ev_ssize_t decr);
int bufferevent_rate_limit_group_decrement_write(
        struct bufferevent_rate_limit_group *grp, ev_ssize_t decr);
```

这些函数在 `bufferevent` 或速率限制组中递减当前的读取或写入桶。请注意，递减是有符号的：如果您想增加桶，请传递一个负值。

---

### 在速率限制组中设置最小份额

通常，您不想在每个 tick 中均匀地将速率限制组中的字节分配给所有 `bufferevent`。例如，如果您在速率限制组中有 10,000 个活动 `bufferevent`，并且每个 tick 有 10,000 字节可用用于写入，那么让每个 `bufferevent` 每个 tick 只写入 1 字节并不是高效的，因为系统调用和 TCP 头部的开销。

为了解决这个问题，每个速率限制组都有一个“最小份额”的概念。在上述情况下，而不是允许每个 `bufferevent` 每个 tick 写入 1 字节，允许 10,000/SHARE 个 `bufferevent` 每个 tick 写入 SHARE 字节，其余的将无法写入。每个 tick 首先允许写入的 `bufferevent` 是随机选择的。

默认的最小份额选择为给出合理性能，并且目前（截至 2.0.6-rc）设置为 64。可以通过以下函数调整此值：

#### 接口

```c
int bufferevent_rate_limit_group_set_min_share(
        struct bufferevent_rate_limit_group *group, size_t min_share);
```

将 `min_share` 设置为 0 会完全禁用最小份额代码。

Libevent 的速率限制自首次引入以来一直具有最小份额。更改最小份额的函数首次在 Libevent 2.0.6-rc 中公开。

---

以下是关于速率限制实现的局限性的中文翻译：

---

### 速率限制实现的局限性

截至 Libevent 2.0，您应该了解速率限制实现的一些局限性：

1. **不是所有的 bufferevent 类型都支持速率限制，或者支持得不好。**

2. **bufferevent 速率限制组不能嵌套，一个 bufferevent 一次只能属于一个速率限制组。**

3. **速率限制实现仅计算在 TCP 数据包中传输的字节作为数据，不包括 TCP 头部。**

4. **读取限制的实现依赖于 TCP 堆栈注意到应用程序以某种速率仅消耗数据，并在其缓冲区满时向 TCP 连接的另一侧施加回退。**

5. **某些 bufferevent 的实现（特别是 Windows IOCP 实现）可能会过度承诺。**

6. **桶从一个完整的 tick 的流量开始。这意味着 bufferevent 可以立即开始读取或写入，而无需等待完整的 tick 过去。这也意味着，经过 N.1 个 tick 速率限制的 bufferevent 可能会转移 N+1 个 tick 的流量。**

7. **tick 不能小于 1 毫秒，且所有小于 1 毫秒的分数都会被忽略。**

---

### 示例待写

/// TODO: 编写速率限制的示例

### Bufferevents 和 SSL

Bufferevents 可以使用 OpenSSL 库来实现 SSL/TLS 安全传输层。由于许多应用程序不需要或不想链接 OpenSSL，因此该功能实现为单独的库，称为 "libevent_openssl"。未来的 Libevent 版本可能会支持其他 SSL/TLS 库，如 NSS 或 GnuTLS，但目前只有 OpenSSL 可用。

OpenSSL 功能是在 Libevent 2.0.3-alpha 中引入的，但在 Libevent 2.0.5-beta 或 Libevent 2.0.6-rc 之前效果不佳。

本节不是 OpenSSL、SSL/TLS 或一般密码学的教程。

这些函数都在头文件 "event2/bufferevent_ssl.h" 中声明。

### 设置和使用基于 OpenSSL 的 bufferevent

#### 接口

```c
enum bufferevent_ssl_state {
        BUFFEREVENT_SSL_OPEN = 0,
        BUFFEREVENT_SSL_CONNECTING = 1,
        BUFFEREVENT_SSL_ACCEPTING = 2
};
```

```c
struct bufferevent *
bufferevent_openssl_filter_new(struct event_base *base,
    struct bufferevent *underlying,
    SSL *ssl,
    enum bufferevent_ssl_state state,
    int options);
```

```c
struct bufferevent *
bufferevent_openssl_socket_new(struct event_base *base,
    evutil_socket_t fd,
    SSL *ssl,
    enum bufferevent_ssl_state state,
    int options);
```

您可以创建两种类型的 SSL bufferevent：一种是基于过滤器的 bufferevent，它通过另一个底层 bufferevent 进行通信；另一种是基于套接字的 bufferevent，它直接告诉 OpenSSL 通过网络进行通信。在这两种情况下，您都必须提供一个 SSL 对象以及对该 SSL 对象状态的描述。状态应为 BUFFEREVENT_SSL_CONNECTING（如果 SSL 正在作为客户端执行协商）、BUFFEREVENT_SSL_ACCEPTING（如果 SSL 正在作为服务器执行协商）或 BUFFEREVENT_SSL_OPEN（如果 SSL 握手已完成）。

通常接受的选项包括：`BEV_OPT_CLOSE_ON_FREE` 会在 openssl bufferevent 自身关闭时关闭 SSL 对象和底层的 fd 或 bufferevent。

一旦握手完成，新的 bufferevent 的事件回调将会以 `BEV_EVENT_CONNECTED` 的标志被调用。

如果您正在创建一个基于套接字的 bufferevent，并且 SSL 对象已经设置了一个套接字，则不需要自己提供套接字：只需传递 -1。您也可以稍后使用 `bufferevent_setfd()` 设置 fd。

> **TODO**: 一旦 `bufferevent_shutdown()` API 完成，请删除此注释。

请注意，当在 SSL bufferevent 上设置 `BEV_OPT_CLOSE_ON_FREE` 时，SSL 连接将不会进行干净的关闭。这会产生两个问题：首先，连接将被认为是被对方“破坏”的，而不是被干净关闭；另一方将无法判断您是关闭了连接，还是被攻击者或第三方破坏。其次，OpenSSL 会将会话视为“坏的”，并从会话缓存中移除。这会导致在负载下 SSL 应用程序的显著性能下降。

目前唯一的解决方法是手动执行懒惰的 SSL 关闭。虽然这违反了 TLS RFC，但它将确保会话在关闭后仍然保留在缓存中。以下代码实现了这一解决方法。

### 示例代码

```c
SSL *ctx = bufferevent_openssl_get_ssl(bev);

/*
 * SSL_RECEIVED_SHUTDOWN 告诉 SSL_shutdown 作用如同我们已经
 * 收到了来自另一端的关闭通知。SSL_shutdown 然后将
 * 发送最终的关闭通知作为回复。另一端将接收关闭通知并发送他们的关闭通知。
 * 在此之前，我们已经关闭了套接字，而另一端的真正关闭通知将永远不会被接收。
 * 实际上，双方将认为它们已完成干净的关闭，并保持其会话有效。
 * 如果套接字未准备好写入，这一策略将失败，
 * 在这种情况下，此 hack 将导致不干净的关闭和另一端的会话丢失。
 */
SSL_set_shutdown(ctx, SSL_RECEIVED_SHUTDOWN);
SSL_shutdown(ctx);
bufferevent_free(bev);
```

### 接口

```c
SSL *bufferevent_openssl_get_ssl(struct bufferevent *bev);
```

此函数返回由 OpenSSL bufferevent 使用的 SSL 对象；如果 bev 不是基于 OpenSSL 的 bufferevent，则返回 NULL。

### 接口

```c
unsigned long bufferevent_get_openssl_error(struct bufferevent *bev);
```

此函数返回给定 bufferevent 操作的第一个待处理 OpenSSL 错误；如果没有待处理的错误，则返回 0。错误格式与 OpenSSL 库中的 `ERR_get_error()` 返回的格式相同。

### 接口

```c
int bufferevent_ssl_renegotiate(struct bufferevent *bev);
```

调用此函数将告知 SSL 进行重新协商，并使 bufferevent 调用适当的回调。这是一个高级主题；您通常应该避免使用，特别是因为许多 SSL 版本已知存在与重新协商相关的安全问题。

### 接口

```c
int bufferevent_openssl_get_allow_dirty_shutdown(struct bufferevent *bev);
void bufferevent_openssl_set_allow_dirty_shutdown(struct bufferevent *bev, int allow_dirty_shutdown);
```

所有良好的 SSL 协议版本（即 SSLv3 和所有 TLS 版本）都支持一种经过验证的关闭操作，允许各方区分有意关闭与意外或恶意引起的底层缓冲区终止。默认情况下，我们将任何不正确的关闭视为连接错误。然而，如果 `allow_dirty_shutdown` 标志设置为 1，我们将把连接中的关闭视为 `BEV_EVENT_EOF`。

`allow_dirty_shutdown` 函数是在 Libevent 2.1.1-alpha 中添加的。

### 示例：简单的基于 SSL 的回显服务器

```c
/* 使用 OpenSSL bufferevents 的简单回显服务器 */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

#include <openssl/ssl.h>
#include <openssl/err.h>
#include <openssl/rand.h>

#include <event.h>
#include <event2/listener.h>
#include <event2/bufferevent_ssl.h>

static void ssl_readcb(struct bufferevent *bev, void *arg)
{
    struct evbuffer *in = bufferevent_get_input(bev);

    printf("接收到 %zu 字节\n", evbuffer_get_length(in));
    printf("----- 数据 ----\n");
    printf("%.*s\n", (int)evbuffer_get_length(in), evbuffer_pullup(in, -1));

    bufferevent_write_buffer(bev, in);
}

static void ssl_acceptcb(struct evconnlistener *serv, int sock, struct sockaddr *sa,
                         int sa_len, void *arg)
{
    struct event_base *evbase;
    struct bufferevent *bev;
    SSL_CTX *server_ctx;
    SSL *client_ctx;

    server_ctx = (SSL_CTX *)arg;
    client_ctx = SSL_new(server_ctx);
    evbase = evconnlistener_get_base(serv);

    bev = bufferevent_openssl_socket_new(evbase, sock, client_ctx,
                                         BUFFEREVENT_SSL_ACCEPTING,
                                         BEV_OPT_CLOSE_ON_FREE);

    bufferevent_enable(bev, EV_READ);
    bufferevent_setcb(bev, ssl_readcb, NULL, NULL, NULL);
}

static SSL_CTX *evssl_init(void)
{
    SSL_CTX *server_ctx;

    /* 初始化 OpenSSL 库 */
    SSL_load_error_strings();
    SSL_library_init();
    /* 我们必须有熵，否则加密没有意义。 */
    if (!RAND_poll())
        return NULL;

    server_ctx = SSL_CTX_new(SSLv23_server_method());

    if (!SSL_CTX_use_certificate_chain_file(server_ctx, "cert") ||
        !SSL_CTX_use_PrivateKey_file(server_ctx, "pkey", SSL_FILETYPE_PEM)) {
        puts("无法读取 'pkey' 或 'cert' 文件。要生成密钥\n"
             "和自签名证书，请运行：\n"
             "  openssl genrsa -out pkey 2048\n"
             "  openssl req -new -key pkey -out cert.req\n"
             "  openssl x509 -req -days 365 -in cert.req -signkey pkey -out cert");
        return NULL;
    }
    SSL_CTX_set_options(server_ctx, SSL_OP_NO_SSLv2);

    return server_ctx;
}

int main(int argc, char **argv)
{
    SSL_CTX *ctx;
    struct evconnlistener *listener;
    struct event_base *evbase;
    struct sockaddr_in sin;

    memset(&sin, 0, sizeof(sin));
    sin.sin_family = AF_INET;
    sin.sin_port = htons(9999);
    sin.sin_addr.s_addr = htonl(0x7f000001); /* 127.0.0.1 */

    ctx = evssl_init();
    if (ctx == NULL)
        return 1;
    evbase = event_base_new();
    listener = evconnlistener_new_bind(
        evbase, ssl_acceptcb, (void *)ctx,
        LEV_OPT_CLOSE_ON_FREE | LEV_OPT_REUSEABLE, 1024,
        (struct sockaddr *)&sin, sizeof(sin));

    event_base_loop(evbase, 0);

    evconnlistener_free(listener);
    SSL_CTX_free(ctx);

    return 0;
}
```

### 关于线程和 OpenSSL 的一些说明

Libevent 的内置线程机制并不涵盖 OpenSSL 的锁定。由于 OpenSSL 使用了大量全局变量，您仍然必须配置 OpenSSL 以确保线程安全。虽然这个过程超出了 Libevent 的范围，但由于这个话题经常出现，值得讨论。

### 示例：如何启用线程安全的 OpenSSL

```c
/*
 * 请参阅 OpenSSL 文档以验证您这样做是否正确，
 * Libevent 不保证此代码是完整的，仅作为示例使用。
 */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include <pthread.h>
#include <openssl/ssl.h>
#include <openssl/crypto.h>

pthread_mutex_t *ssl_locks;
int ssl_num_locks;

/* 实现 OpenSSL 要求的线程 ID 函数 */
static unsigned long get_thread_id_cb(void)
{
    return (unsigned long)pthread_self();
}

static void thread_lock_cb(int mode, int which, const char *f, int l)
{
    if (which < ssl_num_locks) {
        if (mode & CRYPTO_LOCK) {
            pthread_mutex_lock(&(ssl_locks[which]));
        } else {
            pthread_mutex_unlock(&(ssl_locks[which]));
        }
    }
}

int init_ssl_locking(void)
{
    int i;

    ssl_num_locks = CRYPTO_num_locks();
    ssl_locks = malloc(ssl_num_locks * sizeof(pthread_mutex_t));
    if (ssl_locks == NULL)
        return -1;

    for (i = 0; i < ssl_num_locks; i++) {
        pthread_mutex_init(&(ssl_locks[i]), NULL);
    }

    CRYPTO_set_id_callback(get_thread_id_cb);
    CRYPTO_set_locking_callback(thread_lock_cb);

    return 0;
}
```
