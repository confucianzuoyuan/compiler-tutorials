# 从零开始编写一个risc-v 64位操作系统

## 第一步，编写bootloader

### 环境准备

操作系统使用`Ubuntu 20.2`。

安装必要的工具：

```bash
$ sudo apt-get install git build-essential gdb-multiarch qemu-system-misc gcc-riscv64-linux-gnu binutils-riscv64-linux-gnu
$ sudo apt-get remove qemu-system-misc
$ sudo apt-get install qemu-system-misc=1:4.2-3ubuntu6
```

### 编写`entry.S`汇编文件

```assembly
        # qemu -kernel 会把内核加载到内存地址为 0x80000000 的地方
        # 然后使每个 CPU 都跳转到那里。
        # kernel.ld 链接文件会将下面的代码放在 0x80000000 这个地方。
.section .text
.global _entry
_entry:
        # 为 C 语言开辟一个栈空间。
        # stack0 是在 start.c 中声明的。
        # 我们会为每个 CPU 开辟一个 4096 个字节的栈空间。
        # sp = stack0 + (hartid * 4096)
        la sp, stack0     # 将栈的起始地址加载到 sp 栈指针寄存器中。
        li a0, 1024*4     # 此时 a0 寄存器中的值是 4096
        csrr a1, mhartid  # 将当前的 hart id 读取到 a1 寄存器中
        addi a1, a1, 1    # 执行操作：a1 <- a1 + 1
        mul a0, a0, a1    # 执行操作：a0 <- a0 * a1
        add sp, sp, a0    # 执行操作：sp <- sp + a0 ，此时栈指针指向栈的空间的尾部，因为栈地址是从高地址向低地址生长的。
        # 跳转到 start.c 中的 start() 函数
        call start
spin:
        j spin # 悬置在这里
```

**mhartid**这个寄存器用于指示运行当前代码的硬件线程（Hart）。操作系统 SMP （对称多处理） 时会用到这个寄存器来判断运行当前代码的 Hart 。

**Hart** ：英文全称 hardware thread ，表示一个硬件线程，对软件层面来说，就是一个独立的处理器。但是实际上也有可能**它不是完整的独立的一个 CPU 核心**。这里我们需要理解一个概念，那就是超线程技术，超线程技术就是将一个处理器运算单元被多个硬件线程复用，每个硬件线程有自己独立的一套通用寄存器等上下文资源。

所以上面的`addi a1, a1, 1`视为了多核运行os做准备的。因为每个核都有自己独立的栈空间。

### 编写`start.c`文件

```c
#include <stdint.h>

#define UART        0x10000000
#define UART_THR    (uint8_t*)(UART+0x00) // THR:transmitter holding register
#define UART_LSR    (uint8_t*)(UART+0x05) // LSR:line status register
#define UART_LSR_EMPTY_MASK 0x40          // LSR Bit 6: Transmitter empty; both the THR and LSR are empty
#define NCPU          8  // maximum number of CPUs

int lib_putc(char ch) {
  while ((*UART_LSR & UART_LSR_EMPTY_MASK) == 0);
  return *UART_THR = ch;
}

void lib_puts(char *s) {
  while (*s) lib_putc(*s++);
}

// 这里开辟了栈空间，每个CPU都有自己的独有的栈空间。
__attribute__ ((aligned (16))) char stack0[4096 * NCPU];

// entry.S 会在机器模式下跳转到这里，当然栈是 stack0 。
void start() {
  lib_puts("Hello 世界!\n");
  while (1) {}
}
```

### 编写`kernel.ld`链接文件

```
OUTPUT_ARCH( "riscv" )
ENTRY( _entry )

SECTIONS
{
  /*
   * 保证 entry.S / _entry 位于内存地址为 0x80000000 的地方，
   * 这样 qemu's -kernel 可以跳转过去。
   */
  . = 0x80000000;
}
```

### 编写`Makefile`文件

```makefile
K=kernel
U=user

OBJS = \
  $K/entry.o \
  $K/start.o

QEMU = qemu-system-riscv64

CC = riscv64-unknown-elf-gcc
AS = riscv64-unknown-elf-gas
LD = riscv64-unknown-elf-ld
OBJCOPY = riscv64-unknown-elf-objcopy
OBJDUMP = riscv64-unknown-elf-objdump

CFLAGS = -Wall -Werror -O -fno-omit-frame-pointer -ggdb
CFLAGS += -MD
CFLAGS += -mcmodel=medany
CFLAGS += -ffreestanding -fno-common -nostdlib -mno-relax
CFLAGS += -I.
CFLAGS += $(shell $(CC) -fno-stack-protector -E -x c /dev/null >/dev/null 2>&1 && echo -fno-stack-protector)
CFLAGS += -fno-pie -no-pie

LDFLAGS = -z max-page-size=4096

$K/kernel: $(OBJS) $K/kernel.ld
	$(LD) $(LDFLAGS) -T $K/kernel.ld -o $K/kernel $(OBJS) 
	$(OBJDUMP) -S $K/kernel > $K/kernel.asm
	$(OBJDUMP) -t $K/kernel | sed '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $K/kernel.sym

-include kernel/*.d

clean: 
	rm -f *.tex *.dvi *.idx *.aux *.log *.ind *.ilg \
	*/*.o */*.d */*.asm */*.sym

# try to generate a unique GDB port
GDBPORT = $(shell expr `id -u` % 5000 + 25000)

ifndef CPUS
CPUS := 1
endif

QEMUOPTS = -machine virt -bios none -kernel $K/kernel -m 128M -smp $(CPUS) -nographic

qemu: $K/kernel
	$(QEMU) $(QEMUOPTS)
```

### 执行程序

```bash
$ make clean
$ make
$ make qemu
```

使用`ctrl-a x`退出。

此时目录结构如下：

```
os-tutorial
├── kernel
│   ├── entry.S
│   ├── kernel.ld
│   └── start.c
└── Makefile
```
