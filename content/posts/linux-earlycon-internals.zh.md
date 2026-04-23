+++
date = '2024-07-03T18:01:54+08:00'
draft = false
title = 'Linux earlycon：内核如何在驱动就绪前输出日志'
tags = ["linux"]
summary = '深入分析 Linux earlycon 机制 — 内核如何在串口驱动初始化之前提供控制台输出，涵盖命令行参数和设备树两条初始化路径。'
+++

## 什么是 earlycon？

内核启动早期，标准的控制台子系统（串口驱动、framebuffer 控制台等）还没有初始化。如果内核在这个阶段 panic 或者你需要调试输出，什么都看不到 — 除非你有 **earlycon**。

earlycon 是一个最小化的控制台机制，在内核进入 C 代码（`start_kernel`）后就能提供输出。它通过直接写 UART 寄存器（MMIO）来工作，不依赖完整的串口驱动栈。当真正的控制台驱动接管后，earlycon 会被自动注销。

## 启用 earlycon

需要两个内核配置选项：

```
CONFIG_SERIAL_EARLYCON=y
CONFIG_OF_EARLY_FLATTREE=y
```

然后在内核命令行中传入 earlycon 参数，格式为：

```
earlycon=<name>,<io_type>,<addr>,<options>
```

例如：

```
earlycon=uart8250,mmio32,0xfe660000,115200n8
earlycon=pxa_serial,0xd4017000
earlycon=sbi                          # RISC-V SBI 控制台
earlycon                              # 从设备树 stdout-path 自动检测
```

当 `earlycon` 不带参数传入时，内核会使用设备树的 `stdout-path` 来找到正确的 UART 和驱动（见下文设备树路径部分）。

---

## 路径一：基于命令行参数的初始化

### 调用链

```
start_kernel()
  → setup_arch()
    → parse_early_param()
      → parse_early_options()
        → do_early_param()        // 遍历所有 __setup("earlycon", ...) 条目
          → param_setup_earlycon()
            → setup_earlycon()
              → register_earlycon()
```

### early_param 的工作原理

内核的 `parse_early_param()` 扫描启动命令行，调用通过 `early_param()` 宏注册的处理函数。对于 earlycon：

```c
// drivers/tty/serial/earlycon.c
early_param("earlycon", param_setup_earlycon);
```

`parse_args()` 将命令行分词为键值对，对每一个调用 `do_early_param()`：

```c
// init/main.c
static int __init do_early_param(char *param, char *val,
                                 const char *unused, void *arg)
{
    const struct obs_kernel_param *p;

    for (p = __setup_start; p < __setup_end; p++) {
        if ((p->early && parameq(param, p->str)) ||
            (strcmp(param, "console") == 0 &&
             strcmp(p->str, "earlycon") == 0 && val &&
             strncmp(val, "uart", 4) == 0)) {
            if (p->setup_func(val) != 0)
                pr_warn("Malformed early option '%s'\n", param);
        }
    }
    return 0;
}
```

这里遍历 `__setup` 段 — 一个由链接器定义的早期参数处理函数表。当找到 `"earlycon"` 条目时，调用 `param_setup_earlycon(val)`，其中 `val` 是 `earlycon=` 后面的字符串。

### setup_earlycon：匹配驱动

```c
// drivers/tty/serial/earlycon.c
int __init setup_earlycon(char *buf)
{
    const struct earlycon_id *match;
    bool empty_compatible = true;

    if (!buf || !buf[0])
        return -EINVAL;

    if (console_is_registered(&early_con))
        return -EALREADY;

again:
    for (match = __earlycon_table; match < __earlycon_table_end; match++) {
        size_t len = strlen(match->name);

        if (strncmp(buf, match->name, len))
            continue;

        /* 优先匹配 compatible 为空的条目（命令行驱动） */
        if (empty_compatible && *match->compatible)
            continue;

        if (buf[len]) {
            if (buf[len] != ',')
                continue;
            buf += len + 1;    // 跳过 "name," → 指向地址/选项
        } else
            buf = NULL;

        return register_earlycon(buf, match);
    }

    if (empty_compatible) {
        empty_compatible = false;
        goto again;
    }

    return -ENOENT;
}
```

函数遍历 `__earlycon_table` — 一个由 `EARLYCON_DECLARE` 和 `OF_EARLYCON_DECLARE` 宏填充的链接器段。它先尝试匹配 `compatible` 字段为空的条目（命令行驱动），然后回退到设备树兼容的条目。

剥离驱动名前缀后，剩余字符串（如 `"0xd4017000"`）传给 `register_earlycon()`。

### __earlycon_table 段

驱动通过两个宏将自己注册到 `__earlycon_table`：

```c
// include/linux/serial_core.h

// 用于设备树匹配：
#define OF_EARLYCON_DECLARE(_name, compat, fn)                \
    static const struct earlycon_id __UNIQUE_ID(__earlycon_##_name) \
        EARLYCON_USED_OR_UNUSED  __section("__earlycon_table")  \
        __aligned(__alignof__(struct earlycon_id))              \
        = { .name = __stringify(_name),                        \
            .compatible = compat,                              \
            .setup = fn }

// 用于命令行匹配（无设备树）：
#define EARLYCON_DECLARE(_name, fn) OF_EARLYCON_DECLARE(_name, "", fn)
```

`earlycon_id` 结构体：

```c
struct earlycon_id {
    char    name[15];
    char    name_term;       /* 确保 null 终止 */
    char    compatible[128];
    int     (*setup)(struct earlycon_device *, const char *options);
};
```

链接脚本将它们放在连续的表中：

```
.init.data : {
    __earlycon_table = .;
    KEEP(*(__earlycon_table))
    __earlycon_table_end = .;
}
```

### 示例：SpacemiT K1（pxa_serial）

```c
// drivers/tty/serial/pxa_k1x.c
static void pxa_early_write(struct console *con, const char *s, unsigned n)
{
    struct earlycon_device *dev = con->data;
    uart_console_write(&dev->port, s, n, serial_pxa_console_putchar);
}

static int __init pxa_early_console_setup(struct earlycon_device *device,
                                          const char *opt)
{
    if (!device->port.membase)
        return -ENODEV;

    device->con->write = pxa_early_write;
    return 0;
}

EARLYCON_DECLARE(pxa_serial, pxa_early_console_setup);
```

当命令行包含 `earlycon=pxa_serial,0xd4017000` 时：
1. `setup_earlycon` 在 `__earlycon_table` 中匹配到 `"pxa_serial"`
2. `buf` 在剥离名称后变为 `"0xd4017000"`
3. `register_earlycon` 解析地址并调用 `pxa_early_console_setup`

### register_earlycon：启动控制台

```c
static int __init register_earlycon(char *buf, const struct earlycon_id *match)
{
    int err;
    struct uart_port *port = &early_console_dev.port;

    /* 解析地址、I/O 类型、波特率 */
    if (buf && !parse_options(&early_console_dev, buf))
        buf = NULL;

    spin_lock_init(&port->lock);
    if (!port->uartclk)
        port->uartclk = BASE_BAUD * 16;
    if (port->mapbase)
        port->membase = earlycon_map(port->mapbase, 64);

    earlycon_init(&early_console_dev, match->name);
    err = match->setup(&early_console_dev, buf);
    earlycon_print_info(&early_console_dev);
    if (err < 0)
        return err;
    if (!early_console_dev.con->write)
        return -ENODEV;

    register_console(early_console_dev.con);
    return 0;
}
```

关键步骤：
1. **parse_options** — 从参数字符串解析 I/O 类型（`mmio`、`mmio32`、`io`）和物理地址，设置 `port->mapbase` 和 `port->iotype`
2. **earlycon_map** — 通过 `ioremap` 将 UART 寄存器的物理地址映射到虚拟地址空间
3. **match->setup** — 调用驱动特定的 setup 函数（如 `pxa_early_console_setup`），设置 `write` 回调
4. **register_console** — 将 earlycon 注册为活跃控制台；此后 `printk` 输出会发送到 UART

### parse_options：地址和波特率解析

```c
static int __init parse_options(struct earlycon_device *device, char *options)
{
    struct uart_port *port = &device->port;
    resource_size_t addr;

    if (uart_parse_earlycon(options, &port->iotype, &addr, &options))
        return -EINVAL;

    switch (port->iotype) {
    case UPIO_MEM:    port->mapbase = addr; break;
    case UPIO_MEM16:  port->regshift = 1; port->mapbase = addr; break;
    case UPIO_MEM32:
    case UPIO_MEM32BE: port->regshift = 2; port->mapbase = addr; break;
    case UPIO_PORT:   port->iobase = addr; break;
    default: return -EINVAL;
    }

    if (options)
        device->baud = simple_strtoul(options, NULL, 0);

    return 0;
}
```

I/O 类型决定了 UART 寄存器的访问方式：
- `UPIO_MEM` — 字节宽度 MMIO
- `UPIO_MEM32` — 32 位 MMIO（ARM/RISC-V SoC 上最常见）
- `UPIO_PORT` — x86 风格的 I/O 端口访问

---

## 路径二：基于设备树的初始化

当 `earlycon` 不带参数传入（或未传入但启用了 `CONFIG_SERIAL_EARLYCON`），内核可以从设备树的 `stdout-path` 属性自动检测 earlycon UART。

```c
// drivers/of/fdt.c
int __init early_init_dt_scan_chosen_stdout(void)
{
    int offset;
    const char *p, *q, *options = NULL;
    const struct earlycon_id *match;
    const void *fdt = initial_boot_params;

    offset = fdt_path_offset(fdt, "/chosen");
    if (offset < 0)
        offset = fdt_path_offset(fdt, "/chosen@0");
    if (offset < 0)
        return -ENOENT;

    p = fdt_getprop(fdt, offset, "stdout-path", &l);
    if (!p)
        p = fdt_getprop(fdt, offset, "linux,stdout-path", &l);
    if (!p || !l)
        return -ENOENT;

    /* 解析 "path:options" 格式 */
    q = strchrnul(p, ':');
    if (*q != '\0')
        options = q + 1;

    offset = fdt_path_offset_namelen(fdt, p, q - p);

    for (match = __earlycon_table; match < __earlycon_table_end; match++) {
        if (!match->compatible[0])
            continue;

        if (fdt_node_check_compatible(fdt, offset, match->compatible))
            continue;

        ret = of_setup_earlycon(match, offset, options);
        if (!ret || ret == -EALREADY)
            return 0;
    }
    return -ENODEV;
}
```

这个函数：
1. 在扁平设备树中找到 `/chosen` 节点
2. 读取 `stdout-path`（如 `"/serial@d4017000:115200"`）
3. 以 `:` 为分隔符拆分路径和选项
4. 将路径解析为 DT 节点，检查其 `compatible` 属性是否与 `__earlycon_table` 中通过 `OF_EARLYCON_DECLARE` 注册的条目匹配
5. 调用 `of_setup_earlycon`，从 DT 节点的 `reg` 属性提取 MMIO 地址并调用驱动的 setup 函数

这是设备树平台的推荐方式 — 命令行上不需要硬编码地址。

---

## 生命周期：earlycon → 正式控制台

earlycon 注册时带有 `CON_BOOT` 标志，告诉控制台子系统这是一个临时的启动控制台：

```c
static struct console early_con = {
    .name  = "uart",
    .flags = CON_PRINTBUFFER | CON_BOOT | CON_ANYTIME,
    .index = -1,
};
```

当真正的串口驱动在启动后期初始化（如 `8250_port.c`、`pxa.c`）并调用 `register_console()` 时，内核会自动注销所有 `CON_BOOT` 控制台。你会在 dmesg 中看到：

```
[    0.000000] earlycon: uart0 at MMIO32 0xfe660000 (options '115200n8')
[    0.000000] printk: legacy bootconsole [uart0] enabled
...
[    1.234567] printk: legacy bootconsole [uart0] disabled
```

切换是无缝的 — 不会丢失输出，因为 `CON_PRINTBUFFER` 确保过渡期间缓冲的消息会在新控制台上重放。
