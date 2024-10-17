### 异步IO简介

大多数初学者程序员从阻塞IO调用开始。IO调用是同步的，如果在调用时，它不会返回，直到操作完成，或者直到经过一定时间后，网络堆栈放弃操作为止。举个例子，当你调用TCP连接的 `connect()` 时，操作系统会将SYN数据包排队发送到对端主机的TCP连接上，直到收到对方主机的SYN ACK数据包，或者直到经过足够的时间操作系统决定放弃，才会将控制权返回给应用程序。

以下是一个非常简单的客户端示例，使用阻塞网络调用。它打开一个与www.google.com的连接，发送一个简单的HTTP请求，并将响应打印到标准输出。

### 示例：简单的阻塞HTTP客户端

```c
/* For sockaddr_in */
#include <netinet/in.h>
/* For socket functions */
#include <sys/socket.h>
/* For gethostbyname */
#include <netdb.h>

#include <unistd.h>
#include <string.h>
#include <stdio.h>

int main(int c, char **v)
{
    const char query[] =
        "GET / HTTP/1.0\r\n"
        "Host: www.google.com\r\n"
        "\r\n";
    const char hostname[] = "www.google.com";
    struct sockaddr_in sin;
    struct hostent *h;
    const char *cp;
    int fd;
    ssize_t n_written, remaining;
    char buf[1024];

    /* Look up the IP address for the hostname. Watch out; this isn't
       threadsafe on most platforms. */
    h = gethostbyname(hostname);
    if (!h) {
        fprintf(stderr, "Couldn't lookup %s: %s", hostname, hstrerror(h_errno));
        return 1;
    }
    if (h->h_addrtype != AF_INET) {
        fprintf(stderr, "No ipv6 support, sorry.");
        return 1;
    }

    /* Allocate a new socket */
    fd = socket(AF_INET, SOCK_STREAM, 0);
    if (fd < 0) {
        perror("socket");
        return 1;
    }

    /* Connect to the remote host. */
    sin.sin_family = AF_INET;
    sin.sin_port = htons(80);
    sin.sin_addr = *(struct in_addr*)h->h_addr;
    if (connect(fd, (struct sockaddr*) &sin, sizeof(sin))) {
        perror("connect");
        close(fd);
        return 1;
    }

    /* Write the query. */
    /* XXX Can send succeed partially? */
    cp = query;
    remaining = strlen(query);
    while (remaining) {
      n_written = send(fd, cp, remaining, 0);
      if (n_written <= 0) {
        perror("send");
        return 1;
      }
      remaining -= n_written;
      cp += n_written;
    }

    /* Get an answer back. */
    while (1) {
        ssize_t result = recv(fd, buf, sizeof(buf), 0);
        if (result == 0) {
            break;
        } else if (result < 0) {
            perror("recv");
            close(fd);
            return 1;
        }
        fwrite(buf, 1, result, stdout);
    }

    close(fd);
    return 0;
}
```

上面的代码中的所有网络调用都是阻塞的：`gethostbyname` 在解析www.google.com时不会返回，直到解析成功或失败；`connect` 在连接完成之前不会返回；`recv` 调用不会返回，直到接收到数据或关闭连接；而 `send` 调用不会返回，直到至少将数据刷写到内核的写缓冲区。

阻塞IO并不是不好的选择。如果在执行期间没有其他任务需要处理，阻塞IO也能很好地工作。但是，假设你需要编写一个程序来处理多个连接，这时阻塞IO可能就不太适用了。具体来说，假设你想要同时读取来自两个连接的输入，而你并不知道哪个连接会先收到数据。你不能像下面这样做：

### 错误示例

```c
/* This won't work. */
char buf[1024];
int i, n;
while (i_still_want_to_read()) {
    for (i=0; i<n_sockets; ++i) {
        n = recv(fd[i], buf, sizeof(buf), 0);
        if (n==0)
            handle_close(fd[i]);
        else if (n<0)
            handle_error(fd[i], errno);
        else
            handle_input(fd[i], buf, n);
    }
}
```
因为如果数据首先到达 `fd[2]`，你的程序甚至不会尝试从 `fd[2]` 读取，直到 `fd[0]` 和 `fd[1]` 的读取操作获取了一些数据并完成。

有时候，人们会通过多线程或多进程服务器来解决这个问题。进行多线程的一个简单方法是为每个连接创建一个独立的进程（或线程）。由于每个连接都有自己的进程，等待一个连接的阻塞IO调用不会导致其他连接的进程被阻塞。

以下是另一个示例程序。它是一个简单的服务器，监听40713端口上的TCP连接，逐行读取输入数据，并将每行数据写出其ROT13编码。它使用Unix的 `fork()` 调用为每个传入的连接创建一个新进程。

### 示例：Forking ROT13服务器

```c
/* For sockaddr_in */
#include <netinet/in.h>
/* For socket functions */
#include <sys/socket.h>

#include <unistd.h>
#include <string.h>
#include <stdio.h>
#include <stdlib.h>

#define MAX_LINE 16384

char
rot13_char(char c)
{
    /* 我们不想在这里使用 isalpha；设置区域会改变
     * 哪些字符被视为字母。 */
    if ((c >= 'a' && c <= 'm') || (c >= 'A' && c <= 'M'))
        return c + 13;
    else if ((c >= 'n' && c <= 'z') || (c >= 'N' && c <= 'Z'))
        return c - 13;
    else
        return c;
}

void
child(int fd)
{
    char outbuf[MAX_LINE+1];
    size_t outbuf_used = 0;
    ssize_t result;

    while (1) {
        char ch;
        result = recv(fd, &ch, 1, 0);
        if (result == 0) {
            break;
        } else if (result == -1) {
            perror("read");
            break;
        }

        /* 我们进行这个测试以防止用户溢出缓冲区。 */
        if (outbuf_used < sizeof(outbuf)) {
            outbuf[outbuf_used++] = rot13_char(ch);
        }

        if (ch == '\n') {
            send(fd, outbuf, outbuf_used, 0);
            outbuf_used = 0;
            continue;
        }
    }
}

void
run(void)
{
    int listener;
    struct sockaddr_in sin;

    sin.sin_family = AF_INET;
    sin.sin_addr.s_addr = 0;
    sin.sin_port = htons(40713);

    listener = socket(AF_INET, SOCK_STREAM, 0);

#ifndef WIN32
    {
        int one = 1;
        setsockopt(listener, SOL_SOCKET, SO_REUSEADDR, &one, sizeof(one));
    }
#endif

    if (bind(listener, (struct sockaddr*)&sin, sizeof(sin)) < 0) {
        perror("bind");
        return;
    }

    if (listen(listener, 16)<0) {
        perror("listen");
        return;
    }

    while (1) {
        struct sockaddr_storage ss;
        socklen_t slen = sizeof(ss);
        int fd = accept(listener, (struct sockaddr*)&ss, &slen);
        if (fd < 0) {
            perror("accept");
        } else {
            if (fork() == 0) {
                child(fd);
                exit(0);
            }
        }
    }
}

int
main(int c, char **v)
{
    run();
    return 0;
}
```

所以，我们真的有一个完美的解决方案来处理多个连接吗？我可以停止写这本书，去做其他事情吗？并不完全如此。首先，进程创建（甚至线程创建）在某些平台上可能相当昂贵。在实际应用中，你可能希望使用线程池，而不是每次都创建新进程。但更根本的是，线程的扩展性不会如你所愿。如果你的程序需要同时处理数千或数万的连接，处理数万个线程的效率不会像试图让每个CPU只有少数几个线程那么高效。

但如果线程不是处理多个连接的答案，那是什么呢？在Unix范式中，你可以将套接字设置为非阻塞。实现这个的Unix调用是：

```c
fcntl(fd, F_SETFL, O_NONBLOCK);
```

其中 `fd` 是套接字的文件描述符。 一旦你将 `fd`（套接字）设置为非阻塞，从那时起，每当你对 `fd` 进行网络调用时，该调用要么会立即完成操作，要么返回一个特殊的错误代码，表示“我现在无法取得任何进展，请再试一次”。因此，我们的两个套接字示例可以简单地写成：

### 糟糕的示例：忙等待所有套接字

```c
/* 这将工作，但性能会非常糟糕。 */
int i, n;
char buf[1024];
for (i = 0; i < n_sockets; ++i)
    fcntl(fd[i], F_SETFL, O_NONBLOCK);

while (i_still_want_to_read()) {
    for (i = 0; i < n_sockets; ++i) {
        n = recv(fd[i], buf, sizeof(buf), 0);
        if (n == 0) {
            handle_close(fd[i]);
        } else if (n < 0) {
            if (errno == EAGAIN)
                 ; /* 内核没有数据可以读取。 */
            else
                 handle_error(fd[i], errno);
        } else {
            handle_input(fd[i], buf, n);
        }
    }
}
```

现在我们使用非阻塞套接字，上面的代码将工作……但只是在勉强运行。性能将非常糟糕，原因有两个。首先，当两个连接都没有数据可读取时，循环将无限旋转，消耗所有的CPU周期。其次，如果你试图用这种方法处理多个连接，将为每个连接执行一次内核调用，无论它是否有任何数据可用。因此，我们需要一种方法来告诉内核“等待，直到这些套接字中的一个准备好给我数据，并告诉我哪些是准备好的。”

人们仍然使用的最古老的解决方案是 `select()`。`select()` 调用接受三个文件描述符集（实现为位数组）：一个用于读取，一个用于写入，还有一个用于“异常”。它会等待直到其中一个集合中的套接字准备就绪，并改变集合以仅包含准备好的套接字。

这里是我们使用 `select` 的例子：

**示例：使用 select**
```c
/* 如果你只有几十个 fd，这个版本不会太糟 */
fd_set readset;
int i, n;
char buf[1024];

while (i_still_want_to_read()) {
    int maxfd = -1;
    FD_ZERO(&readset);

    /* 将所有感兴趣的 fd 添加到 readset */
    for (i=0; i < n_sockets; ++i) {
         if (fd[i]>maxfd) maxfd = fd[i];
         FD_SET(fd[i], &readset);
    }

    /* 等待一个或多个 fd 准备好读取 */
    select(maxfd+1, &readset, NULL, NULL, NULL);

    /* 处理仍在 readset 中设置的所有 fd */
    for (i=0; i < n_sockets; ++i) {
        if (FD_ISSET(fd[i], &readset)) {
            n = recv(fd[i], buf, sizeof(buf), 0);
            if (n == 0) {
                handle_close(fd[i]);
            } else if (n < 0) {
                if (errno == EAGAIN)
                     ; /* 内核没有数据供我们读取。 */
                else
                     handle_error(fd[i], errno);
             } else {
                handle_input(fd[i], buf, n);
             }
        }
    }
}
```

这里是我们用 `select()` 重新实现的 ROT13 服务器。

**示例：基于 select() 的 ROT13 服务器**
```c
/* For sockaddr_in */
#include <netinet/in.h>
/* For socket functions */
#include <sys/socket.h>
/* For fcntl */
#include <fcntl.h>
/* for select */
#include <sys/select.h>

#include <assert.h>
#include <unistd.h>
#include <string.h>
#include <stdlib.h>
#include <stdio.h>
#include <errno.h>

#define MAX_LINE 16384

char
rot13_char(char c)
{
    /* 我们不想在这里使用 isalpha；设置区域会改变
     * 哪些字符被认为是字母。 */
    if ((c >= 'a' && c <= 'm') || (c >= 'A' && c <= 'M'))
        return c + 13;
    else if ((c >= 'n' && c <= 'z') || (c >= 'N' && c <= 'Z'))
        return c - 13;
    else
        return c;
}

struct fd_state {
    char buffer[MAX_LINE];
    size_t buffer_used;

    int writing;
    size_t n_written;
    size_t write_upto;
};

struct fd_state *
alloc_fd_state(void)
{
    struct fd_state *state = malloc(sizeof(struct fd_state));
    if (!state)
        return NULL;
    state->buffer_used = state->n_written = state->writing =
        state->write_upto = 0;
    return state;
}

void
free_fd_state(struct fd_state *state)
{
    free(state);
}

void
make_nonblocking(int fd)
{
    fcntl(fd, F_SETFL, O_NONBLOCK);
}

int
do_read(int fd, struct fd_state *state)
{
    char buf[1024];
    int i;
    ssize_t result;
    while (1) {
        result = recv(fd, buf, sizeof(buf), 0);
        if (result <= 0)
            break;

        for (i=0; i < result; ++i)  {
            if (state->buffer_used < sizeof(state->buffer))
                state->buffer[state->buffer_used++] = rot13_char(buf[i]);
            if (buf[i] == '\n') {
                state->writing = 1;
                state->write_upto = state->buffer_used;
            }
        }
    }

    if (result == 0) {
        return 1;
    } else if (result < 0) {
        if (errno == EAGAIN)
            return 0;
        return -1;
    }

    return 0;
}

int
do_write(int fd, struct fd_state *state)
{
    while (state->n_written < state->write_upto) {
        ssize_t result = send(fd, state->buffer + state->n_written,
                              state->write_upto - state->n_written, 0);
        if (result < 0) {
            if (errno == EAGAIN)
                return 0;
            return -1;
        }
        assert(result != 0);

        state->n_written += result;
    }

    if (state->n_written == state->buffer_used)
        state->n_written = state->write_upto = state->buffer_used = 0;

    state->writing = 0;

    return 0;
}

void
run(void)
{
    int listener;
    struct fd_state *state[FD_SETSIZE];
    struct sockaddr_in sin;
    int i, maxfd;
    fd_set readset, writeset, exset;

    sin.sin_family = AF_INET;
    sin.sin_addr.s_addr = 0;
    sin.sin_port = htons(40713);

    for (i = 0; i < FD_SETSIZE; ++i)
        state[i] = NULL;

    listener = socket(AF_INET, SOCK_STREAM, 0);
    make_nonblocking(listener);

#ifndef WIN32
    {
        int one = 1;
        setsockopt(listener, SOL_SOCKET, SO_REUSEADDR, &one, sizeof(one));
    }
#endif

    if (bind(listener, (struct sockaddr*)&sin, sizeof(sin)) < 0) {
        perror("bind");
        return;
    }

    if (listen(listener, 16)<0) {
        perror("listen");
        return;
    }

    FD_ZERO(&readset);
    FD_ZERO(&writeset);
    FD_ZERO(&exset);

    while (1) {
        maxfd = listener;

        FD_ZERO(&readset);
        FD_ZERO(&writeset);
        FD_ZERO(&exset);

        FD_SET(listener, &readset);

        for (i=0; i < FD_SETSIZE; ++i) {
            if (state[i]) {
                if (i > maxfd)
                    maxfd = i;
                FD_SET(i, &readset);
                if (state[i]->writing) {
                    FD_SET(i, &writeset);
                }
            }
        }

        if (select(maxfd+1, &readset, &writeset, &exset, NULL) < 0) {
            perror("select");
            return;
        }

        if (FD_ISSET(listener, &readset)) {
            struct sockaddr_storage ss;
            socklen_t slen = sizeof(ss);
            int fd = accept(listener, (struct sockaddr*)&ss, &slen);
            if (fd < 0) {
                perror("accept");
            } else if (fd > FD_SETSIZE) {
                close(fd);
            } else {
                make_nonblocking(fd);
                state[fd] = alloc_fd_state();
                assert(state[fd]); /* XXX */
            }
        }

        for (i=0; i < maxfd+1; ++i) {
            int r = 0;
            if (i == listener)
                continue;

            if (FD_ISSET(i, &readset)) {
                r = do_read(i, state[i]);
            }
            if (r == 0 && FD_ISSET(i, &writeset)) {
                r = do_write(i, state[i]);
            }
            if (r) {
                free_fd_state(state[i]);
                state[i] = NULL;
                close(i);
            }
        }
    }
}

int
main(int c, char **v)
{
    setvbuf(stdout, NULL, _IONBF, 0);

    run();
    return 0;
}
```

但我们仍然没有结束。因为生成和读取 `select()` 位数组需要的时间与您为 `select()` 提供的最大 fd 成比例，因此在高并发套接字数量的情况下，`select()` 调用的扩展性非常差【2】。

不同的操作系统提供了不同的替代函数来替代 `select`。这些包括 `poll()`、`epoll()`、`kqueue()`、`evports` 和 `/dev/poll`。所有这些方法都比 `select()` 提供更好的性能，除了 `poll()` 之外，其他所有方法在添加、移除套接字以及检测套接字是否准备好进行 IO 时，均提供 O(1) 的性能。
很遗憾，没有一种高效的接口是普遍标准。Linux 有 `epoll()`，BSD（包括 Darwin）有 `kqueue()`，Solaris 有 `evports` 和 `/dev/poll`……而且这些操作系统之间没有任何共享接口。因此，如果您想编写一个可移植的高性能异步应用程序，您需要一个包装所有这些接口的抽象，提供计算机上可用的最高效的接口。

这就是 Libevent API 最低级别为您提供的功能。它提供了对各种 `select()` 替代品的一致接口，使用在运行计算机上可用的最高效版本。

以下是我们异步 ROT13 服务器的又一个版本。这次它使用 Libevent 2，而不是 `select()`。请注意，`fd_sets` 已经不再使用：相反，我们将事件与 `struct event_base` 关联和分离，该结构可能是基于 `select()`、`poll()`、`epoll()`、`kqueue()` 等实现的。

### 示例：使用 Libevent 的低级 ROT13 服务器

```c
/* For sockaddr_in */
#include <netinet/in.h>
/* For socket functions */
#include <sys/socket.h>
/* For fcntl */
#include <fcntl.h>

#include <event2/event.h>

#include <assert.h>
#include <unistd.h>
#include <string.h>
#include <stdlib.h>
#include <stdio.h>
#include <errno.h>

#define MAX_LINE 16384

void do_read(evutil_socket_t fd, short events, void *arg);
void do_write(evutil_socket_t fd, short events, void *arg);

char rot13_char(char c)
{
    /* 我们不想在这里使用 isalpha；设置区域会改变
     * 哪些字符被认为是字母。 */
    if ((c >= 'a' && c <= 'm') || (c >= 'A' && c <= 'M'))
        return c + 13;
    else if ((c >= 'n' && c <= 'z') || (c >= 'N' && c <= 'Z'))
        return c - 13;
    else
        return c;
}

struct fd_state {
    char buffer[MAX_LINE];
    size_t buffer_used;

    size_t n_written;
    size_t write_upto;

    struct event *read_event;
    struct event *write_event;
};

struct fd_state *alloc_fd_state(struct event_base *base, evutil_socket_t fd)
{
    struct fd_state *state = malloc(sizeof(struct fd_state));
    if (!state)
        return NULL;
    state->read_event = event_new(base, fd, EV_READ|EV_PERSIST, do_read, state);
    if (!state->read_event) {
        free(state);
        return NULL;
    }
    state->write_event = event_new(base, fd, EV_WRITE|EV_PERSIST, do_write, state);

    if (!state->write_event) {
        event_free(state->read_event);
        free(state);
        return NULL;
    }

    state->buffer_used = state->n_written = state->write_upto = 0;

    assert(state->write_event);
    return state;
}

void free_fd_state(struct fd_state *state)
{
    event_free(state->read_event);
    event_free(state->write_event);
    free(state);
}

void do_read(evutil_socket_t fd, short events, void *arg)
{
    struct fd_state *state = arg;
    char buf[1024];
    int i;
    ssize_t result;
    while (1) {
        assert(state->write_event);
        result = recv(fd, buf, sizeof(buf), 0);
        if (result <= 0)
            break;

        for (i = 0; i < result; ++i) {
            if (state->buffer_used < sizeof(state->buffer))
                state->buffer[state->buffer_used++] = rot13_char(buf[i]);
            if (buf[i] == '\n') {
                assert(state->write_event);
                event_add(state->write_event, NULL);
                state->write_upto = state->buffer_used;
            }
        }
    }

    if (result == 0) {
        free_fd_state(state);
    } else if (result < 0) {
        if (errno == EAGAIN) // XXXX use evutil macro
            return;
        perror("recv");
        free_fd_state(state);
    }
}

void do_write(evutil_socket_t fd, short events, void *arg)
{
    struct fd_state *state = arg;

    while (state->n_written < state->write_upto) {
        ssize_t result = send(fd, state->buffer + state->n_written,
                              state->write_upto - state->n_written, 0);
        if (result < 0) {
            if (errno == EAGAIN) // XXX use evutil macro
                return;
            free_fd_state(state);
            return;
        }
        assert(result != 0);

        state->n_written += result;
    }

    if (state->n_written == state->buffer_used)
        state->n_written = state->write_upto = state->buffer_used = 1;

    event_del(state->write_event);
}

void do_accept(evutil_socket_t listener, short event, void *arg)
{
    struct event_base *base = arg;
    struct sockaddr_storage ss;
    socklen_t slen = sizeof(ss);
    int fd = accept(listener, (struct sockaddr*)&ss, &slen);
    if (fd < 0) { // XXXX eagain??
        perror("accept");
    } else if (fd > FD_SETSIZE) {
        close(fd); // XXX replace all closes with EVUTIL_CLOSESOCKET
    } else {
        struct fd_state *state;
        evutil_make_socket_nonblocking(fd);
        state = alloc_fd_state(base, fd);
        assert(state); /*XXX err*/
        assert(state->write_event);
        event_add(state->read_event, NULL);
    }
}

void run(void)
{
    evutil_socket_t listener;
    struct sockaddr_in sin;
    struct event_base *base;
    struct event *listener_event;

    base = event_base_new();
    if (!base)
        return; /*XXXerr*/

    sin.sin_family = AF_INET;
    sin.sin_addr.s_addr = 0;
    sin.sin_port = htons(40713);

    listener = socket(AF_INET, SOCK_STREAM, 0);
    evutil_make_socket_nonblocking(listener);

#ifndef WIN32
    {
        int one = 1;
        setsockopt(listener, SOL_SOCKET, SO_REUSEADDR, &one, sizeof(one));
    }
#endif

    if (bind(listener, (struct sockaddr*)&sin, sizeof(sin)) < 0) {
        perror("bind");
        return;
    }

    if (listen(listener, 16) < 0) {
        perror("listen");
        return;
    }

    listener_event = event_new(base, listener, EV_READ|EV_PERSIST, do_accept, (void*)base);
    /*XXX check it */
    event_add(listener_event, NULL);

    event_base_dispatch(base);
}

int main(int c, char **v)
{
    setvbuf(stdout, NULL, _IONBF, 0);

    run();
    return 0;
}
```

（代码中的其他注意事项：为了避免将套接字类型定义为“int”，我们使用 `evutil_socket_t` 类型。为了使套接字非阻塞，我们调用 `evutil_make_socket_nonblocking`，而不是使用 `fcntl(O_NONBLOCK)`。这些更改使我们的代码与 Win32 网络 API 的不同部分兼容

**关于便利性？（那Windows呢？）**

你可能已经注意到，随着我们的代码变得更加高效，它也变得更加复杂。回到我们进行分叉的时侯，我们不需要为每个连接管理一个缓冲区：我们只需为每个进程分配一个栈分配的缓冲区。我们不需要明确跟踪每个套接字是正在读取还是写入：这在代码中的位置是隐含的。我们也不需要一个结构来跟踪每个操作完成的进度：我们只是使用循环和栈变量。

此外，如果你对Windows上的网络编程非常熟悉，你会意识到上面的示例中使用Libevent可能并没有获得最佳性能。在Windows上，快速异步IO的方式并不是使用类似select()的接口，而是通过使用IOCP（IO完成端口）API。与所有快速网络API不同，IOCP并不会在套接字准备好进行操作时提醒你的程序，而是程序告诉Windows网络栈开始网络操作，而IOCP会告诉程序何时操作完成。

幸运的是，Libevent 2的“bufferevents”接口解决了这两个问题：它使得程序的编写变得更加简单，并提供了一个Libevent可以在Windows和Unix上高效实现的接口。

以下是我们最后一次使用bufferevents API的ROT13服务器示例。

**示例：使用Libevent的更简单的ROT13服务器**

```c
/* For sockaddr_in */
#include <netinet/in.h>
/* For socket functions */
#include <sys/socket.h>
/* For fcntl */
#include <fcntl.h>

#include <event2/event.h>
#include <event2/buffer.h>
#include <event2/bufferevent.h>

#include <assert.h>
#include <unistd.h>
#include <string.h>
#include <stdlib.h>
#include <stdio.h>
#include <errno.h>

#define MAX_LINE 16384

void do_read(evutil_socket_t fd, short events, void *arg);
void do_write(evutil_socket_t fd, short events, void *arg);

char
rot13_char(char c)
{
    /* We don't want to use isalpha here; setting the locale would change
     * which characters are considered alphabetical. */
    if ((c >= 'a' && c <= 'm') || (c >= 'A' && c <= 'M'))
        return c + 13;
    else if ((c >= 'n' && c <= 'z') || (c >= 'N' && c <= 'Z'))
        return c - 13;
    else
        return c;
}

void
readcb(struct bufferevent *bev, void *ctx)
{
    struct evbuffer *input, *output;
    char *line;
    size_t n;
    int i;
    input = bufferevent_get_input(bev);
    output = bufferevent_get_output(bev);

    while ((line = evbuffer_readln(input, &n, EVBUFFER_EOL_LF))) {
        for (i = 0; i < n; ++i)
            line[i] = rot13_char(line[i]);
        evbuffer_add(output, line, n);
        evbuffer_add(output, "\n", 1);
        free(line);
    }

    if (evbuffer_get_length(input) >= MAX_LINE) {
        /* Too long; just process what there is and go on so that the buffer
         * doesn't grow infinitely long. */
        char buf[1024];
        while (evbuffer_get_length(input)) {
            int n = evbuffer_remove(input, buf, sizeof(buf));
            for (i = 0; i < n; ++i)
                buf[i] = rot13_char(buf[i]);
            evbuffer_add(output, buf, n);
        }
        evbuffer_add(output, "\n", 1);
    }
}

void
errorcb(struct bufferevent *bev, short error, void *ctx)
{
    if (error & BEV_EVENT_EOF) {
        /* connection has been closed, do any clean up here */
        /* ... */
    } else if (error & BEV_EVENT_ERROR) {
        /* check errno to see what error occurred */
        /* ... */
    } else if (error & BEV_EVENT_TIMEOUT) {
        /* must be a timeout event handle, handle it */
        /* ... */
    }
    bufferevent_free(bev);
}

void
do_accept(evutil_socket_t listener, short event, void *arg)
{
    struct event_base *base = arg;
    struct sockaddr_storage ss;
    socklen_t slen = sizeof(ss);
    int fd = accept(listener, (struct sockaddr*)&ss, &slen);
    if (fd < 0) {
        perror("accept");
    } else if (fd > FD_SETSIZE) {
        close(fd);
    } else {
        struct bufferevent *bev;
        evutil_make_socket_nonblocking(fd);
        bev = bufferevent_socket_new(base, fd, BEV_OPT_CLOSE_ON_FREE);
        bufferevent_setcb(bev, readcb, NULL, errorcb, NULL);
        bufferevent_setwatermark(bev, EV_READ, 0, MAX_LINE);
        bufferevent_enable(bev, EV_READ|EV_WRITE);
    }
}

void
run(void)
{
    evutil_socket_t listener;
    struct sockaddr_in sin;
    struct event_base *base;
    struct event *listener_event;

    base = event_base_new();
    if (!base)
        return; /*XXXerr*/

    sin.sin_family = AF_INET;
    sin.sin_addr.s_addr = 0;
    sin.sin_port = htons(40713);

    listener = socket(AF_INET, SOCK_STREAM, 0);
    evutil_make_socket_nonblocking(listener);

#ifndef WIN32
    {
        int one = 1;
        setsockopt(listener, SOL_SOCKET, SO_REUSEADDR, &one, sizeof(one));
    }
#endif

    if (bind(listener, (struct sockaddr*)&sin, sizeof(sin)) < 0) {
        perror("bind");
        return;
    }

    if (listen(listener, 16)<0) {
        perror("listen");
        return;
    }

    listener_event = event_new(base, listener, EV_READ|EV_PERSIST, do_accept, (void*)base);
    /*XXX check it */
    event_add(listener_event, NULL);

    event_base_dispatch(base);
}

int
main(int c, char **v)
{
    setvbuf(stdout, NULL, _IONBF, 0);

    run();
    return 0;
}
```

**这一切的效率到底有多高？**

XXXX 在这里写一个效率部分。libevent页面上的基准测试已经非常过时。

1. 文件描述符是内核在你打开套接字时分配给套接字的数字。你使用这个数字来进行与套接字相关的Unix调用。
2. 在用户空间，生成和读取位数组可以使得时间与提供给select()的fd数量成正比。但在内核空间，读取位数组的时间与位数组中最大fd的数量成正比，这往往大约是程序中使用的fd总数，无论在select中添加了多少fd。
