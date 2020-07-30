---
layout: post
title: qemu调试linux内核启动过程
date: 2019-12-09 16:27
category: 内核
author: 
tags: [内核,qemu]
summary: 
---

## 实验环境

* OS: ubuntu 18.04 x64
* CPU: ryzen 1700 @3.5 GHz

## 编译linux内核

```bash
git clone https://github.com/torvalds/linux.git
cd linux
git checkout v5.4 # 切换分支
cp /boot/config-4.15.0-72-generic .config
make oldconfig # 根据提示安装`flex bison libelf-dev`等依赖包
make localmodconfig # 仅安装已有的module
make menuconfig
make -j16
```
编译成功后会得到镜像文件`arch/x86/boot/bzImage`

## 运行镜像

```bash
qemu-system-x86_64 arch/x86/boot/bzImage
```

显示如下

```text
Booting from Hard Disk...
Use a boot loader.

Remove disk and press any key to reboot...

```

相关源码

`arch/x86/boot/header.S`

```asm
	.code16
	.section ".bstext", "ax"

	.global bootsect_start
bootsect_start:
#ifdef CONFIG_EFI_STUB
	# "MZ", MS-DOS header
	.byte 0x4d
	.byte 0x5a
#endif

	# Normalize the start address
	ljmp	$BOOTSEG, $start2

start2:
	movw	%cs, %ax
	movw	%ax, %ds
	movw	%ax, %es
	movw	%ax, %ss
	xorw	%sp, %sp
	sti
	cld

	movw	$bugger_off_msg, %si

msg_loop:
	lodsb
	andb	%al, %al
	jz	bs_die
	movb	$0xe, %ah
	movw	$7, %bx
	int	$0x10
	jmp	msg_loop

bs_die:
	# Allow the user to press a key, then reboot
	xorw	%ax, %ax
	int	$0x16
	int	$0x19

	# int 0x19 should never return.  In case it does anyway,
	# invoke the BIOS reset code...
	ljmp	$0xf000,$0xfff0

#ifdef CONFIG_EFI_STUB
	.org	0x3c
	#
	# Offset to the PE header.
	#
	.long	pe_header
#endif /* CONFIG_EFI_STUB */

	.section ".bsdata", "a"
bugger_off_msg:
	.ascii	"Use a boot loader.\r\n"
	.ascii	"\n"
	.ascii	"Remove disk and press any key to reboot...\r\n"
	.byte	0
```

linux内核启动机制一直在变化，早期的如《Linux内核完全剖析-基于0.12内核》中的启动过程是有用到`boot sector`的，新版中则必须通过`bootloader`（`GRUB`、`UBOOT`等）来启动。所以上面的代码的主要作用就是打印提示信息，等待用户任意键，然后重启。

## 调试`boot sector`

启动内核

```bash
qemu-system-x86_64 arch/x86_64/boot/bzImage -s -S
```

另起一个shell

```bash
$ gdb
(gdb) target remote :1234
Remote debugging using :1234
warning: No executable has been specified and target does not support
determining executable automatically.  Try using the "file" command.
0x000000000000fff0 in ?? ()
(gdb) info registers
rax            0x0      0
rbx            0x0      0
rcx            0x0      0
rdx            0x663    1635
rsi            0x0      0
rdi            0x0      0
rbp            0x0      0x0
rsp            0x0      0x0
r8             0x0      0
r9             0x0      0
r10            0x0      0
r11            0x0      0
r12            0x0      0
r13            0x0      0
r14            0x0      0
r15            0x0      0
rip            0xfff0   0xfff0
eflags         0x2      [ ]
cs             0xf000   61440
ss             0x0      0
ds             0x0      0
es             0x0      0
fs             0x0      0
gs             0x0      0
(gdb)
```

CPU通电后，`$cs=0xf000 $ip=0xfff0`，指向BIOS的代码，此时CPU处于16bit实模式，需要切换arch才能反汇编，但gdb不支持实时切换，会报以下错误：

```bash
(gdb) x/5i $cs*16+$pc
   0xffff0:     (bad)
   0xffff1:     pop    %rbx
   0xffff2:     loopne 0xffff4
   0xffff4:     lock xor %dh,(%rsi)
   0xffff7:     (bad)
(gdb) set arch i8086
warning: Selected architecture i8086 is not compatible with reported target architecture i386:x86-64
warning: A handler for the OS ABI "GNU/Linux" is not built into this configuration
of GDB.  Attempting to continue with the default i8086 settings.

The target architecture is assumed to be i8086
Remote 'g' packet reply is too long (expected 308 bytes, got 536 bytes): 0000000000000000000000000000000000000000000000006306000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000f0ff0000000000000200000000f00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000007f0300000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000801f0000
(gdb)
```

最简单的处理方式是换`qemu-system-i386`模拟，kernel启动前期是不需要支持64位指令的，需要调试64位代码的时候，再换做`qemu-system-x86_64`并跳过前期代码就行。

```bash
qemu-system-i386 arch/x86_64/boot/bzImage -s -S
```

```bash
(gdb) target remote :1234
Remote debugging using :1234
warning: No executable has been specified and target does not support
determining executable automatically.  Try using the "file" command.
0x0000fff0 in ?? ()
(gdb) x/5i $cs*16+$pc
   0xffff0:     ljmp   $0x3630,$0xf000e05b
   0xffff7:     das    
   0xffff8:     xor    (%ebx),%dh
   0xffffa:     das    
   0xffffb:     cmp    %edi,(%ecx)
(gdb) set arch i8086
The target architecture is assumed to be i8086
(gdb) x/5i $cs*16+$pc
   0xffff0:     ljmp   $0x3630,$0xf000e05b
   0xffff7:     das    
   0xffff8:     xor    (%ebx),%dh
   0xffffa:     das    
   0xffffb:     cmp    %edi,(%ecx)
(gdb)
```

CPU刚启动时，先运行预加载到内存中的BIOS代码，BIOS会将bzImage前512B（即第一个扇区）加载到0x7c00处，并执行

```bash
(gdb) break *0x7c00
Breakpoint 1 at 0x7c00
(gdb) c
Continuing.

Breakpoint 1, 0x00007c00 in ?? ()
(gdb) x/5i $cs*16+$pc
=> 0x7c00:      dec    %ebp
   0x7c01:      pop    %edx
   0x7c02:      ljmp   $0xc88c,$0x7c00007
   0x7c09:      mov    %eax,%ds
   0x7c0b:      mov    %eax,%es
(gdb) x/b 0x7c00
0x7c00: 0x4d
(gdb) x/b 0x7c01
0x7c01: 0x5a
(gdb) 
```

`4d 5a`就和`arch/x86/boot/header.S`中的代码对应起来了

```nasm
	.code16
	.section ".bstext", "ax"

	.global bootsect_start
bootsect_start:
#ifdef CONFIG_EFI_STUB
	# "MZ", MS-DOS header
	.byte 0x4d
	.byte 0x5a
#endif

	# Normalize the start address
	ljmp	$BOOTSEG, $start2
```

但这里的反汇编结果仍然有问题

```text
   0x7c02:      ljmp   $0xc88c,$0x7c00007
```

用`objdump`来看

```bash
$ objdump --adjust-vma=0x7c00 -m i8086 -b binary -D arch/x86/boot/bzImage| more                                                                                                                                          


arch/x86/boot/bzImage:     file format binary


Disassembly of section .data:

00007c00 <.data>:
    7c00:       4d                      dec    %bp
    7c01:       5a                      pop    %dx
    7c02:       ea 07 00 c0 07          ljmp   $0x7c0,$0x7
    7c07:       8c c8                   mov    %cs,%ax
    7c09:       8e d8                   mov    %ax,%ds
    7c0b:       8e c0                   mov    %ax,%es
    7c0d:       8e d0                   mov    %ax,%ss
    7c0f:       31 e4                   xor    %sp,%sp
    7c11:       fb                      sti
    7c12:       fc                      cld
    7c13:       be 40 00                mov    $0x40,%si
    7c16:       ac                      lods   %ds:(%si),%al
    7c17:       20 c0                   and    %al,%al
    7c19:       74 09                   je     0x7c24
    7c1b:       b4 0e                   mov    $0xe,%ah
    7c1d:       bb 07 00                mov    $0x7,%bx
    7c20:       cd 10                   int    $0x10
    7c22:       eb f2                   jmp    0x7c16
    7c24:       31 c0                   xor    %ax,%ax
    7c26:       cd 16                   int    $0x16
    7c28:       cd 19                   int    $0x19
    7c2a:       ea f0 ff 00 f0          ljmp   $0xf000,$0xfff0
        ...
    7c3b:       00 82 00 00             add    %al,0x0(%bp,%si)
    7c3f:       00 55 73                add    %dl,0x73(%di)
```

这里`    7c02:       ea 07 00 c0 07          ljmp   $0x7c0,$0x7`才是合理的


解决方法(参考链接5)：

```bash
$ wget https://gist.githubusercontent.com/MatanShahar/1441433e19637cf1bb46b1aa38a90815/raw/2687fb5daf60cf6aa8435efc8450d89f1ccf2205/target.xml
$ gdb -ex "target remote :1234" -ex "set tdesc filename target.xml" -ex "set arch i8086" -ex "display/i \$cs*16+\$pc"
Remote debugging using :1234
warning: No executable has been specified and target does not support
determining executable automatically.  Try using the "file" command.
0x0000fff0 in ?? ()
warning: A handler for the OS ABI "GNU/Linux" is not built into this configuration
of GDB.  Attempting to continue with the default i8086 settings.

1: x/i $cs*16+$pc
   0xffff0:     ljmp   $0xf000,$0xe05b
(gdb) break *0x7c00
Breakpoint 1 at 0x7c00
(gdb) c
Continuing.

Breakpoint 1, 0x00007c00 in ?? ()
1: x/i $cs*16+$pc
=> 0x7c00:      dec    %bp
(gdb) x/5i $cs*16+$pc
=> 0x7c00:      dec    %bp
   0x7c01:      pop    %dx
   0x7c02:      ljmp   $0x7c0,$0x7
   0x7c07:      mov    %cs,%ax
   0x7c09:      mov    %ax,%ds
(gdb)
```

## 调试UEFI的启动过程



## 调试UEFI+GRUB的启动过程


## 参考链接

1. 《Linux内核完全剖析-基于0.12内核》 - 赵炯 编著
2. <https://www.kernel.org/doc/Documentation/x86/boot.txt>
3. <https://0xax.gitbooks.io/linux-insides/content/>
4. <https://stackoverflow.com/questions/48620622/how-to-solve-qemu-gdb-debug-error-remote-g-packet-reply-is-too-long?rq=1>
5. <https://stackoverflow.com/questions/32955887/how-to-disassemble-16-bit-x86-boot-sector-code-in-gdb-with-x-i-pc-it-gets-tr/32960272>
6. <https://software.intel.com/en-us/articles/unified-extensible-firmware-interface>
