인터럽트와 인터럽트 핸들링. Part 6.
================================================================================

Non-maskable 인터럽트 핸들러
--------------------------------------------------------------------------------

이 파트는 [인터럽트와 인터럽트 처리](https://github.com/daeseokyoun/linux-insides/blob/korean-trans/interrupts/README.md)의 6번째이고 이전 [파트](https://github.com/daeseokyoun/linux-insides/blob/korean-trans/interrupts/interrupts-5.md) 에서 [General Protection Fault](https://en.wikipedia.org/wiki/General_protection_fault) 예외, 0으로 나누기 예외, 유효하지 않는 [opcode](https://en.wikipedia.org/wiki/Opcode) 예외를 위한 핸들러의 구현을 보았다. 이전 파트에서 언급했지만, 우리는 이 파트에서 나머지 예외의 구현에 대해 볼 것이다. 우리가 볼 핸들러의 구현은 다음과 같다:

* [Non-Maskable](https://en.wikipedia.org/wiki/Non-maskable_interrupt) 인터럽트;
* [BOUND](http://pdos.csail.mit.edu/6.828/2005/readings/i386/BOUND.htm) 범위 초과 예외;
* [Coprocessor](https://en.wikipedia.org/wiki/Coprocessor) 예외;
* [SIMD](https://en.wikipedia.org/wiki/SIMD) coprocessor 예외.

시작해보자.

Non-Maskable(넌 마스커블) 인터럽트 핸들러
--------------------------------------------------------------------------------

하나의 [Non-Maskable](https://en.wikipedia.org/wiki/Non-maskable_interrupt) 인터럽트는 표준의 마스킹 기술들에 의해 무시될 수 없는 하드웨어 인터럽트이다. 일반적인 방법으로, non-maskable 인터럽트는 두 가지 방법으로 생성될 수 있다.:

* 외부 하드웨어 자원들이 CPU 의 non-maskable 인터럽트 [pin](https://en.wikipedia.org/wiki/CPU_socket) 으로 발생된다.
* 그 프로세서는 시스템 버스나 배달(delivery) 모드 `NMI` 와 함께 APIC 시리얼 버스로 부터 받는다.

프로세서가 위에서 설명한 것 중에 하나로 부터 `NMI` 를 받으면, 프로세서는 인터럽트 벡터 2번이 가리키는 `NMI` 핸들러를 호출 함으로써 즉시 처리한다.(인터럽트 번호는 인터럽트의 첫번째 [파트](https://github.com/daeseokyoun/linux-insides/blob/korean-trans/interrupts/interrupts-1.md)에서 확인하자) 우리는 이미 [Interrupt Descriptor Table](https://en.wikipedia.org/wiki/Interrupt_descriptor_table)를 [vector number](https://en.wikipedia.org/wiki/Interrupt_vector_table), `nmi` 인터럽트 핸들러의 주소와 `NMI_STACK` [Interrupt Stack Table entry](https://github.com/torvalds/linux/blob/master/Documentation/x86/kernel-stacks)로 채웠다:

```C
set_intr_gate_ist(X86_TRAP_NMI, &nmi, NMI_STACK);
```

[arch/x86/kernel/traps.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/traps.c) 소스 코드 파일에 정의된 `trap_init` 함수내에서 수행한다. 이전 파트들에서 우리는 모든 인터럽트 핸들러의 엔트리 포인트들이 아래와 같이 선언되어 있었다는 것을 살펴 보았었다.:

```assembly
.macro idtentry sym do_sym has_error_code:req paranoid=0 shift_ist=-1
ENTRY(\sym)
...
...
...
END(\sym)
.endm
```

매크로는 [arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/entry/entry_64.S) 어셈블리 소스 파일에 있다. 하지만, `Non-Maskable` 인터럽트 핸들러는 이 매크로와 정의되지 않았다. 그것은 자신만의 엔트리 포인트를 가진다.:

```assembly
ENTRY(nmi)
...
...
...
END(nmi)
```

같은 [arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/entry/entry_64.S) 어셈블리 파일에 있다. 이제 어떻게 `Non-Maskable` 인터럽트 핸들러가 동작하는지 더 자세히 살펴보도록 하자. `nmi` 핸들러는 아래의 호출로 부터 시작한다.:

```assembly
PARAVIRT_ADJUST_EXCEPTION_FRAME
```

우리는 이 부분에 대해서는 더 자세히 살펴보지 않을 것이다. 이유는 이 매크로는 다른 챕터에서 볼 [Paravirtualization](https://en.wikipedia.org/wiki/Paravirtualization)에 관련된 것이기 때문이다. 이 호출 다음에 `rdx` 레지스터의 내용을 스택에 저장한다.:

```assembly
pushq	%rdx
```

그리고 non-maskable 인터럽트가 발생했을 때, `cs` 가 커널 세그먼트인지 검사한다.:

```assembly
cmpl	$__KERNEL_CS, 16(%rsp)
jne	first_nmi
```

`__KERNEL_CS` 는 [arch/x86/include/asm/segment.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/segment.h) 에 선언되어 있고 [Global Descriptor Table](https://en.wikipedia.org/wiki/Global_Descriptor_Table) 에서 두 번째 디스크립터를 나타낸다.:

```C
#define GDT_ENTRY_KERNEL_CS	2
#define __KERNEL_CS	(GDT_ENTRY_KERNEL_CS*8)
```

`GDT` 에 대한 자세한 내용은 리눅스 커널 부팅 과정 챕터에서 두 번째 [파트](https://github.com/daeseokyoun/linux-insides/blob/korean-trans/Initialization/linux-initialization-2.md) 에서 볼 수 있다. 만약 `cs` 가 커널 세그먼트가 아니라면, 그것은 중첩된 `NMI` 가 아니라는 의미이며, 우리는 `first_nmi` 라벨로 점프한다. 이 경우를 고려해보자. 무엇보다 우선은 우리가 현재 스택 포인터의 주소를 `rdx`에 넣고 `1` 을 `first_nmi` 라벨에서 스택에 넣는다.:

```assembly
first_nmi:
	movq	(%rsp), %rdx
	pushq	$1
```

왜 `1` 을 스택에 넣을까? 거기에 comment 는 `우리는 NMI 들 내에 브레이크 포인트를 허가한다(We allow breakpoints in NMIs)`게 있다. [x86_64](https://en.wikipedia.org/wiki/X86-64)에서는, 다른 아키텍처와 마찬가지로, CPU 는 첫 `NMI` 가 완료되기 전까지는 다른 `NMI`를 실행하지 않을 것이다. 하나의 `NMI` 인터럽트는 다른 명령어와 예외 처리가 하는 것 처럼 [iret](http://faydoc.tripod.com/cpu/iret.htm) 명령와 함께 마무리 된다. 만약 `NMI` 핸들러가 [페이지 폴트](https://en.wikipedia.org/wiki/Page_fault) 또는 [breakpoint](https://en.wikipedia.org/wiki/Breakpoint) 또는 `iret` 명령어를 사용하는 다른 예외와 같다고 하고 `NMI` 문맥이 실행중에 이런 일들이(Page fault 등) 일어난다면, CPU 는 즉시 `NMI` 문맥을 떠나 다른 `NMI` 가 수행될 것이다. 이런 예외들이 복귀할 때 사용되는 `iret` 는 `NMIs`를 재진입할 수 있을 것이고, nested non-maskable 인터럽트를 얻을 수 있을 것이다. 이문제는 예외가 발생하면, `NMI` 핸들러가 그것의 상태가 어땠는지 반환하지 않을 것이고, 대신에 수행주인 `NMI` 해들러를 새로운 `NMI`에게 허용하는 상태를 반환할 것이다. 만약 다른 `NMI`가 첫 NMI 핸들러가 수행완료 전에 온다면, 새로운 NMI 는 선점된 `NMIs` 스택을 덮어 쓸것이다. 우리는 다음 `NMI`를 이전 `NMI` 의 스택의 꼭대기에 사용하도록 하여 nested `NMIs` 를 가질 수 있다. 그것은 우리가 그것을 실행할 수 없다는 의미이다. 왜냐하면 하나의 nested non-maskable 인터럽트는 이전 non-maskable 인터럽트의 스택을 더럽힐 것이기 때문이다. 그렇기 때문에 우리가 임시 변수를 위해 스택의 공간에 할당하는 것이다. 우리는 이 값을 이전 `NMI`가 실행될 때 확인 할 것이고 만약 nested `NMI`가 아니라면 클리어 해줄 것이다. 우리가 여기에 `1`을 넣는 것은 `non-maskable` 인터럽트가 현재 실행되었다는 것을 알리기 위해스택에 할당된 공간이 이전에 있었다는 것을 위해서 이다. `NMI` 또는 다은 예외는 다음과 같은 [스택 프레임](https://en.wikipedia.org/wiki/Call_stack)를 가지고 사용한다.:

```
+------------------------+
|         SS             |
|         RSP            |
|        RFLAGS          |
|         CS             |
|         RIP            |
+------------------------+
```

and also an error code if an exception has it. So, after all of these manipulations our stack frame will look like this:

```
+------------------------+
|         SS             |
|         RSP            |
|        RFLAGS          |
|         CS             |
|         RIP            |
|         RDX            |
|          1             |
+------------------------+
```

In the next step we allocate yet another `40` bytes on the stack:

```assembly
subq	$(5*8), %rsp
```

and pushes the copy of the original stack frame after the allocated space:

```C
.rept 5
pushq	11*8(%rsp)
.endr
```

with the [.rept](http://tigcc.ticalc.org/doc/gnuasm.html#SEC116) assembly directive. We need in the copy of the original stack frame. Generally we need in two copies of the interrupt stack. First is `copied` interrupts stack: `saved` stack frame and `copied` stack frame. Now we pushes original stack frame to the `saved` stack frame which locates after the just allocated `40` bytes (`copied` stack frame). This stack frame is used to fixup the `copied` stack frame that a nested NMI may change. The second - `copied` stack frame modified by any nested `NMIs` to let the first `NMI` know that we triggered a second `NMI` and we should repeat the first `NMI` handler. Ok, we have made first copy of the original stack frame, now time to make second copy:

```assembly
addq	$(10*8), %rsp

.rept 5
pushq	-6*8(%rsp)
.endr
subq	$(5*8), %rsp
```

After all of these manipulations our stack frame will be like this:

```
+-------------------------+
| original SS             |
| original Return RSP     |
| original RFLAGS         |
| original CS             |
| original RIP            |
+-------------------------+
| temp storage for rdx    |
+-------------------------+
| NMI executing variable  |
+-------------------------+
| copied SS               |
| copied Return RSP       |
| copied RFLAGS           |
| copied CS               |
| copied RIP              |
+-------------------------+
| Saved SS                |
| Saved Return RSP        |
| Saved RFLAGS            |
| Saved CS                |
| Saved RIP               |
+-------------------------+
```

After this we push dummy error code on the stack as we did it already in the previous exception handlers and allocate space for the general purpose registers on the stack:

```assembly
pushq	$-1
ALLOC_PT_GPREGS_ON_STACK
```

We already saw implementation of the `ALLOC_PT_GREGS_ON_STACK` macro in the third part of the interrupts [chapter](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-3.html). This macro defined in the [arch/x86/entry/calling.h](https://github.com/torvalds/linux/blob/master/arch/x86/entry/calling.h) and yet another allocates `120` bytes on stack for the general purpose registers, from the `rdi` to the `r15`:

```assembly
.macro ALLOC_PT_GPREGS_ON_STACK addskip=0
addq	$-(15*8+\addskip), %rsp
.endm
```

After space allocation for the general registers we can see call of the `paranoid_entry`:

```assembly
call	paranoid_entry
```

We can remember from the previous parts this label. It pushes general purpose registers on the stack, reads `MSR_GS_BASE` [Model Specific register](https://en.wikipedia.org/wiki/Model-specific_register) and checks its value. If the value of the `MSR_GS_BASE` is negative, we came from the kernel mode and just return from the `paranoid_entry`, in other way it means that we came from the usermode and need to execute `swapgs` instruction which will change user `gs` with the kernel `gs`:

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

Note that after the `swapgs` instruction we zeroed the `ebx` register. Next time we will check content of this register and if we executed `swapgs` than `ebx` must contain `0` and `1` in other way. In the next step we store value of the `cr2` [control register](https://en.wikipedia.org/wiki/Control_register) to the `r12` register, because the `NMI` handler can cause `page fault` and corrupt the value of this control register:

```C
movq	%cr2, %r12
```

Now time to call actual `NMI` handler. We push the address of the `pt_regs` to the `rdi`, error code to the `rsi` and call the `do_nmi` handler:

```assembly
movq	%rsp, %rdi
movq	$-1, %rsi
call	do_nmi
```

We will back to the `do_nmi` little later in this part, but now let's look what occurs after the `do_nmi` will finish its execution. After the `do_nmi` handler will be finished we check the `cr2` register, because we can got page fault during `do_nmi` performed and if we got it we restore original `cr2`, in other way we jump on the label `1`. After this we test content of the `ebx` register (remember it must contain `0` if we have used `swapgs` instruction and `1` if we didn't use it) and execute `SWAPGS_UNSAFE_STACK` if it contains `1` or jump to the `nmi_restore` label. The `SWAPGS_UNSAFE_STACK` macro just expands to the `swapgs` instruction. In the `nmi_restore` label we restore general purpose registers, clear allocated space on the stack for this registers, clear our temporary variable and exit from the interrupt handler with the `INTERRUPT_RETURN` macro:

```assembly
	movq	%cr2, %rcx
	cmpq	%rcx, %r12
	je	1f
	movq	%r12, %cr2
1:
	testl	%ebx, %ebx
	jnz	nmi_restore
nmi_swapgs:
	SWAPGS_UNSAFE_STACK
nmi_restore:
	RESTORE_EXTRA_REGS
	RESTORE_C_REGS
	/* Pop the extra iret frame at once */
	REMOVE_PT_GPREGS_FROM_STACK 6*8
	/* Clear the NMI executing stack variable */
	movq	$0, 5*8(%rsp)
	INTERRUPT_RETURN
```

where `INTERRUPT_RETURN` is defined in the [arch/x86/include/irqflags.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/irqflags.h) and just expands to the `iret` instruction. That's all.

Now let's consider case when another `NMI` interrupt occurred when previous `NMI` interrupt didn't finish its execution. You can remember from the beginning of this part that we've made a check that we came from userspace and jump on the `first_nmi` in this case:

```assembly
cmpl	$__KERNEL_CS, 16(%rsp)
jne	first_nmi
```

Note that in this case it is first `NMI` every time, because if the first `NMI` catched page fault, breakpoint or another exception it will be executed in the kernel mode. If we didn't come from userspace, first of all we test our temporary variable:

```assembly
cmpl	$1, -8(%rsp)
je	nested_nmi
```

and if it is set to `1` we jump to the `nested_nmi` label. If it is not `1`, we test the `IST` stack. In the case of nested `NMIs` we check that we are above the `repeat_nmi`. In this case we ignore it, in other way we check that we above than `end_repeat_nmi` and jump on the `nested_nmi_out` label.

Now let's look on the `do_nmi` exception handler. This function defined in the [arch/x86/kernel/nmi.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/nmi.c) source code file and takes two parameters:

* address of the `pt_regs`;
* error code.

as all exception handlers. The `do_nmi` starts from the call of the `nmi_nesting_preprocess` function and ends with the call of the `nmi_nesting_postprocess`. The `nmi_nesting_preprocess` function checks that we likely do not work with the debug stack and if we on the debug stack set the `update_debug_stack` [per-cpu](http://0xax.gitbooks.io/linux-insides/content/Concepts/per-cpu.html) variable to `1` and call the `debug_stack_set_zero` function from the [arch/x86/kernel/cpu/common.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/cpu/common.c). This function increases the `debug_stack_use_ctr` per-cpu variable and loads new `Interrupt Descriptor Table`:

```C
static inline void nmi_nesting_preprocess(struct pt_regs *regs)
{
        if (unlikely(is_debug_stack(regs->sp))) {
                debug_stack_set_zero();
                this_cpu_write(update_debug_stack, 1);
        }
}
```

The `nmi_nesting_postprocess` function checks the `update_debug_stack` per-cpu variable which we set in the `nmi_nesting_preprocess` and resets debug stack or in another words it loads origin `Interrupt Descriptor Table`. After the call of the `nmi_nesting_preprocess` function, we can see the call of the `nmi_enter` in the `do_nmi`. The `nmi_enter` increases `lockdep_recursion` field of the interrupted process, update preempt counter and informs the [RCU](https://en.wikipedia.org/wiki/Read-copy-update) subsystem about `NMI`. There is also `nmi_exit` function that does the same stuff as `nmi_enter`, but vice-versa. After the `nmi_enter` we increase `__nmi_count` in the `irq_stat` structure and call the `default_do_nmi` function. First of all in the `default_do_nmi` we check the address of the previous nmi and update address of the last nmi to the actual:

```C
if (regs->ip == __this_cpu_read(last_nmi_rip))
    b2b = true;
else
    __this_cpu_write(swallow_nmi, false);

__this_cpu_write(last_nmi_rip, regs->ip);
```

After this first of all we need to handle CPU-specific `NMIs`:

```C
handled = nmi_handle(NMI_LOCAL, regs, b2b);
__this_cpu_add(nmi_stats.normal, handled);
```

And then non-specific `NMIs` depends on its reason:

```C
reason = x86_platform.get_nmi_reason();
if (reason & NMI_REASON_MASK) {
	if (reason & NMI_REASON_SERR)
		pci_serr_error(reason, regs);
	else if (reason & NMI_REASON_IOCHK)
		io_check_error(reason, regs);

	__this_cpu_add(nmi_stats.external, 1);
	return;
}
```

That's all.

Range Exceeded Exception
--------------------------------------------------------------------------------

The next exception is the `BOUND` range exceeded exception. The `BOUND` instruction determines if the first operand (array index) is within the bounds of an array specified the second operand (bounds operand). If the index is not within bounds, a `BOUND` range exceeded exception or `#BR` is occurred. The handler of the `#BR` exception is the `do_bounds` function that defined in the [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/traps.c). The `do_bounds` handler starts with the call of the `exception_enter` function and ends with the call of the `exception_exit`:

```C
prev_state = exception_enter();

if (notify_die(DIE_TRAP, "bounds", regs, error_code,
	           X86_TRAP_BR, SIGSEGV) == NOTIFY_STOP)
    goto exit;
...
...
...
exception_exit(prev_state);
return;
```

After we have got the state of the previous context, we add the exception to the `notify_die` chain and if it will return `NOTIFY_STOP` we return from the exception. More about notify chains and the `context tracking` functions you can read in the [previous part](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-5.html). In the next step we enable interrupts if they were disabled with the `contidional_sti` function that checks `IF` flag and call the `local_irq_enable` depends on its value:

```C
conditional_sti(regs);

if (!user_mode(regs))
	die("bounds", regs, error_code);
```

and check that if we didn't came from user mode we send `SIGSEGV` signal with the `die` function. After this we check is [MPX](https://en.wikipedia.org/wiki/Intel_MPX) enabled or not, and if this feature is disabled we jump on the `exit_trap` label:

```C
if (!cpu_feature_enabled(X86_FEATURE_MPX)) {
	goto exit_trap;
}

where we execute `do_trap` function (more about it you can find in the previous part):

```C
exit_trap:
	do_trap(X86_TRAP_BR, SIGSEGV, "bounds", regs, error_code, NULL);
	exception_exit(prev_state);
```

If `MPX` feature is enabled we check the `BNDSTATUS` with the `get_xsave_field_ptr` function and if it is zero, it means that the `MPX` was not responsible for this exception:

```C
bndcsr = get_xsave_field_ptr(XSTATE_BNDCSR);
if (!bndcsr)
		goto exit_trap;
```

After all of this, there is still only one way when `MPX` is responsible for this exception. We will not dive into the details about Intel Memory Protection Extensions in this part, but will see it in another chapter.

Coprocessor exception and SIMD exception
--------------------------------------------------------------------------------

The next two exceptions are [x87 FPU](https://en.wikipedia.org/wiki/X87) Floating-Point Error exception or `#MF` and [SIMD](https://en.wikipedia.org/wiki/SIMD) Floating-Point Exception or `#XF`. The first exception occurs when the `x87 FPU` has detected floating point error. For example divide by zero, numeric overflow and etc. The second exception occurs when the processor has detected [SSE/SSE2/SSE3](https://en.wikipedia.org/wiki/SSE3) `SIMD` floating-point exception. It can be the same as for the `x87 FPU`. The handlers for these exceptions are `do_coprocessor_error` and `do_simd_coprocessor_error` are defined in the [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/traps.c) and very similar on each other. They both make a call of the `math_error` function from the same source code file but pass different vector number. The `do_coprocessor_error` passes `X86_TRAP_MF` vector number to the `math_error`:

```C
dotraplinkage void do_coprocessor_error(struct pt_regs *regs, long error_code)
{
	enum ctx_state prev_state;

	prev_state = exception_enter();
	math_error(regs, error_code, X86_TRAP_MF);
	exception_exit(prev_state);
}
```

and `do_simd_coprocessor_error` passes `X86_TRAP_XF` to the `math_error` function:

```C
dotraplinkage void
do_simd_coprocessor_error(struct pt_regs *regs, long error_code)
{
	enum ctx_state prev_state;

	prev_state = exception_enter();
	math_error(regs, error_code, X86_TRAP_XF);
	exception_exit(prev_state);
}
```

First of all the `math_error` function defines current interrupted task, address of its fpu, string which describes an exception, add it to the `notify_die` chain and return from the exception handler if it will return `NOTIFY_STOP`:

```C
	struct task_struct *task = current;
	struct fpu *fpu = &task->thread.fpu;
	siginfo_t info;
	char *str = (trapnr == X86_TRAP_MF) ? "fpu exception" :
						"simd exception";

	if (notify_die(DIE_TRAP, str, regs, error_code, trapnr, SIGFPE) == NOTIFY_STOP)
		return;
```

After this we check that we are from the kernel mode and if yes we will try to fix an excetpion with the `fixup_exception` function. If we cannot we fill the task with the exception's error code and vector number and die:

```C
if (!user_mode(regs)) {
	if (!fixup_exception(regs)) {
		task->thread.error_code = error_code;
		task->thread.trap_nr = trapnr;
		die(str, regs, error_code);
	}
	return;
}
```

If we came from the user mode, we save the `fpu` state, fill the task structure with the vector number of an exception and `siginfo_t` with the number of signal, `errno`, the address where exception occurred and signal code:

```C
fpu__save(fpu);

task->thread.trap_nr	= trapnr;
task->thread.error_code = error_code;
info.si_signo		= SIGFPE;
info.si_errno		= 0;
info.si_addr		= (void __user *)uprobe_get_trap_addr(regs);
info.si_code = fpu__exception_code(fpu, trapnr);
```

After this we check the signal code and if it is non-zero we return:

```C
if (!info.si_code)
	return;
```

Or send the `SIGFPE` signal in the end:

```C
force_sig_info(SIGFPE, &info, task);
```

That's all.

Conclusion
--------------------------------------------------------------------------------

It is the end of the sixth part of the [Interrupts and Interrupt Handling](http://0xax.gitbooks.io/linux-insides/content/interrupts/index.html) chapter and we saw implementation of some exception handlers in this part, like `non-maskable` interrupt, [SIMD](https://en.wikipedia.org/wiki/SIMD) and [x87 FPU](https://en.wikipedia.org/wiki/X87) floating point exception. Finally we have finsihed with the `trap_init` function in this part and will go ahead in the next part. The next our point is the external interrupts and the `early_irq_init` function from the [init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c).

If you have any questions or suggestions write me a comment or ping me at [twitter](https://twitter.com/0xAX).

**Please note that English is not my first language, And I am really sorry for any inconvenience. If you find any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-insides).**

Links
--------------------------------------------------------------------------------

* [General Protection Fault](https://en.wikipedia.org/wiki/General_protection_fault)
* [opcode](https://en.wikipedia.org/wiki/Opcode)
* [Non-Maskable](https://en.wikipedia.org/wiki/Non-maskable_interrupt)
* [BOUND instruction](http://pdos.csail.mit.edu/6.828/2005/readings/i386/BOUND.htm)
* [CPU socket](https://en.wikipedia.org/wiki/CPU_socket)
* [Interrupt Descriptor Table](https://en.wikipedia.org/wiki/Interrupt_descriptor_table)
* [Interrupt Stack Table](https://github.com/torvalds/linux/blob/master/Documentation/x86/kernel-stacks)
* [Paravirtualization](https://en.wikipedia.org/wiki/Paravirtualization)
* [.rept](http://tigcc.ticalc.org/doc/gnuasm.html#SEC116)
* [SIMD](https://en.wikipedia.org/wiki/SIMD)
* [Coprocessor](https://en.wikipedia.org/wiki/Coprocessor)
* [x86_64](https://en.wikipedia.org/wiki/X86-64)
* [iret](http://faydoc.tripod.com/cpu/iret.htm)
* [page fault](https://en.wikipedia.org/wiki/Page_fault)
* [breakpoint](https://en.wikipedia.org/wiki/Breakpoint)
* [Global Descriptor Table](https://en.wikipedia.org/wiki/Global_Descriptor_Table)
* [stack frame](https://en.wikipedia.org/wiki/Call_stack)
* [Model Specific regiser](https://en.wikipedia.org/wiki/Model-specific_register)
* [percpu](http://0xax.gitbooks.io/linux-insides/content/Concepts/per-cpu.html)
* [RCU](https://en.wikipedia.org/wiki/Read-copy-update)
* [MPX](https://en.wikipedia.org/wiki/Intel_MPX)
* [x87 FPU](https://en.wikipedia.org/wiki/X87)
* [Previous part](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-5.html)
