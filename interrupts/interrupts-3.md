인터럽트와 인터럽트 핸들링. Part 3.
================================================================================

예외 처리
--------------------------------------------------------------------------------

리눅스 커널에서 인터럽트와 예외 처리에 관련된 3 번째 파트이고 이전 [파트](https://github.com/daeseokyoun/linux-insides/blob/korean-trans/interrupts/interrupts-2.md)에서 [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blame/master/arch/x86/kernel/setup.c) 소스 코드 파일에 있는 `setup_arch` 함수에서 마무리되었다.

우리는 이미 이 함수가 아키텍처 의존적인 것들의 초기화는 수행했다는 것을 알고 있다. 우리의 경우에 `setup_arch` 함수는 [x86_64](https://en.wikipedia.org/wiki/X86-64) 아키텍처와 관련된 초기화를 수행한다. `setup_arch` 함수는 매우 큰 함수이고, 이전 파트에서 우리는 아래에 기술되는 두개의 예외 핸들러를 위한 설정에서 마무리되었다.:

* `#DB` - 디버깅 예외, 인터럽트된 프로세스로 부터 디버그 핸들러로 제어권을 넘김
* `#BP` - 중단점(breakpoint) 예외, `int 3` 명령어에 의해 발생한다.

이 예외들은 [kgdb](https://en.wikipedia.org/wiki/KGDB) 를 통해 디버깅의 목적을 위한 초기 예외처리를 하기 위해 `x86_64` 아키텍처에서 허용한다.

기억하겠지만, 이 예외 처리 핸들러들은 `early_trap_init` 함수에서 설정된다.:

```C
void __init early_trap_init(void)
{
        set_intr_gate_ist(X86_TRAP_DB, &debug, DEBUG_STACK);
        set_system_intr_gate_ist(X86_TRAP_BP, &int3, DEBUG_STACK);
        load_idt(&idt_descr);
}
```

이 함수는 [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/tree/master/arch/x86/kernel/traps.c) 에 구현되어 있다. 우리는 이미 이전 파트에서 `set_intr_gate_ist` 와 `set_system_intr_gate_ist` 함수의 구현을 보았고 이제 우리는 이 두 예외 처리 핸들러의 구현을 살펴 볼 것이다.

디버깅과 중단점(Breakpoint) 예외들
--------------------------------------------------------------------------------

좋다, 우리는 `early_trap_init` 함수에서 `#DB` 와 `#BP` 예외처리를 위한 핸들러를 설정했고 이제는 그것들의 구현을 살펴보아야 할 시간이다. 살펴보기 앞서 이 예외 처리의 내용을 조금 더 알아보자.

첫 번째 예외는 - `#DB` 또는 `debug` 예외 - 디버깅 이벤트가 발생했을 때 발생한다. 예를 들어 [디버깅 레지스터](http://en.wikipedia.org/wiki/X86_debug_register)의 내용 변경을 시도할때 발생한다. 디버깅 레지스터 들은 [Intel 80386](http://en.wikipedia.org/wiki/Intel_80386) 에서 부터 `x86` 계열 프로세서들에 존재하고 이름에서 알 수 있듯이, 이 레지스터 들의 주요 목적은 디버깅이다.

이런 레지스터들은 코드에 중단점을 설정하거나 특정 변수를 추적하기 위한 데이터를 읽기/쓰기를 허용한다. 디버깅 레지스터들은 특권 모드에서만 접근 가능할 것이며, 특권 레벨이 아닌 상태에서 이 디버깅 레지스터들을 읽거나 쓰기를 시도한다면 [general protection fault](https://en.wikipedia.org/wiki/General_protection_fault) 예외가 발생할 것이다. 그렇기 때문에 `#DB` 예외를 위해 `set_intr_gate_ist` 함수가 사용된다, 하지만 `set_system_intr_gate_ist` 함수는 그럴 필요가 없다.

`#DB` 예외의 벡터 번호는 `1` 이다.(우리는 그것을 `X86_TRAP_DB`로 넘겨준다) 그리고 스펙을 읽어본다면, 이 예외는 에러코드를 가지지 않는다는 것을 알수 있다.:

```
+-----------------------------------------------------+
|Vector|Mnemonic|Description         |Type |Error Code|
+-----------------------------------------------------+
|1     | #DB    |Reserved            |F/T  |NO        |
+-----------------------------------------------------+
```

두 번째 예외는 `#BP` 또는 `중단점(breakpoint)` 이고 프로세서가 [int 3](http://en.wikipedia.org/wiki/INT_%28x86_instruction%29#INT_3) 명령어의 수행으로 발생하는 예외이다. `DB` 예외와는 다르게, `#BP` 예외는 사용자 영역에서 발생한다. 우리는 중단점을 우리의 코드 어디에나 추가할 수 있다. 예를 들어 간단한 프로그램을 보자.:

```C
// breakpoint.c
#include <stdio.h>

int main() {
    int i;
    while (i < 6){
	    printf("i equal to: %d\n", i);
	    __asm__("int3");
		++i;
    }
}
```

만약, 우리가 이 코드를 컴파일하고 실행한다면, 아래와 같은 출력을 볼 수 있을 것이다.:

```
$ gcc breakpoint.c -o breakpoint
i equal to: 0
Trace/breakpoint trap
```

하지만, 만약 이것이 `gdb`에서 실행된다면, 우리가 설정한 중단점을 볼 수 있고, 실행된 프로그램을 중단 없이 계속 진행할 수 있을 것이다.:

```
$ gdb breakpoint
...
...
...
(gdb) run
Starting program: /home/alex/breakpoints
i equal to: 0

Program received signal SIGTRAP, Trace/breakpoint trap.
0x0000000000400585 in main ()
=> 0x0000000000400585 <main+31>:	83 45 fc 01	add    DWORD PTR [rbp-0x4],0x1
(gdb) c
Continuing.
i equal to: 1

Program received signal SIGTRAP, Trace/breakpoint trap.
0x0000000000400585 in main ()
=> 0x0000000000400585 <main+31>:	83 45 fc 01	add    DWORD PTR [rbp-0x4],0x1
(gdb) c
Continuing.
i equal to: 2

Program received signal SIGTRAP, Trace/breakpoint trap.
0x0000000000400585 in main ()
=> 0x0000000000400585 <main+31>:	83 45 fc 01	add    DWORD PTR [rbp-0x4],0x1
...
...
...
```

이 순간 부터 우리는 이 두 예외들에 대해 조금 알게 되었고 이제 그것들의 핸들러를 살펴 보도록 할 것이다.

예최 핸들러 전에 준비 과정
--------------------------------------------------------------------------------

알겠지만, `set_intr_gate_ist` 와 `set_system_intr_gate_ist` 함수는 예외 핸들러의 주소를 그 함수들의 2번째 인자로 받는다. 우리의 경우에, 두 개의 예외 핸들러들은 아래와 같다.:


* `debug`;
* `int3`.

당신은 C 코드에서 이 함수들을 찾을 수 없을 것이다. 커널의 `*.c/*.h` 파일들에서는 [arch/x86/include/asm/traps.h](https://github.com/torvalds/linux/tree/master/arch/x86/include/asm/traps.h) 커널 헤더 파일에 위치한 그 함수들의 선언만 볼 수 있을 것이다.:

```C
asmlinkage void debug(void);
```

그리고

```C
asmlinkage void int3(void);
```

이 함수들은 `asmlinkage` 키워드와 함께 선언되어 있다. 이 선언은 [gcc](http://en.wikipedia.org/wiki/GNU_Compiler_Collection)의 특별한 선언자이다. 실제 에셈블리로 부터 불려지는  `C` 함수를 위해 우리는 함수 호출 규약에 따라 명시적인 선언이 필요하다. 우리의 경우, 만약 함수가 `asmlinkage` 와 함께 선언되어 있다면, `gcc`는 이 함수의 인자를 무조건 스택으로 부터 가져오게 할 것이다.

그래서, 이 두 핸들러들은 [arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/entry/entry_64.S) 어셈블리 코드에 `idtentry` 매크로와 함께 선언되어 있다.:

```assembly
idtentry debug do_debug has_error_code=0 paranoid=1 shift_ist=DEBUG_STACK고
```

그리고

```assembly
idtentry int3 do_int3 has_error_code=0 paranoid=1 shift_ist=DEBUG_STACK
```

각 예외 핸들러는 두 개의 파트로 구성되어 있을 것이다. 첫 번째 부분은 일반적인 부분으로 모든 예외 핸들러에서 같은 부분이다. 예외 핸들러는 반드시 스택에 [범용 레지스터](https://en.wikipedia.org/wiki/Processor_register) 를 저장하고 만약 예외가 사용자 영역에서 왔다면 커널 스택으로 전환하고 예외 핸들러의 두 번째 부분으로 제어권을 넘겨야 한다. 예외 처리의 두 번째 부분은 특정 예외에 의존적인 특정 일을 하는 것이다. 예를 들어 페이지 폴트(page fault) 예외 처리 핸들러는 주어진 주소를 위한 가상 페이지를 찾고, 유효하지 않는 opcode 예외 처리 핸들러는 `SIGILL` [시그널](https://en.wikipedia.org/wiki/Unix_signal)을 보내는 것과 같은 것이다.

방금 우리가 본 대로, 하나의 예외 처리 핸들러는 [arch/x86/kernel/entry_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/entry_64.S) 어셈블리 코드 파일에서 `idtentry` 매크로의 선언으로 부터 시작한다. 그럼 이 매크로의 구현을 살펴보자. 보시다 시피, `idtentry` 매크로는 5개의 인자를 받는다.:

* `sym` - 예외 처리 핸들러의 엔트리가 될 것인 심볼을 `.globl name` 로 정의한다.
* `do_sym` - 예회 처리 핸들러의 두번째 엔트리를 표현하는 심볼 이름
* `has_error_code` - 예외의 에러코드 유무의 정보

마지막 두개의 인자는 선택적인 사항이다.:

* `paranoid` - 현재 모드를 확인할 필요가 있는지 보여준다.(나중에 삺펴보자.)
* `shift_ist` - 예외가 `Interrupt Stack Table`에서 수행되는지 보여준다.

`.idtentry` 매크로의 정의는 아래와 같다.

```assembly
.macro idtentry sym do_sym has_error_code:req paranoid=0 shift_ist=-1
ENTRY(\sym)
...
...
...
END(\sym)
.endm
```

`idtentry` 매크로의 내부를 살펴보기 전에, 우리는 예외가 발생했을 때, 스택의 상태를 반드시 알아야 한다. [Intel® 64 and IA-32 Architectures Software Developer’s Manual 3A](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html) 를 읽어본다면, 예외가 발생했을 때, 다음과 같은 스택의 상태를 볼 수 있다.:

```
    +------------+
+40 | %SS        |
+32 | %RSP       |
+24 | %RFLAGS    |
+16 | %CS        |
 +8 | %RIP       |
  0 | ERROR CODE | <-- %RSP
    +------------+
```

이제 `idtentry` 매크로의 구현을 살펴보자. `#DB` 와 `#BP` 예외 핸들러는 아래처럼 선언되어 있다.:

```assembly
idtentry debug do_debug has_error_code=0 paranoid=1 shift_ist=DEBUG_STACK
idtentry int3 do_int3 has_error_code=0 paranoid=1 shift_ist=DEBUG_STACK
```

만약 우리가 이와 같은 선언들을 본다면, 우리는 `debug` 와 `int3` 라는 이름의 두 개의 루틴들을 생성할 것이며 이 예외 핸들러는 어떤 준비작업 이후에 `do_debug` 와 `do_int3` 두 번째 부분의 핸들러들을 호출 할 것이다. 3 번째 인자는 에러 코드의 유무를 선언하는데 보시다 시피, 우리가 살펴보는 두개의 예외는 에러코드를 가지지 않는다. 위의 다이어그램을 보면, 프로세서는 예외가 에러코드를 제공한다면 스택에 넣도록 되어 있다. 우리의 경우 `debug` 와 `int3` 예외는 에러코드들을 가지지 않는다. 이것은 스택을 에러코드가 제공되는 예외와 그렇지 않은 예외를 다르게 봐야 하기 때문에, 처리에 약간의 어려움은 있다. 그래서 `idtentry` 매크로는 에러코드를 제공하지 않는 예외들을 위해 스택에 가짜의 에러코드를 넣는다.:

```assembly
.ifeq \has_error_code
    pushq	$-1
.endif
```

하지만, 그것은 단지 가짜의 에러 코드만은 아니다. 게다가 `-1` 은 유효하지 않는 시스템 콜 번호를 사용하는 것과 같은 에러 코드이다. 그래서 시스템 콜 재시작 로직은 진행하지 않을 것이다.

`idtentry` 매크로의 마지막 2 개의 인자로 `shift_ist` 와 `paranoid`는 예외 처리가 `Interrupt Stack Table` 로 부터 온 스택에서 실행되는지를 알아 내기 위해 사용된다. 당신은 이미 시스템에서 각 커널 쓰레드는 자신만의 스택을 가진다는 것을 알고 있을 것이다. 추가적으로, 시스템에는 각 프로세서와 연관된 특별한 스택들도 있다. 이런 특별한 스택 중에 하나인 것이 예외 스택이다. [x86_64](https://en.wikipedia.org/wiki/X86-64) 아키텍처는 `Interrupt Stack Table` 라 불리는 특별한 기능을 제공한다. 이 기능은 `double fault` 와 같은 원자적(atomic) 예외 처럼 지정된 이벤트를 위해 새로운 스택으로 전환하는 것을 허용한다. 그래서 `shift_ist` 인자는 우리들에서 예외 핸들러가 `IST`로 전환할 필요가 있는지 알게 해준다.

두 번째 인자는 - `paranoid` 는 예외 처리 핸들러로 사용자 영역에서 부터 왔는지를 알게 도와 주는 방법을 정의한다. 가장 쉬운 방법은 `CS` 세그먼트 레지스터에 있는 `CPL`(`Current Privilege Level`)를 통해 이를 확인 할 수 있다. 만약 이것이 `3`과 같다면, 사용자 영역에서 온것이고, `0`이라면 커널 영역에서 온것이다.:

```
testl $3,CS(%rsp)
jnz userspace
...
...
...
// we are from the kernel space
```

하지만 불행하게도 이 방법은 100% 신뢰할 수 있는 방법은 아니다. 커널 문서에 기술되어 있는 내용에 따르면:

> if we are in an NMI/MCE/DEBUG/whatever super-atomic entry context,
> which might have triggered right after a normal entry wrote CS to the
> stack but before we executed SWAPGS, then the only safe way to check
> for GS is the slower method: the RDMSR.
> 만약 우리가 NMI/MCE/DEBUG 또는 super-atomic entry context 에서 수행중이라면,
> 일반적인 엔트리에서 CS 를 스택에 쓴 바로뒤에 그리고 SWAPGS 수행전에 트리거되었다면,
> GS 확인을 위한 단 하나의 안전한 방법은 느리지만 RDMSR 을 사용하는 것이다.

예를 들어 보자, [swapgs](http://www.felixcloutier.com/x86/SWAPGS.html) 명령어의 임계영역(critical section) 내에서 `NMI` 가 일어났다고 해보자. per-cpu 영역의 시작을 가리키고 있는 `MSR_GS_BASE` [model specific register](https://en.wikipedia.org/wiki/Model-specific_register)  의 값을 확인해야 한다. 그래서 사용자 영역에서 온것인지 아닌지 확인하기 위해, 우리는 `MSR_GS_BASE` 모델 정의 레지스터의 값을 확인해야 할 것이고, 만약 그것이 음수라면 커널 영역에서 온것이고 아니라면 사용자 영역에서 온 것이다.:

```assembly
movl $MSR_GS_BASE,%ecx
rdmsr
testl %edx,%edx
js 1f
```

첫 두 라인은 `MSR_GS_BASE` 모델 정의 레지스터를 읽어 값을 `edx:eax` 쌍에 넣는다. 우리는 사용자 영역에서 온 `gs` 의 값을 음수로 설정할 수 없다. 하지만 커널에서 왔다면, 우리는 이미 물리 메모리의 직접 맵핑은 `0xffff880000000000` 가상 주소에서 시작한다는 것을 알고 있다. 이런 식으로 `MSR_GS_BASE` 는 주소를 `0xffff880000000000` 부터 `0xffffc7ffffffffff`까지 포함할 것이다. `rdmsr` 명령가 실행되고 나면, `%edx` 레지스터에서 가장 작은 가능한 값이 `0xffff8800` 로 될 것이다.(`-30720`). 그래서 `per-cpu` 영역의 시작을 가리키고 있는 커널 영역 `gs`는 음수의 값을 포함하게 될 것이다.

스택에 가짜 에러 코드를 넣은 다음에, 우리는 범용 레지스터들을 위한 공간을 할당해야 한다.:

```assembly
ALLOC_PT_GPREGS_ON_STACK
```

이 매크로는 [arch/x86/entry/calling.h](https://github.com/torvalds/linux/blob/master/arch/x86/entry/calling.h) 헤터 파일에 정의되어 있다. 이 매크로는 단지 벙용 레지스터들을 보관하기 위해 스택에 15*8 바이트의 공간을 할당한다.:

```assembly
.macro ALLOC_PT_GPREGS_ON_STACK addskip=0
    addq	$-(15*8+\addskip), %rsp
.endm
```

그래서 그 스택은 `ALLOC_PT_GPREGS_ON_STACK` 의 수행 이후에는 아래와 같이 보일 것이다.:

```
     +------------+
+160 | %SS        |
+152 | %RSP       |
+144 | %RFLAGS    |
+136 | %CS        |
+128 | %RIP       |
+120 | ERROR CODE |
     |------------|
+112 |            |
+104 |            |
 +96 |            |
 +88 |            |
 +80 |            |
 +72 |            |
 +64 |            |
 +56 |            |
 +48 |            |
 +40 |            |
 +32 |            |
 +24 |            |
 +16 |            |
  +8 |            |
  +0 |            | <- %RSP
     +------------+
```

범용 레지스터들을 위한 공간을 할당하고 나면, 우리는 사용자영역에서 부터 온 예외의 행동을 이해하기 위해 몇가지 확인을 해야 한다. 만약 사용자 공간에서 왔다면, 우리는 인터럽트된 프로세스 스택으로 돌아가던가 예외 스택에 머물러야 한다.:

```assembly
.if \paranoid
    .if \paranoid == 1
	    testb	$3, CS(%rsp)
	    jnz	1f
	.endif
	call	paranoid_entry
.else
	call	error_entry
.endif
```

위의 모든 경우에 내용을 생각해보자.

사용자 영역에서 발생한 예외
--------------------------------------------------------------------------------

먼저 예외가 우리의 `debug` 와 `int3` 과 같이 `paranoid=1` 를 가질때의 경우를 보자. 이 경우는 우리가 `CS` 세그먼트 레지스터로 부터 셀렉터를 확인하고 사용자 영역에서 부터 왔거다면 `1f` 라벨로 점프한다. 아니라면 다른 방법으로 `paranoid_entry` 를 호출 할 것이다.

첫번째 경우세어 사용자 영역에서 예외 처리 핸들러로 왔을때를 고려해보자. 위에서 기술했듯이, 우리는 `1` 라벨로 점프해야 한다. `1` 라벨은 아래의 호출에서 부터 시작한다.

```assembly
call	error_entry
```

스택에 있는 이전에 할당되었던 영역에 있는 모든 범용 레지스터를 저장하는 루틴이다.:

```assembly
SAVE_C_REGS 8
SAVE_EXTRA_REGS 8
```

이 두개의 매크로는 [arch/x86/entry/calling.h](https://github.com/torvalds/linux/blob/master/arch/x86/entry/calling.h) 헤더 파일에 선언되어 있고, 단지 범용 레지스터 값들을 스택의 특정 영역으로 옮기는 일을 한다, 예를 들어:

```assembly
.macro SAVE_EXTRA_REGS offset=0
	movq %r15, 0*8+\offset(%rsp)
	movq %r14, 1*8+\offset(%rsp)
	movq %r13, 2*8+\offset(%rsp)
	movq %r12, 3*8+\offset(%rsp)
	movq %rbp, 4*8+\offset(%rsp)
	movq %rbx, 5*8+\offset(%rsp)
.endm
```

`SAVE_C_REGS` 와 `SAVE_EXTRA_REGS`가 수행되고 나면, 스택은 아래와 같은 모습을 하고 있을 것이다.:

```
     +------------+
+160 | %SS        |
+152 | %RSP       |
+144 | %RFLAGS    |
+136 | %CS        |
+128 | %RIP       |
+120 | ERROR CODE |
     |------------|
+112 | %RDI       |
+104 | %RSI       |
 +96 | %RDX       |
 +88 | %RCX       |
 +80 | %RAX       |
 +72 | %R8        |
 +64 | %R9        |
 +56 | %R10       |
 +48 | %R11       |
 +40 | %RBX       |
 +32 | %RBP       |
 +24 | %R12       |
 +16 | %R13       |
  +8 | %R14       |
  +0 | %R15       | <- %RSP
     +------------+
```

스택에 범용 레지스터의 값을 저장한 후에, 우리는 사용자 영역에서 왔는지 확인을 다시 해야 한다.:

```assembly
testb	$3, CS+8(%rsp)
jz	.Lerror_kernelspace
```

왜냐하면 우리는 아마도 잘려진(truncated) `%RIP` 가 보고된다면 기술된 문서에서는 잠재적인 실패(fault)를 가질수 있다고 보기 때문이다. 어쨌든, [SWAPGS](http://www.felixcloutier.com/x86/SWAPGS.html) 명령어가 수행이 될 것이든 `MSR_KERNEL_GS_BASE` 와 `MSR_GS_BASE` 의 값이 서로 바꿔치기 되는 2 가지 경우가 있다. 이 순간부터 `%gs` 레지스터는 커널 구조체들의 베이스 주소를 가리킬 것이다. 그래서, `SWAPGS` 명령어는 호출되고 `error_entry` 루틴의 주요 기점이 된다.

이제 `idtentry` 매크로로 돌아오자. 우리는 `error_entry` 의 호출 이후에 아래와 같은 어셈블리 코드를 볼 수 있을 것이다.:

```assembly
movq	%rsp, %rdi
call	sync_regs
```

여기서 우리는 스택 포인터의 베이스 주소를 `sync_regs` 함수의 첫번째 인자([x86_64 ABI](https://www.uclibc.org/docs/psABI-x86_64.pdf) 에 따라)로 사용될 `%rdi` 레지스터에 넣고 [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/traps.c) 소스 코드 파일에 정의된 이 함수를 호출한다.:

```C
asmlinkage __visible notrace struct pt_regs *sync_regs(struct pt_regs *eregs)
{
	struct pt_regs *regs = task_pt_regs(current);
	*regs = *eregs;
	return regs;
}
```

이 함수는 [arch/x86/include/asm/processor.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/processor.h) 헤더 파일에 구현되어 있는 `task_ptr_regs` 매크로의 결과를 받아 스택 포인터에 그것을 저장하고 반환한다. `task_ptr_regs` 매크로는 일반 커널 스택을 가리키고 있는 `thread.sp0` 의 주소를 사용하는 매크로이다.:

```C
#define task_pt_regs(tsk)       ((struct pt_regs *)(tsk)->thread.sp0 - 1)
```

사용자 영역에서 부터 왔으니, 이 것은 real 프로세스 문맥(real process context) 에서 예외 처리 핸들러가 수행된다는 것이다. `sync_regs` 로 부터 스택 포인터를 얻은 후에, 스택을 전환한다.:

```assembly
movq	%rax, %rsp
```

예외 처리내에서 두 번째 핸들러를 호출 하기 전에 마지막 두 단계는:

1. 저장된 범용 레지스터들을 포함하는 `pt_regs` 구조체의 포인터를 `%rdi` 레지스터로 전달한다.:

```assembly
movq	%rsp, %rdi
```

이러므로 두번째 예외 핸들러의 첫번째 인자로 전달될 것이다.

2. `%rsi` 레지스터에 에러 코드를 전달한다. 예외처리 핸들러의 두 번째 인자로 사용될 것이며, `-1` 로의 설정은 이전에 설명했던 시스템 콜 호출을 재시도 하는 것을 방지하기 위해 한다.:

```
.if \has_error_code
	movq	ORIG_RAX(%rsp), %rsi
	movq	$-1, ORIG_RAX(%rsp)
.else
	xorl	%esi, %esi
.endif
```

추가적으로 에러 코드가 제공되지 않는 예외라면 `%esi` 레지스터가 0 으로 채워진다.

마침내 두 번째 예외 처리 핸들러가 호출된다.:

```assembly
call	\do_sym
```

이것은:

```C
dotraplinkage void do_debug(struct pt_regs *regs, long error_code);
```

`debug` 예외가 될 것이다.:

```C
dotraplinkage void notrace do_int3(struct pt_regs *regs, long error_code);
```

이것은 `int 3` 예외 처리를 위한 것이다. 이 파트에서는 두 번째 핸들러들의 구현을 살펴보지 않을 것이다. 왜냐하면 그것들은 매우 특화된 코드이다. 하지만 다음 파트에서 몇몇 부분을 보도록 하자.

사용자 영역에서 발생한 예외를 위해 첫번째 경우를 고려해 보았다. 마지막 2 개도 보자.

커널 영역에서 발생한 paranoid > 0 의 예외
--------------------------------------------------------------------------------

이 경우의 예외는 커널 영역에서 발생했고, `idtentry` 매크로는 이 예외를 위해 `paranoid=1`로 선언되어 있다. `paranoid` 의 값은 이 파트의 초반 부에 다루어졌던 예외가 커널영역에서 부터 온 것인지 아닌지 확인하는 것을 좀 더 느린 방법으로 해야 한다는 의미이다. `paranoid_entry` 루틴은 이 값을 알게 해준다.:

```assembly
ENTRY(paranoid_entry)
	cld
	SAVE_C_REGS 8
	SAVE_EXTRA_REGS 8
	movl	$1, %ebx
	movl	$MSR_GS_BASE, %ecx
	rdmsr
	testl	%edx, %edx
	js	1f
	SWAPGS
	xorl	%ebx, %ebx
1:	ret
END(paranoid_entry)
```

당신이 보듯이, 이 함수는 이전에 다루었던 것과 같은것이다. 우리는 인터럽트된 태스크의 이전 상태에 관한 정보를 얻기 위한 두 번째(느린) 방법을 사용한다. 우리가 이 정보를 확인하고 사용자 영역에서 부터 왔을 때 `SWAPGS`를 실행하게 되면, 우리는 이전에 했던 같은 일을 해야만 한다.: 우리는 범용 레지스터들을 갖고 있는 구조테의 포인터를 `%rdi`(두번째 핸들러의 첫번째 인자로 사용될 것이다.) 에 넣어주고 예외가 만약에 에러 코드를 지원하면 에러 코드를 `%rsi`(두 번째 핸들러의 두번째 인자로 사용 될 것이다.)에 넣는다.:

```assembly
movq	%rsp, %rdi

.if \has_error_code
	movq	ORIG_RAX(%rsp), %rsi
	movq	$-1, ORIG_RAX(%rsp)
.else
	xorl	%esi, %esi
.endif
```

예외처리의 두 번째 핸들러 수행 전에 하는 마지막 단계에서 호출 될 것은 새로운 `IST` 스택 프레임을 정리하는 것이다.:

```assembly
.if \shift_ist != -1
	subq	$EXCEPTION_STKSZ, CPU_TSS_IST(\shift_ist)
.endif
```

당신은 `idtentry` 매크로의 인자로 `shift_ist`를 넘겨 준것을 기억할 것이다. 여기서 우리는 그 값을 확인하고 만약 `-1` 과 같지 않으면 `shift_ist` 인덱스로 `Interrupt Stack Table` 에서 스택을 가리키는 포인터를 얻고 설정할 것이다.

이 것이 두 번째(느린) 방법의 두 번째 예외 핸들러의 호출 바로 직전까지 하는 일들을 정리했다.:

```assembly
call	\do_sym
```

마지막 방법은 이전 두 방법과 비슷하지만, `paranoid=0` 와 예외가 발생하고 빠른 방법으로 어디서 부터 예외가 발생했는지 확인 할 것이다.

예외 처리의 종료
--------------------------------------------------------------------------------

두 번째 핸들러가 일을 마무리 한 뒤에, 우리는 `idtentry` 매크로로 부터 복귀할 것이고 다음 단계는 `error_exit` 로 점프하는 것이다.:

```assembly
jmp	error_exit
```

`error_exit` 함수는 [arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/entry/entry_64.S) 어셈블리 소스 파일에 구현되어 있고 이 함수의 주요 목적은 우리가 어디서 부터 왔는지(사용자 영역 또는 커널 영역) 알게 하고, 그것에 의존하여 `SWPAGS` 를 실행한다. 이전 상태의 레지스터들을 복구하고 `iret` 명령어를 실행하여 인터럽트 되었던 태스크로 제어권을 넘겨준다.

이상.

결론
--------------------------------------------------------------------------------

리눅스 커널의 인터럽트와 인터럽트 처리의 3번째 파트가 마무리되었다. 우리는 이전 파트에서 `#DB`, `#BP` 게이트와 함께 [Interrupt descriptor table](https://en.wikipedia.org/wiki/Interrupt_descriptor_table) 의 초기화를 보았고, 이 파트에서 제어권을 예외 처리 핸들러로 넘기기 전에 준비 작업을 살펴 보고 몇몇의 인터럽트 핸들러의 구현도 보았다. 다음 파트에서는 계속에서 인터럽트와 관련된 내용을 볼 것이다. `setup_arch` 함수의 다음 부분에서 인터럽트 핸들링과 연관된 것들을 이해해보도록 하자.

어떤 질문이나 제안이 있다면, twitter [0xAX](https://twitter.com/0xAX), [email](anotherworldofworld@gmail.com) 또는 [issue](https://github.com/0xAX/linux-insides/issues/new) 를 만들어 주길 바란다.

**나는 영어권의 사람이 아니고 이런 것에 대해 매우 미안해 하고 있다. 만약 어떤 실수를 발견한다면, 나에게 PR을 [linux-insides](https://github.com/0xAX/linux-internals)을 보내줘**

Links
--------------------------------------------------------------------------------

* [Debug registers](http://en.wikipedia.org/wiki/X86_debug_register)
* [Intel 80385](http://en.wikipedia.org/wiki/Intel_80386)
* [INT 3](http://en.wikipedia.org/wiki/INT_%28x86_instruction%29#INT_3)
* [gcc](http://en.wikipedia.org/wiki/GNU_Compiler_Collection)
* [TSS](http://en.wikipedia.org/wiki/Task_state_segment)
* [GNU assembly .error directive](https://sourceware.org/binutils/docs/as/Error.html#Error)
* [dwarf2](http://en.wikipedia.org/wiki/DWARF)
* [CFI directives](https://sourceware.org/binutils/docs/as/CFI-directives.html)
* [IRQ](http://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29)
* [system call](http://en.wikipedia.org/wiki/System_call)
* [swapgs](http://www.felixcloutier.com/x86/SWAPGS.html)
* [SIGTRAP](https://en.wikipedia.org/wiki/Unix_signal#SIGTRAP)
* [Per-CPU variables](http://0xax.gitbooks.io/linux-insides/content/Concepts/per-cpu.html)
* [kgdb](https://en.wikipedia.org/wiki/KGDB)
* [ACPI](https://en.wikipedia.org/wiki/Advanced_Configuration_and_Power_Interface)
* [Previous part](http://0xax.gitbooks.io/linux-insides/content/interrupts/index.html)
