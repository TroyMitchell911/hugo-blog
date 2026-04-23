+++
date = '2024-07-03T18:01:54+08:00'
draft = false
title = 'Linux earlycon: How the Kernel Gets Console Output Before Drivers Are Ready'
tags = ["linux"]
summary = 'A deep dive into the Linux earlycon mechanism — how the kernel provides console output during early boot before the real serial driver is initialized, covering both command-line and device-tree paths.'
+++

## What Is earlycon?

During early kernel boot, the standard console subsystem (serial drivers, framebuffer consoles, etc.) hasn't been initialized yet. If the kernel panics or you need debug output during this phase, you get nothing — unless you have **earlycon**.

earlycon is a minimal console mechanism that provides output as soon as the kernel enters C code (`start_kernel`). It works by directly writing to UART registers (via MMIO) without relying on the full serial driver stack. Once the real console driver takes over, earlycon is automatically unregistered.

## Enabling earlycon

Two kernel config options are required:

```
CONFIG_SERIAL_EARLYCON=y
CONFIG_OF_EARLY_FLATTREE=y
```

Then pass the earlycon parameter on the kernel command line. The format is:

```
earlycon=<name>,<io_type>,<addr>,<options>
```

For example:

```
earlycon=uart8250,mmio32,0xfe660000,115200n8
earlycon=pxa_serial,0xd4017000
earlycon=sbi                          # RISC-V SBI console
earlycon                              # auto-detect from device tree stdout-path
```

When `earlycon` is passed without arguments, the kernel uses the device tree `stdout-path` to find the right UART and driver (covered in the DT path section below).

---

## Path 1: Command-Line Based Initialization

### Call Chain

```
start_kernel()
  → setup_arch()
    → parse_early_param()
      → parse_early_options()
        → do_early_param()        // iterates all __setup("earlycon", ...) entries
          → param_setup_earlycon()
            → setup_earlycon()
              → register_earlycon()
```

### How early_param Works

The kernel's `parse_early_param()` scans the boot command line and calls handler functions registered via the `early_param()` macro. For earlycon, this is:

```c
// drivers/tty/serial/earlycon.c
early_param("earlycon", param_setup_earlycon);
```

`parse_args()` tokenizes the command line into key-value pairs and calls `do_early_param()` for each one:

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

This iterates the `__setup` section — a linker-defined table of early parameter handlers. When it finds the `"earlycon"` entry, it calls `param_setup_earlycon(val)` where `val` is the string after `earlycon=`.

### setup_earlycon: Matching the Driver

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

        /* prefer entries with empty compatible (command-line drivers) */
        if (empty_compatible && *match->compatible)
            continue;

        if (buf[len]) {
            if (buf[len] != ',')
                continue;
            buf += len + 1;    // skip "name," → points to address/options
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

The function iterates `__earlycon_table` — a linker section populated by the `EARLYCON_DECLARE` and `OF_EARLYCON_DECLARE` macros. It first tries to match entries with an empty `compatible` field (command-line drivers), then falls back to DT-compatible entries.

After stripping the driver name prefix, the remaining string (e.g., `"0xd4017000"`) is passed to `register_earlycon()`.

### The __earlycon_table Section

Drivers register themselves into `__earlycon_table` using two macros:

```c
// include/linux/serial_core.h

// For device-tree based matching:
#define OF_EARLYCON_DECLARE(_name, compat, fn)                \
    static const struct earlycon_id __UNIQUE_ID(__earlycon_##_name) \
        EARLYCON_USED_OR_UNUSED  __section("__earlycon_table")  \
        __aligned(__alignof__(struct earlycon_id))              \
        = { .name = __stringify(_name),                        \
            .compatible = compat,                              \
            .setup = fn }

// For command-line based matching (no device tree):
#define EARLYCON_DECLARE(_name, fn) OF_EARLYCON_DECLARE(_name, "", fn)
```

The `earlycon_id` struct:

```c
struct earlycon_id {
    char    name[15];
    char    name_term;       /* ensures null termination */
    char    compatible[128];
    int     (*setup)(struct earlycon_device *, const char *options);
};
```

The linker script places these in a contiguous table:

```
.init.data : {
    __earlycon_table = .;
    KEEP(*(__earlycon_table))
    __earlycon_table_end = .;
}
```

### Example: SpacemiT K1 (pxa_serial)

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

With `earlycon=pxa_serial,0xd4017000` on the command line:
1. `setup_earlycon` matches `"pxa_serial"` in `__earlycon_table`
2. `buf` becomes `"0xd4017000"` after stripping the name
3. `register_earlycon` parses the address and calls `pxa_early_console_setup`

### register_earlycon: Bringing the Console Online

```c
static int __init register_earlycon(char *buf, const struct earlycon_id *match)
{
    int err;
    struct uart_port *port = &early_console_dev.port;

    /* Parse address, I/O type, baud rate from buf */
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

Key steps:
1. **parse_options** — parses the I/O type (`mmio`, `mmio32`, `io`) and physical address from the parameter string, sets `port->mapbase` and `port->iotype`
2. **earlycon_map** — maps the physical UART register address into virtual memory via `ioremap` so the kernel can access it
3. **match->setup** — calls the driver-specific setup function (e.g., `pxa_early_console_setup`), which sets the `write` callback
4. **register_console** — registers the earlycon as an active console; from this point, `printk` output goes to the UART

### parse_options: Address and Baud Rate Parsing

```c
static int __init parse_options(struct earlycon_device *device, char *options)
{
    struct uart_port *port = &device->port;
    int length;
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

    if (options) {
        device->baud = simple_strtoul(options, NULL, 0);
        /* optional: parse uart clock after baud */
    }

    return 0;
}
```

The I/O type determines how the UART registers are accessed:
- `UPIO_MEM` — byte-width MMIO
- `UPIO_MEM32` — 32-bit MMIO (most common on ARM/RISC-V SoCs)
- `UPIO_PORT` — x86-style I/O port access

---

## Path 2: Device Tree Based Initialization

When `earlycon` is passed without arguments on the command line (or not passed at all but `CONFIG_SERIAL_EARLYCON` is enabled), the kernel can auto-detect the earlycon UART from the device tree's `stdout-path` property.

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

    /* Parse "path:options" format */
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

This function:
1. Finds the `/chosen` node in the flattened device tree
2. Reads `stdout-path` (e.g., `"/serial@d4017000:115200"`)
3. Splits the path and options at the `:` delimiter
4. Resolves the path to a DT node and checks its `compatible` property against `__earlycon_table` entries registered via `OF_EARLYCON_DECLARE`
5. Calls `of_setup_earlycon` which extracts the MMIO address from the DT node's `reg` property and calls the driver's setup function

This is the preferred approach for device-tree platforms — no hardcoded addresses on the command line.

---

## Lifecycle: earlycon → Real Console

earlycon is registered with the `CON_BOOT` flag, which tells the console subsystem it's a temporary boot console:

```c
static struct console early_con = {
    .name  = "uart",
    .flags = CON_PRINTBUFFER | CON_BOOT | CON_ANYTIME,
    .index = -1,
};
```

When the real serial driver initializes later in boot (e.g., `8250_port.c`, `pxa.c`) and calls `register_console()`, the kernel automatically unregisters all `CON_BOOT` consoles. You'll see this in dmesg:

```
[    0.000000] earlycon: uart0 at MMIO32 0xfe660000 (options '115200n8')
[    0.000000] printk: legacy bootconsole [uart0] enabled
...
[    1.234567] printk: legacy bootconsole [uart0] disabled
```

The handoff is seamless — no output is lost because `CON_PRINTBUFFER` ensures any messages buffered during the transition are replayed on the new console.
