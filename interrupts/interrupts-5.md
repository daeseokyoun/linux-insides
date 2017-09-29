인터럽트와 인터럽트 핸들링. Part 5.
================================================================================

예외 핸들러의 구현
--------------------------------------------------------------------------------

이 문서는 리눅스 커널의 인터럽트와 예외 처리를 위한 5번째 파트이고 이전 [파트](https://github.com/daeseokyoun/linux-insides/blob/korean-trans/interrupts/interrupts-4.md) 에서는 인터럽트 게이트를 [Interrupt descriptor Table](https://en.wikipedia.org/wiki/Interrupt_descriptor_table)에 설정하는 부분에서 마무리되었다. 우리는 그 설정을 [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/tree/master/arch/x86/kernel/traps.c) 파일의 `trap_init` 함수에서 진행했었다. 우리는 이전 파트에서는 이런 인터럽트의 설정 부분만 보았지만 현재 파트에서는 이런 게이트들을 위한 예외 처리 핸들러의 구현을 보게될 것이다. 예외 처리 핸들러가 수행되기 전의 준비는 [arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/entry/entry_64.S) 어셈블리 파일에 있고 예외처리 엔트리 포인트를 정의하는 [idtentry](https://github.com/torvalds/linux/blob/master/arch/x86/entry/entry_64.S#L820) 매크로에서 발생한다.:

```assembly
idtentry divide_error			        do_divide_error			       has_error_code=0
idtentry overflow			            do_overflow			           has_error_code=0
idtentry invalid_op			            do_invalid_op			       has_error_code=0
idtentry bounds				            do_bounds			           has_error_code=0
idtentry device_not_available		    do_device_not_available		   has_error_code=0
idtentry coprocessor_segment_overrun	do_coprocessor_segment_overrun has_error_code=0
idtentry invalid_TSS			        do_invalid_TSS			       has_error_code=1
idtentry segment_not_present		    do_segment_not_present		   has_error_code=1
idtentry spurious_interrupt_bug		    do_spurious_interrupt_bug	   has_error_code=0
idtentry coprocessor_error		        do_coprocessor_error		   has_error_code=0
idtentry alignment_check		        do_alignment_check		       has_error_code=1
idtentry simd_coprocessor_error		    do_simd_coprocessor_error	   has_error_code=0
```

`idtentry` 매크로는 실제 예외 핸들러가 제어를 넘겨 받기 전에 다음과 같은 준비를 한다.(`divide_error`를 위한 `do_divide_error`,`overflow`를 위한 `do_overflow`과 같은) 그 준비 작업은 스택에 레지스터 저장을 위한 공간을 할당, 만약 인터럽트/예외가 에러코드를 가지지 않는다면, 스택 일관성(stack consistency)을 위해 가짜의 에러코드를 넣고, `cs` 세그먼트 레지스터에 세그머트 셀렉터를 확인 그리고 이전 상태의 의존적인 전환을 하게 된다.(사용자 영역 또는 커널 영역) 이 모든 준비 후에 실제 인터럽트/예외 핸들러의 호출을 가능하게 한다.:

```assembly
.macro idtentry sym do_sym has_error_code:req paranoid=0 shift_ist=-1
ENTRY(\sym)
	...
	...
	...
	call	\do_sym
	...
	...
	...
END(\sym)
.endm
```

예외 핸들러가 자신의 일을 마무리하고 나면, `idtentry` 매크로는 인터럽트된 태스크의 스택과 범용 레지스터를 복구하고 [iret](http://x86.renejeschke.de/html/file_module_x86_id_145.html) 명령어를 수행한다.:

```assembly
ENTRY(paranoid_exit)
	...
	...
	...
	RESTORE_EXTRA_REGS
	RESTORE_C_REGS
	REMOVE_PT_GPREGS_FROM_STACK 8
	INTERRUPT_RETURN
END(paranoid_exit)
```

`INTERRUPT_RETURN` 는 아래와 같이 선언:

```assembly
#define INTERRUPT_RETURN	jmp native_iret
...
ENTRY(native_iret)
.global native_irq_return_iret
native_irq_return_iret:
iretq
```

`idtentry` 매크로에 대해 조금더 자세한 사항은 [3번째 파트](https://github.com/daeseokyoun/linux-insides/blob/korean-trans/interrupts/interrupts-3.md)를 읽어보길 바란다. 좋다. 우리는 예외 핸들러가 수행하기 전에 준비작업을 살펴보았고 이제 핸들러 내부를 보도록 하자. 우리는 다음과 같은 핸들러를 살펴 볼것이다.:

* divide_error
* overflow
* invalid_op
* coprocessor_segment_overrun
* invalid_TSS
* segment_not_present
* stack_segment
* alignment_check

이 모든 핸들러들은 [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/traps.c)에 `DO_ERROR` 매크로로 선언되어 있다.

```C
DO_ERROR(X86_TRAP_DE,     SIGFPE,  "divide error",                divide_error)
DO_ERROR(X86_TRAP_OF,     SIGSEGV, "overflow",                    overflow)
DO_ERROR(X86_TRAP_UD,     SIGILL,  "invalid opcode",              invalid_op)
DO_ERROR(X86_TRAP_OLD_MF, SIGFPE,  "coprocessor segment overrun", coprocessor_segment_overrun)
DO_ERROR(X86_TRAP_TS,     SIGSEGV, "invalid TSS",                 invalid_TSS)
DO_ERROR(X86_TRAP_NP,     SIGBUS,  "segment not present",         segment_not_present)
DO_ERROR(X86_TRAP_SS,     SIGBUS,  "stack segment",               stack_segment)
DO_ERROR(X86_TRAP_AC,     SIGBUS,  "alignment check",             alignment_check)
```

`DO_ERROR` 는 4개의 인자를 받는다.:

* 인터럽트 벡터 번호
* 인터럽트된 프로세스에게 보내어질 시그널 번호
* 예외를 설명하는 문자열
* 예외 핸들러의 엔트리 포인트

이 매크로는 같은 소스파일에 있으며, `do_<핸들러 이름>` 으로 확장된다.:

```C
#define DO_ERROR(trapnr, signr, str, name)                              \
dotraplinkage void do_##name(struct pt_regs *regs, long error_code)     \
{                                                                       \
        do_error_trap(regs, error_code, str, trapnr, signr);            \
}
```

`##` 토큰에 주목해보자. 이것은 주어진 두개의 문자열을 연결하는 [GCC macro Concatenation](https://gcc.gnu.org/onlinedocs/cpp/Concatenation.html#Concatenation) 이다. 예를 들어, 우리의 경우에서 첫 `DO_ERROR` 를 예제로 들면:

```C
dotraplinkage void do_divide_error(struct pt_regs *regs, long error_code)     \
{
	...
}
```

우리는 `DO_ERROR` 매크로로 생성된 모든 함수는 [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/tree/master/arch/x86/kernel/traps.c)에 있는 `do_error_trap` 함수를 호출한다는 것을 알 수 있다. `do_error_trap` 함수의 구현을 살펴보자.

트랩 핸들러(Trap handlers)
--------------------------------------------------------------------------------

`do_error_trap` 함수는 아래 두 함수를 시작과 끝에 사용한다.:

```C
enum ctx_state prev_state = exception_enter();
...
...
...
exception_exit(prev_state);
```

이 두 함수는 [include/linux/context_tracking.h](https://github.com/torvalds/linux/tree/master/include/linux/context_tracking.h) 에 선언되어 있다. 커널 경계를 제공하는 리눅스 커널내의 문맥 추적은 `사용자` 혹은 `커널` 과 같은 기본적인 문맥에서 레벨 문맥 사이의 전환의 추적을 유지하기 위한것이다. `exception_enter` 함수는 문맥 추적 기능이 활성화 되어 있는지 확인한다. 만약 그것이 활성화 되어 있다면, `exception_enter` 는 이전 문맥을 읽고 `CONTEXT_KERNEL` 와 비교한다. 만약 이전 문맥이 `사용자` 문맥이라면, 우리는 프로세서가 사용자 영역을 빠져나와 커널 영역으로 진입한다는 것을 문맥 추적 서브 시스템에 알려주는 [kernel/context_tracking.c](https://github.com/torvalds/linux/blob/master/kernel/context_tracking.c) 소스 코드에 구현된 `context_tracking_exit` 함수를 호출한다.:

```C
if (!context_tracking_is_enabled())
	return 0;

prev_ctx = this_cpu_read(context_tracking.state);
if (prev_ctx != CONTEXT_KERNEL)
	context_tracking_exit(prev_ctx);

return prev_ctx;
```

만약 이전 문맥이 `사용자`가 아니라면, 우리는 그냥 그 문맥을 반환한다. `pre_ctx` 은 [include/linux/context_tracking_state.h](https://github.com/torvalds/linux/tree/master/include/linux/context_tracking_state.h) 에 선언된 `enum ctx_state` 타입을 갖고 있으며, 아래와 같다.:

```C
enum ctx_state {
	CONTEXT_KERNEL = 0,
	CONTEXT_USER,
	CONTEXT_GUEST,
} state;
```

두 번째 함수는 [include/linux/context_tracking.h](https://github.com/torvalds/linux/tree/master/include/linux/context_tracking.h) 에 구현된 `exception_exit` 이고 문맥 추적이 활성화 되어 있는지 확인하고 이전 문맥이 `사용자`였다면 `context_tracking_enter` 함수를 호출한다.:

```C
static inline void exception_exit(enum ctx_state prev_ctx)
{
	if (context_tracking_is_enabled()) {
		if (prev_ctx != CONTEXT_KERNEL)
			context_tracking_enter(prev_ctx);
	}
}
```

`context_tracking_enter` 함수는 프로세서가 커널 모드에서 사용자 모드로 진입할 것이라는 것을 문맥 추적 서브시스템에 알려준다. 우리는 `exception_enter` 와 `exception_exit` 사이에 아래와 같은 코드가 있는 것을 볼 수 있다.:

```C
if (notify_die(DIE_TRAP, str, regs, error_code, trapnr, signr) !=
		NOTIFY_STOP) {
	conditional_sti(regs);
	do_trap(trapnr, signr, str, regs, error_code,
		fill_trap_info(regs, signr, trapnr, &info));
}
```

첫 번째로 보이는 호출은 `notify_die` 함수인데 [kernel/notifier.c](https://github.com/torvalds/linux/tree/master/kernel/notifier.c) 에 구현되어 있다. [kernel panic](https://en.wikipedia.org/wiki/Kernel_panic), [kernel oops](https://en.wikipedia.org/wiki/Linux_kernel_oops), [Non-Maskable Interrupt](https://en.wikipedia.org/wiki/Non-maskable_interrupt) 또는 다른 이벤트의 호출자를 위한 알림을 받기 위해서는 `notify_die` 체인을 직접 넣어줄 필요가 있고 `notify_die` 함수가 그 일을 해준다. 리눅스 커널은 어떤 일이 언제 일어났는지 커널에게 요청하는 것을 허용하는 특별한 매커니즘이 있고 이를 `notifiers` 혹은 `notifier chains` 이라 부른다. 이 매커니즘을 위한 가장 좋은 예제는 `USB` hotplug 이벤트, ([drivers/usb/core/notify.c](https://github.com/torvalds/linux/tree/master/drivers/usb/core/notify.c)를 보면 된다.), 메모리 [hotplug](https://en.wikipedia.org/wiki/Hot_swapping) -([include/linux/memory.h](https://github.com/torvalds/linux/tree/master/include/linux/memory.h) 을 보면, `hotplug_memory_notifier` 매크로 등등이 있다.), 시스템 리부팅 등이 있다. notifier chain 은 간단하며, 단일 연결리스트이다. 리눅스 커널 서브시스템이 특정 이벤트의 알림을 받기 원할 때, 그것은 `notifier_block` 구조체에 내요을 채우고 `notifier_chain_register` 함수로 넘겨준다. 하나의 이벤트는 `notifier_call_chain` 의 호출과 함께 전달 될 수 있다. `notify_die` 함수는 트랩 번호, 레지스터들과 다른 몇몇의 값을 `die_args`에 채운다.:

```C
struct die_args args = {
       .regs   = regs,
       .str    = str,
       .err    = err,
       .trapnr = trap,
       .signr  = sig,
}
```

그리고 `die_chain` 과 함께 `atomic_notifier_call_chain` 함수의 결과를 반환한다.:

```C
static ATOMIC_NOTIFIER_HEAD(die_chain);
return atomic_notifier_call_chain(&die_chain, val, &args);
```

이것은 단순히 lock 과 `notifier_block`을 가지는 `atomic_notifier_head` 구조체를 확장한다.:

```C
struct atomic_notifier_head {
        spinlock_t lock;
        struct notifier_block __rcu__ *head*; // TODO rcu 뒤에 언더바 두개, head 뒤에 별하나
};
```

`atomic_notifier_call_chain` 함수는 차례로 notifier chain 에 있는 각 함수들을 호출하고 마지막 호출된 notifier 함수의 값을 반환한다. 만약 `do_error_trap` 내에 `notify_die` 가 `NOTIFY_STOP` 를 반환하지 않는다면, 우리는 [interrupt flag](https://en.wikipedia.org/wiki/Interrupt_flag) 값을 확인하고 필요하다면 인터럽트를 활성화하는 [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/traps.c)에 구현된 `conditional_sti` 함수를 수행한다.:

```C
static inline void conditional_sti(struct pt_regs *regs)
{
        if (regs->flags & X86_EFLAGS_IF)
                local_irq_enable();
}
```

`local_irq_enable` 매크로에 대해 더 자세히 알고 싶다면 인터럽트 두번째 [파트](https://github.com/daeseokyoun/linux-insides/blob/korean-trans/interrupts/interrupts-2.md) 를 읽어보길 바란다. `do_error_trap` 함수에서 마지막으로 호출되는 `do_trap` 함수를 보자. `do_trap` 함수는 `task_struct` 구조체 타입인 `tsk`를 선언하고 현재 인터럽트된 프로세스를 할당한다. `tsk`의 선언 후에는, `do_trap_no_signal` 함수의 호출을 볼 수 있다.:

```C
struct task_struct *tsk = current;

if (!do_trap_no_signal(tsk, trapnr, str, regs, error_code))
	return;
```

`do_trap_no_signal` 함수는 두가지의 확인을 한다.:

* [Virtual 8086](https://en.wikipedia.org/wiki/Virtual_8086_mode) 모드에서 부터 왔는지
* 커널 영역에서 부터 왔는지

```C
if (v8086_mode(regs)) {
	...
}

if (!user_mode(regs)) {
	...
}

return -1;
```

우리는 첫 번째 확인은 고려하지 않을 것이다. 왜냐하면 [long mode](https://en.wikipedia.org/wiki/Long_mode) 는 [Virtual 8086](https://en.wikipedia.org/wiki/Virtual_8086_mode) 를 지원하지 않기 때문이다. 두 번째 경우는 fault 를 복구하려고 시도하고 복구가 안된다면 `die`를 호출하는 `fixup_exception` 함수를 실행한다.:

```C
if (!fixup_exception(regs)) {
	tsk->thread.error_code = error_code;
	tsk->thread.trap_nr = trapnr;
	die(str, regs, error_code);
}
```

`die` 함수는 [arch/x86/kernel/dumpstack.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/dumpstack.c) 소스 코드 파일에 구현되어 있고, 스택, 레지스터, 커널 모듈들과 야기된 커널 [oops](https://en.wikipedia.org/wiki/Linux_kernel_oops) 관련 유용한 정보를 출력한다. 만약 사용자 영역으로 부터 왔다면, `do_trap_no_signal` 함수는 `-1`을 반환할 것이고 `do_trap` 함수의 실행을 계속할 것이다. 만약 우리가 `do_trap_no_signal` 함수를 지나왔고 `do_trap`으로 부터 빠져나오지 않았다면, 그것은 이전 문맥이 `사용자`였다는 것을 의미한다. 프로세서에 의해 야기된 대부분의 예외는 에러 상태의 리눅스에 의해 처리 될 수 있는데, 예를 들면 0으로 나누기, 유효하지 않는 명령코드(opcode) 등이 있다. 예외가 발생할 때, 리눅스 커널은 [signal](https://en.wikipedia.org/wiki/Unix_signal) 을 예외가 발생한 인터럽트된 프로세스에게 전달하여 적절치 않은 상태에 대한 통보를 한다. 그래서, `do_trap`함수에서 주어진 번호와 함께 시그널을 전달할 필요가 있다.(0으로 나누기에 대한 `SIGFPE`, 오버플로우 예외를 위한 `SIGILL` 등) 무엇보다도 먼저 `thread.error_code` 와 `thread_trap_nr` 을 값을 현재 인터럽트된 프로세스에 에러 코드와 벡터 번호를 저장한다.:

```C
tsk->thread.error_code = error_code;
tsk->thread.trap_nr = trapnr;
```

우리는 `show_unhandled_signals` 설정되어 있는지 확인, [kernel/signal.c](https://github.com/torvalds/linux/blob/master/kernel/signal.c) 에 있는 처리하지 못한 시그널(들)을 반환하는 `unhandled_signal` 함수가 반환하는 값 확인 그리고 [printk](https://en.wikipedia.org/wiki/Printk) rate 제한을 확인하여 인터럽트된 프로세스를 위해 처리하지 못한 시그널에 관련된 정보를 출력한다.:

```C
#ifdef CONFIG_X86_64
	if (show_unhandled_signals && unhandled_signal(tsk, signr) &&
	    printk_ratelimit()) {
		pr_info("%s[%d] trap %s ip:%lx sp:%lx error:%lx",
			tsk->comm, tsk->pid, str,
			regs->ip, regs->sp, error_code);
		print_vma_addr(" in ", regs->ip);
		pr_cont("\n");
	}
#endif
```

그리고 주어진 시그널을 인터럽트된 프로세스에게 전달한다.:

```C
force_sig_info(signr, info ?: SEND_SIG_PRIV, tsk);
```

`do_trap`가 마무리 되었다. 우리는 단지 위에서 `DO_ERROR` 매크로로 선언한 8 개의 다른 예외를 위한 일반적인 구현을 보았다. 이제 다른 예외 처리 핸들러를 살펴보자.

Double fault
--------------------------------------------------------------------------------

다음 예외는 `#DF`/`Double fault` 이다. 이 예외는 프로세서가 이전 예외 처리를 하는 동안에 두 번째 예외가 발견되면 발생한다. 우리는 이전 파트에서 이 예외를 위한 트랩 게이트를 설정했다.:

```C
set_intr_gate_ist(X86_TRAP_DF, &double_fault, DOUBLEFAULT_STACK);
```

이 예외는 인덱스가 `1` 인 `DOUBLEFAULT_STACK` [Interrupt Stack Table](https://www.kernel.org/doc/Documentation/x86/x86_64/kernel-stacks) 에서 수행된다.:

```C
#define DOUBLEFAULT_STACK 1
```

`double_fault` 는 이 예외를 위한 핸들러이고 [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/tree/master/arch/x86/kernel/traps.c)에 구현되어 있다. `double_fault` 핸들러는 두 개의 변수를 선언하는 것부터 시작한다.: 예외를 기술하는 문자열과 현재 인터럽트된 프로세스:

```C
static const char str[] = "double fault";
struct task_struct *tsk = current;
```

double fault 예외의 핸들러는 두 부분으로 분리된다. 첫 번째 부분은 발생한 폴트가 `espfix64` 스택에서 일어나는 `non-IST` 폴트인지 확인한다. 실제 `iret` 명령어는 `16` 비트 세그먼트로 반환될 때, 단지 하위 `16` 비트만을 복구한다. `espfix` 기능은 이 문제를 해결할 수 있다. 그래서 만약 espfix64 스택의 `non-IST` 폴트라면, 우리는 스택을 수정하여 그것을 `General Protection Fault` 로 보이게 한다.:

```C
struct pt_regs *normal_regs = task_pt_regs(current);

memmove(&normal_regs->ip, (void *)regs->sp, 5*8);
ormal_regs->orig_ax = 0;
regs->ip = (unsigned long)general_protection;
regs->sp = (unsigned long)&normal_regs->orig_ax;
return;
```

두 번째 경우는 이전 예외 핸들러 내에서 했던 것과 거의 같은 일을 한다. 첫번째는 이전 문맥을 버리는 `ist_enter` 함수의 호출이 있다. 우리의 경우는 `사용자` 문맥이다.:

```C
ist_enter(regs);
```

그리고 나서 우리는 이전 핸들러내에서 했던 것과 같은 `Double fault` 예외의 벡터 번호와 에러코드를 인터럽트된 프로세스에 채운다.:

```C
tsk->thread.error_code = error_code;
tsk->thread.trap_nr = X86_TRAP_DF;
```

다음에는 double fault 에 관련된 유용한 정보를 출력한다.([PID](https://en.wikipedia.org/wiki/Process_identifier) 번호, 레지스터 내용):

```C
#ifdef CONFIG_DOUBLEFAULT
	df_debug(regs, error_code);
#endif
```

그리고 die 를 호출한다.:

```
	for (;;)
		die(str, regs, error_code);
```

이상이다.

장치가 가용하지 않을 때 발생하는 예외 핸들러
--------------------------------------------------------------------------------

다음 살펴볼 예외는 `#NM`/`Device not available`. `Device not available` 예외는 아래와 같은 것들에 의존적으로 발생할 수 있다.:

* [control register](https://en.wikipedia.org/wiki/Control_register) `cr0` 에 EM 플래그가 설정되어 있는 상태에서 [x87 FPU](https://en.wikipedia.org/wiki/X87)  floating-point 명령이 프로세서에서 실행되는 경우.
* `cr0` 레지스터에서 `MP`와 `TS` 플래그가 설정되어 있는데, `wait` 또는 `fwait` 명령이 프로세서에서 수행될 때
* `cr0` 레지스터에서 `TS` 플래그가 설정되어 있고 `EM` 플래그가 설정되어 있지 않는 상태에서 [x87 FPU](https://en.wikipedia.org/wiki/X87), [MMX](https://en.wikipedia.org/wiki/MMX_%28instruction_set%29) 나 [SSE](https://en.wikipedia.org/wiki/Streaming_SIMD_Extensions) 명령이 프로세서에서 수행될 때

`Device not available` 예외의 핸들러는 `do_device_not_available` 함수이고 [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/tree/master/arch/x86/kernel/traps.c) 소스 파일에 구현되어 있다. 그것은 이 파트의 앞부분에서 보았던 이전 문맥을 얻는 것에서 부터 시작하고 끝내는 것을 아래와 같이 한다.:

```C
enum ctx_state prev_state;
prev_state = exception_enter();
...
...
...
exception_exit(prev_state);
```

다음 단계는 FPU 의 사용이 eagar(열망하는) 한지 아닌지 확인한다.:

```C
BUG_ON(use_eager_fpu());
```

우리가 태스크나 인터럽트로의 전환이 될 때, `FPU` 상태의 로딩을 피하고 싶은 것이다. 만약 태스가 그것을 사용하고 있다면, 우리는 `Device not Available exception` 예외를 받을 것이다. 만약 태스크 전환중에 `FPU` 상태를 로드하면, `FPU`는 `eager` 상태이다. 다음 단계는 `cr0` 제어 레지스터를 확인하여 `x87` floating point 유닛의 상태를 보여주는 `EM` 플래그가 설정되어 있는지 아닌지 본다.:

```C
#ifdef CONFIG_MATH_EMULATION
	if (read_cr0() & X86_CR0_EM) {
		struct math_emu_info info = { };

		conditional_sti(regs);

		info.regs = regs;
		math_emulate(&info);
		exception_exit(prev_state);
		return;
	}
#endif
```

만약 `x87` floating point 유닛이 작동하지 않는다면, 우리는 `conditional_sti` 로 인터럽트를 활성화 하고, `math_emu_info` 구조체([arch/x86/include/asm/math_emu.h](https://github.com/torvalds/linux/tree/master/arch/x86/include/asm/math_emu.h) 에 선언)를 인터럽트 태스크의 레지스터로 채우고, [arch/x86/math-emu/fpu_entry.c](https://github.com/torvalds/linux/tree/master/arch/x86/math-emu/fpu_entry.c) 에 있는 `math_emulate` 함수를 호출한다. 함수 이름에서 부터 유추 가능할 것인 이 함수는 `X87 FPU` 유닛을 에뮬레이트한다. 다른 말로는, 만약 `X87 FPU` 유닛이 사용상태인 `X86_CR0_EM` 플래그가 설정되어 있지 않는다면, 우리는 `fpustate` 로 `FPU` 레지스터를 실제 하드웨어 레지스터에 복사하는 [arch/x86/kernel/fpu/core.c](https://github.com/torvalds/linux/tree/master/arch/x86/kernel/fpu/core.c)에 구현되어 있는 `fpu__restore` 함수를 호출한다. 이후에 `FPU` 명령어는 사용 가능하다.:

```C
fpu__restore(&current->thread.fpu);
```

일반 보호 실패(General protection fault) 예외 처리
--------------------------------------------------------------------------------

다음 살펴볼 예외는 `#GP` / `General protection fault` 이다. 이 예외는 프로세서가 `general-protection violations` 라 불리는 보호 위반의 하나를 감지 했을 때 발생한다.:

* `cs`, `ds`, `es`, `fs` 또는 `gs` 세그먼트들을 접근할 때, 세그먼트 제한을 초과하여 접근하려고 했을 때
* 시스템 세그먼트를 위해 세그먼트 셀렉터를 `ss`, `ds`, `es`, `fs` 또는 `gs` 레지스터에 로드 하려고 했을 때
* 특권 레벨사용에 대한 규칙을 어겼을 때
* 몇몇의 규칙 위반?

이 예외를 위한 핸들러는 [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/tree/master/arch/x86/kernel/traps.c)에 구현된 `do_general_protection` 함수이다. `do_general_protection` 함수는 다른 예외들 처럼 첫부분과 끝부분에 이전 문맥을 얻어오기 위해 다음과 같은 구조로 되어 있다.:

```C
prev_state = exception_enter();
...
exception_exit(prev_state);
```

여기서는 [Virtual 8086](https://en.wikipedia.org/wiki/Virtual_8086_mode) 모드가 확인이 되고 만약 인터럽트들이 비활성화 되어 있다면, 인터럽트를 활성화 시킨다.

```C
conditional_sti(regs);

if (v8086_mode(regs)) {
	local_irq_enable();
	handle_vm86_fault((struct kernel_vm86_regs *) regs, error_code);
	goto exit;
}
```

As long mode does not support this mode, we will not consider exception handling for this case. In the next step check that previous mode was kernel mode and try to fix the trap. If we can't fix the current general protection fault exception we fill the interrupted process with the vector number and error code of the exception and add it to the `notify_die` chain:

```C
if (!user_mode(regs)) {
	if (fixup_exception(regs))
		goto exit;

	tsk->thread.error_code = error_code;
	tsk->thread.trap_nr = X86_TRAP_GP;
	if (notify_die(DIE_GPF, "general protection fault", regs, error_code,
		       X86_TRAP_GP, SIGSEGV) != NOTIFY_STOP)
		die("general protection fault", regs, error_code);
	goto exit;
}
```

If we can fix exception we go to the `exit` label which exits from exception state:

```C
exit:
	exception_exit(prev_state);
```

If we came from user mode we send `SIGSEGV` signal to the interrupted process from user mode as we did it in the `do_trap` function:

```C
if (show_unhandled_signals && unhandled_signal(tsk, SIGSEGV) &&
		printk_ratelimit()) {
	pr_info("%s[%d] general protection ip:%lx sp:%lx error:%lx",
		tsk->comm, task_pid_nr(tsk),
		regs->ip, regs->sp, error_code);
	print_vma_addr(" in ", regs->ip);
	pr_cont("\n");
}

force_sig_info(SIGSEGV, SEND_SIG_PRIV, tsk);
```

That's all.

Conclusion
--------------------------------------------------------------------------------

It is the end of the fifth part of the [Interrupts and Interrupt Handling](http://0xax.gitbooks.io/linux-insides/content/interrupts/index.html) chapter and we saw implementation of some interrupt handlers in this part. In the next part we will continue to dive into interrupt and exception handlers and will see handler for the [Non-Maskable Interrupts](https://en.wikipedia.org/wiki/Non-maskable_interrupt), handling of the math [coprocessor](https://en.wikipedia.org/wiki/Coprocessor) and [SIMD](https://en.wikipedia.org/wiki/SIMD) coprocessor exceptions and many many more.

If you have any questions or suggestions write me a comment or ping me at [twitter](https://twitter.com/0xAX).

**Please note that English is not my first language, And I am really sorry for any inconvenience. If you find any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-insides).**

Links
--------------------------------------------------------------------------------

* [Interrupt descriptor Table](https://en.wikipedia.org/wiki/Interrupt_descriptor_table)
* [iret instruction](http://x86.renejeschke.de/html/file_module_x86_id_145.html)
* [GCC macro Concatenation](https://gcc.gnu.org/onlinedocs/cpp/Concatenation.html#Concatenation)
* [kernel panic](https://en.wikipedia.org/wiki/Kernel_panic)
* [kernel oops](https://en.wikipedia.org/wiki/Linux_kernel_oops)
* [Non-Maskable Interrupt](https://en.wikipedia.org/wiki/Non-maskable_interrupt)
* [hotplug](https://en.wikipedia.org/wiki/Hot_swapping)
* [interrupt flag](https://en.wikipedia.org/wiki/Interrupt_flag)
* [long mode](https://en.wikipedia.org/wiki/Long_mode)
* [signal](https://en.wikipedia.org/wiki/Unix_signal)
* [printk](https://en.wikipedia.org/wiki/Printk)
* [coprocessor](https://en.wikipedia.org/wiki/Coprocessor)
* [SIMD](https://en.wikipedia.org/wiki/SIMD)
* [Interrupt Stack Table](https://www.kernel.org/doc/Documentation/x86/x86_64/kernel-stacks)
* [PID](https://en.wikipedia.org/wiki/Process_identifier)
* [x87 FPU](https://en.wikipedia.org/wiki/X87)
* [control register](https://en.wikipedia.org/wiki/Control_register)
* [MMX](https://en.wikipedia.org/wiki/MMX_%28instruction_set%29)
* [Previous part](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-4.html)
