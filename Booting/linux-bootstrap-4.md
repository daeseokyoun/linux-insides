커널 부팅 과정. part 4.
================================================================================

64 비트 모드로 전환
--------------------------------------------------------------------------------

`커널 부팅 과정` 의 4 번째 파트이고 우리는 [protected mode](http://en.wikipedia.org/wiki/Protected_mode) 모드에서 첫 단계에 진입을 했고, [long mode](http://en.wikipedia.org/wiki/Long_mode), [SSE](http://en.wikipedia.org/wiki/Streaming_SIMD_Extensions), [paging](http://en.wikipedia.org/wiki/Paging) 를 CPU 에서 지원하는지 확인하고, 페이지 테이블을 초기화 한다. 그 다음에 마지막으로 [long mode](https://en.wikipedia.org/wiki/Long_mode) 전환을 살펴 보자.

**NOTE: 이 파트에서 어셈블리 코드가 더 많이 보일 것이고 만약 당신이 어셈블리에 익숙치 않다면 관련된 책을 추천 받아 보는 것을 권장한다.**

이전 [파트](https://github.com/daeseokyoun/linux-insides/blob/master/Booting/linux-bootstrap-3.md) 에서 우리는 [arch/x86/boot/pmjump.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/pmjump.S)에 32 비트 엔트리 포인트로 점프하는 것까지 진행했다.:

```assembly
jmpl	*%eax
```

당신은 32 비트 엔트리 포인트의 주소를 갖고 있는 `eax` 레지스터를 다시 보게 될 것이다. 우리는 이것에 관해 [linux kernel x86 boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt)에서 확인 할 수 있다.:

```
When using bzImage, the protected-mode kernel was relocated to 0x100000
bzImage 를 사용할 때, 보호 모드 커널은 0x100000 로 재배치 된다.
```

32 비트 엔트리 포인트에 있는 레지스터 값을 확인함으로써 이 얘기가 진짜인지 확인해보자.:

```
eax            0x100000	1048576
ecx            0x0	    0
edx            0x0	    0
ebx            0x0	    0
esp            0x1ff5c	0x1ff5c
ebp            0x0	    0x0
esi            0x14470	83056
edi            0x0	    0
eip            0x100000	0x100000
eflags         0x46	    [ PF ZF ]
cs             0x10	16
ss             0x18	24
ds             0x18	24
es             0x18	24
fs             0x18	24
gs             0x18	24
```

우리는 여기서 `cs` 레지스터가 `0x10` (이전 파트에서 기억을 한다면, GDT(Global Descriptor Table) 에서 두 번째 인덱스이다.) 갖고 있고, `eip` 레지스터는 `0x100000` 그리고 코드 세그먼트를 포함한 모든 세그먼트의 베이스 주소는 0 이다. 이상태로 물리주소를 얻을 수 있는데, 그것은 부트 프로토콜에 따르면 `0:0x100000` 이거나 그냥 `0x100000` 가 될 것이다. 이제는 32 비트 엔트리 포인트에서 시작할 수 있다.

32 비트 엔트리 포인트(32-bit entry point)
--------------------------------------------------------------------------------

우리는 [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_64.S) 어셈블리 소스 코드 파일에서 32 비트 엔트리 포인트의 정의를 찾을 수 있다:

```assembly
	__HEAD
	.code32
ENTRY(startup_32)
....
....
....
ENDPROC(startup_32)
```

왜 `compressed` 디렉토리 하위일까? 실제 bzImage 는 gzip 으로 `vmlinux + 헤더 + 커널 설정 코드` 압축된 파일이다. 우리는 이전 파트에서 커널 설정코드에 대한 것을 보았다. 그래서, `head_64.S` 의 주요 목표는 long 모드로 진입하기 위한 준비를 하고, long 모드로 진입한다. 그리고 커널 압축을 해제한다. 우리는 이 파트에서 커널의 압축해제 단계를 살펴 볼 것이다.

`arch/x86/boot/compressed` 디렉토리에 2개의 파일이 있다:

* [head_32.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_32.S)
* [head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_64.S)

그러나 우리는 `head_64.S` 만 볼 것이다, 이유는, 기억할지 모르겠지만, 이 책은 `x86_64` 아키텍처를 위한 것이다.; `head_32.S` 는 살펴보지 않을 것이다. [arch/x86/boot/compressed/Makefile](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/Makefile)을 보자. 거기에는 타겟 지정을 볼 수 있다.:

```Makefile
vmlinux-objs-y := $(obj)/vmlinux.lds $(obj)/head_$(BITS).o $(obj)/misc.o \
	$(obj)/string.o $(obj)/cmdline.o \
	$(obj)/piggy.o $(obj)/cpuflags.o
```

`$(obj)/head_$(BITS).o` 를 보면, `$(BITS)` 의 설정의 무엇이냐에 따라 파일을 선택 빌드 하게되어 있다. (head_32.o 또는 head_64.o). `$(BITS)` [arch/x86/Makefile](https://github.com/torvalds/linux/blob/master/arch/x86/Makefile) 에서 .config 파일에 설정된 내용 기준으로 값을 설정한다.:

```Makefile
ifeq ($(CONFIG_X86_32),y)
        BITS := 32
        ...
        ...
else
        BITS := 64
        ...
        ...
endif
```

이제 우리가 어디서 시작해야 하는지 알았다, 그럼 확인 해보자.

필요하다면, 세그먼트 리로드
--------------------------------------------------------------------------------

위에서 얘기한 것처럼, [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_64.S) 어셈블리 코드 파일에서 시작할 것이다. 첫 째로 우리는 `startup_32` 정의 전에 있는 특별한 섹션 속성을 알아 봐야 한다.:

```assembly
    __HEAD
	.code32
ENTRY(startup_32)
```

`__HEAD` 는 [include/linux/init.h](https://github.com/torvalds/linux/blob/master/include/linux/init.h) 헤더 파일에 정의되어 있는 매크로 이며 다음 섹션의 정의를 확장한다.:

```C
#define __HEAD		.section	".head.text","ax"
```

`.head.text` 이름과 `ax` 플래크와 함께 정의되어 있다. 우리의 경우, 이 플래그들은 [executable](https://en.wikipedia.org/wiki/Executable) 이거나 다른 말로 코드를 포함한다는 의미이다. 우리는 이 섹션의 정의를 [arch/x86/boot/compressed/vmlinux.lds.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/vmlinux.lds.S) 링커 스크립트에서 찾아볼 수 있다.:

```
SECTIONS
{
	. = 0;
	.head.text : {
		_head = . ;
		HEAD_TEXT
		_ehead = . ;
	}
```

만약 `GNU LD` 링커 스크립트 언어의 문법에 익숙하지 않다면, 당신은 [documentation](https://sourceware.org/binutils/docs/ld/Scripts.html#Scripts) 를 읽어보면 좋을 것이다. 짧게 설명하면, `.` 심볼은 링커의 특별한 변수이다(위치 카운터-location counter). 여기에 값을 할당하는 것은 위치 카운터(location counter)가 이동하도록 할 것이다. 여기서는 위치 카운터에 0을 할당할 것이다. 이것은 우리의 코드는 메모리 오프셋 `0` 에서 수행되기 위해 링크된다는 의미이다. 게다가, 우리는 주석에 이와 같은 정보도 확인할 수 있다.:

```
Be careful parts of head_64.S assume startup_32 is at address 0.
head_64.S 에서 주의해서 볼 부분은 startup_32 가 주소 0에 있다고 가정한 것이다.
```

좋다, 이제 우리는 어디에 있는지 그리고 `startup_32` 함수를 들여다 보기에 적절한 시점에 왔다.

`startup_32` 함수의 초기에, 우리는 [flags](https://en.wikipedia.org/wiki/FLAGS_register) 레지스터에서 `DF` 비트를 클리어 하는 `cld` 명령어를 볼 수 있다. `DF`(direction flag)가 클리어 되면, 모든 문자열/배열 관련된 명령들이, 예를 들면 [stos](http://x86.renejeschke.de/html/file_module_x86_id_306.html), [scas](http://x86.renejeschke.de/html/file_module_x86_id_287.html) 나 다른 명령어들이 `esi` 나 `edi` 를 증가시켜 준다. 우리는 방향 플래그(DF) 를 클리어 해줄 필요가 있다. 이유는 나중에 페이지 테이블을 위해 공간을 클리어 하기 위한 문자열 명령수행을 사용할 것이기 때문이다.

우리가 `DF` 비트를 클리어 하고 나면, 다음 단계는 커널 설정 헤더 항목에서 `loadflags` 로 부터 `KEEP_SEGMENTS` 플래그를 확인하는 것이다. 당신은 이 책의 첫 번째 [파트](https://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-1.html)에서 `loadflags` 가 기억날 것이다. 우리는 힙의 사용 가능 여부를 위해 `CAN_USE_HEAP` 플래그를 확인했다. 이제 `KEEP_SEGMENTS` 플래그를 확인할 필요가 있다. 이 플래그는 리눅스 [boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt) 문서에 아래와 같이 설명한다:

```
Bit 6 (write): KEEP_SEGMENTS
  Protocol: 2.07+
  - If 0, reload the segment registers in the 32bit entry point.
  - If 1, do not reload the segment registers in the 32bit entry point.
    Assume that %cs %ds %ss %es are all set to flat segments with
	a base of 0 (or the equivalent for their environment).

비트 6 (쓰기): KEEP_SEGMENTS
  프로토콜: 2.07+
	- 만약 0 이면, 32 비트 엔트리 포인트에 세그먼트 레지스터를 리로드.
	- 만약 1 이면, 32 비트 엔트리 포인트에 세그먼트 레지스터를 로드 하지 않음.
	  %cs %ds %ss %es 모두 0 의 베이스와 함께 flat 세그먼트로 설정되어 있다는 가정
```

그래서, 만약 `loadflags`에서 `KEEP_SEGMENTS` 비트가 설정되어 있지않다면, 우리는 `ds`, `ss` 그리고 `es` 세그먼트 레지스터를 베이스 `0` 과 함께 flat 세그먼트로 리셋을 할 필요가 있다. 다음과 같이 하면 된다.:

```C
	testb $(1 << 6), BP_loadflags(%esi)
	jnz 1f

	cli
	movl	$(__BOOT_DS), %eax
	movl	%eax, %ds
	movl	%eax, %es
	movl	%eax, %ss
```

여기서 `__BOOT_DS` 는 `0x18` 이라는 것을 기억하자.([Global Descriptor Table](https://en.wikipedia.org/wiki/Global_Descriptor_Table) 에서 데이터 세그먼트의 인덱스). 만약 `KEEP_SEGMENTS` 이 셋되어 있다면, 우리는 가장 가까이에 있는 `1f` 라벨로 점프하거나, 셋되어 있지 않다면 `__BOOT_DS` 와 함께 세그먼트 레지스터를 업데이트 한다. 그것은 꽤 쉬운 작업이지만 여기에 흥미로운 한가지가 있다. 만약 당신이 이전 [파트](https://github.com/0xAX/linux-insides/blob/master/Booting/linux-bootstrap-3.md)를 읽었다면, 당신은 우리가 이미 [arch/x86/boot/pmjump.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/pmjump.S) 파일 내에서 [보호 모드](https://en.wikipedia.org/wiki/Protected_mode)로 전환하자 마자 이 레지스터들을 업데이트 했다는 것을 기억할 수 있을 것이다. 그런데 왜 다시 이 세그먼트 레지스터들을 신경써줘야 하는가? 그 대답은 간단하다. 리눅스 커널 또한 32 비트 부트 프로토콜을 갖고 준수하고 만약 부트로더도 리눅스 커널을 로드하기 위해 부트 프로토콜을 이용하고 모든 코드는 `startup_32` 이전에 모두 사라질 것이다. `startup_32` 는 부트로더 수행이 끝나고 리눅스 커널의 첫 엔트리 포인트이며, 세그먼트 레지스터의 어떤 내용도 우리가 알고 있는 상태로 있다는 것을 보장 할 수 없기 때문이다.

`KEEP_SEGMENTS` 플래그를 확인하고 세그먼트 레지스터에 알맞은 값을 넣은 다음에는 실행중인 주소 값과 컴파일된 주소의 차이를 계산하여 커널이 로드된 곳을 알아낸다. `setup.ld.S` 는 `.head.text` 의 시작주소를 `. = 0` 로 설정했다는 것을 기억하자. 이것은 이 섹션의 코드는 `0` 주소로 부터 시작된다는 것을 의미한다. 우리는 `objdump` 명령어의 결과로 이것을 볼 수 있다.:

```
arch/x86/boot/compressed/vmlinux:     file format elf64-x86-64


Disassembly of section .head.text:

0000000000000000 <startup_32>:
   0:   fc                      cld
   1:   f6 86 11 02 00 00 40    testb  $0x40,0x211(%rsi)
```

`objdump` 유틸은 `startup_32` 의 주소가 `0` 이라는 것을 알려준다. 그러나 실제로는 그렇지 않다. 우리의 현재 목표는 실제로 어디에 위치하는지 알아보는 것이다. 그것은 꽤나 [long 모드](https://en.wikipedia.org/wiki/Long_mode) 내에서 `rip` 상대 주소를 지원하기 때문에 단순하게 동작한다. 그러나 현재는 [보호 모드](https://en.wikipedia.org/wiki/Protected_mode) 동작 중이라는 것이다. 여기서 `startup_32` 의 주소를 알아내기 위해 일반적인 패턴을 사용할 것이다. 우리는 라벨을 정의하고 이 라벨을 호출하며 스택의 맨 위의 내용을 레지스터에 넣어줘야 한다:

```assembly
call label
label: pop %reg
```

이 레지스터는 라벨의 주소를 가질 것이다. 리눅스 커널에서 `startup_32` 의 주소를 검색해서 비슷한 코드를 찾아보자.:

```assembly
	leal	(BP_scratch+4)(%esi), %esp
	call	1f
1:  popl	%ebp
	subl	$1b, %ebp
```

이전 파트에서 기억한다면, `esi` 레지스터는 보호 모드로 전환되기 전에 채워진 [boot_params](https://github.com/torvalds/linux/blob/master/arch/x86/include/uapi/asm/bootparam.h#L113) 구조체의 주소를 갖고 있다는 것을 알것이다. `boot_params` 구조체는 `0x1e4` 오프셋위치에 `scratch` 라는 특별한 항목을 가진다. 4 바이트의 이 필드는 `call` 명령어를 위한 임시 스택이 될 것이다. 우리는 `scratch` 필드 + 4 바이트의 주소를 얻고 그것을 `esp` 레지스터에 넣어둔다. 우리는 `BP_scratch` 의 베이스에 `4` 바이트를 더하는 이유는, 앞서 설명했듯이, 그것은 임시 스택으로 사용될 것이며, `x86_64` 아키텍처에서는 위에서 아래로 주소가 움직이게 되어 있다. 다음은 내가 위해 말한 패턴을 알아볼 것이다. 우리는 `1f` 라벨을 만들고 이 라벨의 주소를 `esp` 레지스터에 넣어두었다. 왜냐하면 우리는 `call` 명령어 이후에 스택의 맨 위의 주소를 반환할 것이기 때문이다. 그래서 지금은 `1f` 라벨의 주소를 만들어서 `startup_32` 주소를 쉽게 얻을 수 있게 했다. 우리는 단지 라벨의 주소를 스택에서 얻은 주소에서 빼주면 된다.:

```
startup_32 (0x0)     +-----------------------+
                     |                       |
                     |                       |
                     |                       |
                     |                       |
                     |                       |
                     |                       |
                     |                       |
                     |                       |
1f (0x0 + 1f offset) +-----------------------+ %ebp - real physical address
                     |                       |
                     |                       |
                     +-----------------------+
```

`startup_32` 는 `0x0` 주소에서 수행되도록 링킹되었고 이것의 의미는 `1f` 는 `0x0 + 1f 라벨의 오프셋`의 주소를 가진다는 의미이다. 대략적으로 `0x21` 바이트가 된다. `ebp` 레지스터는 `1f` 라벨의 실제 물리 주소를 갖고 있다. 그래서 만약 `ebp`에서 `1f`를 뺀다면 우리는 `startup_32` 의 실제 물리 주소를 얻을 수 있을 것이다. 리눅스 커널 [부트 프로토콜](https://www.kernel.org/doc/Documentation/x86/boot.txt) 은 보호 모드 커널의 베이스 주소는 `0x100000` 라고 기술한다. 우리는 [gdb](https://en.wikipedia.org/wiki/GNU_Debugger) 를 갖고 검증할 수 있다. 디버거를 실행하고 `1f` 주소(`0x100021`)에 브레이크 포인트를 만들자. 만약 이 주소가 맞다면 `ebp` 레지스터에 `0x100021` 이 있는 것을 볼 수 있을 것이다.:

```
$ gdb
(gdb)$ target remote :1234
Remote debugging using :1234
0x0000fff0 in ?? ()
(gdb)$ br *0x100022
Breakpoint 1 at 0x100022
(gdb)$ c
Continuing.

Breakpoint 1, 0x00100022 in ?? ()
(gdb)$ i r
eax            0x18	0x18
ecx            0x0	0x0
edx            0x0	0x0
ebx            0x0	0x0
esp            0x144a8	0x144a8
ebp            0x100021	0x100021
esi            0x142c0	0x142c0
edi            0x0	0x0
eip            0x100022	0x100022
eflags         0x46	[ PF ZF ]
cs             0x10	0x10
ss             0x18	0x18
ds             0x18	0x18
es             0x18	0x18
fs             0x18	0x18
gs             0x18	0x18
```

만약 우리가 다음 명령어를 `subl $1b, %ebp` 한다면, 다음을 볼 수 있다:
```
nexti
...
ebp            0x100000	0x100000
...
```

그렇다. `startup_32` 주소는 `0x100000` 가 맞다. 우리는 `startup_32` 라벨의 주소를 알았고, [long mode](https://en.wikipedia.org/wiki/Long_mode) 로 전환할 준비를 할 수 있다. 다음 목표는 스택을 설정하고 CPU 가 long 모드와 [SSE](http://en.wikipedia.org/wiki/Streaming_SIMD_Extensions)를 확인해 보자.

Stack setup and CPU verification
--------------------------------------------------------------------------------

We could not setup the stack while we did not know the address of the `startup_32` label. We can imagine the stack as an array and the stack pointer register `esp` must point to the end of this array. Of course we can define an array in our code, but we need to know its actual address to configure the stack pointer in a correct way. Let's look at the code:

```assembly
	movl	$boot_stack_end, %eax
	addl	%ebp, %eax
	movl	%eax, %esp
```

The `boot_stack_end` label, defined in the same [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_64.S) assembly source code file and located in the [.bss](https://en.wikipedia.org/wiki/.bss) section:

```assembly
	.bss
	.balign 4
boot_heap:
	.fill BOOT_HEAP_SIZE, 1, 0
boot_stack:
	.fill BOOT_STACK_SIZE, 1, 0
boot_stack_end:
```

First of all, we put the address of `boot_stack_end` into the `eax` register, so the `eax` register contains the address of `boot_stack_end` where it was linked, which is `0x0 + boot_stack_end`. To get the real address of `boot_stack_end`, we need to add the real address of the `startup_32`. As you remember, we have found this address above and put it to the `ebp` register. In the end, the register `eax` will contain real address of the `boot_stack_end` and we just need to put to the stack pointer.

After we have set up the stack, next step is CPU verification. As we are going to execute transition to the `long mode`, we need to check that the CPU supports `long mode` and `SSE`. We will do it by the call of the `verify_cpu` function:

```assembly
	call	verify_cpu
	testl	%eax, %eax
	jnz	no_longmode
```

This function defined in the [arch/x86/kernel/verify_cpu.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/verify_cpu.S) assembly file and just contains a couple of calls to the [cpuid](https://en.wikipedia.org/wiki/CPUID) instruction. This instruction is used for getting information about the processor. In our case it checks `long mode` and `SSE` support and returns `0` on success or `1` on fail in the `eax` register.

If the value of the `eax` is not zero, we jump to the `no_longmode` label which just stops the CPU by the call of the `hlt` instruction while no hardware interrupt will not happen:

```assembly
no_longmode:
1:
	hlt
	jmp     1b
```

If the value of the `eax` register is zero, everything is ok and we are able to continue.

Calculate relocation address
--------------------------------------------------------------------------------

The next step is calculating relocation address for decompression if needed. First we need to know what it means for a kernel to be `relocatable`. We already know that the base address of the 32-bit entry point of the Linux kernel is `0x100000`, but that is a 32-bit entry point. The default base address of the Linux kernel is determined by the value of the `CONFIG_PHYSICAL_START` kernel configuration option. Its default value is `0x1000000` or `16 MB`. The main problem here is that if the Linux kernel crashes, a kernel developer must have a `rescue kernel` for [kdump](https://www.kernel.org/doc/Documentation/kdump/kdump.txt) which is configured to load from a different address. The Linux kernel provides special configuration option to solve this problem: `CONFIG_RELOCATABLE`. As we can read in the documentation of the Linux kernel:

```
This builds a kernel image that retains relocation information
so it can be loaded someplace besides the default 1MB.

Note: If CONFIG_RELOCATABLE=y, then the kernel runs from the address
it has been loaded at and the compile time physical address
(CONFIG_PHYSICAL_START) is used as the minimum location.
```

In simple terms this means that the Linux kernel with the same configuration can be booted from different addresses. Technically, this is done by compiling the decompressor as [position independent code](https://en.wikipedia.org/wiki/Position-independent_code). If we look at [arch/x86/boot/compressed/Makefile](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/Makefile), we will see that the decompressor is indeed compiled with the `-fPIC` flag:

```Makefile
KBUILD_CFLAGS += -fno-strict-aliasing -fPIC
```

When we are using position-independent code an address is obtained by adding the address field of the command and the value of the program counter. We can load code which uses such addressing from any address. That's why we had to get the real physical address of `startup_32`. Now let's get back to the Linux kernel code. Our current goal is to calculate an address where we can relocate the kernel for decompression. Calculation of this address depends on `CONFIG_RELOCATABLE` kernel configuration option. Let's look at the code:

```assembly
#ifdef CONFIG_RELOCATABLE
	movl	%ebp, %ebx
	movl	BP_kernel_alignment(%esi), %eax
	decl	%eax
	addl	%eax, %ebx
	notl	%eax
	andl	%eax, %ebx
	cmpl	$LOAD_PHYSICAL_ADDR, %ebx
	jge	1f
#endif
	movl	$LOAD_PHYSICAL_ADDR, %ebx
1:
	addl	$z_extract_offset, %ebx
```

Remember that the value of the `ebp` register is the physical address of the `startup_32` label. If the `CONFIG_RELOCATABLE` kernel configuration option is enabled during kernel configuration, we put this address in the `ebx` register, align it to a multiple of `2MB` and compare it with the `LOAD_PHYSICAL_ADDR` value. The `LOAD_PHYSICAL_ADDR` macro is defined in the [arch/x86/include/asm/boot.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/boot.h) header file and it looks like this:

```C
#define LOAD_PHYSICAL_ADDR ((CONFIG_PHYSICAL_START \
				+ (CONFIG_PHYSICAL_ALIGN - 1)) \
				& ~(CONFIG_PHYSICAL_ALIGN - 1))
```

As we can see it just expands to the aligned `CONFIG_PHYSICAL_ALIGN` value which represents the physical address of where to load the kernel. After comparison of the `LOAD_PHYSICAL_ADDR` and value of the `ebx` register, we add the offset from the `startup_32` where to decompress the compressed kernel image. If the `CONFIG_RELOCATABLE` option is not enabled during kernel configuration, we just put the default address where to load kernel and add `z_extract_offset` to it.

After all of these calculations we will have `ebp` which contains the address where we loaded it and `ebx` set to the address of where kernel will be moved after decompression.

Preparation before entering long mode
--------------------------------------------------------------------------------

When we have the base address where we will relocate the compressed kernel image, we need to do one last step before we can transition to 64-bit mode. First we need to update the [Global Descriptor Table](https://en.wikipedia.org/wiki/Global_Descriptor_Table):

```assembly
	leal	gdt(%ebp), %eax
	movl	%eax, gdt+2(%ebp)
	lgdt	gdt(%ebp)
```

Here we put the base address from `ebp` register with `gdt` offset into the `eax` register. Next we put this address into `ebp` register with offset `gdt+2` and load the `Global Descriptor Table` with the `lgdt` instruction. To understand the magic with `gdt` offsets we need to look at the definition of the `Global Descriptor Table`. We can find its definition in the same source code [file](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_64.S):

```assembly
	.data
gdt:
	.word	gdt_end - gdt
	.long	gdt
	.word	0
	.quad	0x0000000000000000	/* NULL descriptor */
	.quad	0x00af9a000000ffff	/* __KERNEL_CS */
	.quad	0x00cf92000000ffff	/* __KERNEL_DS */
	.quad	0x0080890000000000	/* TS descriptor */
	.quad   0x0000000000000000	/* TS continued */
gdt_end:
```

We can see that it is located in the `.data` section and contains five descriptors: `null` descriptor, kernel code segment, kernel data segment and two task descriptors. We already loaded the `Global Descriptor Table` in the previous [part](https://github.com/0xAX/linux-insides/blob/master/Booting/linux-bootstrap-3.md), and now we're doing almost the same here, but descriptors with `CS.L = 1` and `CS.D = 0` for execution in `64` bit mode. As we can see, the definition of the `gdt` starts from two bytes: `gdt_end - gdt` which represents last byte in the `gdt` table or table limit. The next four bytes contains base address of the `gdt`. Remember that the `Global Descriptor Table` is stored in the `48-bits GDTR` which consists of two parts:

* size(16-bit) of global descriptor table;
* address(32-bit) of the global descriptor table.

So, we put address of the `gdt` to the `eax` register and then we put it to the `.long	gdt` or `gdt+2` in our assembly code. From now we have formed structure for the `GDTR` register and can load the `Global Descriptor Table` with the `lgtd` instruction.

After we have loaded the `Global Descriptor Table`, we must enable [PAE](http://en.wikipedia.org/wiki/Physical_Address_Extension) mode by putting the value of the `cr4` register into `eax`, setting 5 bit in it and loading it again into `cr4`:

```assembly
	movl	%cr4, %eax
	orl	$X86_CR4_PAE, %eax
	movl	%eax, %cr4
```

Now we are almost finished with all preparations before we can move into 64-bit mode. The last step is to build page tables, but before that, here is some information about long mode.

Long mode
--------------------------------------------------------------------------------

[Long mode](https://en.wikipedia.org/wiki/Long_mode) is the native mode for [x86_64](https://en.wikipedia.org/wiki/X86-64) processors. First let's look at some differences between `x86_64` and the `x86`.

The `64-bit` mode provides features such as:

* New 8 general purpose registers from `r8` to `r15` + all general purpose registers are 64-bit now;
* 64-bit instruction pointer - `RIP`;
* New operating mode - Long mode;
* 64-Bit Addresses and Operands;
* RIP Relative Addressing (we will see an example of it in the next parts).

Long mode is an extension of legacy protected mode. It consists of two sub-modes:

* 64-bit mode;
* compatibility mode.

To switch into `64-bit` mode we need to do following things:

* Enable [PAE](https://en.wikipedia.org/wiki/Physical_Address_Extension);
* Build page tables and load the address of the top level page table into the `cr3` register;
* Enable `EFER.LME`;
* Enable paging.

We already enabled `PAE` by setting the `PAE` bit in the `cr4` control register. Our next goal is to build the structure for [paging](https://en.wikipedia.org/wiki/Paging). We will see this in next paragraph.

Early page table initialization
--------------------------------------------------------------------------------

So, we already know that before we can move into `64-bit` mode, we need to build page tables, so, let's look at the building of early `4G` boot page tables.

**NOTE: I will not describe the theory of virtual memory here. If you need to know more about it, see links at the end of this part.**

The Linux kernel uses `4-level` paging, and we generally build 6 page tables:

* One `PML4` or `Page Map Level 4` table with one entry;
* One `PDP` or `Page Directory Pointer` table with four entries;
* Four Page Directory tables with a total of `2048` entries.

Let's look at the implementation of this. First of all we clear the buffer for the page tables in memory. Every table is `4096` bytes, so we need clear `24` kilobyte buffer:

```assembly
	leal	pgtable(%ebx), %edi
	xorl	%eax, %eax
	movl	$((4096*6)/4), %ecx
	rep	stosl
```

We put the address of `pgtable` plus `ebx` (remember that `ebx` contains the address to relocate the kernel for decompression) in the `edi` register, clear the `eax` register and set the `ecx` register to `6144`. The `rep stosl` instruction will write the value of the `eax` to `edi`, increase value of the `edi` register by `4` and decrease the value of the `ecx` register by `1`. This operation will be repeated while the value of the `ecx` register is greater than zero. That's why we put `6144` in `ecx`.

`pgtable` is defined at the end of [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_64.S) assembly file and is:

```assembly
	.section ".pgtable","a",@nobits
	.balign 4096
pgtable:
	.fill 6*4096, 1, 0
```

As we can see, it is located in the `.pgtable` section and its size is `24` kilobytes.

After we have got buffer for the `pgtable` structure, we can start to build the top level page table - `PML4` - with:

```assembly
	leal	pgtable + 0(%ebx), %edi
	leal	0x1007 (%edi), %eax
	movl	%eax, 0(%edi)
```

Here again, we put the address of the `pgtable` relative to `ebx` or in other words relative to address of the `startup_32` to the `edi` register. Next we put this address with offset `0x1007` in the `eax` register. The `0x1007` is `4096` bytes which is the size of the `PML4` plus `7`. The `7` here represents flags of the `PML4` entry. In our case, these flags are `PRESENT+RW+USER`. In the end we just write first the address of the first `PDP` entry to the `PML4`.

In the next step we will build four `Page Directory` entries in the `Page Directory Pointer` table with the same `PRESENT+RW+USE` flags:

```assembly
	leal	pgtable + 0x1000(%ebx), %edi
	leal	0x1007(%edi), %eax
	movl	$4, %ecx
1:  movl	%eax, 0x00(%edi)
	addl	$0x00001000, %eax
	addl	$8, %edi
	decl	%ecx
	jnz	1b
```

We put the base address of the page directory pointer which is `4096` or `0x1000` offset from the `pgtable` table in `edi` and the address of the first page directory pointer entry in `eax` register. Put `4` in the `ecx` register, it will be a counter in the following loop and write the address of the first page directory pointer table entry to the `edi` register. After this `edi` will contain the address of the first page directory pointer entry with flags `0x7`. Next we just calculate the address of following page directory pointer entries where each entry is `8` bytes, and write their addresses to `eax`. The last step of building paging structure is the building of the `2048` page table entries with `2-MByte` pages:

```assembly
	leal	pgtable + 0x2000(%ebx), %edi
	movl	$0x00000183, %eax
	movl	$2048, %ecx
1:  movl	%eax, 0(%edi)
	addl	$0x00200000, %eax
	addl	$8, %edi
	decl	%ecx
	jnz	1b
```

Here we do almost the same as in the previous example, all entries will be with flags - `$0x00000183` - `PRESENT + WRITE + MBZ`. In the end we will have `2048` pages with `2-MByte` page or:

```python
>>> 2048 * 0x00200000
4294967296
```

`4G` page table. We just finished to build our early page table structure which maps `4` gigabytes of memory and now we can put the address of the high-level page table - `PML4` - in `cr3` control register:

```assembly
	leal	pgtable(%ebx), %eax
	movl	%eax, %cr3
```

That's all. All preparation are finished and now we can see transition to the long mode.

Transition to the 64-bit mode
--------------------------------------------------------------------------------

First of all we need to set the `EFER.LME` flag in the [MSR](http://en.wikipedia.org/wiki/Model-specific_register) to `0xC0000080`:

```assembly
	movl	$MSR_EFER, %ecx
	rdmsr
	btsl	$_EFER_LME, %eax
	wrmsr
```

Here we put the `MSR_EFER` flag (which is defined in [arch/x86/include/uapi/asm/msr-index.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/uapi/asm/msr-index.h#L7)) in the `ecx` register and call `rdmsr` instruction which reads the [MSR](http://en.wikipedia.org/wiki/Model-specific_register) register. After `rdmsr` executes, we will have the resulting data in `edx:eax` which depends on the `ecx` value. We check the `EFER_LME` bit with the `btsl` instruction and write data from `eax` to the `MSR` register with the `wrmsr` instruction.

In the next step we push the address of the kernel segment code to the stack (we defined it in the GDT) and put the address of the `startup_64` routine in `eax`.

```assembly
	pushl	$__KERNEL_CS
	leal	startup_64(%ebp), %eax
```

After this we push this address to the stack and enable paging by setting `PG` and `PE` bits in the `cr0` register:

```assembly
	movl	$(X86_CR0_PG | X86_CR0_PE), %eax
	movl	%eax, %cr0
```

and execute:

```assembly
lret
```

instruction. Remember that we pushed the address of the `startup_64` function to the stack in the previous step, and after the `lret` instruction, the CPU extracts the address of it and jumps there.

After all of these steps we're finally in 64-bit mode:

```assembly
	.code64
	.org 0x200
ENTRY(startup_64)
....
....
....
```

That's all!

Conclusion
--------------------------------------------------------------------------------

This is the end of the fourth part linux kernel booting process. If you have questions or suggestions, ping me in twitter [0xAX](https://twitter.com/0xAX), drop me [email](anotherworldofworld@gmail.com) or just create an [issue](https://github.com/0xAX/linux-insides/issues/new).

In the next part we will see kernel decompression and many more.

**Please note that English is not my first language and I am really sorry for any inconvenience. If you find any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-internals).**

Links
--------------------------------------------------------------------------------

* [Protected mode](http://en.wikipedia.org/wiki/Protected_mode)
* [Intel® 64 and IA-32 Architectures Software Developer’s Manual 3A](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html)
* [GNU linker](http://www.eecs.umich.edu/courses/eecs373/readings/Linker.pdf)
* [SSE](http://en.wikipedia.org/wiki/Streaming_SIMD_Extensions)
* [Paging](http://en.wikipedia.org/wiki/Paging)
* [Model specific register](http://en.wikipedia.org/wiki/Model-specific_register)
* [.fill instruction](http://www.chemie.fu-berlin.de/chemnet/use/info/gas/gas_7.html)
* [Previous part](https://github.com/0xAX/linux-insides/blob/master/Booting/linux-bootstrap-3.md)
* [Paging on osdev.org](http://wiki.osdev.org/Paging)
* [Paging Systems](https://www.cs.rutgers.edu/~pxk/416/notes/09a-paging.html)
* [x86 Paging Tutorial](http://www.cirosantilli.com/x86-paging/)
* [링커 스크립트-한글](http://korea.gnu.org/manual/release/ld/ld-sjp/ld-ko_3.html)
* [CLD/SLD](http://blog.naver.com/PostView.nhn?blogId=krquddnr37&logNo=20193347417)
