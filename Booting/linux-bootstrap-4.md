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

그렇다. `startup_32` 주소는 `0x100000` 가 맞다. 우리는 `startup_32` 라벨의 주소를 알았고, [long mode](https://en.wikipedia.org/wiki/Long_mode) 로 전환할 준비를 할 수 있다. 다음 목표는 스택을 설정하고 CPU 가 long 모드와 [SSE](http://en.wikipedia.org/wiki/Streaming_SIMD_Extensions)를 지원하는지 확인하는 과정을 보자.

스택 설정 및 CPU 검증
--------------------------------------------------------------------------------

우리는 `startup_32` 라벨 주소를 알수 없었던 동안 스택을 설정할 수 없었다. 스택을 배열로 생각하고 스택 포인터 레지스터인 `esp` 는 이 배열의 맨끝을 가리켜야 한다. 물론, 코드 내에서 배열을 정의 할 수 있지만, 우리는 스택 포인터를 설정하기 위해 그것의 실제 주소를 정상적인 방법으로 알아야 할 필요가 있다. 아래의 코드를 보자:

```assembly
	movl	$boot_stack_end, %eax
	addl	%ebp, %eax
	movl	%eax, %esp
```

`boot_stack_end` 라벨은 [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_64.S) 어셈블리 코드 파일에 정의되어 있고, [.bss](https://en.wikipedia.org/wiki/.bss) 에 위치한다.:

```assembly
	.bss
	.balign 4
boot_heap:
	.fill BOOT_HEAP_SIZE, 1, 0
boot_stack:
	.fill BOOT_STACK_SIZE, 1, 0
boot_stack_end:
```

첫 번째로, 우리는 `boot_stack_end` 주소를 `eax` 레지스터에 넣는다. 그러면 `eax` 레지스터는 `0x0 + boot_stack_end` 라는 주소 값을 가진다. `boot_stack_end` 의 실제 주소를 알기 위해서는 `startup_32` 의 실제 주소에 더하면 될 것이다. 기억해보면, 우리는 `startup_32` 주소를 `ebp` 레지스터에 넣었다는 것을 알 수 있다. 결론적으로, `eax` 는 `boot_stack_end` 의 실제 주소를 가질 것이고 우리는 이 주소를 스택 포인터 레지스터에 넣어 둘 필요가 있다.

스택을 설정한 이후 단계는 CPU 검증이다. 우리는 `long 모드` 로 전환중에 있으며, 우리는 CPU 가 `long 모드` 와 `SSE` 의 지원여부를 확인해야 한다. 우리는 `verify_cpu` 함수를 호출하여 이 확인을 할 것이다:

```assembly
	call	verify_cpu
	testl	%eax, %eax
	jnz	no_longmode
```

[arch/x86/kernel/verify_cpu.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/verify_cpu.S) 어셈블리 파일에 `verify_cpu`는 정의되어 있고, 몇 번의 [cpuid](https://en.wikipedia.org/wiki/CPUID) 명령어 관련된 호출을 시도한다. 이 명령어는 프로세서의 정보를 얻기 위해 사용된다. 이 경우에는 `long mode` 와 `SSE` 지원 여부를 확인하는데 `eax`레지스터에 결과를 `0` 이면 성공을, `1` 이면 실패로 저장한다.

만약 `eax` 가 0이 아니면, 우리는 `hlt` 명령어로 호출로 CPU 를 멈추게 하고 무한루프를 수행하는 `no_longmode` 라벨로 점프한다.:

```assembly
no_longmode:
1:
	hlt
	jmp     1b
```

만약 `eax` 레지스터 값이 0이면, 모든 것이 확인 완료되었고 계속해도 좋다는 얘기이다.

재배치 주소 계산
--------------------------------------------------------------------------------

다음 단계는 필요하다면 압축해제한 커널의 재배치 주소를 계산하는 것이다. 먼저, 우리는 커널을 위해 `재배치 가능(relocatable)` 라는 의미를 알아봐야 한다. 우리는 이미 리눅스 커널의 32 비트 베이스 주소를 `0x100000` 으로 알고 있지만 그것은 32비트 엔트리 포인드이다. 리눅스 커널의 기본 베이스 주소는 커널 설정 옵션에서 `CONFIG_PHYSICAL_START` 의 값에 의해 결정된다. 그것의 기본 값은 `0x1000000` 또는 `16MB` 이다. 여기서 문제는 만약 커널이 문제가 발생해서 죽으면(panic), 다른 주소에 로드되도록 설정되어 있는 신뢰성 있는 [kdump](https://www.kernel.org/doc/Documentation/kdump/kdump.txt) 라는 `rescue kernel/dump-capture kernel` 을 통해 부팅 할 수 있도록 한다. 리눅스 커널은 이 문제를 해결하기 위해 특별한 구성 옵션을 제공한다 `CONFIG_RELOCATABLE`. 우리는 리눅스 커널 문서를 통해 이 내용을 알 수 있다.:

```
This builds a kernel image that retains relocation information
so it can be loaded someplace besides the default 1MB.
재배치 정보를 가지고 커널 이미지가 빌드되고 그것은 기본 1MB 주소 이외에 어떤 곳에 로드 될 수 있다.

Note: If CONFIG_RELOCATABLE=y, then the kernel runs from the address
it has been loaded at and the compile time physical address
(CONFIG_PHYSICAL_START) is used as the minimum location.
Note: 만약 CONFIG_RELOCATABLE=y 라면, 커널은 그 주소로 로드되고 수행된다. 그리고 컴파일 시에 적용된
물리주소는(CONFIG_PHYSICAL_START) 최소 위치로써 사용된다.
```

같은 구성 옵션의 리눅스 커널은 서로 다른 주소에서 로드 될 수 있다는 의미이다. 기술적으로 이것은 [위치 독립 코드(position independent code)](https://en.wikipedia.org/wiki/Position-independent_code)로써 압축해제기(decompressor)를 커파일 함으로써 가능해진다. 만약 우리가 [arch/x86/boot/compressed/Makefile](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/Makefile) 파일을 살펴 본다면, 우리는 이 압축해제기(decompressor)는 `-fPIC` 플래그와 함께 컴파일 되었다는 것을 알수 있다:

```Makefile
KBUILD_CFLAGS += -fno-strict-aliasing -fPIC
```

우리가 위치-독립 코드(position independent code) 를 사용 할때, 주소는 프로그램 카운터(Program counter-PC) 의 값과 명령 주소의 필드에 더 해져서 얻을 수 있다. 우리는 어떤 주소에도 사용할 수 있는 코드를 로드 할 것이다. 이것이야 말로 우리가 `startup_32` 의 실제 물리 주소를 얻어야 하는 이유이다. 이제 리눅스 커널 코드로 돌아가보자. 우리의 현재 목표는 압축해제된 커널을 어디로 재배치 할 수 있는지에 대한 주소 계산이다. 이 주소의 계산은 커널 구성 옵션에서 `CONFIG_RELOCATABLE` 의 값에 의존적이다. 아래의 코드를 보자:

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

여기서 `ebp` 레지스터는 `startup_32` 라벨의 물리 주소를 갖고 있다는 것을 기억하자. 만약 `CONFIG_RELOCATABLE` 옵션이 활성화 되어 있다면, 우리는 이 주소를 `ebx`레지스터에 넣을 것이고, 그것을 `2MB` 의 배수로 정렬하고 `LOAD_PHYSICAL_ADDR` 의 값과 비교할 것이다. `LOAD_PHYSICAL_ADDR` 매크로는 [arch/x86/include/asm/boot.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/boot.h)에 정의되어 있고 아래 처럼 구현되어 있다.:

```C
#define LOAD_PHYSICAL_ADDR ((CONFIG_PHYSICAL_START \
				+ (CONFIG_PHYSICAL_ALIGN - 1)) \
				& ~(CONFIG_PHYSICAL_ALIGN - 1))
```

커널이 어디에 로드되는지에 대한 물리 주소의 값을 가지는 정렬된 `CONFIG_PHYSICAL_ALIGN` 값에 확장을 하는 것을 볼 수 잇다. `LOAD_PHYSICAL_ADDR`와 `ebx` 레지스터의 값과 비교후에, 우리는 압축된 커널 이미지를 압축해제하는 곳을 `startup_32`에 오프셋을 더해서 계산할 수 있다. 만약 `CONFIG_RELOCATABLE` 옵션이 활성화 되지 않았다면, 우리는 커널이 로드되어야 하는 고정 주소가 있고 그것과 `z_extract_offset` 을 더해서 사용한다.

이 모든 계산이 끝나면, 우리는 `ebp` 에는 커널이 어디에 로드외어 있는지 주소를 갖고 있고 `ebx` 에는 압축 해제 후에 커널이 어디로 이동을 하는지에 대한 주소를 담고 있다.

long 모드로 진입 전 준비
--------------------------------------------------------------------------------

우리는 압축된 커널이미지를 재배치 할 수 있는 베이스 주소를 가졌다면, 우리는 64 비트 모드로 전환전에 마지막 한가지를 더 처리 해야 한다. 그중 제일 처음은 [Global Descriptor Table-GDT](https://en.wikipedia.org/wiki/Global_Descriptor_Table) 의 업데이트 이다:

```assembly
	leal	gdt(%ebp), %eax
	movl	%eax, gdt+2(%ebp)
	lgdt	gdt(%ebp)
```

`ebp` 레지스터의 기본 주소를 `gdt` 의 오프셋과 함께 `eax` 레지스터에 넣어주는 것이다. 다음은 이 주소를  `ebp` 레지스터 주소에서 `gdt+2` 오프셋에 넣고 `lgdt` 명령어로  `Global Descriptor Table` 를 로드한다. `gdt` 오프셋이라는 것을 이해하기 위해 우리는 `Global Descriptor Table` 의 정의를 살펴봐야 할 것이다. 우리는 [head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_64.S) 어셈블리 소스 코드에서 GDT 의 정의를 볼 수 있다:

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
GDT 는 `.data` 섹션에 위치하고 5개의 디스크립터를 갖고 있다: `null` 디스크립터, 커널 코드 세그먼트, 커널 데이다 세그먼트와 마지막으로 두개의 타스크 디스크립터이다(TS). 이미 이전 [파트](https://github.com/daeseokyoun/linux-insides/blob/master/Booting/linux-bootstrap-3.md) 에서 `Global Descriptor Table`을 로드 했다. 그리고 이제 우리는 거의 같은 것을 하지만, 64 비트 모드에서 실행을 위한 GDT 에서 `CS.L = 1` 과 `CS.D = 0` 를 설정하는 것은 다르다. `gdt` 의 정의는 2 바이트로 부터 시작한다: `gdt` 테이블 또는 테이블 제한 내에 마지막 바이트를 표현하는 `gdt_end - gdt` 이다. 다음 4바이트는 `gdt` 의 베이스 주소를 포함한다. `Global Descriptor Table` 은 두 부분으로 이루어진 `48 비트 GDTR` 에 저장된다는 것을 기억하자:

* global descriptor table 의 크기(16비트);
* global descriptor table 의 주소(32비트)

그래서 우리는 `gdt` 의 주소를 `eax` 레지스터에 넣었고 그다음에 그것을 어셈블리 코드로 `.long gdt` 또는 `gdt+2`로 넣었다. 이제 부터는 `GDTR` 레지스터를 위한 정형화된 구조체를 가졌고 `lgdt` 명령어로 `Global Descriptor Table` 을 로드할 수 있다.

우리가 `Global Descriptor Table` 로드 한 후에, `cr4`  레지스터의 값을 써주줌으로 해서 [PAE](http://en.wikipedia.org/wiki/Physical_Address_Extension) 모드를 활성화 해야 한다:

```assembly
	movl	%cr4, %eax
	orl	$X86_CR4_PAE, %eax
	movl	%eax, %cr4
```

이제 우리는 64 비트 모드로 진입하기 위한 모든 준비를 거의 다 마쳤다. 마지막 단계로 페이제 테이블을 만들고, 그러나 그전에, long 모드에 대한 정보를 보자.

Long mode(롱 모드)
--------------------------------------------------------------------------------

[Long 모드](https://en.wikipedia.org/wiki/Long_mode) 는 [x86_64](https://en.wikipedia.org/wiki/X86-64) 을 위한 네이티브 모드이다. 먼저 `x86_64` 와 `x86`에 대해서 알아보자.

`64 비트` 모드는 아래와 같은 항목을 제공한다:

* `r8` 에서 `r15`까지의 8 개의 새로운 범용 레지스터 + 모든 레지스터들이 64 비트가 된다.;
* 64 비트 명령어 포인터 레지스터(instruction Pointer) - `RIP`;
* 새로운 운영 모드 - Long 모드;
* 64 비트 주소와 피연산자;
* RIP 상대 어드레싱(다음 파트에서 이것의 예제를 볼 것이다.)

Long 모드는 기존 보호보드에서 확장한 것이다. 이것은 아래 두 모드를 포함한다.:

* 64 비트 모드;
* 호완 모드.

`64 비트` 모드로 전환하기 위해서는 아래의 일들이 진행되어야 한다.:

* [PAE](https://en.wikipedia.org/wiki/Physical_Address_Extension) 허용;
* 페이지 테이블을 구성하고 `cr3` 레지스터에 최상위 단계 페이지 테이블 주소를 로드한다.;
* `EFER.LME` 허용;
* 페이징 허용.

`cr4` 컨트롤 레지스터에 `PAE`비트를 셋팅함으로써 `PAE`는 활성화가 이미 되었다. 다음 목표는 [paging](https://en.wikipedia.org/wiki/Paging) 을 위한 구조체를 구성한다. 다음 장을 보자.

초기 페이지 테이블 초기화
--------------------------------------------------------------------------------

그래서, 우리는 `64 비트` 모드로 이동할 수 있고, 페이지 테이블을 구성해야 한다는 것도 알았다. 이제 초기 `4G` 부트 페이지 테이블의 구성방법을 알아보자.

**NOTE: 나는 가상 메모리의 이론에 대해서는 기술 하지 않을 것이다. 만얀 이것에 대해 더 자세히 알고 싶다면 이 파트 맨아래 링크에서 찾아 보길 바란다.**

리눅스 커널은 `4-level` 페이징을 사용한다, 그리고 우리는 일반적으로 아래와 같이 페이지 테이블을 구성해야 한다.:

* 하나의 `PML4` 나 `Page Map Level 4`가 하나의 엔트리를 가짐;
* 하나의 `PDP` 나 `Page Directory Pointer` 테이블은 4개의 엔트리를 가짐;
* 4개의 페이지 디렉터리 테이블은 `2048`개의 엔트리를 가진다

이것의 구현을 살펴보자. 메모리의 페이지 테이블을 위한 버퍼를 클리어 해야 한다. 모든 테이블은 `4096` 바이트이고, 모든 버퍼를 클리어 하려면 `24` 킬로 바이트를 해야 한다.:

```assembly
	leal	pgtable(%ebx), %edi
	xorl	%eax, %eax
	movl	$((4096*6)/4), %ecx
	rep	stosl
```

`ebx`(압축 해제된 커널이 재배치 되는 주소가 있다.)에 `pgtable` 의 주소를 더해서 `edi` 레지스터에 넣어준다. `eax` 레지스터를 클리어 하고, `ecx` 레지스터에 `6144` 값을 넣는다. `rep stosl` 명령어는 `eax` 의 값을 `edi` 에 쓰고, `edi` 레지스터의 값을 `4` 만큼 증가시킨다. 그리고 `ecx` 레지스터의 값은 `1` 감소한다. 이 수행은 `ecx` 레지스터의 값이 0 보다 클때까지 계속 반복적으로 한다. `ecx` 에 `6144` 값을 넣은 이유도 이와 같은 것을 하기 위함이다.

`pgtable` 은 [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_64.S) 어셈블리 마지막 부분에 정의되어있고 아래 처럼 되어 있다:

```assembly
	.section ".pgtable","a",@nobits
	.balign 4096
pgtable:
	.fill 6*4096, 1, 0
```

우리가 이미 살펴 봤듯이, `.pgtable` 섹션에 위치하고 그 크기는 `24` 킬로 바이트이다.

`pgtable` 구조체를 위한 버퍼를 얻은 후에, 우리는 맨 상위 레벨의 페이지 테이블인 `PML4`를 아래와 같이 구성할 수 있다:

```assembly
	leal	pgtable + 0(%ebx), %edi // %edi = %ebx + 0 + pgtable?
	leal	0x1007 (%edi), %eax // %eax = (%edi + 0x1007)
	movl	%eax, 0(%edi) // (%edi + 0) = *(%eax)
```

우리는 `ebx`에서 상대적으로 `pgtable` 주소 만큼 떨어진 주소를 `edi` 레지스터에 넣어준다.(`startup_32` 의 주소에서 `pgtable` 만큼 떨어진) 다음은 `eax` 레지스터에 `edi` 주소에서 `0x1007` 오프셋 이동한 주소를 넣어준다. `0x1007` 은 `PML4` 더하기 7 의 크기인 `4096 + 7` 바이트 이다. 여기서 `7` 은 `PML4` 엔트리의 플래그를 표시한다. 여기서는 이 플래그는 `PRESENT+RW+USER` 이다. 마지막으로 첫 `PDP` 엔트리의 주소값을 `PML4`에 써준다.

다음 단계에서 우리는 `Page Directory Pointer` 테이블에서 같은 `PRESENT+RW+USE` 플래그를 가지고 4 개의 `Page Directory` 엔트리를 구성하는 것을 알아볼 것이다.:

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

우리는 page directory pointer 베이스 주소에서 `pgtable` 테이블 의 `4096` 또는 `0x1000` 오프셋을 더한 주소를 `edi` 에 넣고 그 첫 page directory pointer 엔트리 주소를 `eax` 레지스터에 넣어준다. `ecx` 레지스터에 `4` 를 넣는데, 그것은 다음에 올 루프에서 카운터로 사용될 것이며, `edi` 레지스터에 첫 page directory pointer 테이블의 주소를 쓴다. 그 다음 `edi`는 `0x7`의 플래그를 갖고 첫 page directory pointer 엔트리의 주소를 갖게 될 것이다. 다음 우리는 각 `8` 바이트의 크기를 가지는 다음 엔트리의 주소 값을 계산하여 `edi` 에 넣어준다. 페지징 구조체를 구성하는 마지막 단계는 `2 MB` 의 페이지들을 관리하는 `2048` 개의 페이지 테이블 엔트리를 구성하는 것이다.:

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

여기서 우리는 이전 단계와 거의 비슷한 일을 한다. 모든 엔트리는 `$0x00000183` - `PRESENT + WRITE + MBZ` 플래그를 갖게 될 것이다. 마지막으로 우리는 `2 MB` 크리의 페이지를 `2048` 개를 가질 수 있다. 또는:

```python
>>> 2048 * 0x00200000
4294967296
```

`4G` 페이지 테이블을 갖게 된다. 우리는 `4GB` 의 메모리 맵을 가질 수 있는 초기 페이지 테이블 구조체를 구성했고, 이제는 상위 레벨 페이지 테이블인 `PML4` 의 주소 값을 `cr3` 컨트롤 레지스터에 넣기만 하면 된다.:

```assembly
	leal	pgtable(%ebx), %eax
	movl	%eax, %cr3
```

이제 끝이다. 모든 준비를 마치고 이제 long 모드로 전환하는 것을 살펴 보자.

64 비트 모드로 전환
--------------------------------------------------------------------------------

무엇 보다더 먼저, 우리는 [MSR](http://en.wikipedia.org/wiki/Model-specific_register) 의 `EFER.LME` 플래그를 `0xC0000080` 으로 설정할 필요가 있다.:

```assembly
	movl	$MSR_EFER, %ecx
	rdmsr
	btsl	$_EFER_LME, %eax
	wrmsr
```

여기서 우리는 `MSR_EFER` 플래그를 ([arch/x86/include/uapi/asm/msr-index.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/uapi/asm/msr-index.h#L7) 여기에 정의되어 있다.) `ecx` 레지스터에 넣고 [MSR](http://en.wikipedia.org/wiki/Model-specific_register) 레지스터를 읽는 `rdmsr` 명령어를 호출한다. 다음에 `rdmsr` 을 실행하면, 우리는 MSR 번호(`ecx`) 에 해당하는 레지스터의 값을 `edx:eax` 에 읽어온다. `btsl` 명령어로 `EFER_LME` 비트를 확인하고 `wrmsr` 명령어를 호출하여 `edx:eax` 의 데이터를 `MSR` 레지스터로 써준다.

다음 단계는 커널 코드 세그먼트 주소를 스택(우리는 그것을 GDT 에 정의해두었다.)에 넣어주고 `eax` 에 `startup_64` 루틴의 주소를 넣어준다.

```assembly
	pushl	$__KERNEL_CS
	leal	startup_64(%ebp), %eax // eax = (ebp + startup_64);
```

이 주소를 스택에 넣고 나서 `cr0` 레지스터의 `PG`와 `PE` 비트를 셋팅함으로써 페이징을 활성화한다.:

```assembly
	movl	$(X86_CR0_PG | X86_CR0_PE), %eax
	movl	%eax, %cr0
```

그리고 `lret` 실행한다:

```assembly
lret
```

이전 단계에서 `startup_64` 함수의 주소를 스택에 넣었고, `lret` 명령어 이후에, CPU 는 그것의 주소를 꺼내고 거기로 점프(jump) 한다.

마침내 64 비트 모드로 진입할 수 있게 되었다.:

```assembly
	.code64
	.org 0x200
ENTRY(startup_64)
....
....
....
```

끝!

결론
--------------------------------------------------------------------------------

리눅스 부팅 과정의 4번째 파트의 끝이다. 당신이 어떤 질문이나 제안이 있다면 [twitter](https://twitter.com/0xAX) - 원저자 에게 알려주길 바란다.

다음 파트에서는 커널 압축해제와 다른 많은 것을 볼 것이다.

**나는 영어권의 사람이 아니고 이런 것에 대해 매우 미안해 하고 있다. 만약 어떤 실수를 발견한다면, 나에게 PR을 [linux-insides](https://github.com/0xAX/linux-internals)을 보내줘**

링크
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
* [kdump-한글](https://ko.wikipedia.org/wiki/Kdump)
* [Virtual Memory in IA-64 리눅스 커널](http://www.informit.com/articles/article.aspx?p=29961&seqNum=3)
* [리눅스 페이지 테이블](http://www.iamroot.org/xe/index.php?document_srl=26333&mid=Kernel)
* [IA32e 페이징-iamroot](http://www.iamroot.org/xe/index.php?document_srl=26289&mid=Kernel)
