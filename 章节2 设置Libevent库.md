# 设置Libevent库

Libevent有一些全局设置，这些设置在整个进程中共享，并影响整个库。

在调用Libevent库的任何其他部分之前，您必须对这些设置进行任何更改。如果不这样做，Libevent可能会处于不一致的状态。

## Libevent中的日志消息

Libevent可以记录内部错误和警告。如果在编译时启用了日志支持，它还会记录调试消息。默认情况下，这些消息写入标准错误(stderr)。您可以通过提供自己的日志记录函数来覆盖此行为。

### 接口

```c
#define EVENT_LOG_DEBUG 0
#define EVENT_LOG_MSG   1
#define EVENT_LOG_WARN  2
#define EVENT_LOG_ERR   3

/* 已弃用；请参见本节末尾的说明 */
#define _EVENT_LOG_DEBUG EVENT_LOG_DEBUG
#define _EVENT_LOG_MSG   EVENT_LOG_MSG
#define _EVENT_LOG_WARN  EVENT_LOG_WARN
#define _EVENT_LOG_ERR   EVENT_LOG_ERR

typedef void (*event_log_cb)(int severity, const char *msg);

void event_set_log_callback(event_log_cb cb);
```

要覆盖Libevent的日志记录行为，请编写一个与`event_log_cb`签名匹配的函数，并将其作为参数传递给`event_set_log_callback()`。每当Libevent想要记录一条消息时，它将把该消息传递给您提供的函数。您可以通过再次调用`event_set_log_callback()`并将`NULL`作为参数来使Libevent恢复到其默认行为。

### 示例

```c
#include <event2/event.h>
#include <stdio.h>

static void discard_cb(int severity, const char *msg)
{
    /* 此回调不执行任何操作。 */
}

static FILE *logfile = NULL;
static void write_to_file_cb(int severity, const char *msg)
{
    const char *s;
    if (!logfile)
        return;
    switch (severity) {
        case _EVENT_LOG_DEBUG: s = "debug"; break;
        case _EVENT_LOG_MSG:   s = "msg";   break;
        case _EVENT_LOG_WARN:  s = "warn";  break;
        case _EVENT_LOG_ERR:   s = "error"; break;
        default:               s = "?";     break; /* 永远不会到达 */
    }
    fprintf(logfile, "[%s] %s\n", s, msg);
}

/* 关闭Libevent的所有日志记录。 */
void suppress_logging(void)
{
    event_set_log_callback(discard_cb);
}

/* 将所有Libevent日志消息重定向到C标准IO文件'f'。 */
void set_logfile(FILE *f)
{
    logfile = f;
    event_set_log_callback(write_to_file_cb);
}
```

### 注意

在用户提供的`event_log_cb`回调内部调用Libevent函数是不安全的！例如，如果您尝试编写一个日志回调，使用bufferevents将警告消息发送到网络套接字，您可能会遇到奇怪且难以诊断的错误。在未来版本的Libevent中，此限制可能会对某些函数移除。

通常情况下，调试日志默认未启用，并且不会发送到日志回调。如果Libevent构建时支持调试日志，您可以手动启用它们。

### 接口

```c
#define EVENT_DBG_NONE 0
#define EVENT_DBG_ALL 0xffffffffu

void event_enable_debug_logging(ev_uint32_t which);
```

调试日志非常详细，并且在大多数情况下未必有用。调用`event_enable_debug_logging()`并传递`EVENT_DBG_NONE`将获得默认行为；调用它并传递`EVENT_DBG_ALL`将启用所有支持的调试日志。未来版本可能会支持更细粒度的选项。

这些函数在`<event2/event.h>`中声明。它们首次出现在Libevent 1.0c中，除了`event_enable_debug_logging()`，它首次出现在Libevent 2.1.1-alpha中。

### 兼容性说明

在Libevent 2.0.19-stable之前，`EVENT_LOG_*`宏的名称以一个下划线开头：`_EVENT_LOG_DEBUG`、`_EVENT_LOG_MSG`、`_EVENT_LOG_WARN`和`_EVENT_LOG_ERR`。这些旧名称已弃用，仅应与Libevent 2.0.18-stable及更早版本的向后兼容使用。它们可能在未来版本的Libevent中被移除。

## 处理致命错误

当Libevent检测到不可恢复的内部错误（例如，损坏的数据结构）时，其默认行为是调用`exit()`或`abort()`来终止当前运行的进程。这些错误几乎总是意味着在您的代码或Libevent本身中存在一个错误。

如果您希望应用程序以更优雅的方式处理致命错误，可以通过提供一个函数来覆盖Libevent的行为，该函数将被Libevent在遇到致命错误时调用。

### 接口

```c
typedef void (*event_fatal_cb)(int err);
void event_set_fatal_callback(event_fatal_cb cb);
```

要使用这些函数，首先定义一个新的函数，当遇到致命错误时Libevent应该调用该函数，然后将其传递给`event_set_fatal_callback()`。稍后，如果Libevent遇到致命错误，它将调用您提供的函数。

您的函数不应将控制权返回给Libevent；这样做可能会导致未定义行为，并且Libevent可能仍会退出以避免崩溃。一旦调用了您的函数，您不应再调用任何其他Libevent函数。

这些函数在`<event2/event.h>`中声明。它们首次出现在Libevent 2.0.3-alpha中。

### 内存管理

默认情况下，Libevent 使用 C 库的内存管理函数从堆中分配内存。你可以通过提供自己的 `malloc`、`realloc` 和 `free` 函数来让 Libevent 使用另一个内存管理器。如果你有一个更高效的分配器，想让 Libevent 使用，或者你有一个带有内存泄漏检测功能的分配器，想让 Libevent 使用，你可以做这样的替换。

### 接口

```c
void event_set_mem_functions(void *(*malloc_fn)(size_t sz),
                             void *(*realloc_fn)(void *ptr, size_t sz),
                             void (*free_fn)(void *ptr));
```

以下是一个简单的示例，它用一个变体来替换 Libevent 的分配函数，这个变体会计算总共分配的字节数。实际上，你可能需要在这里添加锁，以防 Libevent 在多个线程中运行时发生错误。

### 示例

```c
#include <event2/event.h>
#include <sys/types.h>
#include <stdlib.h>

/* 这个联合体的目的就是使它的大小与其中最大的类型一致。 */
union alignment {
    size_t sz;
    void *ptr;
    double dbl;
};

/* 我们需要确保返回的内存具有正确的对齐方式，能够容纳任何类型的数据，包括 double 类型。 */
#define ALIGNMENT sizeof(union alignment)

/* 我们需要使用指针类型转换为 char* 来调整指针；对 void* 指针做算术运算在标准中是不允许的。 */
#define OUTPTR(ptr) (((char*)ptr) + ALIGNMENT)
#define INPTR(ptr)  (((char*)ptr) - ALIGNMENT)

static size_t total_allocated = 0;

static void *replacement_malloc(size_t sz)
{
    void *chunk = malloc(sz + ALIGNMENT);
    if (!chunk) return chunk;
    total_allocated += sz;
    *(size_t*)chunk = sz;
    return OUTPTR(chunk);
}

static void *replacement_realloc(void *ptr, size_t sz)
{
    size_t old_size = 0;
    if (ptr) {
        ptr = INPTR(ptr);
        old_size = *(size_t*)ptr;
    }
    ptr = realloc(ptr, sz + ALIGNMENT);
    if (!ptr)
        return NULL;
    *(size_t*)ptr = sz;
    total_allocated = total_allocated - old_size + sz;
    return OUTPTR(ptr);
}

static void replacement_free(void *ptr)
{
    ptr = INPTR(ptr);
    total_allocated -= *(size_t*)ptr;
    free(ptr);
}

void start_counting_bytes(void)
{
    event_set_mem_functions(replacement_malloc,
                            replacement_realloc,
                            replacement_free);
}
```

### 注意事项

- 替换内存管理函数会影响 Libevent 未来所有分配、调整大小或释放内存的操作。因此，确保在调用任何其他 Libevent 函数之前替换这些函数。否则，Libevent 会使用你的 `free` 函数来释放通过 C 库的 `malloc` 返回的内存。
  
- 你的 `malloc` 和 `realloc` 函数需要返回具有与 C 库相同对齐方式的内存块。

- 你的 `realloc` 函数需要正确处理 `realloc(NULL, sz)`（即将其当作 `malloc(sz)` 处理）。

- 你的 `realloc` 函数需要正确处理 `realloc(ptr, 0)`（即将其当作 `free(ptr)` 处理）。

- 你的 `free` 函数不需要处理 `free(NULL)`。

- 你的 `malloc` 函数不需要处理 `malloc(0)`。

- 如果你在多线程环境中使用 Libevent，则替换的内存管理函数需要是线程安全的。

- Libevent 将使用这些函数分配它返回给你的内存。因此，如果你想释放通过 Libevent 函数分配并返回的内存，并且你已替换了 `malloc` 和 `realloc` 函数，那么你可能需要使用替换的 `free` 函数来释放它。

`event_set_mem_functions()` 函数在 `<event2/event.h>` 中声明。它首次出现在 Libevent 2.0.1-alpha 中。

Libevent 可以通过禁用 `event_set_mem_functions()` 来构建。如果禁用，那么使用 `event_set_mem_functions` 的程序将无法编译或链接。在 Libevent 2.0.2-alpha 及更高版本中，你可以通过检查 `EVENT_SET_MEM_FUNCTIONS_IMPLEMENTED` 宏是否定义来检测 `event_set_mem_functions()` 的存在。

### 锁与线程

正如你所知，如果你正在编写多线程程序，同时从多个线程访问相同的数据并不总是安全的。

Libevent 结构通常可以通过三种方式与多线程配合使用。

某些结构天生是单线程的：在任何时候都不能安全地从多个线程使用它们。

某些结构是可选锁定的：你可以告诉 Libevent 每个对象是否需要从多个线程同时使用。

某些结构始终被锁定：如果 Libevent 运行时支持锁定，那么它们始终可以安全地从多个线程同时使用。

要在 Libevent 中实现锁定，你必须告诉 Libevent 使用哪些锁定函数。你需要在调用任何需要在多个线程之间共享的结构的 Libevent 函数之前执行此操作。

如果你使用的是 pthreads 库或本机 Windows 线程代码，那么你很幸运。已经定义了预定义函数，可以为你设置 Libevent 以使用正确的 pthreads 或 Windows 函数。

接口
```c
#ifdef WIN32
int evthread_use_windows_threads(void);
#define EVTHREAD_USE_WINDOWS_THREADS_IMPLEMENTED
#endif
#ifdef _EVENT_HAVE_PTHREADS
int evthread_use_pthreads(void);
#define EVTHREAD_USE_PTHREADS_IMPLEMENTED
#endif
```
这两个函数在成功时返回 0，在失败时返回 -1。

如果你需要使用不同的线程库，那么你将需要做更多的工作。你需要定义使用你的库来实现以下内容的函数：

- 锁
  - 锁定
  - 解锁
  - 锁分配
  - 锁销毁

- 条件
  - 条件变量创建
  - 条件变量销毁
  - 等待条件变量
  - 向条件变量发送信号/广播

- 线程
  - 线程 ID 检测

然后你可以使用 `evthread_set_lock_callbacks` 和 `evthread_set_id_callback` 接口告诉 Libevent 这些函数。

接口
```c
#define EVTHREAD_WRITE  0x04
#define EVTHREAD_READ   0x08
#define EVTHREAD_TRY    0x10

#define EVTHREAD_LOCKTYPE_RECURSIVE 1
#define EVTHREAD_LOCKTYPE_READWRITE 2

#define EVTHREAD_LOCK_API_VERSION 1

struct evthread_lock_callbacks {
       int lock_api_version;
       unsigned supported_locktypes;
       void *(*alloc)(unsigned locktype);
       void (*free)(void *lock, unsigned locktype);
       int (*lock)(unsigned mode, void *lock);
       int (*unlock)(unsigned mode, void *lock);
};

int evthread_set_lock_callbacks(const struct evthread_lock_callbacks *);

void evthread_set_id_callback(unsigned long (*id_fn)(void));

struct evthread_condition_callbacks {
        int condition_api_version;
        void *(*alloc_condition)(unsigned condtype);
        void (*free_condition)(void *cond);
        int (*signal_condition)(void *cond, int broadcast);
        int (*wait_condition)(void *cond, void *lock,
            const struct timeval *timeout);
};

int evthread_set_condition_callbacks(
        const struct evthread_condition_callbacks *);
```
`evthread_lock_callbacks` 结构描述了你的锁定回调及其能力。对于上述描述的版本，`lock_api_version` 字段必须设置为 `EVTHREAD_LOCK_API_VERSION`。`supported_locktypes` 字段必须设置为一个位掩码，包含你支持的锁类型的 `EVTHREAD_LOCKTYPE_*` 常量。（截至 2.0.4-alpha，`EVTHREAD_LOCK_RECURSIVE` 是强制的，而 `EVTHREAD_LOCK_READWRITE` 是未使用的。）`alloc` 函数必须返回指定类型的新锁。`free` 函数必须释放由指定类型的锁持有的所有资源。`lock` 函数必须尝试以指定模式获取锁，成功时返回 0，失败时返回非零值。`unlock` 函数必须尝试解锁，成功时返回 0，失败时返回非零值。

已识别的锁类型有：

- 0
  - 一个普通的、并不一定是递归的锁。

- `EVTHREAD_LOCKTYPE_RECURSIVE`
  - 一个不会阻止已经持有的线程再次请求的锁。其他线程可以在持有锁的线程解锁后获取锁，次数与最初锁定的次数相同。

- `EVTHREAD_LOCKTYPE_READWRITE`
  - 一个允许多个线程同时读取，但只允许一个线程同时写入的锁。写入者会排除所有读取者。

已识别的锁模式有：

- `EVTHREAD_READ`
  - 仅对 READWRITE 锁：获取或释放读取锁。

- `EVTHREAD_WRITE`
  - 仅对 READWRITE 锁：获取或释放写入锁。

- `EVTHREAD_TRY`
  - 仅对锁定：仅在锁可以立即获取的情况下获取锁。

`id_fn` 参数必须是一个返回无符号长整型的函数，标识调用该函数的线程。对于同一个线程，它必须始终返回相同的数字，并且在两个不同线程同时执行时，绝不应返回相同的数字。

`evthread_condition_callbacks` 结构描述了与条件变量相关的回调。对于上述描述的版本，`condition_api_version` 字段必须设置为 `EVTHREAD_CONDITION_API_VERSION`。`alloc_condition` 函数必须返回指向新条件变量的指针。它接收 0 作为参数。`free_condition` 函数必须释放条件变量持有的存储和资源。`wait_condition` 函数接受三个参数：由 `alloc_condition` 分配的条件，由你提供的 `evthread_lock_callbacks.alloc` 函数分配的锁，以及一个可选的超时。每当调用该函数时，锁将被持有；该函数必须释放锁，并等待条件被发送信号或（可选）超时到期。`wait_condition` 函数应在出错时返回 -1，在条件被发送信号时返回 0，在超时时返回 1。在返回之前，它应确保再次持有锁。最后，`signal_condition` 函数应使一个等待条件的线程唤醒（如果其 `broadcast` 参数为 false），并使当前所有等待条件的线程唤醒（如果其 `broadcast` 参数为 true）。它将仅在持有与条件相关的锁时保持锁定。

有关条件变量的更多信息，请查阅 pthreads 的 `pthread_cond_*` 函数或 Windows 的 `CONDITION_VARIABLE` 函数的文档。

**示例**

有关如何使用这些函数的示例，请参见 Libevent 源代码分发中的 `evthread_pthread.c` 和 `evthread_win32.c`。本节中的函数声明在 `<event2/thread.h>` 中。大多数函数首次出现在 Libevent 2.0.4-alpha 版本中。Libevent 版本从 2.0.1-alpha 到 2.0.3-alpha 使用了旧的接口来设置锁定函数。`event_use_pthreads()` 函数要求您将程序链接到 `event_pthreads` 库。

条件变量函数是在 Libevent 2.0.7-rc 中新增的；它们是为了解决一些难以处理的死锁问题而添加的。

Libevent 可以构建为禁用锁定支持。如果禁用，则构建使用上述线程相关函数的程序将无法运行。

**调试锁的使用**

为了帮助调试锁的使用，Libevent 提供了一个可选的“锁调试”功能，该功能封装其锁定调用，以捕获典型的锁错误，包括：

- 解锁一个我们实际上并不持有的锁
- 重新锁定一个非递归锁

如果发生这些锁错误之一，Libevent 将以断言失败的方式退出。

**接口**
```c
void evthread_enable_lock_debugging(void);
#define evthread_enable_lock_debuging() evthread_enable_lock_debugging()
```
**注意**
此函数必须在创建或使用任何锁之前调用。为了安全起见，请在设置线程函数后立即调用此函数。该函数在 Libevent 2.0.4-alpha 中首次引入，原名拼写错误为 `evthread_enable_lock_debuging()`，在 2.1.2-alpha 中已更正为 `evthread_enable_lock_debugging()`；目前支持这两个名称。

**调试事件的使用**

Libevent 可以检测并报告您在使用事件时的一些常见错误。包括：

- 将未初始化的 `struct event` 视为已初始化。
- 尝试重新初始化一个挂起的 `struct event`。

跟踪哪些事件已初始化需要 Libevent 使用额外的内存和 CPU，因此您应仅在实际调试程序时启用调试模式。

**接口**
```c
void event_enable_debug_mode(void);
```
此函数必须仅在创建任何 `event_base` 之前调用。

在使用调试模式时，如果您的程序使用大量通过 `event_assign()`（而不是 `event_new()`）创建的事件，您可能会耗尽内存。这是因为 Libevent 无法判断使用 `event_assign()` 创建的事件何时不再使用。（当您对其调用 `event_free()` 时，它可以判断 `event_new()` 创建的事件已失效。）如果您希望在调试时避免耗尽内存，可以明确告知 Libevent 这些事件不再被视为已分配：

**接口**
```c
void event_debug_unassign(struct event *ev);
```
当未启用调试时，调用 `event_debug_unassign()` 没有任何效果。

**示例**
```c
#include <event2/event.h>
#include <event2/event_struct.h>
#include <stdlib.h>

void cb(evutil_socket_t fd, short what, void *ptr)
{
    /* 对于堆分配的事件，我们将 'NULL' 作为回调指针，
     * 对于栈分配的事件，我们将事件本身作为回调指针。 */
    struct event *ev = ptr;

    if (ev)
        event_debug_unassign(ev);
}

/* 这是一个简单的主循环，等待 fd1 和 fd2 都准备好读取。 */
void mainloop(evutil_socket_t fd1, evutil_socket_t fd2, int debug_mode)
{
    struct event_base *base;
    struct event event_on_stack, *event_on_heap;

    if (debug_mode)
       event_enable_debug_mode();

    base = event_base_new();

    event_on_heap = event_new(base, fd1, EV_READ, cb, NULL);
    event_assign(&event_on_stack, base, fd2, EV_READ, cb, &event_on_stack);

    event_add(event_on_heap, NULL);
    event_add(&event_on_stack, NULL);

    event_base_dispatch(base);

    event_free(event_on_heap);
    event_base_free(base);
}
```
详细的事件调试是一个只能在编译时通过使用 CFLAGS 环境变量 `-DUSE_DEBUG` 启用的功能。启用此标志后，任何与 Libevent 编译的程序将输出非常详细的日志，详细记录后台的低级活动。这些日志包括但不限于以下内容：

- 事件添加
- 事件删除
- 特定平台的事件通知信息

此功能不能通过 API 调用启用或禁用，因此必须仅在开发者构建中使用。

这些调试函数是在 Libevent 2.0.4-alpha 中添加的。

**检测 Libevent 的版本**

新版本的 Libevent 可以添加新功能并修复错误。有时您可能需要检测 Libevent 的版本，以便可以：

- 检测安装的 Libevent 版本是否足够好以构建您的程序。
- 显示 Libevent 版本以便进行调试。
- 检测 Libevent 的版本，以便可以警告用户有关错误，或为其提供解决方法。

**接口**
```c
#define LIBEVENT_VERSION_NUMBER 0x02000300
#define LIBEVENT_VERSION "2.0.3-alpha"
const char *event_get_version(void);
ev_uint32_t event_get_version_number(void);
```
这些宏提供了 Libevent 库的编译时版本；而这些函数返回运行时版本。请注意，如果您已将程序动态链接到 Libevent，这些版本可能会不同。

您可以以两种格式获取 Libevent 版本：作为适合显示给用户的字符串，或作为适合数值比较的 4 字节整数。整数格式使用高字节表示主版本，第二个字节表示次版本，第三个字节表示补丁版本，低字节表示发布状态（0 表示正式发布，非零表示在给定发布之后的开发系列）。

因此，发布的 Libevent 2.0.1-alpha 的版本号为 [02 00 01 00]，即 0x02000100。介于 2.0.1-alpha 和 2.0.2-alpha 之间的开发版本可能具有版本号 [02 00 01 08]，即 0x02000108。

**示例：编译时检查**
```c
#include <event2/event.h>

#if !defined(LIBEVENT_VERSION_NUMBER) || LIBEVENT_VERSION_NUMBER < 0x02000100
#error "此版本的 Libevent 不受支持；请获取 2.0.1-alpha 或更高版本。"
#endif

int make_sandwich(void)
{
    /* 假设 Libevent 6.0.5 引入了一个制作三明治的函数。 */
#if LIBEVENT_VERSION_NUMBER >= 0x06000500
    evutil_make_me_a_sandwich();
    return 0;
#else
    return -1;
#endif
}
```

**示例：运行时检查**
```c
#include <event2/event.h>
#include <string.h>

int check_for_old_version(void)
{
    const char *v = event_get_version();
    /* 这是一种笨拙的做法，但在 Libevent 2.0 之前，这是唯一有效的方法。 */
    if (!strncmp(v, "0.", 2) ||
        !strncmp(v, "1.1", 3) ||
        !strncmp(v, "1.2", 3) ||
        !strncmp(v, "1.3", 3)) {

        printf("您的 Libevent 版本非常旧。如果遇到错误，"
               "请考虑升级。\n");
        return -1;
    } else {
        printf("当前使用的 Libevent 版本 %s\n", v);
        return 0;
    }
}

int check_version_match(void)
{
    ev_uint32_t v_compile, v_run;
    v_compile = LIBEVENT_VERSION_NUMBER;
    v_run = event_get_version_number();
    if ((v_compile & 0xffff0000) != (v_run & 0xffff0000)) {
        printf("当前运行的 Libevent 版本 (%s) 与我们构建时使用的版本 (%s) 相差很大。\n", 
               event_get_version(), LIBEVENT_VERSION);
        return -1;
    }
    return 0;
}
```

本节中的宏和函数在 `<event2/event.h>` 中定义。`event_get_version()` 函数首次出现在 Libevent 1.0c 中；其他函数首次出现在 Libevent 2.0.1-alpha 中。

**释放全局 Libevent 结构**

即使您已释放所有通过 Libevent 分配的对象，仍然会有一些全局分配的结构剩下。这通常不是问题：一旦进程退出，它们都会被清理掉。但是，存在这些结构可能会让一些调试工具误认为 Libevent 正在泄漏资源。如果您需要确保 Libevent 已释放所有内部库全局数据结构，可以调用：

**接口**
```c
void libevent_global_shutdown(void);
```

此函数不会释放任何由 Libevent 函数返回给您的结构。如果您希望在退出前释放所有内容，您需要自己释放所有事件、事件基础（event_bases）、缓冲事件（bufferevents）等。

调用 `libevent_global_shutdown()` 将使其他 Libevent 函数表现得不可预测；除了作为程序调用的最后一个 Libevent 函数外，不要调用它。一个例外是，`libevent_global_shutdown()` 是幂等的：即使它已经被调用，您也可以再次调用它。

该函数在 `<event2/event.h>` 中声明。它是在 Libevent 2.1.1-alpha 中引入的。
