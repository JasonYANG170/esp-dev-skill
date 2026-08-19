# Zephyr RTOS 集成：HP 主核 + LP 辅核，sysbuild + mbox 通信

> **适用摘要**: 按 `samples/zephyr/` 的范式，用 `west build ... --sysbuild` 在 ESP32-C6 上同时构建 HP（主）核 Zephyr 应用与 LP（辅）核 BIST 自检固件，通过 mbox IPC 在两核之间收发 ping/pong，并在 LP 核上用 `ulp_lp_core_intr_disable()/enable()` 包裹 `bist_cpu_regs_test()` / `bist_cpu_csr_regs_test()` / `bist_ram_test_march_x()`。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-bist/resources/`, source/examples in `repos/esp-bist/`, and this recipe path `repos/esp-bist/recipes/zephyr_integration.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "Zephyr + BIST"
- "ESP32-C6 双核 / HP + LP"
- "west sysbuild"
- "LP 核跑 BIST"
- "mbox IPC / ping pong"
- "ulp_lp_core 自检"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考工程 | `samples/zephyr/`（HP 主程序 `src/main.c`、LP 远端 `remote/src/main.c`） |
| 支持板子 | `esp32c6_devkitc`（仅此一块；`Kconfig.sysbuild` 默认 REMOTE_BOARD 绑定 `esp32c6_devkitc/esp32c6/lpcore`） |
| 工具链 | Zephyr 环境（`west`、`zephyr` workspace；参考 Zephyr Getting Started），ESP-BIST 仓库 |
| 头文件 | `bist_esp.h`、Zephyr 的 `<zephyr/kernel.h>`、`<zephyr/drivers/mbox.h>`、`<ulp_lp_core_interrupts.h>` |
| 配置 | HP `prj.conf`：`CONFIG_PRINTK=y`、`CONFIG_MBOX=y`；LP `remote/prj.conf`：`CONFIG_ESP_BIST=y` + 各 `CONFIG_ESP_BIST_*_TEST=y` |

## 分步说明

### 1. 拓扑（HP host + LP remote��sysbuild 一次构建两核）

```
west build -p -b esp32c6_devkitc/esp32c6/hpcore ... --sysbuild
        │
        ├── HP image  ← samples/zephyr/src/main.c   (ping/pong host, mbox TX+RX)
        └── LP image  ← samples/zephyr/remote/src/main.c
                          ├── runPOST()   ← bist_cpu_regs_test / csr / ram_test_march_x
                          ├── runtime_tests() ← bist_cpu_regs_test / csr
                          └── mbox TX+RX (ping/pong)
```

sysbuild 依据 `samples/zephyr/Kconfig.sysbuild` 的 `REMOTE_BOARD` 自动把 LP 镜像选为 `esp32c6_devkitc/esp32c6/lpcore`，无需手动指定。

### 2. 把 ESP-BIST 加入 Zephyr workspace（两种方式）

**方式 A — 本地仓库（无需改 manifest）：**

```sh
west build -p -b esp32c6_devkitc/esp32c6/hpcore \
    <path/to/esp-bist/samples/zephyr> \
    -D ZEPHYR_EXTRA_MODULES=<path/to/esp-bist/> \
    --sysbuild
```

**方式 B — 写进 `west.yml`（团队共享）：**

```yaml
# 在 Zephyr workspace 的 west.yml 中
remotes:
  - name: espressif
    url-base: https://github.com/espressif

projects:
  - name: esp-bist
    remote: espressif
    revision: main
    path: modules/lib/esp-bist
```

```sh
west update
west build -p -b esp32c6_devkitc/esp32c6/hpcore \
    modules/lib/esp-bist/samples/zephyr --sysbuild
```

> 必须用 `--sysbuild`：HP 与 LP 是两个独立镜像，sysbuild 一次性构建并合并烧录镜像。漏掉 `--sysbuild` 只会构建 HP 核。

### 3. 烧录并监控

```sh
west flash && west espressif monitor
```

LP 核日志走独立的 LP UART（TX: GPIO5、RX: GPIO4、GND），需另接 USB-UART 适配器查看远端输出（`printf` 来自 LP 核）。

### 4. HP 主核代码（ping/pong host，`samples/zephyr/src/main.c`）

```c
#include <zephyr/kernel.h>
#include <zephyr/drivers/mbox.h>
#include <zephyr/sys/printk.h>

#ifdef CONFIG_RX_ENABLED
static void callback(const struct device *dev, mbox_channel_id_t channel_id,
                     void *user_data, struct mbox_msg *data)
{
    printk("Pong (on channel %d)\n", channel_id);   // LP 核回的 pong
}
#endif

int main(void)
{
    int ret;
    printk("Hello from HOST - %s\n", CONFIG_BOARD_TARGET);

#ifdef CONFIG_RX_ENABLED
    const struct mbox_dt_spec rx_channel =
        MBOX_DT_SPEC_GET(DT_PATH(mbox_consumer), rx);
    printk("Maximum RX channels: %d\n", mbox_max_channels_get_dt(&rx_channel));

    ret = mbox_register_callback_dt(&rx_channel, callback, NULL);
    if (ret < 0) { printk("Could not register callback (%d)\n", ret); return 0; }

    ret = mbox_set_enabled_dt(&rx_channel, true);
    if (ret < 0) { printk("Could not enable RX (%d)\n", ret); return 0; }
#endif

#ifdef CONFIG_TX_ENABLED
    const struct mbox_dt_spec tx_channel =
        MBOX_DT_SPEC_GET(DT_PATH(mbox_consumer), tx);
    printk("Maximum bytes of data in the TX message: %d\n",
           mbox_mtu_get_dt(&tx_channel));

    while (1) {
#if defined(CONFIG_MULTITHREADING)
        k_sleep(K_MSEC(2000));
#else
        k_busy_wait(2000000);
#endif
        printk("Ping (on channel %d)\n", tx_channel.channel_id);
        ret = mbox_send_dt(&tx_channel, NULL);     // 给 LP 核发 ping
        if (ret < 0) { printk("Could not send (%d)\n", ret); return 0; }
    }
#endif
    return 0;
}
```

### 5. LP 远端代码（BIST 自检 + mbox，`samples/zephyr/remote/src/main.c`）

关键点：每个 BIST 测试块前后用 `ulp_lp_core_intr_disable()` / `ulp_lp_core_intr_enable()` 包裹，避免 LP 核中断打断寄存器/CSR 自检导致误报。

```c
#include <zephyr/kernel.h>
#include <zephyr/drivers/mbox.h>
#include <bist_esp.h>
#include <ulp_lp_core_interrupts.h>

static void fail_safe_exit(void)
{
    printf("Fail safe exit\n");
    while (1) { ; }                                 // LP 核进入已知安全状态
}

// 一次性上电自检（POST）：CPU 寄存器 + CSR + RAM March X
static void runPOST(void)
{
    bist_esp_err_t test_err = BIST_ESP_OK;

    ulp_lp_core_intr_disable();                     // 关 LP 中断，保护自检窗口

    test_err = bist_cpu_regs_test();
    if (test_err == BIST_ESP_CPU_TEST_ERR) {
        printf("CPU register test failed\n"); fail_safe_exit();
    }

    test_err = bist_cpu_csr_regs_test();
    if (test_err == BIST_ESP_CPU_CSR_TEST_ERR) {
        printf("CPU CSR register test failed\n"); fail_safe_exit();
    }

    test_err = bist_ram_test_march_x();
    if (test_err == BIST_ESP_RAM_TEST_ERR) {
        printf("RAM test failed\n"); fail_safe_exit();
    }

    ulp_lp_core_intr_enable();                      // 自检完成，恢复中断
    printf("All POST tests passed!\n");
}

// 运行时周期自检（每轮 mbox ping 之后）：CPU 寄存器 + CSR
static void runtime_tests(void)
{
    bist_esp_err_t test_err = BIST_ESP_OK;
    ulp_lp_core_intr_disable();

    test_err = bist_cpu_regs_test();
    if (test_err == BIST_ESP_CPU_TEST_ERR) {
        printf("CPU register test failed\n"); fail_safe_exit();
    }
    test_err = bist_cpu_csr_regs_test();
    if (test_err == BIST_ESP_CPU_CSR_TEST_ERR) {
        printf("CPU CSR register test failed\n"); fail_safe_exit();
    }

    ulp_lp_core_intr_enable();
    printf("All RUNTIME tests passed!\n");
}
```

LP `main()` 流程：`runPOST()` → 注册 mbox RX 回调（pong）→ 循环每 3 s `mbox_send_dt()` 发 ping + `runtime_tests()`：

```c
int main(void)
{
    printf("Hello from REMOTE - %s\n", CONFIG_BOARD_TARGET);
    runPOST();

    const struct mbox_dt_spec rx_channel =
        MBOX_DT_SPEC_GET(DT_PATH(mbox_consumer), rx);
    mbox_register_callback_dt(&rx_channel, callback, NULL);
    mbox_set_enabled_dt(&rx_channel, true);

    const struct mbox_dt_spec tx_channel =
        MBOX_DT_SPEC_GET(DT_PATH(mbox_consumer), tx);

    while (1) {
        k_busy_wait(3000000);                       // 3 s
        printf("Ping (on channel %d)\n", tx_channel.channel_id);
        mbox_send_dt(&tx_channel, NULL);
        runtime_tests();
    }
    return 0;
}
```

### 6. LP 核中断门控（关键约束）

| 操作 | 必须配对 | 原因 |
|---|---|---|
| `ulp_lp_core_intr_disable()` | 后续必须有 `ulp_lp_core_intr_enable()` | BIST 写 `0xAAAAAAAA`/`0x55555555` 到寄存器/CSR 并回读；若期间 LP 中断触发，中断处理可能改写被测寄存器，导致误报 `BIST_ESP_CPU_TEST_ERR` |
| POST 块（regs + csr + ram） | 整体包在中断门控内 | RAM March X 备份/回写期间更不能被打断 |
| runtime 块（regs + csr） | 同样包裹 | 运行时周期自检同理 |

> 该模式来自 `samples/zephyr/remote/src/main.c` 的 `runPOST()` 与 `runtime_tests()`，两处都用 `ulp_lp_core_intr_disable()/enable()` 成对包裹。

### 7. 板级 overlay（`samples/zephyr/remote/boards/esp32c6_devkitc_lpcore.overlay`）

overlay 定义 mbox consumer、共享内存布局，并开启 LP UART：

```dts
/ {
    chosen {
        zephyr,ipc_shm = &ipc_shm;      // IPC 共享内存
        zephyr,ipc = &mbox0;             // mbox 设备
    };

    mbox-consumer {
        compatible = "vnd,mbox-consumer";
        mboxes = <&mbox0 1>, <&mbox0 0>; // tx 通道 1，rx 通道 0
        mbox-names = "tx", "rx";
    };
};

&ulp_ram  { reg = <0x0 0x3e00>; };
&ulp_shm  { reg = <0x3df0 0x10>; };
&ipc_shm  { reg = <0x3ef8 0x10>; };
&lp_uart  { status = "okay"; };          // LP 核 printf 走 LP_UART（GPIO4/5）
```

### 8. 配置文件

**HP `samples/zephyr/prj.conf`：**
```
CONFIG_PRINTK=y
CONFIG_MBOX=y
```

**LP `samples/zephyr/remote/prj.conf`（BIST 在 LP 核编译进来）：**
```
CONFIG_STDOUT_CONSOLE=n
CONFIG_MBOX=y
CONFIG_OUTPUT_DISASSEMBLY=y
CONFIG_ESP_BIST=y
CONFIG_ESP_BIST_CPU_REG_TEST=y
CONFIG_ESP_BIST_CPU_CSR_REG_TEST=y
CONFIG_ESP_BIST_MEMORY_RAM_TEST=y
```

### 9. Kconfig.sysbuild（REMOTE_BOARD 自动绑定）

`samples/zephyr/Kconfig.sysbuild`：
```
config REMOTE_BOARD
    string "Remote board"
    default "esp32c6_devkitc/esp32c6/lpcore" if "$(BOARD)/${BOARD_QUALIFIERS}" = "esp32c6_devkitc/esp32c6/hpcore"
```

HP 目标限定符命中后，REMOTE_BOARD 自动设为 LP 板；无需手动 `-DBOARD=...` 指定远端。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 只构建出 HP 镜像 | 漏了 `--sysbuild` | 命令末尾必须加 `--sysbuild` |
| `west build` 找不到 ESP-BIST 模块 | 未传 `ZEPHYR_EXTRA_MODULES` 或未 `west update` | 方式 A 加 `-D ZEPHYR_EXTRA_MODULES=<esp-bist 路径>`；方式 B 先 `west update` |
| LP 核 BIST 误报 CPU/CSR 失败 | LP 中断在自检期间触发改写了被测寄存器 | 用 `ulp_lp_core_intr_disable()/enable()` 包裹整个测试块（见 step 6） |
| LP 核无日志输出 | LP UART 未接 | 接 USB-UART 到 GPIO5 (TX)/GPIO4 (RX)/GND；overlay 已 `&lp_uart { status = "okay"; }` |
| `CONFIG_ESP_BIST` 在 LP 工程里报未定义 | 远端 `prj.conf` 未开 | LP `remote/prj.conf` 必须显式 `CONFIG_ESP_BIST=y` 及各 `CONFIG_ESP_BIST_*_TEST=y` |
| 想在 HP 核也跑 BIST | standalone 范式才适合 HP | HP 核用 `recipes/standalone_integration.md`（IDF 单核范式）；本 sample 把 BIST 放在 LP 核 |
| `REMOTE_BOARD` 选错 | 自定义板未匹配 `hpcore` 限定符 | 在自己的 sysbuild Kconfig 里按板子限定符覆盖默认值 |

## 参考

- `samples/zephyr/README.md`（构建/烧录/监控、支持板子、LP UART 接线）
- `samples/zephyr/src/main.c`（HP ping/pong host）
- `samples/zephyr/remote/src/main.c`（LP POST + runtime + mbox）
- `samples/zephyr/Kconfig.sysbuild`（REMOTE_BOARD 默认值）
- `samples/zephyr/prj.conf`、`samples/zephyr/remote/prj.conf`
- `samples/zephyr/remote/boards/esp32c6_devkitc_lpcore.overlay`（mbox/SHM/LP UART）
- `docs/en/get_started.rst`（支持 SoC、Samples 说明）
