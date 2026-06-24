# POSIX 多线程与同步

> **适用摘要**：WASM 应用使用 `pthread_create/join` 创建线程，配合 `pthread_mutex_t` 互斥锁与 `pthread_cond_t` 条件变量做同步。

## 触发意图

- "WASM 多线程"
- "pthread 线程"
- "互斥锁 条件变量"
- "并发任务"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `<stdio.h>`、`<pthread.h>`（由 `components/wamr/libc-wasi/include/pthread.h` 提供） |
| sdkconfig | 多线程需 `CONFIG_COMPILER_WASI_USE_SHARED_MEMORY=y` |
| 参考 | `examples/multi_thread/` |

## 分步说明

### 1. 应用代码（取自 examples/multi_thread）

```c
#include <stdio.h>
#include <pthread.h>

static pthread_mutex_t mutex;
static pthread_cond_t cond;

static void *
thread(void *arg)
{
    int *num = (int *)arg;

    pthread_mutex_lock(&mutex);
    printf("thread start \n");

    for (int i = 0; i < 10; i++) {
        *num = *num + 1;
        printf("num: %d\n", *num);
    }

    pthread_cond_signal(&cond);      /* 通知主线程 */
    pthread_mutex_unlock(&mutex);

    printf("thread exit \n");
    return NULL;
}

int
main(int argc, char *argv[])
{
    pthread_t tid;
    int num = 0, ret = -1;

    if (pthread_mutex_init(&mutex, NULL) != 0) {
        printf("Failed to init mutex.\n");
        return -1;
    }
    if (pthread_cond_init(&cond, NULL) != 0) {
        printf("Failed to init cond.\n");
        goto fail1;
    }

    pthread_mutex_lock(&mutex);
    if (pthread_create(&tid, NULL, thread, &num) != 0) {
        printf("Failed to create thread.\n");
        goto fail2;
    }

    printf("cond wait start\n");
    pthread_cond_wait(&cond, &mutex);     /* 等待子线程通知 */
    pthread_mutex_unlock(&mutex);
    printf("cond wait success.\n");

    if (pthread_join(tid, NULL) != 0) {
        printf("Failed to join thread.\n");
    }
    ret = 0;

fail2:
    pthread_cond_destroy(&cond);
fail1:
    pthread_mutex_destroy(&mutex);

    return ret;
}
```

### 2. 关键点

- ESP-WDF 的 `pthread` 由 WAMR 的 libc 层提供（`components/wamr/libc-wasi/include/pthread.h`），用法与标准 POSIX 一致。
- 多线程需共享线性内存，构建时设 `CONFIG_COMPILER_WASI_USE_SHARED_MEMORY=y`（对应链接 `-Wl,--shared-memory`）。
- 网络任务（如 tcp_client）通常放 `pthread` 内执行，主线程 `pthread_join` 等待。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `pthread_create` 失败 | 未共享内存 / 线性内存不足 | 设 `USE_SHARED_MEMORY=y`，调大 `INITIAL_MEMORY` |
| 死锁 | `cond_wait` 未在持锁状态调用 / 漏 `unlock` | `pthread_cond_wait` 前先 `lock`，等待返回后释放 |
| 数据竞争 | 共享变量未加锁 | 用 `pthread_mutex_lock/unlock` 保护 |
| 链接报 pthread 符号缺失 | libc 模式不对 | 确认使用 libc-wasi（多线程需要） |

## 参考

- `examples/multi_thread/main/multi_thread.c`
- `components/wamr/libc-wasi/include/pthread.h`
- `resources/pitfalls.md` —— 第 11 条 线性内存约束
