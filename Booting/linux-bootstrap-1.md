커널 부팅 과정. part 1.
================================================================================

부트로더에서 커널로
--------------------------------------------------------------------------------

만약 당신이 나의 이전 [블로그 포스트](http://0xax.blogspot.com/search/label/asm)를 읽어봤다면, 얼마간 나는 low-level 프로그래밍에 관련된 일을 시작했음을 알 수 있을 것이다. 나는 그와 동시에 리눅스에서 x86_64 어셈블리 프로그래밍에 관련에서 글을 썻고, 리눅스 커널 소스 분석을 시작했다. 나는 low-level 에서 일어나는 일, 컴퓨터에서 프로그램이 어떻게 수행되는지, 프로그램이 메모리에 어떻게 위치하는지, 커널은 프로세스와 메모리를 어떻게 관리하는지, low-level 관점에서 network stack 은 어떻게 동작하는지 그리고 많은 다른 부분에 대해 큰 관심을 갖게 되었다. 그래서, 나는 **x86_64** 를 위한 리눅스 커널에 대해 또 다른 일련의 시리즈의 글을 쓰기로 결정했다.

나는 전문적인 리눅스 커널 해커도 아니고 리눅스 커널과 관련된 일을 하지도 않는다. 이것은 단지 취미일 뿐이다. 나는 low-level 에서 일어나는 것을 좋아한다. 그래서 만약 혼란스러운 어떤 것을 알려주거나 질문들이 있다면, twither [0xAX](https://twitter.com/0xAX) 나 [email](anotherworldofworld@gmail.com)을 보내거나 그냥 [issue](https://github.com/0xAX/linux-insides/issues/new) 를 만들어 달라. 나는 그런 것들에 감사한다. 모든 글을은 [linux-insides](https://github.com/0xAX/linux-insides) 에서 접근가능하고, 영어사용이나 글 내용에 문제가 있다면, 수정 후 pull request 를 보내길 바란다. 한글로 번역된 내용은 [linux-insides](https://github.com/0xAX/linux-insides) 이고, 번역에 문제나 글 내용의 수정 사항이 있다면 [issue](https://github.com/daeseokyoun/linux-insides/issues/new)를 이용 바란다.

*이것은 공식적인 문서가 아니라 단지 배우고 지식을 공유하기 위함이라는 것을 알려드린다.*

**요구 지식**

* C 언어를 이해
* 어셈블리 언어의 이해(AT&T syntax)

여기까지는 단순한 소개였고 이제는 리눅스 커널과 low-level 의 내용을 배워보도록 하자.

모든 코드는 3.18 커널 기준으로 되어 있다. 만약 여기에 변경이 있다면, 나는 그 포스트를 업데이트 할 것이다.

컴퓨터의 전원 인가시(파워 버트을 누르면), 다음에 일어나는 일은 무엇일까?
--------------------------------------------------------------------------------

비록 이 문서는 리눅스 커널에 관련된 시리즈의 글이지만, 우리는 리눅스 커널에서 부터 시작하지는 않을 것이다. 당신의 노트북이나 PC의 파워 버튼을 눌렀을 때, 일어나는 일부터 시작한다. 마더보드는 처음 신호를 [power supply-한글](https://namu.wiki/w/%ED%8C%8C%EC%9B%8C%20%EC%84%9C%ED%94%8C%EB%9D%BC%EC%9D%B4)로 보낸다. 신호를 받고 나면, Power supply 는 컴퓨터에 알맞은 전력으로 전원을 공급한다. 일단 마더보드에서 [power good signal](https://en.wikipedia.org/wiki/Power_good_signal)를 받으면, 마더보드는 CPU를 수행하려고 한다. CPU 는 모든 레지스트의 남은 데이터를 리셋하고 각각을 위해 미리 정의된 값으로 설정한다.

[80386-한글](https://ko.wikipedia.org/wiki/%EC%9D%B8%ED%85%94_80386) 과 이후의 CPU 들은 컴퓨터 리셋 이후에 CPU의 레지스터에 다음과 같은 미리 정의된 값으로 설정한다:

```
IP          0xfff0
CS selector 0xf000
CS base     0xffff0000
```

프로세서는 리얼 모드([real mode-한글](https://ko.wikipedia.org/wiki/%EB%A6%AC%EC%96%BC_%EB%AA%A8%EB%93%9C))에서 시작한다. 이 모드에서 메모리 세그먼트를 이해 해보도록 하자. Real mode 는 모든 x86 호완 프로세서([8086](https://ko.wikipedia.org/wiki/%EC%9D%B8%ED%85%94_8086) 에서 부터 최근 64비트 CPU 까지)에서 지원된다. 8086 프로세서는 20 비트의 주소 버스를 갖고 있고, 이것은 주소 공간 0 ~ 0xFFFFF (1 MB)에서 수행가능 하다는 의미 이다. 하지만 그 CPU는 16비트 레지스터만 갖고 있고, 이것은 최대 주소는 2^16 - 1(0xffff) -64 KB- 이라는 것이다. [메모리 세그먼트](https://ko.wikipedia.org/wiki/%EB%A9%94%EB%AA%A8%EB%A6%AC_%EC%84%B8%EA%B7%B8%EB%A8%BC%ED%8A%B8)는 모든 가용한 주소 공간을 사용할 수 있도록 한다. 모든 메모리는 작은, 65536 바이트(64KB)의 고정 크기 세그먼트로 나누어진다. 16비트 레지스터로 64 KB 이상의 메모리 접근이 불가능 하기 때문에 고안된 방법이다. 하나의 주소는 두 개의 부분으로 구성된다.: 베이스 주소인 세그먼트 셀렉터, 그리고 베이스 주소의 오프셋이다. Real mode 에서, 세그먼트 셀렉터의 연관된 기본 주소는 `세그먼트 셀렉터 * 16` 이다. 그러므로, 메모리의 물리적 주소를 얻기 위해서는, 우리는 세그먼트 셀렉터 부분에 16을 곱하고 오프셋을 더하면 된다:

```
물리 주소 = 세그먼트 셀렉터 * 16 + 오프셋
(PhysicalAddress = Segment Selector * 16 + Offset)
```

예를 들어, 만약 `CD:IP` 가 `0x2000:0x0010`이라면, 그에 대응되는 물리 주소는 아래와 같다:
```python
>>> hex((0x2000 << 4) + 0x0010)
'0x20010'
```

그러나, 만약 우리가 가장 큰 주소를 사용하려면, `0xffff:0xffff`을 사용할 수 있다. 그러면 아래와 같은 주소를 얻을 수 있다:
```python
>>> hex((0xffff << 4) + 0xffff)
'0x10ffef'
```

[A20 라인](https://sites.google.com/site/simplicitydrivendevelopment/un-yeongcheje/x86-un-yeongcheje-guhyeon/interrupt-and-exception)이 켜지기 전에 real mode 에서는 단지 1MB 만 접근이 가능하므로, `0x10ffef`의 주소는 `0x00ffef` 가 될 것이다.

이제 우리는 real mode 와 메모리 주소지정에 대해 알아보았다. 이제 리셋 이후의 레지스터 값에 대한 논의로 돌아가자.:

`CS` 레지스터는 두 개의 부분으로 구성된다: 보이는 부분-Visable part-(세그먼트 셀렉터)와 보이지 않는 부분-Hidden Part-(베이스 주소)로 나뉜다. 베이스 주소는 일반적으로 세그먼트 셀렉터에 16을 곱한 값으로 만들어지는 반면에 하드웨어 리셋 동안 CS 레지스터의 세그먼트 셀렉터에는 0xf000 이 로드되고 베이스 주소는 0xffff0000 이 로드 된다. 프로세서는 이 특별한 베이스 주소를 'CS' 가 바뀔 때까지 사용한다.
[추가 잠조 자료-한글](http://zkakira.tistory.com/entry/%EC%9A%B4%EC%98%81%EC%B2%B4%EC%A0%9C%EC%9D%98-%EB%A9%94%EB%AA%A8%EB%A6%AC-%EC%A3%BC%EC%86%8C-%EC%A7%80%EC%A0%95%EB%B0%A9%EC%8B%9D-Segment)

시작 주소는 EIP 레지스터 값에 베이스 주소를 더한 형태이다.:

```python
>>> 0xffff0000 + 0xfff0
'0xfffffff0'
```

우리는 `0xfffffff0`(4GB - 16 bytes) 를 얻었다. 이 지점은 [리셋 벡터](http://en.wikipedia.org/wiki/Reset_vector) 라 불린다. 리셋 이후에 실행되기 위한 첫 명령어를 찾을 수 있는 메모리 위치 이다. 그것은 BIOS 엔트리 포인트로 진입할 수 있는 [jump](https://ko.wikipedia.org/wiki/%EB%AA%85%EB%A0%B9%EC%96%B4_%EC%A7%91%ED%95%A9) (`jmp`) 명령어를 포함한다. 예를 들어, 만약 우리가 [coreboot](http://www.coreboot.org/) 소스를 보게 된다면, 아래와 같이 나온다:

```assembly
    .section ".reset"
    .code16
.globl  reset_vector
reset_vector:
    .byte  0xe9
    .int   _start - ( . + 2 )
    ...
```

여기에 우리가 `jmp` 명령의 [opcode](http://ref.x86asm.net/coder32.html#xE9)를 보면, 0xe9 라는 것을 알수 있고, `_start - ( . + 2)`에 목적지 주소가 있다. 우리는 또한 `reset` 섹션이 16바이트라는 것을 볼 수 있고 그것의 시작 주소는 `0xfffffff0` 이라는 걸 알 수 있다.:

```
SECTIONS {
    _ROMTOP = 0xfffffff0;
    . = _ROMTOP;
    .reset . : {
        *(.reset)
        . = 15 ;
        BYTE(0x00);
    }
}
```

이제 BIOS 가 시작된다; 하드웨어의 확인 및 초기화 이후에, BIOS는 부팅 가능한 장치를 찾는다. 부트 순서는 BIOS 설정에 저장되어 있고, BIOS 가 어떤 장치로 부터 시작해야 하는지 컨트롤 한다. 하드 드라이브로 부터 부팅을 시도할 때, BIOS는 부트섹터를 찾으려고 한다. 하드 드라이브 MBR 파티션을 포함하여 파티션이 나누어져 있는데, 부트 섹터는 첫번째 섹터의 446 바이트에 저장되어 있다. 첫 섹터의 마지막 두 바이트는 `0x55` 와 `0xaa` 인데 이것은 이 장치는 부팅에 사용 가능한 장치라는 것을 알려준다. 예를 들면:

```assembly
;
; Note: this example is written in Intel Assembly syntax
;
[BITS 16]
[ORG  0x7c00]

boot:
    mov al, '!'
    mov ah, 0x0e
    mov bh, 0x00
    mov bl, 0x07

    int 0x10
    jmp $

times 510-($-$$) db 0

db 0x55
db 0xaa
```

위 어셈블리를 빌드하고 실행하기:

```
nasm -f bin boot.nasm && qemu-system-x86_64 boot
```

이것은 디스크 이미지로 빌드된 `boot` 바이너리로 [QEMU](http://qemu.org)에서 사용되어 질 것이다. 부트섹터의 요구사항을 만족하는 위의 어셈블리 코드로 생성된 바이너리를 이용하면(BIOS는 이 코드를 0x7c00 에 로드하여 수행한다.), QEMU는 디스크 이미지의 마스터 부트 레코드(MBR-Master Boot Record) 로 생각할 것이다.

결과는:

![`!` 찍는 간단한 부트로더](http://oi60.tinypic.com/2qbwup0.jpg)

In this example we can see that the code will be executed in 16 bit real mode and will start at `0x7c00` in memory. After starting, it calls the [0x10](http://www.ctyme.com/intr/rb-0106.htm) interrupt, which just prints the `!` symbol; it fills the remaining 510 bytes with zeros and finishes with the two magic bytes `0xaa` and `0x55`.

You can see a binary dump of this using the `objdump` utility:

```
nasm -f bin boot.nasm
objdump -D -b binary -mi386 -Maddr16,data16,intel boot
```

A real-world boot sector has code for continuing the boot process and a partition table instead of a bunch of 0's and an exclamation mark :) From this point onwards, the BIOS hands over control to the bootloader.

**NOTE**: As explained above, the CPU is in real mode; in real mode, calculating the physical address in memory is done as follows:

```
PhysicalAddress = Segment Selector * 16 + Offset
```

just as explained before. We have only 16 bit general purpose registers; the maximum value of a 16 bit register is `0xffff`, so if we take the largest values, the result will be:

```python
>>> hex((0xffff * 16) + 0xffff)
'0x10ffef'
```

where `0x10ffef` is equal to `1MB + 64KB - 16b`. A [8086](https://en.wikipedia.org/wiki/Intel_8086) processor (which was the first processor with real mode), in contrast, has a 20 bit address line. Since `2^20 = 1048576` is 1MB, this means that the actual available memory is 1MB.

General real mode's memory map is as follows:

```
0x00000000 - 0x000003FF - Real Mode Interrupt Vector Table
0x00000400 - 0x000004FF - BIOS Data Area
0x00000500 - 0x00007BFF - Unused
0x00007C00 - 0x00007DFF - Our Bootloader
0x00007E00 - 0x0009FFFF - Unused
0x000A0000 - 0x000BFFFF - Video RAM (VRAM) Memory
0x000B0000 - 0x000B7777 - Monochrome Video Memory
0x000B8000 - 0x000BFFFF - Color Video Memory
0x000C0000 - 0x000C7FFF - Video ROM BIOS
0x000C8000 - 0x000EFFFF - BIOS Shadow Area
0x000F0000 - 0x000FFFFF - System BIOS
```

In the beginning of this post, I wrote that the first instruction executed by the CPU is located at address `0xFFFFFFF0`, which is much larger than `0xFFFFF` (1MB). How can the CPU access this address in real mode? The answer is in the [coreboot](http://www.coreboot.org/Developer_Manual/Memory_map) documentation:

```
0xFFFE_0000 - 0xFFFF_FFFF: 128 kilobyte ROM mapped into address space
```

At the start of execution, the BIOS is not in RAM, but in ROM.

Bootloader
--------------------------------------------------------------------------------

There are a number of bootloaders that can boot Linux, such as [GRUB 2](https://www.gnu.org/software/grub/) and [syslinux](http://www.syslinux.org/wiki/index.php/The_Syslinux_Project). The Linux kernel has a [Boot protocol](https://github.com/torvalds/linux/blob/master/Documentation/x86/boot.txt) which specifies the requirements for a bootloader to implement Linux support. This example will describe GRUB 2.

Continuing from before, now that the BIOS has chosen a boot device and transferred control to the boot sector code, execution starts from [boot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/boot.S;hb=HEAD). This code is very simple, due to the limited amount of space available, and contains a pointer which is used to jump to the location of GRUB 2's core image. The core image begins with [diskboot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/diskboot.S;hb=HEAD), which is usually stored immediately after the first sector in the unused space before the first partition. The above code loads the rest of the core image, which contains GRUB 2's kernel and drivers for handling filesystems, into memory. After loading the rest of the core image, it executes [grub_main](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/kern/main.c).

`grub_main` initializes the console, gets the base address for modules, sets the root device, loads/parses the grub configuration file, loads modules, etc. At the end of execution, `grub_main` moves grub to normal mode. `grub_normal_execute` (from `grub-core/normal/main.c`) completes the final preparations and shows a menu to select an operating system. When we select one of the grub menu entries, `grub_menu_execute_entry` runs, executing the grub `boot` command and booting the selected operating system.

As we can read in the kernel boot protocol, the bootloader must read and fill some fields of the kernel setup header, which starts at the `0x01f1` offset from the kernel setup code. The kernel header [arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S) starts from:

```assembly
    .globl hdr
hdr:
    setup_sects: .byte 0
    root_flags:  .word ROOT_RDONLY
    syssize:     .long 0
    ram_size:    .word 0
    vid_mode:    .word SVGA_MODE
    root_dev:    .word 0
    boot_flag:   .word 0xAA55
```

The bootloader must fill this and the rest of the headers (which are only marked as being type `write` in the Linux boot protocol, such as in [this example](https://github.com/torvalds/linux/blob/master/Documentation/x86/boot.txt#L354)) with values which it has either received from the  command line or calculated. (We will not go over full descriptions and explanations for all fields of the kernel setup header now but instead when the discuss how kernel uses them; you can find a description of all fields in the [boot protocol](https://github.com/torvalds/linux/blob/master/Documentation/x86/boot.txt#L156).)

As we can see in the kernel boot protocol, the memory map will be the following after loading the kernel:

```shell
         | Protected-mode kernel  |
100000   +------------------------+
         | I/O memory hole        |
0A0000   +------------------------+
         | Reserved for BIOS      | Leave as much as possible unused
         ~                        ~
         | Command line           | (Can also be below the X+10000 mark)
X+10000  +------------------------+
         | Stack/heap             | For use by the kernel real-mode code.
X+08000  +------------------------+
         | Kernel setup           | The kernel real-mode code.
         | Kernel boot sector     | The kernel legacy boot sector.
       X +------------------------+
         | Boot loader            | <- Boot sector entry point 0x7C00
001000   +------------------------+
         | Reserved for MBR/BIOS  |
000800   +------------------------+
         | Typically used by MBR  |
000600   +------------------------+
         | BIOS use only          |
000000   +------------------------+

```

So, when the bootloader transfers control to the kernel, it starts at:

```
X + sizeof(KernelBootSector) + 1
```

where `X` is the address of the kernel boot sector being loaded. In my case, `X` is `0x10000`, as we can see in a memory dump:

![kernel first address](http://oi57.tinypic.com/16bkco2.jpg)

The bootloader has now loaded the Linux kernel into memory, filled the header fields, and then jumped to the corresponding memory address. We can now move directly to the kernel setup code.

Start of Kernel Setup
--------------------------------------------------------------------------------

Finally, we are in the kernel! Technically, the kernel hasn't run yet; first, we need to set up the kernel, memory manager, process manager, etc. Kernel setup execution starts from [arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S) at [_start](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L293). It is a little strange at first sight, as there are several instructions before it.

A long time ago, the Linux kernel used to have its own bootloader. Now, however, if you run, for example,

```
qemu-system-x86_64 vmlinuz-3.18-generic
```

then you will see:

![Try vmlinuz in qemu](http://oi60.tinypic.com/r02xkz.jpg)

Actually, `header.S` starts from [MZ](https://en.wikipedia.org/wiki/DOS_MZ_executable) (see image above), the error message printing and following the [PE](https://en.wikipedia.org/wiki/Portable_Executable) header:

```assembly
#ifdef CONFIG_EFI_STUB
# "MZ", MS-DOS header
.byte 0x4d
.byte 0x5a
#endif
...
...
...
pe_header:
    .ascii "PE"
    .word 0
```

It needs this to load an operating system with [UEFI](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface). We won't be looking into its inner workings right now and will cover it in upcoming chapters.

The actual kernel setup entry point is:

```assembly
// header.S line 292
.globl _start
_start:
```

The bootloader (grub2 and others) knows about this point (`0x200` offset from `MZ`) and makes a jump directly to it, despite the fact that `header.S` starts from the `.bstext` section, which prints an error message:

```
//
// arch/x86/boot/setup.ld
//
. = 0;                    // current position
.bstext : { *(.bstext) }  // put .bstext section to position 0
.bsdata : { *(.bsdata) }
```

The kernel setup entry point is:

```assembly
    .globl _start
_start:
    .byte  0xeb
    .byte  start_of_setup-1f
1:
    //
    // rest of the header
    //
```

Here we can see a `jmp` instruction opcode (`0xeb`) that jumps to the `start_of_setup-1f` point. In `Nf` notation, `2f` refers to the following local `2:` label; in our case, it is label `1` that is present right after jump, and it contains the rest of the setup [header](https://github.com/torvalds/linux/blob/master/Documentation/x86/boot.txt#L156). Right after the setup header, we see the `.entrytext` section, which starts at the `start_of_setup` label.

This is the first code that actually runs (aside from the previous jump instructions, of course). After the kernel setup received control from the bootloader, the first `jmp` instruction is located at the `0x200` offset from the start of the kernel real mode, i.e., after the first 512 bytes. This we can both read in the Linux kernel boot protocol and see in the grub2 source code:

```C
segment = grub_linux_real_target >> 4;
state.gs = state.fs = state.es = state.ds = state.ss = segment;
state.cs = segment + 0x20;
```

This means that segment registers will have the following values after kernel setup starts:

```
gs = fs = es = ds = ss = 0x1000
cs = 0x1020
```

In my case, the kernel is loaded at `0x10000`.

After the jump to `start_of_setup`, the kernel needs to do the following:

* Make sure that all segment register values are equal
* Set up a correct stack, if needed
* Set up [bss](https://en.wikipedia.org/wiki/.bss)
* Jump to the C code in [main.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c)

Let's look at the implementation.

Segment registers align
--------------------------------------------------------------------------------

First of all, the kernel ensures that `ds` and `es` segment registers point to the same address. Next, it clears the direction flag using the `cld` instruction:

```assembly
    movw    %ds, %ax
    movw    %ax, %es
    cld
```

As I wrote earlier, grub2 loads kernel setup code at address `0x10000` and `cs` at `0x1020` because execution doesn't start from the start of file, but from

```assembly
_start:
    .byte 0xeb
    .byte start_of_setup-1f
```

`jump`, which is at a 512 byte offset from [4d 5a](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L47). It also needs to align `cs` from `0x10200` to `0x10000`, as well as all other segment registers. After that, we set up the stack:

```assembly
    pushw   %ds
    pushw   $6f
    lretw
```

which pushes the value of `ds` to the stack with the address of the [6](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L494) label and executes the `lretw` instruction. When the `lretw` instruction is called, it loads the address of label `6` into the [instruction pointer](https://en.wikipedia.org/wiki/Program_counter) register and loads `cs` with the value of `ds`. Afterwards, `ds` and `cs` will have the same values.

Stack Setup
--------------------------------------------------------------------------------

Almost all of the setup code is in preparation for the C language environment in real mode. The next [step](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L467) is checking the `ss` register value and making a correct stack if `ss` is wrong:

```assembly
    movw    %ss, %dx
    cmpw    %ax, %dx
    movw    %sp, %dx
    je      2f
```

This can lead to 3 different scenarios:

* `ss` has valid value 0x10000 (as do all other segment registers beside `cs`)
* `ss` is invalid and `CAN_USE_HEAP` flag is set     (see below)
* `ss` is invalid and `CAN_USE_HEAP` flag is not set (see below)

Let's look at all three of these scenarios in turn:

* `ss` has a correct address (0x10000). In this case, we go to label [2](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L481):

```assembly
2:  andw    $~3, %dx
    jnz     3f
    movw    $0xfffc, %dx
3:  movw    %ax, %ss
    movzwl  %dx, %esp
    sti
```

Here we can see the alignment of `dx` (contains `sp` given by bootloader) to 4 bytes and a check for whether or not it is zero. If it is zero, we put `0xfffc` (4 byte aligned address before the maximum segment size of 64 KB) in `dx`. If it is not zero, we continue to use `sp`, given by the bootloader (0xf7f4 in my case). After this, we put the `ax` value into `ss`, which stores the correct segment address of `0x10000` and sets up a correct `sp`. We now have a correct stack:

![stack](http://oi58.tinypic.com/16iwcis.jpg)

* In the second scenario, (`ss` != `ds`). First, we put the value of [_end](https://github.com/torvalds/linux/blob/master/arch/x86/boot/setup.ld#L52) (the address of the end of the setup code) into `dx` and check the `loadflags` header field using the `testb` instruction to see whether we can use the heap. [loadflags](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L321) is a bitmask header which is defined as:

```C
#define LOADED_HIGH     (1<<0)
#define QUIET_FLAG      (1<<5)
#define KEEP_SEGMENTS   (1<<6)
#define CAN_USE_HEAP    (1<<7)
```

and, as we can read in the boot protocol,

```
Field name: loadflags

  This field is a bitmask.

  Bit 7 (write): CAN_USE_HEAP
    Set this bit to 1 to indicate that the value entered in the
    heap_end_ptr is valid.  If this field is clear, some setup code
    functionality will be disabled.
```

If the `CAN_USE_HEAP` bit is set, we put `heap_end_ptr` into `dx` (which points to `_end`) and add `STACK_SIZE` (minimum stack size, 512 bytes) to it. After this, if `dx` is not carried (it will not be carried, dx = _end + 512), jump to label `2` (as in the previous case) and make a correct stack.

![stack](http://oi62.tinypic.com/dr7b5w.jpg)

* When `CAN_USE_HEAP` is not set, we just use a minimal stack from `_end` to `_end + STACK_SIZE`:

![minimal stack](http://oi60.tinypic.com/28w051y.jpg)

BSS Setup
--------------------------------------------------------------------------------

The last two steps that need to happen before we can jump to the main C code are setting up the [BSS](https://en.wikipedia.org/wiki/.bss) area and checking the "magic" signature. First, signature checking:

```assembly
    cmpl    $0x5a5aaa55, setup_sig
    jne     setup_bad
```

This simply compares the [setup_sig](https://github.com/torvalds/linux/blob/master/arch/x86/boot/setup.ld#L39) with the magic number `0x5a5aaa55`. If they are not equal, a fatal error is reported.

If the magic number matches, knowing we have a set of correct segment registers and a stack, we only need to set up the BSS section before jumping into the C code.

The BSS section is used to store statically allocated, uninitialized data. Linux carefully ensures this area of memory is first zeroed using the following code:

```assembly
    movw    $__bss_start, %di
    movw    $_end+3, %cx
    xorl    %eax, %eax
    subw    %di, %cx
    shrw    $2, %cx
    rep; stosl
```

First, the [__bss_start](https://github.com/torvalds/linux/blob/master/arch/x86/boot/setup.ld#L47) address is moved into `di`. Next, the `_end + 3` address (+3 - aligns to 4 bytes) is moved into `cx`. The `eax` register is cleared (using a `xor` instruction), and the bss section size (`cx`-`di`) is calculated and put into `cx`. Then, `cx` is divided by four (the size of a 'word'), and the `stosl` instruction is used repeatedly, storing the value of `eax` (zero) into the address pointed to by `di`, automatically increasing `di` by four, repeating until `cx` reaches zero). The net effect of this code is that zeros are written through all words in memory from `__bss_start` to `_end`:

![bss](http://oi59.tinypic.com/29m2eyr.jpg)

Jump to main
--------------------------------------------------------------------------------

That's all - we have the stack and BSS, so we can jump to the `main()` C function:

```assembly
    calll main
```

The `main()` function is located in [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c). You can read about what this does in the next part.

Conclusion
--------------------------------------------------------------------------------

This is the end of the first part about Linux kernel insides. If you have questions or suggestions, ping me on twitter [0xAX](https://twitter.com/0xAX), drop me an [email](anotherworldofworld@gmail.com), or just create an [issue](https://github.com/0xAX/linux-internals/issues/new). In the next part, we will see the first C code that executes in the Linux kernel setup, the implementation of memory routines such as `memset`, `memcpy`, `earlyprintk`, early console implementation and initialization, and much more.

**Please note that English is not my first language and I am really sorry for any inconvenience. If you find any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-internals).**

Links
--------------------------------------------------------------------------------

  * [Intel 80386 programmer's reference manual 1986](http://css.csail.mit.edu/6.858/2014/readings/i386.pdf)
  * [Minimal Boot Loader for Intel® Architecture](https://www.cs.cmu.edu/~410/doc/minimal_boot.pdf)
  * [8086](http://en.wikipedia.org/wiki/Intel_8086)
  * [80386](http://en.wikipedia.org/wiki/Intel_80386)
  * [Reset vector](http://en.wikipedia.org/wiki/Reset_vector)
  * [Real mode](http://en.wikipedia.org/wiki/Real_mode)
  * [Linux kernel boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt)
  * [CoreBoot developer manual](http://www.coreboot.org/Developer_Manual)
  * [Ralf Brown's Interrupt List](http://www.ctyme.com/intr/int.htm)
  * [Power supply](http://en.wikipedia.org/wiki/Power_supply)
  * [Power good signal](http://en.wikipedia.org/wiki/Power_good_signal)
