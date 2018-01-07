### zynq u-boot

#### 连接脚本

编译zynq_zed使用的连接脚本定义在：u-boot-xlnx/arch/arm/mach-zynq/u-boot.lds。最终生成的连接脚本存放到u-boot根目录,名称为u-boot.lds，内如如下（有删减）：

```assembly
OUTPUT_FORMAT("elf32-littlearm", "elf32-littlearm", "elf32-littlearm")
OUTPUT_ARCH(arm)
ENTRY(_start)
SECTIONS
{
 . = 0x00000000;
 . = ALIGN(4);
 .text :
 {
  *(.__image_copy_start)
  *(.vectors)
  arch/arm/cpu/armv7/start.o (.text*)
  *(.text*)
 }
 . = ALIGN(4);
 .rodata : { *(SORT_BY_ALIGNMENT(SORT_BY_NAME(.rodata*))) }
 . = ALIGN(4);
 .data : {
  *(.data*)
 }
 . = ALIGN(4);
 . = .;
 . = ALIGN(4);
 .u_boot_list : {
  KEEP(*(SORT(.u_boot_list*)));
 }
 . = ALIGN(4);
 .image_copy_end :
 {
  *(.__image_copy_end)
 }
 .rel_dyn_start :
 {
  *(.__rel_dyn_start)
 }
 .rel.dyn : {
  *(.rel*)
 }
 .rel_dyn_end :
 {
  *(.__rel_dyn_end)
 }
 .end :
 {
  *(.__end)
 }
 _image_binary_end = .;
 .bss_start __rel_dyn_start (OVERLAY) : {
  KEEP(*(.__bss_start));
  __bss_base = .;
 }
 .bss __bss_base (OVERLAY) : {
  *(.bss*)
   . = ALIGN(4);
   __bss_limit = .;
 }
 .bss_end __bss_limit (OVERLAY) : {
  KEEP(*(.__bss_end));
 }
```

##### 入口地址

从上面可以看出程序的入口符号为_start,定义在arch\arm\lib\vectors.S，内容如下：

```
.globl _start
	.section ".vectors", "ax"

_start:

#ifdef CONFIG_SYS_DV_NOR_BOOT_CFG
	.word	CONFIG_SYS_DV_NOR_BOOT_CFG
#endif
	b	reset
	ldr	pc, _undefined_instruction
	ldr	pc, _software_interrupt
	ldr	pc, _prefetch_abort
	ldr	pc, _data_abort
	ldr	pc, _not_used
	ldr	pc, _irq
	ldr	pc, _fiq
```

该文件的代码定义在段.vectors段中。

虽然可以看到在连接脚本中 . = 0x00000000;但是起始代码是从0x40 00000开始的，这是在连接的时候传递给ld的参数，连接的命令如下：

```c++
arm-linux-gnueabihf-ld.bfd   -pie  --gc-sections -Bstatic -Ttext 0x4000000 -o u-boot -T u-boot.lds arch/arm/cpu/armv7/start.o --start-group  arch/arm/cpu/built-in.o  arch/arm/cpu/armv7/built-in.o   --end-group arch/arm/lib/eabi_compat.o  arch/arm/lib/lib.a -Map u-boot.map
```

其中-pie表示位置无关。-Ttext 0x4000000指定了text段的入口地址。

#### .text 输出段

```
 .text :
 {
  *(.__image_copy_start)
  *(.vectors)
  arch/arm/cpu/armv7/start.o (.text*)
  *(.text*)
 }
```

从上面的连接脚本可以看到.text输出段，主要包含下面的输入段：

1. ``*(.__image_copy_start)``所有的名为``.__image_copy_start``段
2. 所有的``.vectors``段
3. ``arch/arm/cpu/armv7/start.o``的``.text``段
4. 所有的``.text``段

上面的链接是有顺序的，顺序就按照各个输入段出现的顺序。

##### ``.__image_copy_start``和``.__image_copy_end``

``__image_copy_start``和``__image_copy_end``是配合在一块使用的，定义文件	arch\arm\lib\sections.c，如下：

```c
char __image_copy_start[0] __attribute__((section(".__image_copy_start")));
char __image_copy_end[0] __attribute__((section(".__image_copy_end")));
```

看下__image_copy_end：

```assembly
 .image_copy_end :
 {
  *(.__image_copy_end)
 }
```

其实，``.image_copy_end``输出段的大小为0，因为只有``char __image_copy_end[0]``被定义在该段，而且``char __image_copy_end[0]``的大小为0。同样在.text输出段中``  *(.__image_copy_start)``的大小也为0。

至于为什么需要这样来，可能和使用的参数-pie有关系，需要对arm的位置无关有详细了解。

这个文章对pie有一点说明：https://www.ibm.com/developerworks/cn/linux/l-cn-sdlstatic/

在sections.c文件中，作者做了一个简单的说明：

```c++
/**
 * These two symbols are declared in a C file so that the linker
 * uses R_ARM_RELATIVE relocation, rather than the R_ARM_ABS32 one
 * it would use if the symbols were defined in the linker file.
 * Using only R_ARM_RELATIVE relocation ensures that references to
 * the symbols are correct after as well as before relocation.
 *
 * We need a 0-byte-size type for these symbols, and the compiler
 * does not allow defining objects of C type 'void'. Using an empty
 * struct is allowed by the compiler, but causes gcc versions 4.4 and
 * below to complain about aliasing. Therefore we use the next best
 * thing: zero-sized arrays, which are both 0-byte-size and exempt from
 * aliasing warnings.
 */
```

定义在C文件中，linker使用R_ARM_RELATIVE重定位，而不是使用R_ARM_ABS32，如果symbols定义在linker file（连接脚本）那么就会使用R_ARM_ABS32重定位。

我么需要一个0字节大小的类型来定义这个symbols，编译器不允许定义C语言类型为void的变量。使用一个空的结构体编译是允许的，但是gcc 4.4和一下的编译器会把欧安，因此，使用了更好的：零大小的数组。

######  ``__image_copy_start`` 和 `` __image_copy_end``赋值

这两个变量在编译后的值是多少，u-boot.map中查到

```
 *(.__image_copy_start)
 .__image_copy_start
                0x0000000004000000        0x0 arch/arm/lib/built-in.o
                0x0000000004000000                __image_copy_start

```

.__image_copy_start的值为0x0000000004000000

```
.image_copy_end
                0x0000000004069894        0x0   
 *(.__image_copy_end)
 .__image_copy_end
                0x0000000004069894        0x0 arch/arm/lib/built-in.o
```

__image_copy_end值为0x0000000004069894（会动态变化）

从定义

```c
char __image_copy_start[0] __attribute__((section(".__image_copy_start")));
char __image_copy_end[0] __attribute__((section(".__image_copy_end")));
```

``char __image_copy_start[0]``被放到了段``.__image_copy_start``，那么则表示数组``__image_copy_start``是从段``.__image_copy_start``地址处开始的。段``.__image_copy_start``的地址就是代码的起始地址，所以``__image_copy_start``的值是0x0000000004000000（``__image_copy_start``是数组名，表示一个常量指针）。而且在gcc中零元素的数组恰好大小为0，既不占用空间。

如果将``char __image_copy_start[0] __attribute__((section(".__image_copy_start")));``换为``char __image_copy_start[1] __attribute__((section(".__image_copy_start")));``,``__image_copy_start``的值还是0x0000000004000000，只不过需要占用一个字节内存而已。这样程序的入口就变成__image_copy_start变量的地址了，如下图，反汇编如下：

```assembly
04000000 <__image_copy_start>:
04000020 <_start>:
 4000020:       ea0000be        b       4000320 <reset>
 4000024:       e59ff014        ldr     pc, [pc, #20]   ; 4000040 <_undefined_instruction>
 4000028:       e59ff014        ldr     pc, [pc, #20]   ; 4000044 <_software_interrupt>
 400002c:       e59ff014        ldr     pc, [pc, #20]   ; 4000048 <_prefetch_abort>
 4000030:       e59ff014        ldr     pc, [pc, #20]   ; 400004c <_data_abort>
 4000034:       e59ff014        ldr     pc, [pc, #20]   ; 4000050 <_not_used>
 4000038:       e59ff014        ldr     pc, [pc, #20]   ; 4000054 <_irq>
 400003c:       e59ff014        ldr     pc, [pc, #20]   ; 4000058 <_fiq>
```

所以需要采用那么怪异的定义方式。

#### reset

reset symbal定义在arch\arm\cpu\armv7\start.S,如下：

```assembly
	.globl	reset
	.globl	save_boot_params_ret
#ifdef CONFIG_ARMV7_LPAE
	.global	switch_to_hypervisor_ret
#endif

reset:
	/* Allow the board to save important registers */
	b	save_boot_params
save_boot_params_ret:
```

#### save_boot_params

通过r0寄存器判断是否从spl加载的u-boot，并把结果存放在from_spl变量中，然后返回到save_boot_params_ret。

##### _main

_main定义在arch\arm\lib\crt0.S

主要是准备C语言执行环境，调用board_init_f函数

###### board_init_f

board_init_f定义在arch\arm\mach-zynq\spl.c

```c
void board_init_f(ulong dummy)
{
  	/**board\xilinx\zynq\zynq-zed\ps7_init_gpl.c
  	 *初始化MIO,PLL,Clock, DDR和外设
  	 **/
	ps7_init();
	arch_cpu_init();
}
```

在arch_cpu_init()函数中CONFIG_SYS_SDRAM_BASE宏没有定义，默认值就是0