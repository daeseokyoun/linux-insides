Kernel initialization. Part 5.
================================================================================

Continue of architecture-specific initialization
================================================================================

이전 [파트](https://github.com/daeseokyoun/linux-insides/blob/master/Initialization/linux-initialization-4.md) 에서, 우리는 [setup_arch](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/setup.c#L856) 함수에서 아키텍처 의존적인 것들을 초기화하는 과정에서 멈추었고 그것을 이 파트에서 계속진행하려 한다. [initrd](http://en.wikipedia.org/wiki/Initrd) 를 위해 메모리 블럭을 예약했고, 다음 단계는 [One Laptop Per Child support](http://wiki.laptop.org/go/OFW_FAQ) 를 검출하는 `olpc_ofw_detect` 이다. 이 책에서는 플랫폼과 관련된 것은 다루지 않을 것이고 그것과 관련된 함수들은 넘어갈 것이다. 계속 진행해보자. 다음 단계는 `early_trap_init` 함수이다. 이 함수는 디버그와(`#DB` rflags 의 `TF` 플래그를 셋할 때, 올라온다) `int3`(`#BP`) 인터럽트 게이트를 초기화한다. 만약 인터럽트에 대해 아무것도 모른다면, 너는 [초기 인터럽트와 Exception](https://github.com/daeseokyoun/linux-insides/blob/master/Initialization/linux-initialization-2.md) 을 읽어보도록 하자. `x86` 아키텍처에서 `INT`, `INTO` 그리고 `INT3` 는 명시적으로 인터럽트 핸들러를 호출하도록 허가하는 특별한 명령어이다. `INT3` 명령어는 브레이크포인트(`#BP`) 핸들러를 호출한다. 이것 또한 [이전 파트](https://github.com/daeseokyoun/linux-insides/blob/master/Initialization/linux-initialization-2.md) 에서 인터럽트와 예외 처리를 알아보았다.:

```
----------------------------------------------------------------------------------------------
|Vector|Mnemonic|Description         |Type |Error Code|Source                   |
----------------------------------------------------------------------------------------------
|3     | #BP    |Breakpoint          |Trap |NO        |INT 3                    |
----------------------------------------------------------------------------------------------
```

디버그 인터럽트인 `#DB`는 디버거를 수행하는 주요 함수인 [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/traps.c) 에 구현되어 있는 `early_trap_init` 를 보자. 이 함수는 `#DB` 는 `#BP` 핸들러를 설정하고 [IDT](http://en.wikipedia.org/wiki/Interrupt_descriptor_table) 를 재 로드 한다.:

```C
void __init early_trap_init(void)
{
        set_intr_gate_ist(X86_TRAP_DB, &debug, DEBUG_STACK);
        set_system_intr_gate_ist(X86_TRAP_BP, &int3, DEBUG_STACK);
        load_idt(&idt_descr);
}
```

인터럽트와 관련된 `set_intr_gate` 의 구현을 이전 파트에서 이미 보았다. 여기에 또 다른 두개의 비슷한 함수인 `set_intr_gate_ist` 와 `set_system_intr_gate_ist`가 있다. 이 두개의 함수는 3개의 인자를 받는다.:

* 인터럽트 번호
* 인터럽트/예외 핸들러의 베이스 주소
* 세번째 인자는 `Interrupt Stack Table` 이다.  `IST`  는 `x86_64` 에서 새로운 매커니즘이고 [TSS](http://en.wikipedia.org/wiki/Task_state_segment) 의 일부이다. 모든 활성화된 커널 모드 쓰레드들은 `16` 킬로바이트의 자신만의 커널 스택을 가진다. 반면, 유저 영역의 쓰레드는 이 커널 스택은 비어 있다.

per-thread 스택 이외에, 각 CPU 와 연관된 특별한 몇개의 스택들이 있다. 이 모든 스택은 리눅스 커널 문서인 [Kernel stacks](https://www.kernel.org/doc/Documentation/x86/x86_64/kernel-stacks) 를 보자. `x86_64` 는 넌 마스커블 인터럽트등과 같은 어떤 이벤트를 처리하는 동안 새로운 `특별한` 스택으로 전환할 수 있도록 하는 기능을 제공한다. 그리고 기능의 이름을 `Interrupt Stack Table` 라 한다. 각 CPU 마다 7 개의 `IST` 이상의 엔트리들이 있을 수 있고 모든 엔트리 포인트들은 전용의 스택을 갖고 있다. 우리의 경우에는 `DEBUG_STACK` 이다.

`set_intr_gate_ist` 와 `set_system_intr_gate_ist` 는 단지 하나의 차이점만 있지 기능적으로는 `set_intr_gate` 와 같다. 이 두 함수는 인터럽트 번호확인과 내부적으로 `_set_gate`를 호출한다.:

```C
BUG_ON((unsigned)n > 0xFF);
_set_gate(n, GATE_INTERRUPT, addr, 0, ist, __KERNEL_CS__); // TODO 마지막 언더바 2개
```

`set_intr_gate`도 이와 같은 일을 한다. 하지만, `set_intr_gate` 는 [dpl](http://en.wikipedia.org/wiki/Privilege_level) 0과 함께 `_set_gate` 호출하고, ist - 0, 하지만 `set_intr_gate_ist` 와 `set_system_intr_gate_ist`는 `ist`로 `DEBUG_STACK` 설정하고 `set_system_intr_gate_ist`는 `dpl` 을 가장 낮은 특권인 `0x3` 으로 설정한다. 인터럽트가 발생하고 하드웨어가 디스크립터를 로드할 때, 하드웨어는 자동적으로 IST 값을 기반으로 새로운 스택 포인터로 설정하고, 그 다음 인터럽트를 실행한다. 모든 특별한 커널 스택은 `cpu_init` 함수내에서 설정될 것이다.(우리는 나중에 볼 것이다.)

`idt_descr` 쓰여진 `#DB` 와 `#BP` 게이트는, `ldtr` 명령어를 단지 호출하는 `load_idt` 으로 `IDT` 테이블을 재로드 한다. 이제 인터럽트 핸들러를 봤고, 그것들이 어떻게 동작하는지 이해해보도록 하자. 물론, 여기서 모든 인터럽트 핸들러를 알아볼 수 없다. 그것은 리눅스 커널 소스에서 매우 흥미로운 부분이며, 이 파트에서는 `debug` 핸들어가 어떻게 구현이 되어 있는지 볼 것이고, 다른 인터럽트 핸들러의 구현 방식은 당신이 차차 알아보면 좋을 것 이다.

#DB handler
--------------------------------------------------------------------------------

위에서 봤듯이, 우리는 `#DB` 핸들러의 주소인 `&debug` 를 `set_intr_gate_ist` 함수로 넘긴다. [lxr.free-electrons.com](http://lxr.free-electrons.com/ident) 는 리눅스 커널 소스에서 심볼들을 찾는데 아주 훌륭한 사이트이다. 하지만 안타깝게도 `debug` 핸들러는 찾아볼 수 없을 것이다. `debug` 의 정의는 [arch/x86/include/asm/traps.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/traps.h) 에 있다.:

```C
asmlinkage void debug(void);
```

`debug` 함수는 [assembly](http://en.wikipedia.org/wiki/Assembly_language) 와 함께 작성된 것이라고 알려주는 `asmlinkage` 속성을 볼 수 있다. 다른 핸들러와 마찬가지로 `#DB` 핸들러의 구현은 [arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/entry/entry_64.S) 에 있고, `idtentry` 어셈블리 매크로와 함께 정의되어 있다.:

```assembly
idtentry debug do_debug has_error_code=0 paranoid=1 shift_ist=DEBUG_STACK
```

`idtentry` 는 인터럽트/예외 엔트리 포인트를 정의하는 매크로이다. 이것은 5 개의 인자를 받는다.:

* 인터럽트 엔트리 포인트의 이름
* 인터럽트 핸들러의 이름
* 인터럽트 에러 코드가 있는지에 대한 여부
* paranoid  - 만약 1이면, 특별한 스택으로 전환 (위에서 다시 보자)
* shift_ist - 인터럽트 동안에 스택 전환

이제 `idtentry` 매크로 구현을 살펴보자. 이 매크로는 entry_64.S 파일에 정의되어 있고, `debug` 함수를 `ENTRY` 매크로를 이용해서 정의한다. `idtentry` 는 스페셜 스택으로 전환할 필요한 경우에 주어진 인자들이 알맞은 것인지 확인한다. 다음 단계로는 인터럽트가 에러 코드를 반환하는지 확인한다. 만약 인터럽트가 에러코드를 반환하지 않는다면(우리의 경우에 `#DB` 핸들러는 에러코드를 반환하지 않는다), `INTR_FRAME` 호출하고, 만약 인터럽트가 에러 코드를 가진다면 `XCPT_FRAME` 를 호출한다. `XCPT_FRAME` 와 `INTR_FRAME` 매크로는 아무것도 하지 않고 인터럽트를 위해 초기 프레임 상태를 만들기 위해 필요한 것이다. 그것들은 `CFI`(Call Frame Infomation) 지시문을 디버깅을 위해 사용한다.  [CFI directives](https://sourceware.org/binutils/docs/as/CFI-directives.html) 에서 자세한 사항을 볼 수 있을 것이다. [arch/x86/kernel/entry_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/entry/entry_64.S) 에 관련 커멘트를 보면, `CFI 매크로는 더 나은 백트레이스를 위해 dwarf2 unwind 정보를 생성할 때 사용된다. 어떤 코드도 변경하지 않는다.` 그래서 우리는 이것들을 무시할 것이다.

```assembly
.macro idtentry sym do_sym has_error_code:req paranoid=0 shift_ist=-1
ENTRY(\sym)
	/* Sanity check */
	.if \shift_ist != -1 && \paranoid == 0
	.error "using shift_ist requires paranoid=1"
	.endif

	.if \has_error_code
	XCPT_FRAME
	.else
	INTR_FRAME
	.endif
	...
	...
	...
```

당신은 이전 파트에서 인터럽트 발생 후에 초기 인터럽트/예외 핸들링에 관한 내용을 기억할 수 있고, 현재 스택은 아래의 포멧으로 구성될 것이다.:

```
    +-----------------------+
    |                       |
+40 |         SS            |
+32 |         RSP           |
+24 |        RFLAGS         |
+16 |         CS            |
+8  |         RIP           |
 0  |       Error Code      | <---- rsp
    |                       |
    +-----------------------+
```

`idtentry` 의 구현에서 다음 두개의 매크로를 볼 수 있다:

```assembly
	ASM_CLAC
	PARAVIRT_ADJUST_EXCEPTION_FRAME
```

첫 `ASM_CLAC` 매크로는 `CONFIG_X86_SMAP` 구성 옵션에 의존적이고 보안을 위해 필요하다. 이것에 관해서는 [여기](https://lwn.net/Articles/517475/)를 참고 바란다. 다음 `PARAVIRT_ADJUST_EXCEPTION_FRAME` 매크로는 Xen-type-exceptions 을 핸들링하기 위한 것이다. (커널 초기화 관련인 이 파트에서는 가상화 관련된 내용은 다루지 않을 것이다.)

다음 코드 조각은 만약 인터럽트가 에러 코드를 가지는지 아닌지 확인하고, 스택에 `$-1` 를 넣는다.(-1 은 `x86_64` 에서 `0xffffffffffffffff` 이다.):

```assembly
	.ifeq \has_error_code
	pushq_cfi $-1
	.endif
```

모든 인터럽트의 스택 일관성을 위한 `dummy` 에러코드가 필요한 것이다. 다음 단계는 스택 포인터로 부터 `$ORIG_RAX-R15` 를 빼준다:

```assembly
	subq $ORIG_RAX-R15, %rsp
```

`ORIRG_RAX`, `R15` 등의 매크로들은 [arch/x86/include/asm/calling.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/calling.h) 에 정의되어 있고 `ORIG_RAX-R15` 는 120 바이트이다. 인터럽트 핸들링을 하는 동안 스택에 모든 레지스터의 값을 저장 할 필요가 있고 범용 레스터의 경우 120 바이트를 차지 하기 때문에, 이렇게 하는 것이다. 범용 레지스터를 위한 스택 설정 이후에, 유저 영역에서 인터럽트가 온것인지 아래와 같이 확인한다:

```assembly
testl $3, CS(%rsp)
jnz 1f
```

여기서 `CS` 의 첫 번째와 두 번째의 비트를 확인한다. `CS` 레지스터는 세그먼트 레지스터를 포함하고 첫 두 비트는 `RPL` 를 위한 것이다. 모든 특권 레벨은 정수로 0-3 까지 값이고, 가장 낮은 번호가 가장 높은 특권을 나타낸다. 그래서 만얀 인터럽트가 커널 모드로 부터 왔다면, `save_paranoid` 를 호출하고, 아니라면 라벨 `1`로 점프한다. `save_paranoid` 에서는 스택에 모든 범용 레지스터를 저장하고, 필요하다면 유저 `gs` 에서 커널 `gs` 로 전환한다.:

```assembly
	movl $1,%ebx
	movl $MSR_GS_BASE,%ecx
	rdmsr
	testl %edx,%edx
	js 1f
	SWAPGS
	xorl %ebx,%ebx
1:	ret
```

다음 단계는 `pt_regs` 포인터를 `rdi` 에 넣고, `rsi` 에 있는 에러코드를 저장하고 만약 인터럽트가 발생했거나 호출이 있었다면, - 우리의 경우 [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/traps.c) 에 있는 `do_debug` 이다.  `do_debug` 는 다른 핸들러와 같이 두개의 인자를 받는다:

* pt_regs - 프로세스의 메모리 영역에 저장되어 있는 CPU 레지스터 셋을 갖고 있는 구조체
* error code - 인터럽트 에러코드.

모든 인터럽트 핸들러가 자신의 일을 마무리 했다면, 스택을 복구하고, 유저영역에서의 호출이었다면 다시 복귀하고 `iret` 을 호출하는 `paranoid_exit` 를 호출한다. 이것이 여기서 설명하는 전부이다. 하지만 우리는 다른 챕터에서 인터럽트에 관해 깊게 살펴보도록 하자.

이것은 `#DB` 인터럽트를 위한 `idtentry` 매크로의 일반적인 설명이다. 모든 인터럽트는 idtentry 와 함께 구현된 사항과 비슷할 것이다. `early_trap_init` 이 마무리되면, 다음 함수는 `early_cpu_init` 이다. 이 함수는 [arch/x86/kernel/cpu/common.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/cpu/common.c) 에 정의되어 있고 CPU 와 벤더 정보를 모은다.

초기 ioremap 초기화
--------------------------------------------------------------------------------

다음 단계는 초기 `ioremap` 을 초기화하는 것이다. 일반적으로 장치들과의 통신은 두 가지 방법이 있다.:

* I/O 포트;
* 장치 메모리(Device memory).

우리는 이미 리눅스 커널 부팅 [과정](https://github.com/daeseokyoun/linux-insides/blob/master/Booting/linux-bootstrap-3.md) 에서 첫 번째 방법(`outb/inb` 명령어들)을 보았다. 두 번째 방법은 I/O 물리 주소를 가상 주소에 맵핑해 놓는 것이다. CPU 에 의해 물리 주소가 접근이 될 때, 그것은 I/O 장치의 메모리에 맵핑 될 수 있는 물리적인 램의 영역을 참조 할 것이다. 그래서 `ioremap` 이 장치 메모리를 커널 주소 공간에 맵핑하기 위해 사용된다.

바로 위에 언급했던 함수는 `early_ioremap_init` 이고, 이 함수는 그것이 접근 가능 하도록 I/O 메모리를 커널 주소 공간에 재 맵핑을 해준다. 우리는 `ioremap` 사용 가능하기 전에 임시적으로 I/O 나 메모리 영역을 맵핑 해놓았던 초기화 했던 것을 정리 하기 위해 ioremap 을 초기화 할 필요가 있다. 이 함수의 구현은 [arch/x86/mm/ioremap.c](https://github.com/torvalds/linux/blob/master/arch/x86/mm/ioremap.c) 에 있다. `early_ioremap_init` 함수의 시작을 보면, `pmd_t` 타입(PMD-page middle directory 엔트리를 표현한다. 정의는 `typedef struct { pmdval_t pmd; } pmd_t;` 반면에 `pmdval_t` 는 `unsigned long`이다.)의 포인터인 `pmd`를 선언하고 알맞은 방법으로 `fixmap`에 정렬(aligned)되어 있는지 확인한다.:

```C
pmd_t *pmd;
BUILD_BUG_ON((fix_to_virt(0) + PAGE_SIZE) & ((1 << PMD_SHIFT) - 1));
```

`fixmap` 은 `FIXADDR_START` 에서 `FIXADDR_TOP` 까지 내용을 담은 고정된 가상 주소 맵핑이다. 고정된 가상 주소들은 컴파일 시간에 가상 주소를 알아야 하는 서브 시스템을 위해 필요하다. 이런 확인 후에, `early_ioremap_init`는 [mm/early_ioremap.c](https://github.com/torvalds/linux/blob/master/mm/early_ioremap.c) 에 있는 `early_ioremap_setup` 를 호출한다. `early_ioremap_setup` 은 `unsigned long` 타입의 `slot_virt` 배열에 512 개의 임시 부트 시간(boot-time) 고정 맵핑(fix-mappings)을 채운다.:
```C
for (i = 0; i < FIX_BTMAPS_SLOTS; i++)
    slot_virt[i] = __fix_to_virt(FIX_BTMAP_BEGIN - NR_FIX_BTMAPS*i);
```

`FIX_BTMAP_BEGIN` 을 위한 페이지 미들 디렉토리 엔트리를 얻고, 그 값을 `pmd` 변수에 넣는다. 그리고 부트 타임 페이지 테이블인 `bm_pte` 에 0 을 채우고, 주어진 페이지 미들 디렉토리 엔트리에  페이지 테이블 엔트리를 설정하기 위해 `pmd_populate_kernel`를 호출 한다:

```C
pmd = early_ioremap_pmd(fix_to_virt(FIX_BTMAP_BEGIN));
memset(bm_pte, 0, sizeof(bm_pte));
pmd_populate_kernel(&init_mm, pmd, bm_pte);
```

여기가 전부다. 당신이 약간 당혹스럽긴하지만, 걱정말자. `ioremap` 과 `fixmaps` 에 관해 [리눅스 커널 메모리 관리. Part 2](https://github.com/daeseokyoun/linux-insides/blob/master/mm/linux-mm-2.md) 에서 더 살표 볼 것이다.

루트 장치를 위한 Major / Minor 번호 얻기
--------------------------------------------------------------------------------

초기 `ioremap`이 초기화 되면, 다음 코드를 볼 수 있을 것이다:

```C
ROOT_DEV = old_decode_dev(boot_params.hdr.root_dev);
```

이 코드는 나중에 `do_mount_root` 함수에서 `initrd` 를 마운트 할 루트 장치의 major / minor 번호를 얻어온다. 이 장치의 Major 번호는 장치와 연결된 드라이버를 확인해 준다. Minor 번호는 드라이버에 의해 컨트롤되는 장치를 참조한다. `old_decode_dev`는 `boot_params_structure`에 있는 하나의 필드를 인자로 받는다. x86 리눅스 커널 부트 프로토콜에서 아래와 같이 볼 수 있다:

```
Field name:	root_dev
Type:		modify (optional)
Offset/size:	0x1fc/2
Protocol:	ALL

  The default root device number.  The use of this field is
  deprecated, use the "root=" option on the command line instead.

  기본 루트 장치 번호. 이 필드는 더이상 사용되진 않고, 명령 라인의 "root=" 옵션에서
  값을 넣을 수 있다.
```

이제 `old_decode_dev` 에서 무엇을 하는지 이해해 보자. 실제, 이 함수는 주어진 major 와 minor 번호로 `dev_t`를 만드는 `MKDEV` 를 호출한다. 이 구현은 정말 단순하다.:

```C
static inline dev_t old_decode_dev(u16 val)
{
         return MKDEV((val >> 8) & 255, val & 255);
}
```

`dev_t` 는 major/minor 번호의 쌍을 표현하기 위한 커널 데이터 타입이다. 하지만, 이상한 `old_` 접두사는 무엇인가? 역사적으로 살펴 보면, 장치의 major 와 minor 번호를 관리하는 두 가지 방법이 있다. 첫 번째 방법은 major 와 minor 번호를 위해 2 바이트를 사용하는 것이다. 당신이 이전 코드를 본다면: major 번호를 위해 8 비트, minor 번호를 위해 8 비트를 사용한다. 하지만 이 방법은 major/minor 번호를 각각 256 개만 사용할 수 있다는 것이다. 그래서 16 비트 정수 형을 사용해서 32 비트 정수형에다가 12 비트는 major 번호로 사용하고 20 비트는 minor 번호를 관리하는데 사용한다. `new_decode_dev` 구현을 살펴보자.:

```C
static inline dev_t new_decode_dev(u32 dev)
{
         unsigned major = (dev & 0xfff00) >> 8;
         unsigned minor = (dev & 0xff) | ((dev >> 12) & 0xfff00);
         return MKDEV(major, minor);
}
```

이 계산이 끝나면, 만약 dev 값이 `0xffffffff` 라면, `major` 를 위한 12 비트(`0xfff`) 와 `minor`를 위한 20 비트를 얻을 수 있다. 그래서 `old_decode_dev` 의 수행이 끝나면, 우리는 루트 장치를 위한 major 와 minor 번호를 `ROOT_DEV` 로 할당할 것이다.

메모리 맵 설정
--------------------------------------------------------------------------------

다음 단계에서는 `setup_memory_map` 함수를 호출하여 메모리 맵을 설정한다. 하지만, 그 전에 스크린(현재 열과 행, 비디오 페이지 등-[Video mode initialization and transition to protected mode](http://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-3.html) 여기서 더 자세한 정보를 볼 수 있다.)에 관한 정보, 확장된 디스플레이 정보, 비디오 모드, bootloader_type 등의 설정이 필요하다.:

```C
	screen_info = boot_params.screen_info;
	edid_info = boot_params.edid_info;
	saved_video_mode = boot_params.hdr.vid_mode;
	bootloader_type = boot_params.hdr.type_of_loader;
	if ((bootloader_type >> 4) == 0xe) {
		bootloader_type &= 0xf;
		bootloader_type |= (boot_params.hdr.ext_loader_type+0x10) << 4;
	}
	bootloader_version  = bootloader_type & 0xf;
	bootloader_version |= boot_params.hdr.ext_loader_ver << 4;
```

이 모든 값들은 부팅 시간에 `boot_params` 구조체에 저장된 내용들이다. 이것 이후에, I/O 메모리의 끝을 설정할 필요가 있다. 당신이 알고 있는 리눅스 커널의 주요 목적 중 하나는 자원 관리이다. 그 자원 중 하나는 메모리이다. 이미 알고 있겠지만, 장치들과 통신하는 방법은 장치 포트들과 장치 메모리를 통해서 가능하다. 등록된 자원의 모든 정보는 아래와 같이 접근 가능하다.:

* /proc/ioports - 현재 장치와 입력/출력을 위해 사용되는 등록된 포트 영역의 리스트를 제공한다.
* /proc/iomem   - 각 물리 장치를 위한 시스템의 메모리 맵을 제공한다.

여기서 `/proc/iomem` 에 관해 조금 더 살펴 보자:

```
cat /proc/iomem
00000000-00000fff : reserved
00001000-0009d7ff : System RAM
0009d800-0009ffff : reserved
000a0000-000bffff : PCI Bus 0000:00
000c0000-000cffff : Video ROM
000d0000-000d3fff : PCI Bus 0000:00
000d4000-000d7fff : PCI Bus 0000:00
000d8000-000dbfff : PCI Bus 0000:00
000dc000-000dffff : PCI Bus 0000:00
000e0000-000fffff : reserved
000e0000-000e3fff : PCI Bus 0000:00
000e4000-000e7fff : PCI Bus 0000:00
000f0000-000fffff : System ROM
```

보시다 시피 장치가 갖고 있는 주소의 범위를 16 진수로 보여준다. 리눅스 커널은 일반적으로 어떤 자원을 관리하기 위한 API 를 제공한다. 글로벌 자원(예를 들어, PIC 들 또는 I/O 포트들)들은 연관된 하드웨어 버스 슬롯에 의해 하위 세트로 나누어 질 수 있다. 그리고 이것을 위한 자료 구조인 `resource` 가 있다.:

```C
struct resource {
        resource_size_t start;
        resource_size_t end;
        const char *name;
        unsigned long flags;
        struct resource *parent, *sibling, *child*; // TODO 마지막 별
};
```

시스템 자원들을 트리와 비슷한 서브셋 형태로 추상화로 보여주는 것이다. 이 구조체는 자원이 갖고 있는 `start` 에서 `end` 까지의 주소 영역(`resource_size_t` 는 `x86_64` 에서는 `phys_addr_t` 또는 `u64` 이다.), 자원의 `name`(`/proc/iomem` 출력에서 보여지는 이름들)과 자원의 `flags`([include/linux/ioport.h](https://github.com/torvalds/linux/blob/master/include/linux/ioport.h)에서 모든 자원의 플래그들을 정의되어 있다.)를 제공한다. 이 구조체의 마지막은 3개의 포인터를 선언한다. 이 포인터들은 트리 같은 구조를 만들기 위한 것이다.:

```
+-------------+      +-------------+
|             |      |             |
|    parent   |------|    sibling  |
|             |      |             |
+-------------+      +-------------+
       |
       |
+-------------+
|             |
|    child    |
|             |
+-------------+
```

자원의 모든 서브셋은 루트 범위 자원들을 가진다. `iomem` 를 위해, 아래와 같이 `iomem_resource` 를 정의한다.:

```C
struct resource iomem_resource = {
        .name   = "PCI mem",
        .start  = 0,
        .end    = -1,
        .flags  = IORESOURCE_MEM,
};
EXPORT_SYMBOL(iomem_resource);
```

TODO EXPORT_SYMBOL

`iomem_resource`는 `PCI mem` 이름을 가지는 IO 메모리 주소 범위와 플래그로 `IORESOURCE_MEM` (`0x00000200`) 값을 정의한다. 지금 중요한 것은 `iomem` 의 시작과 끝 주소를 설정한다. 그것은 아래와 같이 될 것이다.:

```C
iomem_resource.end = (1ULL << boot_cpu_data.x86_phys_bits) - 1;
```

1을 `boot_cpu_data.x86_phys_bits` 만큼 쉬프트한다. `boot_cpu_data` 는 `early_cpu_init`의 실행 중에 채워진 `cpuinfo_x86` 구조체이다. `x86_phys_bits` 항목의 이름을 봐서 알겠지만, 그것은 시스템에서 최대 물리 주소의 최대 비트들의 양을 표현한다. 또한 `iomem_resource` 는 `EXPORT_SYMBOL` 매크로에 전달된다. 이 매크로는 주어진 심볼(여기서는 `iomem_resource`)을 동적 로드 가능한 모듈에서 접근이 가능하게 만들거나 동적 링킹을 위해 사용된다.

루트 `iomem` 자원 주소 범위의 시작과 끝을 설정한 뒤에, 메모리 맵의 설정을 할 것이다. `setup_ memory_map` 의 호출로 진행된다.:

```C
void __init setup_memory_map(void)
{
        char *who*; // TODO 마지막 별

        who = x86_init.resources.memory_setup();
        memcpy(&e820_saved, &e820, sizeof(struct e820map));
        printk(KERN_INFO "e820: BIOS-provided physical RAM map:\n");
        e820_print_map(who);
}
```

처음으로 `x86_init.resources.memory_setup`의 호출을 볼 수 있다. `x86_init`은 자원 초기화, PCI 초기화 등, [arch/x86/kernel/x86_init.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/x86_init.c) 에 구현되어 있는 플랫폼 특정 설정을 하는 `x86_init_ops` 구조체이다. 이것은 내용이 너무 많아 여기서는 다루지는 않고 우리가 현재 이 파트에서 관심을 가져야 하는 부분만 살펴보자:

```C
struct x86_init_ops x86_init __initdata = {
	.resources = {
            .probe_roms             = probe_roms,
            .reserve_resources      = reserve_standard_io_resources,
            .memory_setup           = default_machine_specific_memory_setup,
    },
    ...
    ...
    ...
}
```

`memry_setup` 항목은 [부트 시간](https://github.com/daeseokyoun/linux-insides/blob/master/Booting/linux-bootstrap-2.md)에서 모은 [e820](http://en.wikipedia.org/wiki/E820) 개수 엔트리 개수를 얻고, BIOS e820 맵을 검증하고 `e820map` 구조체에 메모리 영역의 값을 채우는 `default_machine_specific_memory_setup` 함수이다. 모든 영역들의 정보가 모아지면, printk로 모든 영역의 정보를 출력한다. `dmesg` 명령어를 통해 관련 내용을 모두 확인 가능하다.:

```
[    0.000000] e820: BIOS-provided physical RAM map:
[    0.000000] BIOS-e820: [mem 0x0000000000000000-0x000000000009d7ff] usable
[    0.000000] BIOS-e820: [mem 0x000000000009d800-0x000000000009ffff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000000e0000-0x00000000000fffff] reserved
[    0.000000] BIOS-e820: [mem 0x0000000000100000-0x00000000be825fff] usable
[    0.000000] BIOS-e820: [mem 0x00000000be826000-0x00000000be82cfff] ACPI NVS
[    0.000000] BIOS-e820: [mem 0x00000000be82d000-0x00000000bf744fff] usable
[    0.000000] BIOS-e820: [mem 0x00000000bf745000-0x00000000bfff4fff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000bfff5000-0x00000000dc041fff] usable
[    0.000000] BIOS-e820: [mem 0x00000000dc042000-0x00000000dc0d2fff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000dc0d3000-0x00000000dc138fff] usable
[    0.000000] BIOS-e820: [mem 0x00000000dc139000-0x00000000dc27dfff] ACPI NVS
[    0.000000] BIOS-e820: [mem 0x00000000dc27e000-0x00000000deffefff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000defff000-0x00000000deffffff] usable
...
...
...
```

Copying of the BIOS Enhanced Disk Device information
--------------------------------------------------------------------------------

The next two steps is parsing of the `setup_data` with `parse_setup_data` function and copying BIOS EDD to the safe place. `setup_data` is a field from the kernel boot header and as we can read from the `x86` boot protocol:

```
Field name:	setup_data
Type:		write (special)
Offset/size:	0x250/8
Protocol:	2.09+

  The 64-bit physical pointer to NULL terminated single linked list of
  struct setup_data. This is used to define a more extensible boot
  parameters passing mechanism.
```

It used for storing setup information for different types as device tree blob, EFI setup data and etc... In the second step we copy BIOS EDD information from the `boot_params` structure that we collected in the [arch/x86/boot/edd.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/edd.c) to the `edd` structure:

```C
static inline void __init copy_edd(void)
{
     memcpy(edd.mbr_signature, boot_params.edd_mbr_sig_buffer,
            sizeof(edd.mbr_signature));
     memcpy(edd.edd_info, boot_params.eddbuf, sizeof(edd.edd_info));
     edd.mbr_signature_nr = boot_params.edd_mbr_sig_buf_entries;
     edd.edd_info_nr = boot_params.eddbuf_entries;
}
```

Memory descriptor initialization
--------------------------------------------------------------------------------

The next step is initialization of the memory descriptor of the init process. As you already can know every process has its own address space. This address space presented with special data structure which called `memory descriptor`. Directly in the linux kernel source code memory descriptor presented with `mm_struct` structure. `mm_struct` contains many different fields related with the process address space as start/end address of the kernel code/data, start/end of the brk, number of memory areas, list of memory areas and etc... This structure defined in the [include/linux/mm_types.h](https://github.com/torvalds/linux/blob/master/include/linux/mm_types.h). As every process has its own memory descriptor, `task_struct` structure contains it in the `mm` and `active_mm` field. And our first `init` process has it too. You can remember that we saw the part of initialization of the init `task_struct` with `INIT_TASK` macro in the previous [part](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-4.html):

```C
#define INIT_TASK(tsk)  \
{
    ...
	...
	...
	.mm = NULL,         \
    .active_mm  = &init_mm, \
	...
}
```

`mm` points to the process address space and `active_mm` points to the active address space if process has no address space such as kernel threads (more about it you can read in the [documentation](https://www.kernel.org/doc/Documentation/vm/active_mm.txt)). Now we fill memory descriptor of the initial process:

```C
	init_mm.start_code = (unsigned long) _text;
	init_mm.end_code = (unsigned long) _etext;
	init_mm.end_data = (unsigned long) _edata;
	init_mm.brk = _brk_end;
```

with the kernel's text, data and brk. `init_mm` is the memory descriptor of the initial process and defined as:

```C
struct mm_struct init_mm = {
    .mm_rb          = RB_ROOT,
    .pgd            = swapper_pg_dir,
    .mm_users       = ATOMIC_INIT(2),
    .mm_count       = ATOMIC_INIT(1),
    .mmap_sem       = __RWSEM_INITIALIZER(init_mm.mmap_sem),
    .page_table_lock =  __SPIN_LOCK_UNLOCKED(init_mm.page_table_lock),
    .mmlist         = LIST_HEAD_INIT(init_mm.mmlist),
    INIT_MM_CONTEXT(init_mm)
};
```

where `mm_rb` is a red-black tree of the virtual memory areas, `pgd` is a pointer to the page global directory, `mm_users` is address space users, `mm_count` is primary usage counter and `mmap_sem` is memory area semaphore. After we setup memory descriptor of the initial process, next step is initialization of the Intel Memory Protection Extensions with `mpx_mm_init`. The next step is initialization of the code/data/bss resources with:

```C
	code_resource.start = __pa_symbol(_text);
	code_resource.end = __pa_symbol(_etext)-1;
	data_resource.start = __pa_symbol(_etext);
	data_resource.end = __pa_symbol(_edata)-1;
	bss_resource.start = __pa_symbol(__bss_start);
	bss_resource.end = __pa_symbol(__bss_stop)-1;
```

We already know a little about `resource` structure (read above). Here we fills code/data/bss resources with their physical addresses. You can see it in the `/proc/iomem`:

```C
00100000-be825fff : System RAM
  01000000-015bb392 : Kernel code
  015bb393-01930c3f : Kernel data
  01a11000-01ac3fff : Kernel bss
```

All of these structures are defined in the [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/setup.c) and look like typical resource initialization:

```C
static struct resource code_resource = {
	.name	= "Kernel code",
	.start	= 0,
	.end	= 0,
	.flags	= IORESOURCE_BUSY | IORESOURCE_MEM
};
```

The last step which we will cover in this part will be `NX` configuration. `NX-bit` or no execute bit is 63-bit in the page directory entry which controls the ability to execute code from all physical pages mapped by the table entry. This bit can only be used/set when the `no-execute` page-protection mechanism is enabled by the setting `EFER.NXE` to 1. In the `x86_configure_nx` function we check that CPU has support of `NX-bit` and it does not disabled. After the check we fill `__supported_pte_mask` depend on it:

```C
void x86_configure_nx(void)
{
        if (cpu_has_nx && !disable_nx)
                __supported_pte_mask |= _PAGE_NX;
        else
                __supported_pte_mask &= ~_PAGE_NX;
}
```

Conclusion
--------------------------------------------------------------------------------

It is the end of the fifth part about linux kernel initialization process. In this part we continued to dive in the `setup_arch` function which makes initialization of architecture-specific stuff. It was long part, but we have not finished with it. As i already wrote, the `setup_arch` is big function, and I am really not sure that we will cover all of it even in the next part. There were some new interesting concepts in this part like `Fix-mapped` addresses, ioremap and etc... Don't worry if they are unclear for you. There is a special part about these concepts - [Linux kernel memory management Part 2.](https://github.com/0xAX/linux-insides/blob/master/mm/linux-mm-2.md). In the next part we will continue with the initialization of the architecture-specific stuff and will see parsing of the early kernel parameters, early dump of the pci devices, direct Media Interface scanning and many many more.

If you have any questions or suggestions write me a comment or ping me at [twitter](https://twitter.com/0xAX).

**Please note that English is not my first language, And I am really sorry for any inconvenience. If you find any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-insides).**

Links
--------------------------------------------------------------------------------

* [mm vs active_mm](https://www.kernel.org/doc/Documentation/vm/active_mm.txt)
* [e820](http://en.wikipedia.org/wiki/E820)
* [Supervisor mode access prevention](https://lwn.net/Articles/517475/)
* [Kernel stacks](https://www.kernel.org/doc/Documentation/x86/x86_64/kernel-stacks)
* [TSS](http://en.wikipedia.org/wiki/Task_state_segment)
* [IDT](http://en.wikipedia.org/wiki/Interrupt_descriptor_table)
* [Memory mapped I/O](http://en.wikipedia.org/wiki/Memory-mapped_I/O)
* [CFI directives](https://sourceware.org/binutils/docs/as/CFI-directives.html)
* [PDF. dwarf4 specification](http://dwarfstd.org/doc/DWARF4.pdf)
* [Call stack](http://en.wikipedia.org/wiki/Call_stack)
* [Previous part](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-4.html)
