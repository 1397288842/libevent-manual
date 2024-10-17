### 使用 Libevent 进行 DNS 解析：高层和低层功能

Libevent 提供了一些 API 用于解析 DNS 名称，并提供了实现简单 DNS 服务器的功能。

我们将首先描述高层名称查找功能，然后再描述低层和服务器功能。

**注意**  
Libevent 当前的 DNS 客户端实现存在已知的限制。它不支持 TCP 查找、DNSSec 或任意记录类型。我们希望在未来的 Libevent 版本中修复所有这些问题，但目前它们尚未实现。

### 准备工作：可移植的阻塞名称解析

为了帮助移植已经使用阻塞名称解析的程序，Libevent 提供了标准 `getaddrinfo()` 接口的可移植实现。当您的程序需要在没有 `getaddrinfo()` 函数的系统上运行，或 `getaddrinfo()` 的实现不符合标准时，这将非常有用（这两种情况的数量都令人震惊）。

`getaddrinfo()` 接口在 RFC 3493 的第 6.1 节中进行了规定。有关我们如何未能实现符合规范的实现的摘要，请参见“兼容性说明”部分。

### 接口

```c
struct evutil_addrinfo {
    int ai_flags;
    int ai_family;
    int ai_socktype;
    int ai_protocol;
    size_t ai_addrlen;
    char *ai_canonname;
    struct sockaddr *ai_addr;
    struct evutil_addrinfo *ai_next;
};
```

#### 标志定义
```c
#define EVUTIL_AI_PASSIVE     /* ... */
#define EVUTIL_AI_CANONNAME   /* ... */
#define EVUTIL_AI_NUMERICHOST /* ... */
#define EVUTIL_AI_NUMERICSERV /* ... */
#define EVUTIL_AI_V4MAPPED    /* ... */
#define EVUTIL_AI_ALL         /* ... */
#define EVUTIL_AI_ADDRCONFIG  /* ... */
```

### 函数原型
```c
int evutil_getaddrinfo(const char *nodename, const char *servname,
    const struct evutil_addrinfo *hints, struct evutil_addrinfo **res);
void evutil_freeaddrinfo(struct evutil_addrinfo *ai);
const char *evutil_gai_strerror(int err);
```

`evutil_getaddrinfo()` 函数尝试根据您提供的 `hints` 规则解析给定的 `nodename` 和 `servname` 字段，并构建一个 `evutil_addrinfo` 结构的链表，将其存储在 `*res` 中。成功时返回 0，失败时返回非零错误代码。

您必须提供 `nodename` 和 `servname` 中的至少一个。如果提供了 `nodename`，它可以是字面量 IPv4 地址（例如 "127.0.0.1"）、字面量 IPv6 地址（例如 "::1"）或 DNS 名称（例如 "www.example.com"）。如果提供了 `servname`，它可以是网络服务的符号名称（例如 "https"）或一个以十进制表示的端口号的字符串（例如 "443"）。

如果未指定 `servname`，则 `*res` 中的端口值将被设置为零。如果未指定 `nodename`，则 `*res` 中的地址将默认为本地主机（localhost），或者在设置了 `EVUTIL_AI_PASSIVE` 的情况下为“任何”（any）。

`hints` 的 `ai_flags` 字段告诉 `evutil_getaddrinfo` 如何执行查找。它可以包含零个或多个以下标志，进行 OR 操作。

- **EVUTIL_AI_PASSIVE**  
  此标志表示我们将使用地址进行监听，而不是连接。通常这没有区别，除非 `nodename` 为 NULL：对于连接，NULL 的 `nodename` 是本地主机（127.0.0.1 或 ::1），而在监听时，NULL 的 `nodename` 是 ANY（0.0.0.0 或 ::0）。

- **EVUTIL_AI_CANONNAME**  
  如果设置了此标志，我们尝试在 `ai_canonname` 字段中报告主机的规范名称。

- **EVUTIL_AI_NUMERICHOST**  
  当设置此标志时，我们仅解析数字 IPv4 和 IPv6 地址；如果 `nodename` 需要进行名称查找，我们将返回 `EVUTIL_EAI_NONAME` 错误。

- **EVUTIL_AI_NUMERICSERV**  
  当设置此标志时，我们仅解析数字服务名称。如果 `servname` 既不是 NULL 也不是十进制整数，则返回 `EVUTIL_EAI_NONAME` 错误。

- **EVUTIL_AI_V4MAPPED**  
  此标志表示如果 `ai_family` 是 `AF_INET6`，并且未找到 IPv6 地址，则结果中的任何 IPv4 地址应作为 v4 映射的 IPv6 地址返回。除非操作系统支持，否则 `evutil_getaddrinfo()` 当前不支持此功能。

- **EVUTIL_AI_ALL**  
  如果同时设置了此标志和 `EVUTIL_AI_V4MAPPED`，则结果中包含的 IPv4 地址作为 4 映射的 IPv6 地址，无论是否有任何 IPv6 地址。除非操作系统支持，否则 `evutil_getaddrinfo()` 当前不支持此功能。

- **EVUTIL_AI_ADDRCONFIG**  
  如果设置了此标志，则仅在系统具有非本地 IPv4 地址的情况下包括 IPv4 地址，只有在系统具有非本地 IPv6 地址的情况下才包括 IPv6 地址。

`hints` 的 `ai_family` 字段用于告诉 `evutil_getaddrinfo()` 应返回哪些地址。它可以是 `AF_INET` 以请求仅 IPv4 地址，`AF_INET6` 以请求仅 IPv6 地址，或 `AF_UNSPEC` 以请求所有可用地址。

`ai_socktype` 和 `ai_protocol` 字段用于告诉 `evutil_getaddrinfo()` 您将如何使用该地址。它们与您传递给 `socket()` 的 socktype 和 protocol 字段相同。

如果 `evutil_getaddrinfo()` 成功，它会分配一个新的 `evutil_addrinfo` 结构的链表，每个结构通过其 `ai_next` 指针指向下一个结构，并将其存储在 `*res` 中。因为这个值是动态分配的，您需要使用 `evutil_freeaddrinfo` 来释放它。

如果失败，则返回以下数字错误代码之一：

- **EVUTIL_EAI_ADDRFAMILY**  
  您请求的地址族与 `nodename` 不兼容。

- **EVUTIL_EAI_AGAIN**  
  名称解析时出现可恢复错误；请稍后再试。

- **EVUTIL_EAI_FAIL**  
  名称解析时出现不可恢复的错误；您的解析器或 DNS 服务器可能存在问题。

- **EVUTIL_EAI_BADFLAGS**  
  `hints` 中的 `ai_flags` 字段无效。

- **EVUTIL_EAI_FAMILY**  
  `hints` 中的 `ai_family` 字段不是我们支持的类型。

- **EVUTIL_EAI_MEMORY**  
  在尝试满足您的请求时，内存不足。

- **EVUTIL_EAI_NODATA**  
  您请求的主机存在，但没有与之关联的地址信息。（或者，它没有您请求的类型的地址信息。）

- **EVUTIL_EAI_NONAME**  
  您请求的主机似乎不存在。

- **EVUTIL_EAI_SERVICE**  
  您请求的服务似乎不存在。

- **EVUTIL_EAI_SOCKTYPE**  
  我们不支持您请求的 socket 类型，或者它与 `ai_protocol` 不兼容。

- **EVUTIL_EAI_SYSTEM**  
  在名称解析期间发生了其他系统错误。请检查 errno 获取更多信息。

- **EVUTIL_EAI_CANCEL**  
  应用程序请求在 DNS 查找完成之前取消该查找。`evutil_getaddrinfo()` 函数永远不会生成此错误，但它可能来自 `evdns_getaddrinfo()`，如下面的部分所述。

您可以使用 `evutil_gai_strerror()` 将这些结果转换为可读字符串。

**注意**：如果您的操作系统定义了 `struct addrinfo`，那么 `evutil_addrinfo` 只是您操作系统内置结构的别名。类似地，如果您的操作系统定义了任何 AI_* 标志，则相应的 `EVUTIL_AI_*` 标志只是对本地标志的别名；如果您的操作系统定义了任何 EAI_* 错误，则相应的 `EVUTIL_EAI_*` 代码与您平台的本地错误代码相同。

## 示例：解析主机名并进行阻塞连接

```c
#include <event2/util.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <stdio.h>
#include <string.h>
#include <assert.h>
#include <unistd.h>

evutil_socket_t
get_tcp_socket_for_host(const char *hostname, ev_uint16_t port)
{
    char port_buf[6];
    struct evutil_addrinfo hints;
    struct evutil_addrinfo *answer = NULL;
    int err;
    evutil_socket_t sock;

    /* 将端口转换为十进制。 */
    evutil_snprintf(port_buf, sizeof(port_buf), "%d", (int)port);

    /* 构建提示以告知 getaddrinfo 如何操作。 */
    memset(&hints, 0, sizeof(hints));
    hints.ai_family = AF_UNSPEC; /* IPv4 或 IPv6 均可。 */
    hints.ai_socktype = SOCK_STREAM;
    hints.ai_protocol = IPPROTO_TCP; /* 我们想要一个 TCP 套接字 */
    /* 仅返回我们可以使用的地址。 */
    hints.ai_flags = EVUTIL_AI_ADDRCONFIG;

    /* 查找主机名。 */
    err = evutil_getaddrinfo(hostname, port_buf, &hints, &answer);
    if (err != 0) {
          fprintf(stderr, "解析 '%s' 时出错：%s",
                  hostname, evutil_gai_strerror(err));
          return -1;
    }

    /* 如果没有错误，我们应该至少有一个答案。 */
    assert(answer);
    /* 只使用第一个答案。 */
    sock = socket(answer->ai_family,
                  answer->ai_socktype,
                  answer->ai_protocol);
    if (sock < 0)
        return -1;
    if (connect(sock, answer->ai_addr, answer->ai_addrlen)) {
        /* 注意我们在这个函数中执行的是阻塞连接。
         * 如果这是非阻塞的，我们需要特殊处理一些错误
         * (例如 EINTR 和 EAGAIN)。 */
        EVUTIL_CLOSESOCKET(sock);
        return -1;
    }

    return sock;
}
```

这些函数和常量是在 Libevent 2.0.3-alpha 中新增的。它们在 `event2/util.h` 中声明。

## 使用 `evdns_getaddrinfo()` 进行非阻塞主机名解析

常规的 `getaddrinfo()` 接口和上面的 `evutil_getaddrinfo()` 的主要问题是它们是阻塞的：当你调用它们时，你所在的线程必须等待，同时查询你的 DNS 服务器并等待响应。由于你正在使用 Libevent，这可能不是你想要的行为。

因此，Libevent 提供了一组函数来启动 DNS 请求，并使用 Libevent 等待服务器的回答。

### 接口

```c
typedef void (*evdns_getaddrinfo_cb)(
    int result, struct evutil_addrinfo *res, void *arg);
struct evdns_getaddrinfo_request;

struct evdns_getaddrinfo_request *evdns_getaddrinfo(
    struct evdns_base *dns_base,
    const char *nodename, const char *servname,
    const struct evutil_addrinfo *hints_in,
    evdns_getaddrinfo_cb cb, void *arg);

void evdns_getaddrinfo_cancel(struct evdns_getaddrinfo_request *req);
```

`evdns_getaddrinfo()` 函数的行为与 `evutil_getaddrinfo()` 相似，不同之处在于它不在 DNS 服务器上阻塞，而是利用 Libevent 的底层 DNS 功能为你查找主机名。因为它不能总是立即返回结果，你需要提供一个回调函数，类型为 `evdns_getaddrinfo_cb`，以及一个可选的用户提供的参数给该回调函数。

此外，你还需要为 `evdns_getaddrinfo()` 提供一个指向 `evdns_base` 的指针。该结构持有 Libevent 的 DNS 解析器的状态和配置。有关如何获取一个的更多信息，请参见下一节。

`evdns_getaddrinfo()` 函数在失败时返回 NULL 或立即成功。否则，它返回一个指向 `evdns_getaddrinfo_request` 的指针。你可以在请求完成之前随时使用它取消请求。

请注意，无论 `evdns_getaddrinfo()` 是否返回 NULL，或者是否调用了 `evdns_getaddrinfo_cancel()`，回调函数最终都会被调用。

当你调用 `evdns_getaddrinfo()` 时，它会对其 `nodename`、`servname` 和 `hints` 参数进行内部复制：你不需要确保它们在名称查找进行时持续存在。

### 示例：使用 `evdns_getaddrinfo()` 进行非阻塞查找

```c
#include <event2/dns.h>
#include <event2/util.h>
#include <event2/event.h>
#include <sys/socket.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <assert.h>

int n_pending_requests = 0;
struct event_base *base = NULL;

struct user_data {
    char *name; /* 我们要解析的名称 */
    int idx; /* 在命令行中的位置 */
};

void callback(int errcode, struct evutil_addrinfo *addr, void *ptr)
{
    struct user_data *data = ptr;
    const char *name = data->name;
    if (errcode) {
        printf("%d. %s -> %s\n", data->idx, name, evutil_gai_strerror(errcode));
    } else {
        struct evutil_addrinfo *ai;
        printf("%d. %s", data->idx, name);
        if (addr->ai_canonname)
            printf(" [%s]", addr->ai_canonname);
        puts("");
        for (ai = addr; ai; ai = ai->ai_next) {
            char buf[128];
            const char *s = NULL;
            if (ai->ai_family == AF_INET) {
                struct sockaddr_in *sin = (struct sockaddr_in *)ai->ai_addr;
                s = evutil_inet_ntop(AF_INET, &sin->sin_addr, buf, 128);
            } else if (ai->ai_family == AF_INET6) {
                struct sockaddr_in6 *sin6 = (struct sockaddr_in6 *)ai->ai_addr;
                s = evutil_inet_ntop(AF_INET6, &sin6->sin6_addr, buf, 128);
            }
            if (s)
                printf("    -> %s\n", s);
        }
        evutil_freeaddrinfo(addr);
    }
    free(data->name);
    free(data);
    if (--n_pending_requests == 0)
        event_base_loopexit(base, NULL);
}

/* 从命令行获取一系列域名并并行解析它们。 */
int main(int argc, char **argv)
{
    int i;
    struct evdns_base *dnsbase;

    if (argc == 1) {
        puts("没有给出地址。");
        return 0;
    }
    base = event_base_new();
    if (!base)
        return 1;
    dnsbase = evdns_base_new(base, 1);
    if (!dnsbase)
        return 2;

    for (i = 1; i < argc; ++i) {
        struct evutil_addrinfo hints;
        struct evdns_getaddrinfo_request *req;
        struct user_data *user_data;
        memset(&hints, 0, sizeof(hints));
        hints.ai_family = AF_UNSPEC;
        hints.ai_flags = EVUTIL_AI_CANONNAME;
        /* 除非我们指定 socktype，否则每个地址至少会得到两个条目：
         * 一个用于 TCP，一个用于 UDP。这不是我们想要的。 */
        hints.ai_socktype = SOCK_STREAM;
        hints.ai_protocol = IPPROTO_TCP;

        if (!(user_data = malloc(sizeof(struct user_data)))) {
            perror("malloc");
            exit(1);
        }
        if (!(user_data->name = strdup(argv[i]))) {
            perror("strdup");
            exit(1);
        }
        user_data->idx = i;

        ++n_pending_requests;
        req = evdns_getaddrinfo(
                          dnsbase, argv[i], NULL /* 没有给出服务名称 */,
                          &hints, callback, user_data);
        if (req == NULL) {
          printf("    [对 %s 的请求立即返回]\n", argv[i]);
          /* 无需释放 user_data 或减少 n_pending_requests；那
           * 在回调中发生。 */
        }
    }

    if (n_pending_requests)
      event_base_dispatch(base);

    evdns_base_free(dnsbase, 0);
    event_base_free(base);

    return 0;
}
```

这些函数是在 Libevent 2.0.3-alpha 中新增的。它们在 `event2/dns.h` 中声明。
### 创建和配置 `evdns_base`

在使用 Libevent 进行非阻塞 DNS 查询之前，您需要配置一个 `evdns_base`。每个 `evdns_base` 都存储着一组名称服务器、DNS 配置选项，并跟踪正在进行的 DNS 请求。

#### 接口

```c
struct evdns_base *evdns_base_new(struct event_base *event_base, int initialize);
void evdns_base_free(struct evdns_base *base, int fail_requests);
```

- **`evdns_base_new()`**：此函数返回一个新的 `evdns_base`，如果成功，返回指向新对象的指针；否则，返回 `NULL`。如果 `initialize` 参数为 `1`，它将尝试根据操作系统的默认设置智能地配置 DNS 基础。如果为 `0`，则会创建一个空的 `evdns_base`，没有配置名称服务器或选项。

- **`evdns_base_free()`**：当不再需要 `evdns_base` 时，可以调用此函数来释放它。如果 `fail_requests` 参数为 `true`，则它会使所有正在进行的请求触发回调并报告已取消的错误代码，然后再释放 `evdns_base`。

### 从系统配置初始化 `evdns_base`

如果您希望更精细地控制如何初始化 `evdns_base`，可以将 `initialize` 参数传递为 `0`，并使用以下函数来配置。

#### 接口

```c
#define DNS_OPTION_SEARCH 1
#define DNS_OPTION_NAMESERVERS 2
#define DNS_OPTION_MISC 4
#define DNS_OPTION_HOSTSFILE 8
#define DNS_OPTIONS_ALL 15
int evdns_base_resolv_conf_parse(struct evdns_base *base, int flags, const char *filename);

#ifdef WIN32
int evdns_base_config_windows_nameservers(struct evdns_base *);
#define EVDNS_BASE_CONFIG_WINDOWS_NAMESERVERS_IMPLEMENTED
#endif
```

- **`evdns_base_resolv_conf_parse()`**：此函数将扫描 `resolv.conf` 格式的文件，并读取 `filename` 中列出的所有选项。可以使用 `flags` 来指定哪些选项应被读取。

  `flags` 参数中可以使用以下选项：

  - **`DNS_OPTION_SEARCH`**：读取 `resolv.conf` 文件中的域名和搜索字段以及 `ndots` 选项，并使用它们来决定哪些域名（如果有）应在解析未完全限定的主机名时进行搜索。
  
  - **`DNS_OPTION_NAMESERVERS`**：此标志告诉 `evdns` 从 `resolv.conf` 文件中学习名称服务器。

  - **`DNS_OPTION_MISC`**：读取 `resolv.conf` 文件中的其他配置选项。

  - **`DNS_OPTION_HOSTSFILE`**：读取 `/etc/hosts` 文件中的主机列表作为加载 `resolv.conf` 文件的一部分。

  - **`DNS_OPTIONS_ALL`**：告诉 `evdns` 学习尽可能多的配置。

- **Windows 特定**：在 Windows 上，由于没有 `resolv.conf` 文件，您可以使用 `evdns_base_config_windows_nameservers()` 来从注册表或网络参数中读取所有名称服务器。

### `resolv.conf` 文件格式

Libevent 支持的 `resolv.conf` 文件格式为文本文件，每一行应该是空行、注释（以 `#` 开头），或者由一个令牌和零个或多个参数组成的行。识别的令牌包括：

- **`nameserver`**：必须跟随一个名称服务器的 IP 地址。Libevent 允许使用 `IP:Port` 或 `[IPv6]:port` 来指定非标准端口。
  
- **`domain`**：本地域名。

- **`search`**：一个列出在解析本地主机名时要搜索的域名列表。任何少于 `ndots` 个点的主机名会被认为是本地的，并且如果无法直接解析，我们会在这些域名中进行查找。例如，如果 `search` 为 `example.com` 且 `ndots` 为 1，当用户请求解析 `www` 时，我们会考虑 `www.example.com`。

- **`options`**：一个以空格分隔的选项列表。每个选项是一个简单的字符串，或采用 `option:value` 格式。已知的选项有：

  - **`ndots:INTEGER`**：配置搜索的行为。详情请见“search”部分。默认值为 1。

  - **`timeout:FLOAT`**：我们等待 DNS 服务器响应的时间（以秒为单位），如果超过这个时间没有响应，我们认为没有收到响应。默认值为 5 秒。

  - **`max-timeouts:INT`**：允许名称服务器连续超时的次数，超过该次数会认为该服务器不可用。默认值为 3。

  - **`max-inflight:INT`**：同时允许挂起的最大 DNS 请求数。如果尝试发送更多的请求，额外的请求会被阻塞，直到之前的请求有响应或超时。默认值为 64。

  - **`attempts:INT`**：在放弃之前，重新发送 DNS 请求的次数。默认值为 3。

  - **`randomize-case:INT`**：如果为非零值，随机化 DNS 请求中的字母大小写，并确保响应中保持与请求相同的大小写。这是所谓的 “0x20 hack”，可以帮助防止 DNS 查询中某些简单的活动事件。默认值为 1。

  - **`bind-to:ADDRESS`**：如果提供了此地址，则每当我们发送数据包到名称服务器时，都绑定到该地址。自 Libevent 2.0.4-alpha 起，仅适用于后续的名称服务器。

  - **`initial-probe-timeout:FLOAT`**：当我们认为一个名称服务器不可用时，使用递减的频率进行探测，直到它重新可用。此选项配置第一次探测超时的时间（以秒为单位）。默认值为 10。

  - **`getaddrinfo-allow-skew:FLOAT`**：当 `evdns_getaddrinfo()` 请求同时请求 IPv4 地址和 IPv6 地址时，它会分别发送两个 DNS 请求包，因为某些服务器无法在一个包中处理两者。一旦得到其中一种地址类型的响应，它会等待一段时间看看另一种地址类型是否也会响应。此选项配置等待的时间（以秒为单位）。默认值为 3 秒。

任何未识别的令牌和选项将会被忽略。

### 手动配置 `evdns`

如果您需要对 `evdns` 的行为进行更精细的控制，可以使用以下函数：
### 接口

```c
int evdns_base_nameserver_sockaddr_add(struct evdns_base *base,
                                 const struct sockaddr *sa, ev_socklen_t len,
                                 unsigned flags);
int evdns_base_nameserver_ip_add(struct evdns_base *base,
                                 const char *ip_as_string);
int evdns_base_load_hosts(struct evdns_base *base, const char *hosts_fname);

void evdns_base_search_clear(struct evdns_base *base);
void evdns_base_search_add(struct evdns_base *base, const char *domain);
void evdns_base_search_ndots_set(struct evdns_base *base, int ndots);

int evdns_base_set_option(struct evdns_base *base, const char *option,
    const char *val);

int evdns_base_count_nameservers(struct evdns_base *base);
```

- **`evdns_base_nameserver_sockaddr_add()`**：通过地址将名称服务器添加到现有的 `evdns_base`。`flags` 参数当前被忽略，应该设置为 `0` 以确保向前兼容。函数成功返回 `0`，失败返回负值。（此功能在 Libevent 2.0.7-rc 中添加。）

- **`evdns_base_nameserver_ip_add()`**：将名称服务器以文本字符串的形式添加到现有的 `evdns_base`。可以是 IPv4 地址、IPv6 地址、带端口的 IPv4 地址（IPv4:Port）或带端口的 IPv6 地址 ([IPv6]:Port)。成功时返回 `0`，失败返回负值。

- **`evdns_base_load_hosts()`**：从 `hosts_fname` 加载一个主机文件（格式与 `/etc/hosts` 相同）。成功时返回 `0`，失败返回负值。

- **`evdns_base_search_clear()`**：从 `evdns_base` 中移除所有当前的搜索后缀（由搜索选项配置）；`evdns_base_search_add()` 函数则用于添加后缀。

- **`evdns_base_set_option()`**：在 `evdns_base` 中设置给定选项的值。每个选项和其值都以字符串形式给出。（在 Libevent 2.0.3 之前，选项名称后面需要有冒号。）

- **`evdns_base_count_nameservers()`**：如果您刚刚解析了一组配置文件并想查看添加了多少名称服务器，可以使用此函数。

### 库级配置

您可以使用以下函数指定 `evdns` 模块的库级设置：

```c
typedef void (*evdns_debug_log_fn_type)(int is_warning, const char *msg);
void evdns_set_log_fn(evdns_debug_log_fn_type fn);
void evdns_set_transaction_id_fn(ev_uint16_t (*fn)(void));
```

- **`evdns_set_log_fn()`**：出于历史原因，`evdns` 子系统自己进行日志记录；您可以使用此函数提供一个回调函数，对其消息执行其他操作，而不是简单丢弃它们。

- **`evdns_set_transaction_id_fn()`**：为安全起见，`evdns` 需要一个良好的随机数源：它用来选择难以猜测的事务 ID，并在使用 0x20 hack 时对查询进行随机化。（有关更多信息，请参见“randomize-case”选项。）旧版本的 Libevent 没有提供自己的安全 RNG，您可以通过调用此函数并提供一个返回难以预测的两字节无符号整数的函数来给 `evdns` 提供更好的随机数生成器。在 Libevent 2.0.4-alpha 及以后的版本中，Libevent 使用其内置的安全 RNG；因此，`evdns_set_transaction_id_fn()` 没有效果。

### 低级 DNS 接口

有时，您可能希望以比 `evdns_getaddrinfo()` 更细粒度的控制能力启动特定的 DNS 请求。Libevent 提供了一些接口来实现这一点。

#### 缺失特性

目前，Libevent 的 DNS 支持缺少一些您期望从低级 DNS 系统中获得的功能，例如对任意请求类型和 TCP 请求的支持。如果您需要 `evdns` 没有的功能，请考虑贡献一个补丁。您也可以考虑使用更全面的 DNS 库，例如 c-ares。

### 接口

```c
#define DNS_QUERY_NO_SEARCH /* ... */
#define DNS_IPv4_A         /* ... */
#define DNS_PTR            /* ... */
#define DNS_IPv6_AAAA      /* ... */

typedef void (*evdns_callback_type)(int result, char type, int count,
    int ttl, void *addresses, void *arg);

struct evdns_request *evdns_base_resolve_ipv4(struct evdns_base *base,
    const char *name, int flags, evdns_callback_type callback, void *ptr);
struct evdns_request *evdns_base_resolve_ipv6(struct evdns_base *base,
    const char *name, int flags, evdns_callback_type callback, void *ptr);
struct evdns_request *evdns_base_resolve_reverse(struct evdns_base *base,
    const struct in_addr *in, int flags, evdns_callback_type callback,
    void *ptr);
struct evdns_request *evdns_base_resolve_reverse_ipv6(
    struct evdns_base *base, const struct in6_addr *in, int flags,
    evdns_callback_type callback, void *ptr);
```

#### DNS 查询函数

这些解析函数会发起对特定记录的 DNS 请求。每个函数都需要传入一个用于请求的 `evdns_base`、一个需要查找的资源（主机名用于正向查找，或地址用于反向查找）、一组标志以确定如何进行查找、一个完成后要调用的回调函数，以及一个用户提供的指针。

- **`flags` 参数**：可以是 `0` 或 `DNS_QUERY_NO_SEARCH`，后者用于显式抑制在原始搜索失败时在搜索列表中进行搜索。对反向查找，`DNS_QUERY_NO_SEARCH` 没有影响，因为反向查找永远不进行搜索。

- 当请求完成时，无论成功与否，回调函数都会被调用。回调接收的参数包括一个表示成功与否或错误代码的结果（参见下面的 DNS 错误表）、记录类型（可以是 `DNS_IPv4_A`、`DNS_IPv6_AAAA` 或 `DNS_PTR`）、地址数量、TTL（生存时间，单位为秒）、地址本身，以及用户提供的参数指针。

- **`addresses` 参数**：在发生错误时为 `NULL`。对于 PTR 记录，它是一个以空字符结束的字符串。对于 IPv4 记录，它是一个以网络字节顺序排列的四字节值数组。对于 IPv6 记录，它是一个以网络字节顺序排列的十六字节记录数组。（注意，即使没有错误，地址数量也可能为 `0`，这可能发生在名称存在但没有请求的记录类型时。）

### 错误代码

可以传递给回调的错误代码如下：

### DNS 错误

| 代码                      | 含义                                |
|-------------------------|-----------------------------------|
| `DNS_ERR_NONE`         | 没有发生错误                          |
| `DNS_ERR_FORMAT`       | 服务器无法理解查询                    |
| `DNS_ERR_SERVERFAILED` | 服务器报告内部错误                    |
| `DNS_ERR_NOTEXIST`     | 给定名称没有记录                      |
| `DNS_ERR_NOTIMPL`      | 服务器不理解这种类型的查询              |
| `DNS_ERR_REFUSED`      | 服务器因政策原因拒绝查询                |
| `DNS_ERR_TRUNCATED`    | DNS 记录无法放入 UDP 数据包              |
| `DNS_ERR_UNKNOWN`      | 未知的内部错误                        |
| `DNS_ERR_TIMEOUT`      | 我们等待了太久没有收到答案              |
| `DNS_ERR_SHUTDOWN`     | 用户要求我们关闭 evdns 系统             |
| `DNS_ERR_CANCEL`       | 用户要求我们取消此请求                 |
| `DNS_ERR_NODATA`       | 响应已到达，但没有包含答案               |

（`DNS_ERR_NODATA` 在 2.0.15-stable 中新增。）

您可以使用以下函数将这些错误代码解码为人类可读的字符串：

```c
const char *evdns_err_to_string(int err);
```

每个解析函数返回一个指向不透明的 `evdns_request` 结构的指针。您可以在回调被调用之前的任何时候使用此指针取消请求：

```c
void evdns_cancel_request(struct evdns_base *base,
    struct evdns_request *req);
```

使用此函数取消请求将使其回调以 `DNS_ERR_CANCEL` 结果代码被调用。

### 暂停 DNS 客户端操作和更改名称服务器

有时，您可能希望在不影响正在进行的 DNS 请求的情况下重新配置或关闭 DNS 子系统。

```c
int evdns_base_clear_nameservers_and_suspend(struct evdns_base *base);
int evdns_base_resume(struct evdns_base *base);
```

如果您在 `evdns_base` 上调用 `evdns_base_clear_nameservers_and_suspend()`，则所有名称服务器将被移除，而待处理的请求将保持悬而未决状态，直到您稍后重新添加名称服务器并调用 `evdns_base_resume()`。

这些函数在成功时返回 `0`，失败时返回 `-1`。它们是在 Libevent 2.0.1-alpha 中引入的。

### DNS 服务器接口

Libevent 提供了简单的功能，用于充当简单的 DNS 服务器并响应 UDP DNS 请求。

本节假设您对 DNS 协议有一定的了解。

#### 创建和关闭 DNS 服务器

```c
struct evdns_server_port *evdns_add_server_port_with_base(
    struct event_base *base,
    evutil_socket_t socket,
    int flags,
    evdns_request_callback_fn_type callback,
    void *user_data);
```

```c
typedef void (*evdns_request_callback_fn_type)(
    struct evdns_server_request *request,
    void *user_data);
```

```c
void evdns_close_server_port(struct evdns_server_port *port);
```

要开始监听 DNS 请求，调用 `evdns_add_server_port_with_base()`。该函数需要一个用于事件处理的 `event_base`、一个用于监听的 UDP 套接字、一个标志变量（目前始终为 `0`）、在收到新 DNS 查询时调用的回调函数，以及一个将传递给回调的用户数据指针。它返回一个新的 `evdns_server_port` 对象。

当您完成 DNS 服务器时，可以将其传递给 `evdns_close_server_port()`。

`evdns_add_server_port_with_base()` 函数在 2.0.1-alpha 中新增；`evdns_close_server_port()` 在 1.3 中引入。

### 检查 DNS 请求

不幸的是，Libevent 目前没有提供良好的程序接口来查看 DNS 请求。相反，您需要包含 `event2/dns_struct.h` 并手动查看 `evdns_server_request` 结构。

希望未来的 Libevent 版本能提供更好的方式来实现这一点。

```c
struct evdns_server_request {
        int flags;
        int nquestions;
        struct evdns_server_question **questions;
};
```

```c
#define EVDNS_QTYPE_AXFR 252
#define EVDNS_QTYPE_ALL  255
```

```c
struct evdns_server_question {
        int type;
        int dns_question_class;
        char name[1];
};
```

请求的 `flags` 字段包含在请求中设置的 DNS 标志；`nquestions` 字段是请求中的问题数量；`questions` 是指向 `struct evdns_server_question` 的指针数组。每个 `evdns_server_question` 包括请求的资源类型（参见下面的 EVDNS_*_TYPE 宏列表）、请求的类别（通常为 `EVDNS_CLASS_INET`）和请求的主机名。

这些结构在 Libevent 1.3 中引入。在 Libevent 1.4 之前，`dns_question_class` 被称为 "class"，这对 C++ 用户造成了问题。仍然使用旧的 "class" 名称的 C 程序将在未来版本中停止工作。

### 接口
```c
int evdns_server_request_get_requesting_addr(struct evdns_server_request *req,
        struct sockaddr *sa, int addr_len);
```
有时你需要知道哪个地址发出了特定的 DNS 请求。你可以通过调用 `evdns_server_request_get_requesting_addr()` 来检查它。你应该传入一个 `sockaddr` 结构，以确保有足够的存储空间来保存地址；建议使用 `struct sockaddr_storage`。

此函数是在 Libevent 1.3c 中引入的。

### 响应 DNS 请求
每当你的 DNS 服务器收到请求时，请求会传递给你提供的回调函数，并附带你的 `user_data` 指针。回调函数必须要么响应请求，要么忽略请求，或者确保最终回答或忽略该请求。

在响应请求之前，你可以添加一个或多个答案：

### 接口
```c
int evdns_server_request_add_a_reply(struct evdns_server_request *req,
    const char *name, int n, const void *addrs, int ttl);
int evdns_server_request_add_aaaa_reply(struct evdns_server_request *req,
    const char *name, int n, const void *addrs, int ttl);
int evdns_server_request_add_cname_reply(struct evdns_server_request *req,
    const char *name, const char *cname, int ttl);
```
上述函数分别向请求的答案部分添加一个 RR（类型为 A、AAAA 或 CNAME）。在每种情况下，参数 `name` 是要添加答案的主机名，`ttl` 是答案的生存时间（以秒为单位）。对于 A 和 AAAA 记录，`n` 是要添加的地址数量，`addrs` 是指向原始地址的指针，A 记录使用 n*4 字节的 IPv4 地址序列，AAAA 记录使用 n*16 字节的 IPv6 地址序列。

这些函数成功时返回 0，失败时返回 -1。

### 接口
```c
int evdns_server_request_add_ptr_reply(struct evdns_server_request *req,
    struct in_addr *in, const char *inaddr_name, const char *hostname,
    int ttl);
```
此函数向请求的答案部分添加一个 PTR 记录。参数 `req` 和 `ttl` 如上。你必须提供 `in`（一个 IPv4 地址）或 `inaddr_name`（.arpa 域中的地址）中的一个，以指示你要提供的响应地址。`hostname` 参数是 PTR 查找的答案。

### 接口
```c
#define EVDNS_ANSWER_SECTION 0
#define EVDNS_AUTHORITY_SECTION 1
#define EVDNS_ADDITIONAL_SECTION 2

#define EVDNS_TYPE_A       1
#define EVDNS_TYPE_NS      2
#define EVDNS_TYPE_CNAME   5
#define EVDNS_TYPE_SOA     6
#define EVDNS_TYPE_PTR    12
#define EVDNS_TYPE_MX     15
#define EVDNS_TYPE_TXT    16
#define EVDNS_TYPE_AAAA   28

#define EVDNS_CLASS_INET   1
```

```c
int evdns_server_request_add_reply(struct evdns_server_request *req,
    int section, const char *name, int type, int dns_class, int ttl,
    int datalen, int is_name, const char *data);
```
此函数向请求 `req` 的 DNS 响应添加一个任意 RR。参数 `section` 描述要添加的部分，应该是 EVDNS_*_SECTION 值之一。`name` 参数是 RR 的名称字段，`type` 参数是 RR 的类型字段，尽可能使用 EVDNS_TYPE_* 值。`dns_class` 参数是 RR 的类字段，通常应该是 EVDNS_CLASS_INET。`ttl` 参数是 RR 的生存时间（以秒为单位）。RR 的 rdata 和 rdlength 字段将根据提供的 `data` 中的 `datalen` 字节生成。如果 `is_name` 为真，数据将作为 DNS 名称编码（即，使用 DNS 名称压缩）。否则，它将被逐字包含。

### 接口
```c
int evdns_server_request_respond(struct evdns_server_request *req, int err);
int evdns_server_request_drop(struct evdns_server_request *req);
```
`evdns_server_request_respond()` 函数向请求发送 DNS 响应，包括你附加的所有 RR，并返回错误代码 `err`。如果你收到了不想响应的请求，可以通过调用 `evdns_server_request_drop()` 来忽略它，以释放所有相关内存和记账结构。

### 接口
```c
#define EVDNS_FLAGS_AA  0x400
#define EVDNS_FLAGS_RD  0x080
```

```c
void evdns_server_request_set_flags(struct evdns_server_request *req,
                                    int flags);
```
如果你想在响应消息上设置任何标志，可以在发送响应之前的任何时间调用此函数。

这一节中的所有函数都是在 Libevent 1.3 中引入的，除了 `evdns_server_request_set_flags()`，它首次出现在 Libevent 2.0.1-alpha 中。

### 示例：一个简单的 DNS 响应器

```c
#include <event2/dns.h>
#include <event2/dns_struct.h>
#include <event2/util.h>
#include <event2/event.h>
#include <sys/socket.h>

#include <stdio.h>
#include <string.h>
#include <assert.h>

/* 我们尝试绑定到 5353 端口。传统上使用 53 端口，但在大多数操作系统上，
   该端口需要根权限。 */
#define LISTEN_PORT 5353

#define LOCALHOST_IPV4_ARPA "1.0.0.127.in-addr.arpa"
#define LOCALHOST_IPV6_ARPA ("1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0."         \
                             "0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.ip6.arpa")

const ev_uint8_t LOCALHOST_IPV4[] = { 127, 0, 0, 1 };
const ev_uint8_t LOCALHOST_IPV6[] = { 0,0,0,0,0,0,0,0, 0,0,0,0,0,0,0,1 };

#define TTL 4242

/* 这个简单的 DNS 服务器回调函数回答对 localhost 的请求（映射到
   127.0.0.1 或 ::1）以及对 127.0.0.1 或 ::1 的请求（映射到 localhost）。
 */
void server_callback(struct evdns_server_request *request, void *data)
{
    int i;
    int error = DNS_ERR_NONE;
    /* 我们应该尝试回答所有问题。然而，一些 DNS 服务器并不总是
       可靠地做到这一点，因此在自己放入一个请求中两个问题之前，你应该仔细考虑。 */
    for (i = 0; i < request->nquestions; ++i) {
        const struct evdns_server_question *q = request->questions[i];
        int ok = -1;
        /* 我们不使用常规的 strcasecmp，因为我们希望进行区域
           独立的比较。 */
        if (0 == evutil_ascii_strcasecmp(q->name, "localhost")) {
            if (q->type == EVDNS_TYPE_A)
                ok = evdns_server_request_add_a_reply(
                       request, q->name, 1, LOCALHOST_IPV4, TTL);
            else if (q->type == EVDNS_TYPE_AAAA)
                ok = evdns_server_request_add_aaaa_reply(
                       request, q->name, 1, LOCALHOST_IPV6, TTL);
        } else if (0 == evutil_ascii_strcasecmp(q->name, LOCALHOST_IPV4_ARPA)) {
            if (q->type == EVDNS_TYPE_PTR)
                ok = evdns_server_request_add_ptr_reply(
                       request, NULL, q->name, "LOCALHOST", TTL);
        } else if (0 == evutil_ascii_strcasecmp(q->name, LOCALHOST_IPV6_ARPA)) {
            if (q->type == EVDNS_TYPE_PTR)
                ok = evdns_server_request_add_ptr_reply(
                       request, NULL, q->name, "LOCALHOST", TTL);
        } else {
            error = DNS_ERR_NOTEXIST;
        }
        if (ok < 0 && error == DNS_ERR_NONE)
            error = DNS_ERR_SERVERFAILED;
    }
    /* 现在发送回复。 */
    evdns_server_request_respond(request, error);
}

int main(int argc, char **argv)
{
    struct event_base *base;
    struct evdns_server_port *server;
    evutil_socket_t server_fd;
    struct sockaddr_in listenaddr;

    base = event_base_new();
    if (!base)
        return 1;

    server_fd = socket(AF_INET, SOCK_DGRAM, 0);
    if (server_fd < 0)
        return 2;
    memset(&listenaddr, 0, sizeof(listenaddr));
    listenaddr.sin_family = AF_INET;
    listenaddr.sin_port = htons(LISTEN_PORT);
    listenaddr.sin_addr.s_addr = INADDR_ANY;
    if (bind(server_fd, (struct sockaddr*)&listenaddr, sizeof(listenaddr)) < 0)
        return 3;
    /* 服务器在接收到第一个请求后将接管事件循环，如果套接字是阻塞的 */
    if(evutil_make_socket_nonblocking(server_fd) < 0)
        return 4;
    server = evdns_add_server_port_with_base(base, server_fd, 0,
                                             server_callback, NULL);

    event_base_dispatch(base);

    evdns_close_server_port(server);
    event_base_free(base);

    return 0;
}
```

### 过时的 DNS 接口

```c
void evdns_base_search_ndots_set(struct evdns_base *base,
                                 const int ndots);
int evdns_base_nameserver_add(struct evdns_base *base,
    unsigned long int address);
void evdns_set_random_bytes_fn(void (*fn)(char *, size_t));

struct evdns_server_port *evdns_add_server_port(evutil_socket_t socket,
    int flags, evdns_request_callback_fn_type callback, void *user_data);
```

调用 `evdns_base_search_ndots_set()` 等效于使用 `evdns_base_set_option()` 设置 "ndots" 选项。

`evdns_base_nameserver_add()` 函数的行为类似于 `evdns_base_nameserver_ip_add()`，但它只能添加带有 IPv4 地址的名称服务器。它以网络顺序接收四个字节，这种方式有些特别。

在 Libevent 2.0.1-alpha 之前，无法为 DNS 服务器端口指定事件基础。你必须使用 `evdns_add_server_port()`，它使用默认的事件基础。

从 Libevent 2.0.1-alpha 到 2.0.3-alpha，你可以使用 `evdns_set_random_bytes_fn` 指定用于生成随机数的函数，而不是 `evdns_set_transaction_id_fn`。由于 Libevent 提供了自己的安全 RNG，因此它不再起作用。

`DNS_QUERY_NO_SEARCH` 标志也称为 `DNS_NO_SEARCH`。

在 Libevent 2.0.1-alpha 之前，并没有单独的 `evdns_base` 概念：所有关于 evdns 子系统的信息都是以全局方式存储的，操作这些信息的函数并不接受 `evdns_base` 作为参数。现在这些函数已被弃用，仅在 `event2/dns_compat.h` 中声明。它们通过一个单一的全局 `evdns_base` 实现；您可以通过调用在 Libevent 2.0.3-alpha 中引入的 `evdns_get_global_base()` 函数来访问这个基础。

### 当前函数与过时的全局 evdns_base 版本

| 当前函数 | 过时的全局 evdns_base 版本 |
|----------|--------------------------|
| event_base_new() | evdns_init() |
| evdns_base_free() | evdns_shutdown() |
| evdns_base_nameserver_add() | evdns_nameserver_add() |
| evdns_base_count_nameservers() | evdns_count_nameservers() |
| evdns_base_clear_nameservers_and_suspend() | evdns_clear_nameservers_and_suspend() |
| evdns_base_resume() | evdns_resume() |
| evdns_base_nameserver_ip_add() | evdns_nameserver_ip_add() |
| evdns_base_resolve_ipv4() | evdns_resolve_ipv4() |
| evdns_base_resolve_ipv6() | evdns_resolve_ipv6() |
| evdns_base_resolve_reverse() | evdns_resolve_reverse() |
| evdns_base_resolve_reverse_ipv6() | evdns_resolve_reverse_ipv6() |
| evdns_base_set_option() | evdns_set_option() |
| evdns_base_resolv_conf_parse() | evdns_resolv_conf_parse() |
| evdns_base_search_clear() | evdns_search_clear() |
| evdns_base_search_add() | evdns_search_add() |
| evdns_base_search_ndots_set() | evdns_search_ndots_set() |
| evdns_base_config_windows_nameservers() | evdns_config_windows_nameservers() |

`EVDNS_CONFIG_WINDOWS_NAMESERVERS_IMPLEMENTED` 宏仅在 `evdns_config_windows_nameservers()` 可用时定义
