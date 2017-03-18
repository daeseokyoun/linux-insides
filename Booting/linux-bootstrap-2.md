커널 부팅 과정. part 2.
================================================================================

커널 설정의 처음 단계
--------------------------------------------------------------------------------

우리는 이전 [part](linux-bootstrap-1.md) 에서 리눅스를 깊게 파보기를 시작했고 커널 설정 코드의 초기화 부분을 보았다. 우리는 `main` 함수[arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c). 의 호출에서 이어 나가기로 한다.(C 로 작성된 첫 함수이다.)

이 파트에서 우리는 커널 설정 코드를 계속해서 살펴볼 것이고
* `보호 모드` 가 무엇인지 보고
* 어떤 준비는 그것에 진입하기 위한 전환을 위한 것이다.
* 힙과 콘솔 초기화,
* memory detection, cpu validation, keyboard initialization
* 메모리 검출, CPU 확인, 키보드 초기화
* 그리고 많은 부분이 남아 있다.

이제 시작해보자.

보호 모드(Protected mode)
--------------------------------------------------------------------------------

우리가 Intel64 [Long mode](http://en.wikipedia.org/wiki/Long_mode) 로 넘어가기 전에, 커널은 CPU 를 보호 모드로 전환해야 한다. ([Long mode-한글](https://ko.wikipedia.org/wiki/%EB%A1%B1_%EB%AA%A8%EB%93%9C))

[보호 모드-한글](https://ko.wikipedia.org/wiki/보호_모드)/[Protected mode](https://en.wikipedia.org/wiki/Protected_mode) 란 무엇인가? 보호 모드는 1982년 x86 아키텍처에 처음 추가된 모드이며, Intel 64 와 long mode 가 나타나지 전에 [80286](http://en.wikipedia.org/wiki/Intel_80286) 프로세서 계열인 인텔 프로세서의 주요(main) 모드였다.

[Real mode](http://wiki.osdev.org/Real_Mode)로 부터 변환되는 주요 이유는 메모리 접근이 매우 제한 적이였기 때문이다. 이전 파트의 내용을 기억한다면, Real mode 에서는 2<sup>20</sup> 바이트 또는 1 메가 바이트만이 있고, 때로는 640 킬로바이트 메모리만 가용적이었다.

보호 모드는 많은 변화를 가져왔고, 메모리 관리에서 주요한 하나의 차이를 갖고 있다. 20 비트 주소 버스는 32 비트 주소 버스로 교체된다. 그것은 real mode 에서 1 MB 접근하던 메모리를 4 GB 까지 접근 허용하겠다는 것이다. 또한 다음 섹션에서 설명할 [paging](http://en.wikipedia.org/wiki/Paging)/[페이징-한글](https://ko.wikipedia.org/wiki/%ED%8E%98%EC%9D%B4%EC%A7%95) 의 지원이 추가된다.

보호 모드에서 메모리 관리는 두 가지로 분리가능 하고, 두 부분이 굉장히 독립적이다.:

* 세그먼트
* 페이징

여기서 우리는 단지 세그먼트의 내용만 볼 것이다. 페이징은 다음 섹션에서 논의해보도록 하자.

이전 파트에서 읽을 수 있었듯이, 주소는 real mode 에서 두 개의 부분으로 구성된다:

* 세그먼트의 베이스 주소
* 베이스 주소에서의 오프셋

만약 우리가 이 두가지의 부분을 다 알고 있다면 물리 주소를 얻을 수 있을 것이다.:

```
PhysicalAddress = Segment Selector * 16 + Offset
```

메모리 세그먼트는 보호 모드에서 완전히 다르게 사용된다. 이제는 64 KB 고정 크기 세그먼트는 없다. 대신에, 각 세그먼트의 크기와 위치는 _세그먼트 디스크립터_ 라 불리는 자료 구조에 연관된 내용이 저장될 것이다. 그 세그먼트 디스크립터는 `Global Descriptor Table`(GDT) 라 불리는 자료구조에 저장된다.

GDT는 메모리에 존재한는 구조체 이다. 이것은 메모리네애 고정된 위치는 아니고, 특수한 `GDTR` 레지스터에 그 주소가 저장된다. 리눅스 커널 코드에서 GDT가 로딩되는 것은 나중에 살펴볼 것이다. 그것을 메모리로 로딩하는 아래와 같은 명령들이 있다.:

```assembly
lgdt gdt
```

`lgdt` 명령어는 베이스 주소와 global descriptor table 의 크기(제약)을 `GDTR` 레지스터에 로드 한다. `GDTR`은 48 비트 레지스터고 두 개의 파트로 구성된다.:

 * global descriptor table 의 크기(16 비트)
 * global descriptor table 의 32비트 주소

위에서 언급했듯이 GDT는 메모리 세그먼트를 기술하는 `세그먼트 디스크립터`를 갖고 있다. 각 디스크립터는 64 비트의 크기를 갖는다. 일반적인 디스크립터의 구성은 다음과 같다:

```
31          24        19      16              7            0
------------------------------------------------------------
|             | |B| |A|       | |   | |0|E|W|A|            |
| BASE 31:24  |G|/|L|V| LIMIT |P|DPL|S|  TYPE | BASE 23:16 | 4
|             | |D| |L| 19:16 | |   | |1|C|R|A|            |
------------------------------------------------------------
|                             |                            |
|        BASE 15:0            |       LIMIT 15:0           | 0
|                             |                            |
------------------------------------------------------------
```

걱정 마라, 나는 real mode 이후에 매우 무섭게 변한 것을 알고 있지만 그것은 알고 보면 쉽다. 예를 들어 LIMIT 15:0 은 디스크립터의 0-15 비트의 값은 제한(limit)의 값을 표현한다. 그리고 LIMIT 19:16 에는 나머지가 있다. 그래서 Limit 의 크기는 0-19 비트로 총 20 비트가 된다. 조금 더 살펴보자:

1. Limit[20 비트] 는 0-15, 16-19 비트에 있다. 그것은 `length_of_segment - 1`을 정의한다. 그것은 `G`(Granularity) 비트에 의존적이다.

  * 만약 `G` (비트 55) 가 0 이고 세그먼트 제한(limit) 이 0 이면 세그먼트의 크기는 1 바이트가 된다.
  * 만약 `G` 가 0 이고 세그먼트 제한이 0xffff 이면, 세그먼트의 크기는 1 MB 이다.
  * 만약 `G` 가 1 이고 세그먼트 제한이 0 이면 세그먼트 크기는 4096 바이트이다.
  * 만약 `G` 가 1 이고 세그먼트 제한이 0xfffff 이면, 세그먼트의 크기는 4GB 이다.

  그래서 그것들이 의미하는 것은,
  * 만약 `G` 가 0 이면, 제한(Limit) 의 크기는 1 바이트 에서 최대 1 MB 가 된다고 해석될 수 있다.
  * 만약 `G` 가 1 이면, 제한(Limit) 의 크기는 4096 바이트 = 4 KB = 1 페이지에서 최대 4 GB 의 크기의 세그먼트를 가질 수 있다고 해석된다. 실제로 `G` 가 1일 때, Limit 의 값은 12 비트 왼쪽 쉬프트가 된다. 그래서 20 비트 + 12 비트 = 32 비트 이고, 2<sup>32</sup> = 4 GB이다.

2. 베이스[32 비트] 는 0-15, 32-39 그리고 56-53 비트에 있다. 그것은 세그먼트의 시작 위치의 물리 주소를 정의한다.

3. 타입/속성 (40-47 비트) 는 세그먼트의 타입과 그것의 접근 권한에 대한 정의를 한다.
  * 44 비트에 있는 `S` 플래그는 디스크립터의 타입을 결정한다. 만얀 `S` 가 0 이면, 그 세그먼트는 시스템 세그먼트 이고, 반대로 `S`가 1 이면, 그 세그먼트는 코드 또는 데이터 세그먼트이다. (스택 세그먼트는 읽기와 쓰기 권한이 주어진 데이터 세그먼트라고 할 수 있다.)

만약 세그먼트가 코드나 데이터 세그먼트인지 결정하기 위해서는 우리는 Ex(43 비트) 의 속성이 0 으로 되어 있는지 확인 해야 한다. 만약 그것이 0이면 데이터 세그먼트 이고 1이면 코드 세그먼트가 된다.

세그먼트는 아래 나오는 타입중에 하나일 것이다.:

```
|           Type Field        | Descriptor Type | Description
|-----------------------------|-----------------|------------------
| Decimal                     |                 |
|             0    E    W   A |                 |
| 0           0    0    0   0 | Data            | Read-Only
| 1           0    0    0   1 | Data            | Read-Only, accessed
| 2           0    0    1   0 | Data            | Read/Write
| 3           0    0    1   1 | Data            | Read/Write, accessed
| 4           0    1    0   0 | Data            | Read-Only, expand-down
| 5           0    1    0   1 | Data            | Read-Only, expand-down, accessed
| 6           0    1    1   0 | Data            | Read/Write, expand-down
| 7           0    1    1   1 | Data            | Read/Write, expand-down, accessed
|                  C    R   A |                 |
| 8           1    0    0   0 | Code            | Execute-Only
| 9           1    0    0   1 | Code            | Execute-Only, accessed
| 10          1    0    1   0 | Code            | Execute/Read
| 11          1    0    1   1 | Code            | Execute/Read, accessed
| 12          1    1    0   0 | Code            | Execute-Only, conforming
| 14          1    1    0   1 | Code            | Execute-Only, conforming, accessed
| 13          1    1    1   0 | Code            | Execute/Read, conforming
| 15          1    1    1   1 | Code            | Execute/Read, conforming, accessed
```

위의 테이블에서 처음 비트가(Ex 43 비트) 0이면 `데이터 세그먼트`고 `1` 이면 `코드 세그먼트` 이다. 다음 3 비트는 (40, 41, 42) 는 각각 `EWA`(Expansion Writeable Accessible) 또는 `CRA` (conforming Readable Accessible) 로 나타난다.
  * 만약 E(비트 42) 가 0 이면, Expand up 세그먼트이고 1이면 expand down 세그먼트다. 더 알고 싶다면, [여기](http://www.sudleyplace.com/dpmione/expanddown.html).
  * 만약 W(비트 41)(데이터 세그먼트를 위함) 이 1이면, 쓰기 접근이 가능하고 0이면 안된다. 읽기 접근은 데이터 세그먼트의 경우 항상 허용된다.
  * A(비트 40) - 프로세서에 의해 이 세그먼트가 접근 가능 한지 아닌지에 대한 설정.
  * C(비트 42) 는 코드 세그먼트를 위한 규칙을 수행하도록 한다. 만약 C 가 1이면, 세그먼트의 코드는 낮은 단계의 특권 레벨로 실행가능 할 것이고 0이라면 같은 특권 레벨에서 수행되어야 할 것이다.
  * R(비트 41)(코드 세그먼트). 만약 1이면 세그먼트의 읽기 허용이 되고 0이면 안된다. 코드 세그먼트로 쓰기 접근은 절대로 허용되지 않는다.

4. DPL[2 비트] (디스크립터 특권 레벨-Privilege level)이고 45-46 비트에 위치한다. 그것은 세그먼트의 특권 레벨을 정의한다. 특권 레벨은 0-3 까지 지정 가능하며 0이 가장 높은 특권을 가리킨다.

5. P 플래그(비트 47) - 만약 세그먼트가 메모리에 현재 로드되어 있는지 안되어 있는지를 알려준다. 만약 0이면 세그먼트는 _invalid_ 상태인 것이며, 프로세서가 그 세그먼트를 읽으려고 할 때, 허용하지 않을 것이다.

6. AVL 플래그(비트 52) - 예약된 비트 공간이다. 리눅스에서는 무시된다.

7. L 플래그(비트 53) - 코드 세그먼트가 64 비트 코드를 포함하는지 여부를 알려준다. 만약 1이면 코드 세그먼트는 64 비트 모드에서 수행 되고 있다는 것이다.

8. D/B 플래그(비트 54) - Default/Big 플래그이며 피연산자(operand)의 크기가 16/32 비트인지를 알려준다. 만약 1이면 32비트이고 아니면 16 비트이다.

세그먼트 레지스터들은 real mode 에서 사용된 세그먼트 셀렉터를 포함한다. 하지만, 보호 모드내에서는, 세그먼트 셀렉터는 다르게 응용된다. 각 세그먼트 디스크립터는 연관된 16비트로 구성된 세그먼트 셀렉터를 가지고 있다.:
```
15              3  2   1  0
-----------------------------
|      Index     | TI | RPL |
-----------------------------
```

위의 테이블에서,
* **Index** GDT 에서 디스크립터의 인덱스 번호를 보여준다.
* **TI**(테이블 지시자-Table Indicator) 은 디스크립의 위치를 알려준다. 만약 0 이면 GDT 에 있다는 것이고 1이면 LDT(Local Descriptor Table) 에 있다는 것이다.
* 그리고 **RPL** 는 요청한 시점에 특권 레벨을 알수 있다.

Every segment register has a visible and hidden part.
모든 세그먼트 레지스터는 보여지는 부분(visible)과 숨겨진(hidden) 부분이 있다.
* 보여지는 부분(Visible) - 세그먼트 셀렉터가 여기에 저장
* 숨겨진 부분(Hidden) - 세그먼트 디스크립터(베이스, 제한(limit), 속성, 플래그들)

The following steps are needed to get the physical address in the protected mode:

* The segment selector must be loaded in one of the segment registers
* The CPU tries to find a segment descriptor by GDT address + Index from selector and load the descriptor into the *hidden* part of the segment register
* Base address (from segment descriptor) + offset will be the linear address of the segment which is the physical address (if paging is disabled).

Schematically it will look like this:

![linear address](http://oi62.tinypic.com/2yo369v.jpg)

The algorithm for the transition from real mode into protected mode is:

* Disable interrupts
* Describe and load GDT with `lgdt` instruction
* Set PE (Protection Enable) bit in CR0 (Control Register 0)
* Jump to protected mode code

We will see the complete transition to protected mode in the linux kernel in the next part, but before we can move to protected mode, we need to do some more preparations.

Let's look at [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c). We can see some routines there which perform keyboard initialization, heap initialization, etc... Let's take a look.

Copying boot parameters into the "zeropage"
--------------------------------------------------------------------------------

We will start from the `main` routine in "main.c". First function which is called in `main` is [`copy_boot_params(void)`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c#L30). It copies the kernel setup header into the field of the `boot_params` structure which is defined in the [arch/x86/include/uapi/asm/bootparam.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/uapi/asm/bootparam.h#L113).

The `boot_params` structure contains the `struct setup_header hdr` field. This structure contains the same fields as defined in [linux boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt) and is filled by the boot loader and also at kernel compile/build time. `copy_boot_params` does two things:

1. Copies `hdr` from [header.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L281) to the `boot_params` structure in `setup_header` field

2. Updates pointer to the kernel command line if the kernel was loaded with the old command line protocol.

Note that it copies `hdr` with `memcpy` function which is defined in the [copy.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/copy.S) source file. Let's have a look inside:

```assembly
GLOBAL(memcpy)
    pushw   %si
    pushw   %di
    movw    %ax, %di
    movw    %dx, %si
    pushw   %cx
    shrw    $2, %cx
    rep; movsl
    popw    %cx
    andw    $3, %cx
    rep; movsb
    popw    %di
    popw    %si
    retl
ENDPROC(memcpy)
```

Yeah, we just moved to C code and now assembly again :) First of all we can see that `memcpy` and other routines which are defined here, start and end with the two macros: `GLOBAL` and `ENDPROC`. `GLOBAL` is described in [arch/x86/include/asm/linkage.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/linkage.h) which defines `globl` directive and the label for it. `ENDPROC` is described in [include/linux/linkage.h](https://github.com/torvalds/linux/blob/master/include/linux/linkage.h) which marks the `name` symbol as a function name and ends with the size of the `name` symbol.

Implementation of `memcpy` is easy. At first, it pushes values from the `si` and `di` registers to the stack to preserve their values because they will change during the `memcpy`. `memcpy` (and other functions in copy.S) use `fastcall` calling conventions. So it gets its incoming parameters from the `ax`, `dx` and `cx` registers.  Calling `memcpy` looks like this:

```c
memcpy(&boot_params.hdr, &hdr, sizeof hdr);
```

So,
* `ax` will contain the address of the `boot_params.hdr`
* `dx` will contain the address of `hdr`
* `cx` will contain the size of `hdr` in bytes.

`memcpy` puts the address of `boot_params.hdr` into `di` and saves the size on the stack. After this it shifts to the right on 2 size (or divide on 4) and copies from `si` to `di` by 4 bytes. After this we restore the size of `hdr` again, align it by 4 bytes and copy the rest of the bytes from `si` to `di` byte by byte (if there is more). Restore `si` and `di` values from the stack in the end and after this copying is finished.

Console initialization
--------------------------------------------------------------------------------

After `hdr` is copied into `boot_params.hdr`, the next step is console initialization by calling the `console_init` function which is defined in [arch/x86/boot/early_serial_console.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/early_serial_console.c).

It tries to find the `earlyprintk` option in the command line and if the search was successful, it parses the port address and baud rate of the serial port and initializes the serial port. Value of `earlyprintk` command line option can be one of these:

* serial,0x3f8,115200
* serial,ttyS0,115200
* ttyS0,115200

After serial port initialization we can see the first output:

```C
if (cmdline_find_option_bool("debug"))
    puts("early console in setup code\n");
```

The definition of `puts` is in [tty.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/tty.c). As we can see it prints character by character in a loop by calling the `putchar` function. Let's look into the `putchar` implementation:

```C
void __attribute__((section(".inittext"))) putchar(int ch)
{
    if (ch == '\n')
        putchar('\r');

    bios_putchar(ch);

    if (early_serial_base != 0)
        serial_putchar(ch);
}
```

`__attribute__((section(".inittext")))` means that this code will be in the `.inittext` section. We can find it in the linker file [setup.ld](https://github.com/torvalds/linux/blob/master/arch/x86/boot/setup.ld#L19).

First of all, `putchar` checks for the `\n` symbol and if it is found, prints `\r` before. After that it outputs the character on the VGA screen by calling the BIOS with the `0x10` interrupt call:

```C
static void __attribute__((section(".inittext"))) bios_putchar(int ch)
{
    struct biosregs ireg;

    initregs(&ireg);
    ireg.bx = 0x0007;
    ireg.cx = 0x0001;
    ireg.ah = 0x0e;
    ireg.al = ch;
    intcall(0x10, &ireg, NULL);
}
```

Here `initregs` takes the `biosregs` structure and first fills `biosregs` with zeros using the `memset` function and then fills it with register values.

```C
    memset(reg, 0, sizeof *reg);
    reg->eflags |= X86_EFLAGS_CF;
    reg->ds = ds();
    reg->es = ds();
    reg->fs = fs();
    reg->gs = gs();
```

Let's look at the [memset](https://github.com/torvalds/linux/blob/master/arch/x86/boot/copy.S#L36) implementation:

```assembly
GLOBAL(memset)
    pushw   %di
    movw    %ax, %di
    movzbl  %dl, %eax
    imull   $0x01010101,%eax
    pushw   %cx
    shrw    $2, %cx
    rep; stosl
    popw    %cx
    andw    $3, %cx
    rep; stosb
    popw    %di
    retl
ENDPROC(memset)
```

As you can read above, it uses the `fastcall` calling conventions like the `memcpy` function, which means that the function gets parameters from `ax`, `dx` and `cx` registers.

Generally `memset` is like a memcpy implementation. It saves the value of the `di` register on the stack and puts the `ax` value into `di` which is the address of the `biosregs` structure. Next is the `movzbl` instruction, which copies the `dl` value to the low 2 bytes of the `eax` register. The remaining 2 high bytes  of `eax` will be filled with zeros.

The next instruction multiplies `eax` with `0x01010101`. It needs to because `memset` will copy 4 bytes at the same time. For example, we need to fill a structure with `0x7` with memset. `eax` will contain `0x00000007` value in this case. So if we multiply `eax` with `0x01010101`, we will get `0x07070707` and now we can copy these 4 bytes into the structure. `memset` uses `rep; stosl` instructions for copying `eax` into `es:di`.

The rest of the `memset` function does almost the same as `memcpy`.

After the `biosregs` structure is filled with `memset`, `bios_putchar` calls the [0x10](http://www.ctyme.com/intr/rb-0106.htm) interrupt which prints a character. Afterwards it checks if the serial port was initialized or not and writes a character there with [serial_putchar](https://github.com/torvalds/linux/blob/master/arch/x86/boot/tty.c#L30) and `inb/outb` instructions if it was set.

Heap initialization
--------------------------------------------------------------------------------

After the stack and bss section were prepared in [header.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S) (see previous [part](linux-bootstrap-1.md)), the kernel needs to initialize the [heap](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c#L116) with the [`init_heap`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c#L116) function.

First of all `init_heap` checks the [`CAN_USE_HEAP`](https://github.com/torvalds/linux/blob/master/arch/x86/include/uapi/asm/bootparam.h#L21) flag from the [`loadflags`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L321) in the kernel setup header and calculates the end of the stack if this flag was set:

```C
    char *stack_end;

    if (boot_params.hdr.loadflags & CAN_USE_HEAP) {
        asm("leal %P1(%%esp),%0"
            : "=r" (stack_end) : "i" (-STACK_SIZE));
```

or in other words `stack_end = esp - STACK_SIZE`.

Then there is the `heap_end` calculation:
```c
    heap_end = (char *)((size_t)boot_params.hdr.heap_end_ptr + 0x200);
```
which means `heap_end_ptr` or `_end` + `512`(`0x200h`). The last check is whether `heap_end` is greater than `stack_end`. If it is then `stack_end` is assigned to `heap_end` to make them equal.

Now the heap is initialized and we can use it using the `GET_HEAP` method. We will see how it is used, how to use it and how the it is implemented in the next posts.

CPU validation
--------------------------------------------------------------------------------

The next step as we can see is cpu validation by `validate_cpu` from [arch/x86/boot/cpu.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/cpu.c).

It calls the [`check_cpu`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/cpucheck.c#L102) function and passes cpu level and required cpu level to it and checks that the kernel launches on the right cpu level.
```c
check_cpu(&cpu_level, &req_level, &err_flags);
if (cpu_level < req_level) {
    ...
    return -1;
}
```
`check_cpu` checks the cpu's flags, presence of [long mode](http://en.wikipedia.org/wiki/Long_mode) in case of x86_64(64-bit) CPU, checks the processor's vendor and makes preparation for certain vendors like turning off SSE+SSE2 for AMD if they are missing, etc.

Memory detection
--------------------------------------------------------------------------------

The next step is memory detection by the `detect_memory` function. `detect_memory` basically provides a map of available RAM to the cpu. It uses different programming interfaces for memory detection like `0xe820`, `0xe801` and `0x88`. We will see only the implementation of **0xE820** here.

Let's look into the `detect_memory_e820` implementation from the [arch/x86/boot/memory.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/memory.c) source file. First of all, the `detect_memory_e820` function initializes the `biosregs` structure as we saw above and fills registers with special values for the `0xe820` call:

```assembly
    initregs(&ireg);
    ireg.ax  = 0xe820;
    ireg.cx  = sizeof buf;
    ireg.edx = SMAP;
    ireg.di  = (size_t)&buf;
```

* `ax` contains the number of the function (0xe820 in our case)
* `cx` register contains size of the buffer which will contain data about memory
* `edx` must contain the `SMAP` magic number
* `es:di` must contain the address of the buffer which will contain memory data
* `ebx` has to be zero.

Next is a loop where data about the memory will be collected. It starts from the call of the `0x15` BIOS interrupt, which writes one line from the address allocation table. For getting the next line we need to call this interrupt again (which we do in the loop). Before the next call `ebx` must contain the value returned previously:

```C
    intcall(0x15, &ireg, &oreg);
    ireg.ebx = oreg.ebx;
```

Ultimately, it does iterations in the loop to collect data from the address allocation table and writes this data into the `e820entry` array:

* start of memory segment
* size  of memory segment
* type of memory segment (which can be reserved, usable and etc...).

You can see the result of this in the `dmesg` output, something like:

```
[    0.000000] e820: BIOS-provided physical RAM map:
[    0.000000] BIOS-e820: [mem 0x0000000000000000-0x000000000009fbff] usable
[    0.000000] BIOS-e820: [mem 0x000000000009fc00-0x000000000009ffff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000000f0000-0x00000000000fffff] reserved
[    0.000000] BIOS-e820: [mem 0x0000000000100000-0x000000003ffdffff] usable
[    0.000000] BIOS-e820: [mem 0x000000003ffe0000-0x000000003fffffff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000fffc0000-0x00000000ffffffff] reserved
```

Keyboard initialization
--------------------------------------------------------------------------------

The next step is the initialization of the keyboard with the call of the [`keyboard_init()`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c#L65) function. At first `keyboard_init` initializes registers using the `initregs` function and calling the [0x16](http://www.ctyme.com/intr/rb-1756.htm) interrupt for getting the keyboard status.
```c
    initregs(&ireg);
    ireg.ah = 0x02;     /* Get keyboard status */
    intcall(0x16, &ireg, &oreg);
    boot_params.kbd_status = oreg.al;
```
After this it calls [0x16](http://www.ctyme.com/intr/rb-1757.htm) again to set repeat rate and delay.
```c
    ireg.ax = 0x0305;   /* Set keyboard repeat rate */
    intcall(0x16, &ireg, NULL);
```

Querying
--------------------------------------------------------------------------------

The next couple of steps are queries for different parameters. We will not dive into details about these queries, but will get back to it in later parts. Let's take a short look at these functions:

The [query_mca](https://github.com/torvalds/linux/blob/master/arch/x86/boot/mca.c#L18) routine calls the [0x15](http://www.ctyme.com/intr/rb-1594.htm) BIOS interrupt to get the machine model number, sub-model number, BIOS revision level, and other hardware-specific attributes:

```c
int query_mca(void)
{
    struct biosregs ireg, oreg;
    u16 len;

    initregs(&ireg);
    ireg.ah = 0xc0;
    intcall(0x15, &ireg, &oreg);

    if (oreg.eflags & X86_EFLAGS_CF)
        return -1;  /* No MCA present */

    set_fs(oreg.es);
    len = rdfs16(oreg.bx);

    if (len > sizeof(boot_params.sys_desc_table))
        len = sizeof(boot_params.sys_desc_table);

    copy_from_fs(&boot_params.sys_desc_table, oreg.bx, len);
    return 0;
}
```

It fills  the `ah` register with `0xc0` and calls the `0x15` BIOS interruption. After the interrupt execution it checks  the [carry flag](http://en.wikipedia.org/wiki/Carry_flag) and if it is set to 1, the BIOS doesn't support [**MCA**](https://en.wikipedia.org/wiki/Micro_Channel_architecture). If carry flag is set to 0, `ES:BX` will contain a pointer to the system information table, which looks like this:

```
Offset  Size    Description
 00h    WORD    number of bytes following
 02h    BYTE    model (see #00515)
 03h    BYTE    submodel (see #00515)
 04h    BYTE    BIOS revision: 0 for first release, 1 for 2nd, etc.
 05h    BYTE    feature byte 1 (see #00510)
 06h    BYTE    feature byte 2 (see #00511)
 07h    BYTE    feature byte 3 (see #00512)
 08h    BYTE    feature byte 4 (see #00513)
 09h    BYTE    feature byte 5 (see #00514)
---AWARD BIOS---
 0Ah  N BYTEs   AWARD copyright notice
---Phoenix BIOS---
 0Ah    BYTE    ??? (00h)
 0Bh    BYTE    major version
 0Ch    BYTE    minor version (BCD)
 0Dh  4 BYTEs   ASCIZ string "PTL" (Phoenix Technologies Ltd)
---Quadram Quad386---
 0Ah 17 BYTEs   ASCII signature string "Quadram Quad386XT"
---Toshiba (Satellite Pro 435CDS at least)---
 0Ah  7 BYTEs   signature "TOSHIBA"
 11h    BYTE    ??? (8h)
 12h    BYTE    ??? (E7h) product ID??? (guess)
 13h  3 BYTEs   "JPN"
 ```

Next we call the `set_fs` routine and pass the value of the `es` register to it. The implementation of `set_fs` is pretty simple:

```c
static inline void set_fs(u16 seg)
{
    asm volatile("movw %0,%%fs" : : "rm" (seg));
}
```

This function contains inline assembly which gets the value of the `seg` parameter and puts it into the `fs` register. There are many functions in [boot.h](https://github.com/torvalds/linux/blob/master/arch/x86/boot/boot.h) like `set_fs`, for example `set_gs`, `fs`, `gs` for reading a value in it etc...

At the end of `query_mca` it just copies the table pointed to by `es:bx` to the `boot_params.sys_desc_table`.

The next step is getting [Intel SpeedStep](http://en.wikipedia.org/wiki/SpeedStep) information by calling the `query_ist` function. First of all it checks the CPU level and if it is correct, calls `0x15` for getting info and saves the result to `boot_params`.

The following [query_apm_bios](https://github.com/torvalds/linux/blob/master/arch/x86/boot/apm.c#L21) function gets [Advanced Power Management](http://en.wikipedia.org/wiki/Advanced_Power_Management) information from the BIOS. `query_apm_bios` calls the `0x15` BIOS interruption too, but with `ah` = `0x53` to check `APM` installation. After the `0x15` execution, `query_apm_bios` functions check the `PM` signature (it must be `0x504d`), carry flag (it must be 0 if `APM` supported) and value of the `cx` register (if it's 0x02, protected mode interface is supported).

Next it calls `0x15` again, but with `ax = 0x5304` for disconnecting the `APM` interface and connecting the 32-bit protected mode interface. In the end it fills `boot_params.apm_bios_info` with values obtained from the BIOS.

Note that `query_apm_bios` will be executed only if `CONFIG_APM` or `CONFIG_APM_MODULE` was set in the configuration file:

```C
#if defined(CONFIG_APM) || defined(CONFIG_APM_MODULE)
    query_apm_bios();
#endif
```

The last is the [`query_edd`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/edd.c#L122) function, which queries `Enhanced Disk Drive` information from the BIOS. Let's look into the `query_edd` implementation.

First of all it reads the [edd](https://github.com/torvalds/linux/blob/master/Documentation/kernel-parameters.txt#L1023) option from the kernel's command line and if it was set to `off` then `query_edd` just returns.

If EDD is enabled, `query_edd` goes over BIOS-supported hard disks and queries EDD information in the following loop:

```C
for (devno = 0x80; devno < 0x80+EDD_MBR_SIG_MAX; devno++) {
    if (!get_edd_info(devno, &ei) && boot_params.eddbuf_entries < EDDMAXNR) {
        memcpy(edp, &ei, sizeof ei);
        edp++;
        boot_params.eddbuf_entries++;
    }
    ...
    ...
    ...
    }
```

where `0x80` is the first hard drive and the value of `EDD_MBR_SIG_MAX` macro is 16. It collects data into the array of [edd_info](https://github.com/torvalds/linux/blob/master/include/uapi/linux/edd.h#L172) structures. `get_edd_info` checks that EDD is present by invoking the `0x13` interrupt with `ah` as `0x41` and if EDD is present, `get_edd_info` again calls the `0x13` interrupt, but with `ah` as `0x48` and `si` containing the address of the buffer where EDD information will be stored.

Conclusion
--------------------------------------------------------------------------------

This is the end of the second part about Linux kernel insides. In the next part we will see video mode setting and the rest of preparations before transition to protected mode and directly transitioning into it.

If you have any questions or suggestions write me a comment or ping me at [twitter](https://twitter.com/0xAX).

**Please note that English is not my first language, And I am really sorry for any inconvenience. If you find any mistakes please send me a PR to [linux-insides](https://github.com/0xAX/linux-internals).**

Links
--------------------------------------------------------------------------------

* [Protected mode](http://en.wikipedia.org/wiki/Protected_mode)
* [Protected mode](http://wiki.osdev.org/Protected_Mode)
* [Long mode](http://en.wikipedia.org/wiki/Long_mode)
* [Nice explanation of CPU Modes with code](http://www.codeproject.com/Articles/45788/The-Real-Protected-Long-mode-assembly-tutorial-for)
* [How to Use Expand Down Segments on Intel 386 and Later CPUs](http://www.sudleyplace.com/dpmione/expanddown.html)
* [earlyprintk documentation](http://lxr.free-electrons.com/source/Documentation/x86/earlyprintk.txt)
* [Kernel Parameters](https://github.com/torvalds/linux/blob/master/Documentation/kernel-parameters.txt)
* [Serial console](https://github.com/torvalds/linux/blob/master/Documentation/serial-console.txt)
* [Intel SpeedStep](http://en.wikipedia.org/wiki/SpeedStep)
* [APM](https://en.wikipedia.org/wiki/Advanced_Power_Management)
* [EDD specification](http://www.t13.org/documents/UploadedDocuments/docs2004/d1572r3-EDD3.pdf)
* [TLDP documentation for Linux Boot Process](http://www.tldp.org/HOWTO/Linux-i386-Boot-Code-HOWTO/setup.html) (old)
* [Previous Part](linux-bootstrap-1.md)
