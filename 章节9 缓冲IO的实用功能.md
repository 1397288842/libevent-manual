### Evbuffers: 缓冲 IO 的实用功能

Libevent 的 evbuffer 功能实现了一个字节队列，优化了向末尾添加数据和从前端移除数据的过程。

evbuffers 的设计目标是为缓冲网络 IO 的“缓冲”部分提供通用功能。它们不提供调度 IO 或在 IO 准备好时触发 IO 的功能；这正是 bufferevents 的作用。

本章中的函数声明位于 `event2/buffer.h`，除非另有说明。

### 创建或释放 evbuffer

#### 接口
```c
struct evbuffer *evbuffer_new(void);
void evbuffer_free(struct evbuffer *buf);
```

这些函数应该相对清晰：`evbuffer_new()` 分配并返回一个新的空 evbuffer，而 `evbuffer_free()` 则删除一个 evbuffer 及其所有内容。

这些函数自 Libevent 0.8 以来一直存在。

### evbuffer 和线程安全

#### 接口
```c
int evbuffer_enable_locking(struct evbuffer *buf, void *lock);
void evbuffer_lock(struct evbuffer *buf);
void evbuffer_unlock(struct evbuffer *buf);
```

默认情况下，多个线程无法同时安全地访问一个 evbuffer。如果需要这样做，可以在 evbuffer 上调用 `evbuffer_enable_locking()`。如果其 lock 参数为 NULL，Libevent 将使用提供给 `evthread_set_lock_creation_callback` 的锁创建函数分配一个新的锁。否则，它将使用作为参数传递的锁。

`evbuffer_lock()` 和 `evbuffer_unlock()` 函数分别获取和释放 evbuffer 上的锁。可以使用它们将一组操作设为原子操作。如果没有在 evbuffer 上启用锁，则这些函数什么都不做。

（请注意，您不需要在单个操作周围调用 `evbuffer_lock()` 和 `evbuffer_unlock()`：如果在 evbuffer 上启用了锁，则单个操作已经是原子的。仅当您需要执行多个操作而不希望其他线程插入时，才需要手动锁定 evbuffer。）

这些函数均在 Libevent 2.0.1-alpha 中引入。

### 检查 evbuffer

#### 接口
```c
size_t evbuffer_get_length(const struct evbuffer *buf);
```
此函数返回存储在 evbuffer 中的字节数。

该函数在 Libevent 2.0.1-alpha 中引入。

#### 接口
```c
size_t evbuffer_get_contiguous_space(const struct evbuffer *buf);
```
此函数返回当前在 evbuffer 前端连续存储的字节数。evbuffer 中的字节可能存储在多个独立的内存块中；此函数返回当前存储在第一个块中的字节数。

该函数在 Libevent 2.0.1-alpha 中引入。

### 向 evbuffer 添加数据：基础

#### 接口
```c
int evbuffer_add(struct evbuffer *buf, const void *data, size_t datlen);
```
此函数将 `data` 中的 `datlen` 字节附加到 `buf` 的末尾。成功时返回 0，失败时返回 -1。

#### 接口
```c
int evbuffer_add_printf(struct evbuffer *buf, const char *fmt, ...)
int evbuffer_add_vprintf(struct evbuffer *buf, const char *fmt, va_list ap);
```
这些函数将格式化的数据附加到 `buf` 的末尾。格式参数及其他剩余参数的处理方式与 C 标准库函数“printf”和“vprintf”相同。函数返回附加的字节数。

#### 接口
```c
int evbuffer_expand(struct evbuffer *buf, size_t datlen);
```
此函数更改缓冲区中的最后一个内存块，或添加一个新块，使缓冲区现在足够大以容纳 `datlen` 字节，而无需进一步的分配。
### 示例

```c
/* 这里有两种将 "Hello world 2.0.1" 添加到缓冲区的方法。 */
/* 直接添加： */
evbuffer_add(buf, "Hello world 2.0.1", 17);

/* 通过 printf 添加： */
evbuffer_add_printf(buf, "Hello %s %d.%d.%d", "world", 2, 0, 1);
```

`evbuffer_add()` 和 `evbuffer_add_printf()` 函数自 Libevent 0.8 引入；`evbuffer_expand()` 是在 Libevent 0.9 中引入的，而 `evbuffer_add_vprintf()` 首次出现在 Libevent 1.1 中。

### 从一个 evbuffer 移动数据到另一个

为了提高效率，Libevent 提供了优化函数，用于从一个 evbuffer 移动数据到另一个。

#### 接口
```c
int evbuffer_add_buffer(struct evbuffer *dst, struct evbuffer *src);
int evbuffer_remove_buffer(struct evbuffer *src, struct evbuffer *dst, size_t datlen);
```

`evbuffer_add_buffer()` 函数将所有数据从 `src` 移动到 `dst` 的末尾。成功时返回 0，失败时返回 -1。

`evbuffer_remove_buffer()` 函数将正好 `datlen` 字节从 `src` 移动到 `dst` 的末尾，尽量减少复制。如果要移动的字节少于 `datlen`，则移动所有字节。它返回移动的字节数。

`evbuffer_add_buffer()` 在 Libevent 0.8 中引入；`evbuffer_remove_buffer()` 在 Libevent 2.0.1-alpha 中新引入。

### 向 evbuffer 前端添加数据

#### 接口
```c
int evbuffer_prepend(struct evbuffer *buf, const void *data, size_t size);
int evbuffer_prepend_buffer(struct evbuffer *dst, struct evbuffer* src);
```

这些函数的行为与 `evbuffer_add()` 和 `evbuffer_add_buffer()` 类似，只不过它们将数据移动到目标缓冲区的前端。

这些函数应谨慎使用，切勿在与 `bufferevent` 共享的 `evbuffer` 上使用。它们在 Libevent 2.0.1-alpha 中引入。

### 重排 evbuffer 的内部布局

有时您想查看 `evbuffer` 前面 N 字节的数据，并将其视为一个连续的字节数组。为此，您必须首先确保缓冲区的前端确实是连续的。

#### 接口
```c
unsigned char *evbuffer_pullup(struct evbuffer *buf, ev_ssize_t size);
```

`evbuffer_pullup()` 函数“线性化” `buf` 的前 `size` 字节，根据需要复制或移动它们，以确保它们都是连续的，并占用相同的内存块。如果 `size` 为负，函数将线性化整个缓冲区。如果 `size` 大于缓冲区中的字节数，函数返回 NULL。否则，`evbuffer_pullup()` 返回 `buf` 中第一个字节的指针。

调用 `evbuffer_pullup()` 时，如果指定的大小过大，可能会非常缓慢，因为它可能需要复制整个缓冲区的内容。

### 示例
```c
#include <event2/buffer.h>
#include <event2/util.h>
#include <string.h>

int parse_socks4(struct evbuffer *buf, ev_uint16_t *port, ev_uint32_t *addr)
{
    /* 让我们解析 SOCKS4 请求的开始！格式很简单：
     * 1 字节的版本，1 字节的命令，2 字节的目标端口，4 字节的目标 IP。 */
    unsigned char *mem;

    mem = evbuffer_pullup(buf, 8);

    if (mem == NULL) {
        /* 缓冲区中的数据不足 */
        return 0;
    } else if (mem[0] != 4 || mem[1] != 1) {
        /* 未识别的协议或命令 */
        return -1;
    } else {
        memcpy(port, mem + 2, 2);
        memcpy(addr, mem + 4, 4);
        *port = ntohs(*port);
        *addr = ntohl(*addr);
        /* 现在我们知道喜欢这些数据了，实际上从缓冲区中删除这些数据。 */
        evbuffer_drain(buf, 8);
        return 1;
    }
}
```

### 注意
调用 `evbuffer_pullup()` 时，指定的大小等于由 `evbuffer_get_contiguous_space()` 返回的值将不会导致任何数据被复制或移动。

`evbuffer_pullup()` 函数在 Libevent 2.0.1-alpha 中引入；在此之前的版本的 Libevent 始终保持 evbuffer 数据连续，无论成本如何。

### 从 evbuffer 中移除数据

#### 接口
```c
int evbuffer_drain(struct evbuffer *buf, size_t len);
int evbuffer_remove(struct evbuffer *buf, void *data, size_t datlen);
```

`evbuffer_remove()` 函数将 `buf` 前面 `datlen` 字节的内容复制并移除到 `data` 指向的内存中。如果可用的字节少于 `datlen`，则函数会将所有字节复制到那里。返回值在失败时为 -1，否则返回复制的字节数。

`evbuffer_drain()` 函数的行为与 `evbuffer_remove()` 类似，只是不复制数据：它只是从缓冲区的前端移除数据。它成功时返回 0，失败时返回 -1。

Libevent 0.8 引入了 `evbuffer_drain()`；`evbuffer_remove()` 出现在 Libevent 0.9 中。

### 从 evbuffer 中复制数据
有时您希望在不移除数据的情况下获取缓冲区开头的数据副本。例如，您可能想查看某种记录是否完整到达，而不想通过 `evbuffer_remove` 移除任何数据，或通过 `evbuffer_pullup` 内部重排缓冲区。

**接口**
```c
ev_ssize_t evbuffer_copyout(struct evbuffer *buf, void *data, size_t datlen);
ev_ssize_t evbuffer_copyout_from(struct evbuffer *buf,
     const struct evbuffer_ptr *pos,
     void *data_out, size_t datlen);
```
`evbuffer_copyout()`的行为与`evbuffer_remove()`相似，但不会从缓冲区中排出任何数据。也就是说，它会将`buf`前面的`datlen`字节复制到`data`内存中。如果可用字节少于`datlen`，则函数会复制所有可用的字节。返回值为-1表示失败，否则返回复制的字节数。

`evbuffer_copyout_from()`函数的行为类似于`evbuffer_copyout()`，但它不是从缓冲区的前面复制字节，而是从提供的`pos`位置开始复制。有关`evbuffer_ptr`结构的信息，请参见“在evbuffer中搜索”部分。

如果从缓冲区复制数据太慢，请使用`evbuffer_peek()`代替。

**示例**
```c
#include <event2/buffer.h>
#include <event2/util.h>
#include <stdlib.h>

int get_record(struct evbuffer *buf, size_t *size_out, char **record_out)
{
    /* 假设我们正在使用某种协议，其中记录包含一个以网络顺序排列的4字节大小字段，后面跟随相应数量的字节。 
       如果我们拥有完整的记录，则返回1并设置'out'字段，如果记录还未到达则返回0，如果出错则返回-1。 */
    size_t buffer_len = evbuffer_get_length(buf);
    ev_uint32_t record_len;
    char *record;

    if (buffer_len < 4)
       return 0; /* 大小字段尚未到达。 */

    /* 我们在这里使用evbuffer_copyout，以便大小字段仍然保留在缓冲区中。 */
    evbuffer_copyout(buf, &record_len, 4);
    /* 将len_buf转换为主机序。 */
    record_len = ntohl(record_len);
    if (buffer_len < record_len + 4)
        return 0; /* 记录尚未到达 */

    /* 好的，现在我们可以删除记录。 */
    record = malloc(record_len);
    if (record == NULL)
        return -1;

    evbuffer_drain(buf, 4);
    evbuffer_remove(buf, record, record_len);

    *record_out = record;
    *size_out = record_len;
    return 1;
}
```
`evbuffer_copyout()`函数在Libevent 2.0.5-alpha中首次出现；`evbuffer_copyout_from()`是在Libevent 2.1.1-alpha中添加的。

**基于行的输入**
**接口**
```c
enum evbuffer_eol_style {
        EVBUFFER_EOL_ANY,
        EVBUFFER_EOL_CRLF,
        EVBUFFER_EOL_CRLF_STRICT,
        EVBUFFER_EOL_LF,
        EVBUFFER_EOL_NUL
};
char *evbuffer_readln(struct evbuffer *buffer, size_t *n_read_out,
    enum evbuffer_eol_style eol_style);
```
许多互联网协议使用基于行的格式。`evbuffer_readln()`函数从`evbuffer`的前面提取一行，并将其以新分配的NUL终止字符串返回。如果`n_read_out`不为NULL，则`*n_read_out`将设置为返回字符串中的字节数。如果没有完整的行可读，函数返回NULL。行终止符不会包括在复制的字符串中。

`evbuffer_readln()`函数理解四种行结束格式：

- **EVBUFFER_EOL_LF**：行的结束是一个换行符（也称为"\n"），其ASCII值为0x0A。
- **EVBUFFER_EOL_CRLF_STRICT**：行的结束是一个回车符，后面跟着一个换行符（也称为"\r\n"），ASCII值为0x0D 0x0A。
- **EVBUFFER_EOL_CRLF**：行的结束是一个可选的回车符，后面跟着一个换行符（换句话说，可以是"\r\n"或"\n"）。这种格式在解析基于文本的互联网协议时非常有用，因为标准通常规定使用"\r\n"作为行结束符，但不符合标准的客户端有时仅发送"\n"。
- **EVBUFFER_EOL_ANY**：行的结束是任意数量的回车符和换行符的任何序列。这种格式并不是很有用；它主要是为了向后兼容。
- **EVBUFFER_EOL_NUL**：行的结束是一个值为0的单字节，即ASCII NUL。

（请注意，如果你使用`event_set_mem_functions()`来重写默认的`malloc`，则`evbuffer_readln`返回的字符串将由你指定的`malloc`替代分配。）

**示例**
```c
char *request_line;
size_t len;

request_line = evbuffer_readln(buf, &len, EVBUFFER_EOL_CRLF);
if (!request_line) {
    /* 第一行尚未到达。 */
} else {
    if (!strncmp(request_line, "HTTP/1.0 ", 9)) {
        /* 检测到HTTP 1.0 ... */
    }
    free(request_line);
}
```
`evbuffer_readln()`接口在Libevent 1.4.14-stable及以后的版本中可用。`EVBUFFER_EOL_NUL`是在Libevent 2.1.1-alpha中添加的。

**在evbuffer中搜索**
`evbuffer_ptr`结构指向`evbuffer`中的某个位置，并包含可用于遍历`evbuffer`的数据。

**接口**
```c
struct evbuffer_ptr {
        ev_ssize_t pos;
        struct {
                /* 内部字段 */
        } _internal;
};
```
`pos`字段是唯一的公共字段；其他字段不应由用户代码使用。它指示`evbuffer`中的位置，作为从开始的偏移量。

**接口**
```c
struct evbuffer_ptr evbuffer_search(struct evbuffer *buffer,
    const char *what, size_t len, const struct evbuffer_ptr *start);
struct evbuffer_ptr evbuffer_search_range(struct evbuffer *buffer,
    const char *what, size_t len, const struct evbuffer_ptr *start,
    const struct evbuffer_ptr *end);
struct evbuffer_ptr evbuffer_search_eol(struct evbuffer *buffer,
    struct evbuffer_ptr *start, size_t *eol_len_out,
    enum evbuffer_eol_style eol_style);
```
`evbuffer_search()`函数扫描缓冲区以查找`what`的`len`字符字符串的出现。它返回一个包含字符串位置的`evbuffer_ptr`，如果未找到字符串，则返回-1。如果提供了`start`参数，则表示搜索应从该位置开始；否则，搜索将从字符串的开头开始。

`evbuffer_search_range()`函数的行为与`evbuffer_search`相同，但仅考虑出现在`evbuffer_ptr` `end`之前的`what`的出现。

`evbuffer_search_eol()`函数检测换行结束，类似于`evbuffer_readln()`，但不会复制出行，而是返回一个指向结束行字符的`evbuffer_ptr`。如果`eol_len_out`不为NULL，它会设置为EOL字符串的长度。

### 接口
```c
enum evbuffer_ptr_how {
        EVBUFFER_PTR_SET,
        EVBUFFER_PTR_ADD
};
int evbuffer_ptr_set(struct evbuffer *buffer, struct evbuffer_ptr *pos,
    size_t position, enum evbuffer_ptr_how how);
```

`evbuffer_ptr_set` 函数用于操作 `evbuffer_ptr` 在 `buffer` 中的位置。如果 `how` 是 `EVBUFFER_PTR_SET`，指针将移动到 `buffer` 中的绝对位置 `position`。如果是 `EVBUFFER_PTR_ADD`，指针将向前移动 `position` 个字节。此函数成功时返回 0，失败时返回 -1。

### 示例代码
```c
#include <event2/buffer.h>
#include <string.h>

/* 计算 'str' 在 'buf' 中出现的总次数。 */
int count_instances(struct evbuffer *buf, const char *str)
{
    size_t len = strlen(str);
    int total = 0;
    struct evbuffer_ptr p;

    if (!len)
        /* 不要尝试计算 0 长度字符串的出现次数。 */
        return -1;

    evbuffer_ptr_set(buf, &p, 0, EVBUFFER_PTR_SET);

    while (1) {
         p = evbuffer_search(buf, str, len, &p);
         if (p.pos < 0)
             break;
         total++;
         evbuffer_ptr_set(buf, &p, 1, EVBUFFER_PTR_ADD);
    }

    return total;
}
```

### 警告
任何修改 `evbuffer` 或其布局的调用都会使所有现有的 `evbuffer_ptr` 值无效，并使其不安全使用。

这些接口在 Libevent 2.0.1-alpha 中首次引入。

### 无需复制的数据检查
有时，你想在不复制的情况下读取 `evbuffer` 中的数据（就像 `evbuffer_copyout()` 做的那样），并且不重新排列 `evbuffer` 的内部内存（就像 `evbuffer_pullup()` 做的那样）。有时你可能想查看 `evbuffer` 中间的数据。

你可以使用以下接口：

```c
struct evbuffer_iovec {
        void *iov_base;
        size_t iov_len;
};

int evbuffer_peek(struct evbuffer *buffer, ev_ssize_t len,
    struct evbuffer_ptr *start_at,
    struct evbuffer_iovec *vec_out, int n_vec);
```

调用 `evbuffer_peek()` 时，你需要提供一个 `evbuffer_iovec` 结构数组作为 `vec_out`。数组的长度为 `n_vec`。它会设置这些结构，使每个结构都包含一个指向 `evbuffer` 内存块的指针（`iov_base`），以及该块内存的长度。

如果 `len` 小于 0，`evbuffer_peek()` 将尝试填充你提供的所有 `evbuffer_iovec` 结构。否则，它会填充它们，直到它们全部使用，或者至少 `len` 字节可见。如果函数能够提供你请求的所有数据，则返回它实际使用的 `evbuffer_iovec` 结构数量。否则，它会返回提供所需数量的结构数。

当 `ptr` 为 NULL 时，`evbuffer_peek()` 从缓冲区开始。如果不是，则从 `ptr` 指定的指针开始。

### 示例
```c
{
    /* 查看 buf 的前两个块，并将它们写入 stderr。 */
    int n, i;
    struct evbuffer_iovec v[2];
    n = evbuffer_peek(buf, -1, NULL, v, 2);
    for (i=0; i<n; ++i) { /* 可能没有两个块可用。 */
        fwrite(v[i].iov_base, 1, v[i].iov_len, stderr);
    }
}

{
    /* 将前 4906 字节发送到 stdout 通过 write。 */
    int n, i, r;
    struct evbuffer_iovec *v;
    size_t written = 0;

    /* 确定我们需要多少块。 */
    n = evbuffer_peek(buf, 4096, NULL, NULL, 0);
    /* 为这些块分配空间。使用 alloca() 可能是个好时机。 */
    v = malloc(sizeof(struct evbuffer_iovec) * n);
    /* 实际填充 v。 */
    n = evbuffer_peek(buf, 4096, NULL, v, n);
    for (i=0; i<n; ++i) {
        size_t len = v[i].iov_len;
        if (written + len > 4096)
            len = 4096 - written;
        r = write(1 /* stdout */, v[i].iov_base, len);
        if (r <= 0)
            break;
        /* 我们单独跟踪写入的字节；如果我们不这样做，
           如果最后一个块使我们超过限制，可能会写入超过 4096 个字节。 */
        written += len;
    }
    free(v);
}

{
    /* 在第一个出现 "start\n" 的字符串之后获取前 16K 数据，并传递给 consume() 函数。 */
    struct evbuffer_ptr ptr;
    struct evbuffer_iovec v[1];
    const char s[] = "start\n";
    int n_written = 0; // 初始化 n_written

    ptr = evbuffer_search(buf, s, strlen(s), NULL);
    if (ptr.pos == -1)
        return; /* 找不到起始字符串。 */

    /* 指针向前移动过起始字符串。 */
    if (evbuffer_ptr_set(buf, &ptr, strlen(s), EVBUFFER_PTR_ADD) < 0)
        return; /* 超出字符串末尾。 */

    while (n_written < 16 * 1024) {
        /* 预览一个块。 */
        if (evbuffer_peek(buf, -1, &ptr, v, 1) < 1)
            break;
        /* 将数据传递给某个用户定义的消费函数 */
        consume(v[0].iov_base, v[0].iov_len);
        n_written += v[0].iov_len;

        /* 指针向前移动，以便下次查看下一个块。 */
        if (evbuffer_ptr_set(buf, &ptr, v[0].iov_len, EVBUFFER_PTR_ADD) < 0)
            break;
    }
}
```

### 注意事项
直接修改 `evbuffer_iovec` 指向的数据可能导致未定义的行为。

如果调用了任何修改 `evbuffer` 的函数，则 `evbuffer_peek()` 生成的指针可能变得无效。

如果你的 `evbuffer` 可能在多个线程中使用，请确保在调用 `evbuffer_peek()` 之前用 `evbuffer_lock()` 锁定它，并在使用完 `evbuffer_peek()` 提供的扩展后解锁它。

该函数在 Libevent 2.0.2-alpha 中引入。

### 直接向 evbuffer 添加数据
有时，你想直接向 `evbuffer` 插入数据，而不必先将其写入字符数组，然后再使用 `evbuffer_add()` 复制进去。你可以使用一对高级函数：`evbuffer_reserve_space()` 和 `evbuffer_commit_space()`。与 `evbuffer_peek()` 一样，这些函数使用 `evbuffer_iovec` 结构提供对 `evbuffer` 内部内存的直接访问。

### 接口
```c
int evbuffer_reserve_space(struct evbuffer *buf, ev_ssize_t size,
    struct evbuffer_iovec *vec, int n_vecs);
int evbuffer_commit_space(struct evbuffer *buf,
    struct evbuffer_iovec *vec, int n_vecs);
```

`evbuffer_reserve_space()` 函数为你提供指向 `evbuffer` 内部空间的指针。它根据需要扩展缓冲区，以便为你提供至少 `size` 字节。指向这些空间的指针及其长度将存储在你通过 `vec` 传递的向量数组中；`n_vec` 是该数组的长度。

`n_vec` 的值必须至少为 1。如果你只提供一个向量，Libevent 会确保你在单个块中获得你请求的所有连续空间，但它可能不得不重新排列缓冲区或浪费内存以做到这一点。为了获得更好的性能，提供至少两个向量。该函数返回满足你请求的空间所需的向量数量。

你写入这些向量的数据在调用 `evbuffer_commit_space()` 之前不算在缓冲区中。该函数实际将你写入的数据算作在缓冲区中。如果你想提交比请求的空间更少的内容，可以减少你所获得的任何 `evbuffer_iovec` 结构中的 `iov_len` 字段。你也可以返回比获得的更少的向量。`evbuffer_commit_space()` 函数成功时返回 0，失败时返回 -1。
### 说明和注意事项

调用任何会重新排列 `evbuffer` 或向其添加数据的函数将使您从 `evbuffer_reserve_space()` 获取的指针失效。

在当前实现中，`evbuffer_reserve_space()` 无论用户提供多少向量，最多只会使用两个向量。将来版本可能会发生变化。

可以安全地多次调用 `evbuffer_reserve_space()`。

如果您的 `evbuffer` 可能在多个线程中使用，请确保在调用 `evbuffer_reserve_space()` 之前使用 `evbuffer_lock()` 进行锁定，并在提交后解锁。

### 示例

```c
/* 假设我们想从 generate_data() 函数填充 2048 字节的输出到缓冲区，而不进行复制。 */
struct evbuffer_iovec v[2];
int n, i;
size_t n_to_add = 2048;

/* 保留 2048 字节。 */
n = evbuffer_reserve_space(buf, n_to_add, v, 2);
if (n <= 0)
   return; /* 无法保留空间。 */

for (i = 0; i < n && n_to_add > 0; ++i) {
   size_t len = v[i].iov_len;
   if (len > n_to_add) /* 不要写入超过 n_to_add 字节。 */
      len = n_to_add;
   if (generate_data(v[i].iov_base, len) < 0) {
      /* 如果在数据生成过程中出现问题，我们可以在此停止；
         不会将任何数据提交到缓冲区。 */
      return;
   }
   /* 设置 iov_len 为我们实际写入的字节数，以便
      不会提交过多的数据。 */
   v[i].iov_len = len;
}

/* 在这里提交空间。请注意，我们传递的是 'i'（实际使用的向量数量），
   而不是 'n'（我们拥有的向量数量）。 */
if (evbuffer_commit_space(buf, v, i) < 0)
   return; /* 提交时出错 */
```

### 错误示例

以下是您可能在使用 `evbuffer_reserve()` 时犯的错误示例。**不要模仿此代码。**

```c
struct evbuffer_iovec v[2];

{
  /* 不要在调用任何会修改缓冲区的函数后使用
     evbuffer_reserve_space() 中的指针。 */
  evbuffer_reserve_space(buf, 1024, v, 2);
  evbuffer_add(buf, "X", 1);
  /* 错误：如果 evbuffer_add 需要重新排列
     缓冲区的内容，这一行将无法正常工作。可能会导致程序崩溃。
     相反，您应该在调用 evbuffer_reserve_space 之前添加数据。 */
  memset(v[0].iov_base, 'Y', v[0].iov_len - 1);
  evbuffer_commit_space(buf, v, 1);
}

{
  /* 不要修改 iov_base 指针。 */
  const char *data = "Here is some data";
  evbuffer_reserve_space(buf, strlen(data), v, 1);
  /* 错误：下一行不会按预期工作。相反，您
     应该将数据的内容复制到 v[0].iov_base 中。 */
  v[0].iov_base = (char*) data;
  v[0].iov_len = strlen(data);
  /* 在这种情况下，如果您运气好，evbuffer_commit_space 可能会返回错误 */
  evbuffer_commit_space(buf, v, 1);
}
```

这些函数自 Libevent 2.0.2-alpha 以来就以其当前接口存在。

### 使用 evbuffer 进行网络 IO

在 Libevent 中，使用 `evbuffer` 的最常见用例是网络 IO。执行网络 IO 的接口如下：

#### 接口

```c
int evbuffer_write(struct evbuffer *buffer, evutil_socket_t fd);
int evbuffer_write_atmost(struct evbuffer *buffer, evutil_socket_t fd, ev_ssize_t howmuch);
int evbuffer_read(struct evbuffer *buffer, evutil_socket_t fd, int howmuch);
```

`evbuffer_read()` 函数从套接字 `fd` 读取最多 `howmuch` 字节到缓冲区的末尾。成功时返回读取的字节数，EOF 时返回 0，出错时返回 -1。请注意，错误可能表示非阻塞操作无法成功；您需要检查错误代码以判断是否为 EAGAIN（或在 Windows 上的 WSAEWOULDBLOCK）。如果 `howmuch` 为负，`evbuffer_read()` 尝试自行猜测要读取多少字节。

`evbuffer_write_atmost()` 函数尝试从缓冲区的前面写入最多 `howmuch` 字节到套接字 `fd`。成功时返回写入的字节数，出错时返回 -1。与 `evbuffer_read()` 一样，您需要检查错误代码以判断错误是否真实，或者只是表示非阻塞 IO 无法立即完成。如果您传入一个负值，`evbuffer_write_atmost()` 将尝试写入缓冲区的全部内容。

调用 `evbuffer_write()` 等同于以负 `howmuch` 参数调用 `evbuffer_write_atmost()`：它尝试尽可能多地刷新缓冲区的内容。

在 Unix 上，这些函数应适用于任何支持读取和写入的文件描述符。在 Windows 上，仅支持套接字。

注意，当您使用 `bufferevents` 时，无需调用这些 IO 函数；`bufferevents` 代码会为您处理这些操作。

`evbuffer_write_atmost()` 函数是在 Libevent 2.0.1-alpha 中引入的。

### evbuffer 和回调

`evbuffer` 的用户通常想知道何时向 `evbuffer` 添加或删除数据。为支持这一点，Libevent 提供了一个通用的 `evbuffer` 回调机制。

#### 接口

```c
struct evbuffer_cb_info {
        size_t orig_size;
        size_t n_added;
        size_t n_deleted;
};

typedef void (*evbuffer_cb_func)(struct evbuffer *buffer,
    const struct evbuffer_cb_info *info, void *arg);
```

`evbuffer` 回调在向 `evbuffer` 添加或删除数据时被调用。它接收缓冲区、指向 `evbuffer_cb_info` 结构的指针，以及用户提供的参数。`evbuffer_cb_info` 结构的 `orig_size` 字段记录缓冲区在大小变化之前的字节数；`n_added` 字段记录添加到缓冲区的字节数，而 `n_deleted` 字段记录从缓冲区中删除的字节数。
### 接口
```c
struct evbuffer_cb_entry;
struct evbuffer_cb_entry *evbuffer_add_cb(struct evbuffer *buffer,
    evbuffer_cb_func cb, void *cbarg);
```
`evbuffer_add_cb()` 函数向 `evbuffer` 添加一个回调，并返回一个不透明指针，可以用于后续引用此特定回调实例。`cb` 参数是将被调用的函数，而 `cbarg` 是传递给该函数的用户提供的指针。

你可以在单个 `evbuffer` 上设置多个回调。添加新回调不会移除旧的回调。

### 示例
```c
#include <event2/buffer.h>
#include <stdio.h>
#include <stdlib.h>

/* 这是一个回调，它记住我们从缓冲区中总共排空的字节数，
   每当我们达到一兆字节时就打印一个点。 */
struct total_processed {
    size_t n;
};

void count_megabytes_cb(struct evbuffer *buffer,
    const struct evbuffer_cb_info *info, void *arg)
{
    struct total_processed *tp = arg;
    size_t old_n = tp->n;
    int megabytes, i;
    tp->n += info->n_deleted;
    megabytes = ((tp->n) >> 20) - (old_n >> 20);
    for (i=0; i<megabytes; ++i)
        putc('.', stdout);
}

void operation_with_counted_bytes(void)
{
    struct total_processed *tp = malloc(sizeof(*tp));
    struct evbuffer *buf = evbuffer_new();
    tp->n = 0;
    evbuffer_add_cb(buf, count_megabytes_cb, tp);

    /* 使用 evbuffer 一段时间后，当我们完成时： */
    evbuffer_free(buf);
    free(tp);
}
```
请注意，释放一个非空的 `evbuffer` 并不会计算为从中排空数据，并且释放 `evbuffer` 不会释放其回调的用户提供的数据指针。

如果你不想让回调在缓冲区上永久处于活动状态，你可以移除它（使其永久消失）或禁用它（暂时关闭）。

### 接口
```c
int evbuffer_remove_cb_entry(struct evbuffer *buffer,
    struct evbuffer_cb_entry *ent);
int evbuffer_remove_cb(struct evbuffer *buffer, evbuffer_cb_func cb,
    void *cbarg);
```
```c
#define EVBUFFER_CB_ENABLED 1
int evbuffer_cb_set_flags(struct evbuffer *buffer,
                          struct evbuffer_cb_entry *cb,
                          ev_uint32_t flags);
int evbuffer_cb_clear_flags(struct evbuffer *buffer,
                          struct evbuffer_cb_entry *cb,
                          ev_uint32_t flags);
```
你可以通过在添加时获得的 `evbuffer_cb_entry` 或使用的回调和指针移除回调。`evbuffer_remove_cb()` 函数成功时返回 0，失败时返回 -1。

`evbuffer_cb_set_flags()` 函数和 `evbuffer_cb_clear_flags()` 函数可以分别使给定的标志被设置或清除。当前，只有一个用户可见的标志支持：`EVBUFFER_CB_ENABLED`。该标志默认为设置状态。当其被清除时，`evbuffer` 的修改不会导致此回调被调用。

### 接口
```c
int evbuffer_defer_callbacks(struct evbuffer *buffer, struct event_base *base);
```
与 `bufferevent` 回调一样，你可以使 `evbuffer` 回调在 `evbuffer` 发生变化时不立即运行，而是推迟到给定事件基的事件循环中运行。这在你有多个 `evbuffer` 的情况下特别有用，这些回调可能会导致相互之间添加和删除数据，并且你希望避免栈崩溃。

如果 `evbuffer` 的回调被推迟，则在它们最终被调用时，可能会总结多个操作的结果。

与 `bufferevent` 一样，`evbuffer` 内部是引用计数的，因此即使有推迟的回调尚未执行，也可以安全地释放 `evbuffer`。

整个回调系统是在 Libevent 2.0.1-alpha 中引入的，而 `evbuffer_cb_(set|clear)_flags()` 函数自 2.0.2-alpha 以来保持现有接口。

### 避免通过 evbuffer 进行 IO 的数据拷贝
在高效的网络编程中，通常需要尽量减少数据拷贝。Libevent 提供了一些机制来帮助实现这一点。

### 接口
```c
typedef void (*evbuffer_ref_cleanup_cb)(const void *data,
    size_t datalen, void *extra);

int evbuffer_add_reference(struct evbuffer *outbuf,
    const void *data, size_t datlen,
    evbuffer_ref_cleanup_cb cleanupfn, void *extra);
```
此函数通过引用将一段数据添加到 `evbuffer` 的末尾。不会进行拷贝：相反，`evbuffer` 仅存储指向 `data` 中存储的 `datlen` 字节的指针。因此，在 `evbuffer` 使用时，该指针必须保持有效。当 `evbuffer` 不再需要数据时，将调用提供的 `cleanupfn` 函数，并传递提供的 `data` 指针、`datlen` 值和 `extra` 指针作为参数。该函数成功时返回 0，失败时返回 -1。

### 示例
```c
#include <event2/buffer.h>
#include <stdlib.h>
#include <string.h>

/* 在此示例中，我们有一些 evbuffer，希望将一个一兆字节的资源
   流向网络。我们这样做时没有在内存中保留多余的资源副本。 */

#define HUGE_RESOURCE_SIZE (1024*1024)
struct huge_resource {
    /* 我们保留对该结构的引用计数，
       以便知道何时可以释放它。 */
    int reference_count;
    char data[HUGE_RESOURCE_SIZE];
};

struct huge_resource *new_resource(void) {
    struct huge_resource *hr = malloc(sizeof(struct huge_resource));
    hr->reference_count = 1;
    /* 这里我们应该填充 hr->data 的内容。在实际应用中，
       我们可能会加载某些内容或进行复杂的计算。
       此处，我们将其填充为 EE。 */
    memset(hr->data, 0xEE, sizeof(hr->data));
    return hr;
}

void free_resource(struct huge_resource *hr) {
    --hr->reference_count;
    if (hr->reference_count == 0)
        free(hr);
}

static void cleanup(const void *data, size_t len, void *arg) {
    free_resource(arg);
}

/* 此函数实际将资源添加到缓冲区。 */
void spool_resource_to_evbuffer(struct evbuffer *buf,
    struct huge_resource *hr)
{
    ++hr->reference_count;
    evbuffer_add_reference(buf, hr->data, HUGE_RESOURCE_SIZE,
        cleanup, hr);
}
```
`evbuffer_add_reference()` 函数自 2.0.2-alpha 以来保持现有接口。

### 将文件添加到 evbuffer
某些操作系统提供了将文件写入网络的方式，而无需将数据复制到用户空间。可以通过简单的接口访问这些机制（如果可用）：
## 接口

### `int evbuffer_add_file(struct evbuffer *output, int fd, ev_off_t offset, size_t length);`

`evbuffer_add_file()` 函数假定有一个可供读取的打开文件描述符（而不是套接字），该描述符为 `fd`。它将从文件中从位置 `offset` 开始的 `length` 字节添加到 `output` 的末尾。成功时返回 `0`，失败时返回 `-1`。

### 警告

在 Libevent 2.0.x 中，通过这种方式添加的数据的唯一可靠操作是使用 `evbuffer_write*()` 将其发送到网络、使用 `evbuffer_drain()` 清除它，或使用 `evbuffer_*_buffer()` 将其移动到另一个 `evbuffer` 中。无法可靠地使用 `evbuffer_remove()` 从缓冲区中提取数据，或使用 `evbuffer_pullup()` 线性化数据。Libevent 2.1.x 尝试修复此限制。

如果您的操作系统支持 `splice()` 或 `sendfile()`，Libevent 将在调用 `evbuffer_write()` 时直接从 `fd` 向网络发送数据，而无需将数据复制到用户内存中。如果不支持 `splice/sendfile`，但您有 `mmap()`，Libevent 将对文件进行 `mmap`，并希望您的内核能够判断它永远不需要将数据复制到用户空间。否则，Libevent 将直接从磁盘读取数据到内存中。

文件描述符将在数据从 `evbuffer` 中刷新后，或在 `evbuffer` 被释放时关闭。如果这不是您想要的，或者如果您想对文件进行更细粒度的控制，请查看下面的文件段功能。

该函数在 Libevent 2.0.1-alpha 中引入。

---

### 细粒度的文件段控制

`evbuffer_add_file()` 接口在多次添加相同文件时效率低下，因为它会接管文件的所有权。

### 接口

```c
struct evbuffer_file_segment;

struct evbuffer_file_segment *evbuffer_file_segment_new(
        int fd, ev_off_t offset, ev_off_t length, unsigned flags);
void evbuffer_file_segment_free(struct evbuffer_file_segment *seg);
int evbuffer_add_file_segment(struct evbuffer *buf,
    struct evbuffer_file_segment *seg, ev_off_t offset, ev_off_t length);
```

`evbuffer_file_segment_new()` 函数创建并返回一个新的 `evbuffer_file_segment` 对象，用于表示存储在 `fd` 中的基础文件的一部分，该部分从 `offset` 开始，包含 `length` 字节。如果出错，返回 `NULL`。

文件段是通过 `sendfile`、`splice`、`mmap`、`CreateFileMapping` 或 `malloc()`-并读取() 实现的，具体取决于适用情况。它们使用最轻量的支持机制创建，并在需要时过渡到更重的机制。（例如，如果您的操作系统支持 `sendfile` 和 `mmap`，则文件段可以仅使用 `sendfile` 实现，直到您尝试实际检查其内容为止。在那时，它需要进行 `mmap`。）您可以使用以下标志控制文件段的细粒度行为：

- **EVBUF_FS_CLOSE_ON_FREE**  
  如果设置了此标志，使用 `evbuffer_file_segment_free()` 释放文件段时将关闭基础文件。

- **EVBUF_FS_DISABLE_MMAP**  
  如果设置了此标志，文件段将永远不会使用映射内存风格的后端（`CreateFileMapping`、`mmap`）来处理该文件，即使这样做是合适的。

- **EVBUF_FS_DISABLE_SENDFILE**  
  如果设置了此标志，文件段将永远不会使用 `sendfile` 风格的后端（`sendfile`、`splice`）来处理该文件，即使这样做是合适的。

- **EVBUF_FS_DISABLE_LOCKING**  
  如果设置了此标志，则不会为文件段分配任何锁：在任何情况下使用它都不安全。

一旦您拥有了 `evbuffer_file_segment`，就可以使用 `evbuffer_add_file_segment()` 将其中的全部或部分添加到 `evbuffer` 中。此处的 `offset` 参数指的是文件段内的偏移量，而不是文件本身内的偏移量。

当您不再想使用文件段时，可以使用 `evbuffer_file_segment_free()` 释放它。实际存储不会在没有任何 `evbuffer` 再持有文件段的引用之前释放。

### 接口

```c
typedef void (*evbuffer_file_segment_cleanup_cb)(
    struct evbuffer_file_segment const *seg, int flags, void *arg);
```

```c
void evbuffer_file_segment_add_cleanup_cb(struct evbuffer_file_segment *seg,
        evbuffer_file_segment_cleanup_cb cb, void *arg);
```

您可以向文件段添加回调函数，该函数将在释放文件段的最后一个引用并且即将被释放时调用。此回调不得尝试复活文件段、将其添加到任何缓冲区等。

这些文件段函数首次出现在 Libevent 2.1.1-alpha 中；`evbuffer_file_segment_add_cleanup_cb()` 在 2.1.2-alpha 中添加。

---

### 通过引用将一个 evbuffer 添加到另一个

您还可以通过引用将一个 `evbuffer` 添加到另一个：而不是移除一个缓冲区的内容并将其添加到另一个缓冲区，您可以给一个 `evbuffer` 一个对另一个的引用，使其表现得就像您复制了所有字节一样。

### 接口

```c
int evbuffer_add_buffer_reference(struct evbuffer *outbuf,
    struct evbuffer *inbuf);
```

`evbuffer_add_buffer_reference()` 函数的行为就像您将所有数据从 `outbuf` 复制到 `inbuf` 一样，但不会执行任何不必要的复制。成功时返回 `0`，失败时返回 `-1`。

请注意，对 `inbuf` 内容的后续更改不会反映在 `outbuf` 中：此函数通过引用添加了 `evbuffer` 的当前内容，而不是 `evbuffer` 本身。

请注意，您无法嵌套缓冲区引用：已经成为一个 `evbuffer_add_buffer_reference` 调用的 `outbuf` 不能再是另一个的 `inbuf`。

该函数在 Libevent 2.1.1-alpha 中引入。

### 将 evbuffer 设置为仅添加或删除模式

#### 接口

```c
int evbuffer_freeze(struct evbuffer *buf, int at_front);
int evbuffer_unfreeze(struct evbuffer *buf, int at_front);
```

您可以使用这些函数暂时禁用对 `evbuffer` 前端或后端的更改。`bufferevent` 代码在内部使用它们，以防止意外修改输出缓冲区的前端或输入缓冲区的后端。

`evbuffer_freeze()` 函数在 Libevent 2.0.1-alpha 中引入。

---

### 弃用的 evbuffer 函数

在 Libevent 2.0 中，`evbuffer` 接口发生了很大变化。在此之前，每个 `evbuffer` 都被实现为一个连续的内存块，这使得访问效率非常低下。

`event.h` 头文件曾经公开了 `struct evbuffer` 的内部结构。这些内部结构不再可用；由于 1.4 和 2.0 之间的变化太大，任何依赖于它们的代码都无法正常工作。

要访问 `evbuffer` 中的字节数，以前有一个 `EVBUFFER_LENGTH()` 宏。实际数据可以通过 `EVBUFFER_DATA()` 访问。这两个宏在 `event2/buffer_compat.h` 中仍然可用。但要注意，`EVBUFFER_DATA(b)` 是 `evbuffer_pullup(b, -1)` 的别名，这可能会非常耗时。

以下是一些其他已弃用的接口：

#### 弃用接口

```c
char *evbuffer_readline(struct evbuffer *buffer);
unsigned char *evbuffer_find(struct evbuffer *buffer,
    const unsigned char *what, size_t len);
```

`evbuffer_readline()` 函数的工作方式与当前的 `evbuffer_readln(buffer, NULL, EVBUFFER_EOL_ANY)` 类似。

`evbuffer_find()` 函数会在缓冲区中搜索字符串的首次出现，并返回指向它的指针。与 `evbuffer_search()` 不同，它只能找到第一个字符串。为了与使用此函数的旧代码兼容，现在它会将整个缓冲区线性化，直到找到的字符串末尾。

#### 回调接口也不同：

#### 弃用接口

```c
typedef void (*evbuffer_cb)(struct evbuffer *buffer,
    size_t old_len, size_t new_len, void *arg);
void evbuffer_setcb(struct evbuffer *buffer, evbuffer_cb cb, void *cbarg);
```

一个 `evbuffer` 一次只能设置一个回调，因此设置新的回调会禁用先前的回调，而设置 `NULL` 回调是禁用回调的推荐方法。

该函数的调用返回的是 `evbuffer_cb_info` 结构的旧长度和新长度。因此，如果 `old_len` 大于 `new_len`，则数据被清除；如果 `new_len` 大于 `old_len`，则数据被添加。无法延迟回调，因此添加和删除操作不会被合并到单个回调调用中。

这里的废弃函数仍然可以在 `event2/buffer_compat.h` 中找到。
