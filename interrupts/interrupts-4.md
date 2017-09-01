인터럽트와 인터럽트 핸들링. Part 4.
================================================================================

비초기(non-early) 인터럽트 게이트 초기화
--------------------------------------------------------------------------------

이것은 리눅스 커널에서 인터럽트와 예외 처리에 관련된 4 번째 파트이고 이전 [파트](https://github.com/daeseokyoun/linux-insides/blob/korean-trans/interrupts/interrupts-3.md)에서 우리는 [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/tree/master/arch/x86/kernel/traps.c)에 있는 초기 `#DB` 와 `#BP` 예외 처리를 살펴 보았다. 우리는 [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/tree/master/arch/x86/kernel/setup.c) 에 구현되어 있는 `setup_arch` 함수에서 불려진 `early_trap_init` 바로 다음에서 마무리 되었었다. 이 파트에서는 `x86_64` 를 위한 리눅스 커널의 인터럽트와 예외 처리에 대해 조금 더 자세히 살펴볼 것이고 지난 파트에서 부터 연결하여 진행할 것이다. 인터럽트와 예외 처리와 관련된 첫번째로 살펴볼 것은 `early_trap_pf_init` 함수를 통해 `#PF` 또는 [페이지 폴트](https://en.wikipedia.org/wiki/Page_fault) 처리를 위한 설정이다. 이제 시작해보자.

초기 페이지 폴트 핸들러
--------------------------------------------------------------------------------

`early_trap_pf_init` 함수는 [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/tree/master/arch/x86/kernel/traps.c) 에 구현되어 있다. 그것은 주어진 엔트리로 [Interrupt Descriptor Table](https://en.wikipedia.org/wiki/Interrupt_descriptor_table) 를 채우는 `set_intr_gate` 매크로를 사용한다.:

```C
void __init early_trap_pf_init(void)
{
#ifdef CONFIG_X86_64
         set_intr_gate(X86_TRAP_PF, page_fault);
#endif
}
```

이 매크로는 [arch/x86/include/asm/desc.h](https://github.com/torvalds/linux/tree/master/arch/x86/include/asm/desc.h) 에 정의되어 있다. 우리는 이미 이전 [파트](https://github.com/daeseokyoun/linux-insides/blob/korean-trans/interrupts/interrupts-3.md)에서 이와 같은 매크로를 보았다. 그 매크로는 `set_system_intr_gate` 와 `set_intr_gate_ist` 이다. 이 매크로는 주어진 벡터 번호를 `255`(벡터 번호 최대값) 보다 큰지 확인하고 `set_system_intr_gate` 와 `set_intr_gate_ist` 했던대로 `_set_gate` 함수를 호출한다.:

```C
#define set_intr_gate(n, addr)                                  \
do {                                                            \
        BUG_ON((unsigned)n > 0xFF);                             \
        _set_gate(n, GATE_INTERRUPT, (void *)addr, 0, 0,        \
                  __KERNEL_CS);                                 \
        _trace_set_gate(n, GATE_INTERRUPT, (void *)trace_##addr,\
                        0, 0, __KERNEL_CS__);                     \ //TODO 마지막 언더바 두개
} while (0)
```

`set_intr_gate` 매크로는 두개의 인자들을 받는다.:

* 인터럽트 벡터 번호
* 인터럽트 핸들러의 주소

우리의 경우에 그것들은:

* `X86_TRAP_PF` - `14`;
* `page_fault` - 인터럽트 핸들러 엔트리 포인트

`X86_TRAP_PF` 는 [arch/x86/include/asm/traprs.h](https://github.com/torvalds/linux/tree/master/arch/x86/include/asm/traprs.h) 에 정의된 enum 의 요소중 하나이다.:

```C
enum {
	...
	...
	...
	...
	X86_TRAP_PF,            /* 14, Page Fault */* // TODO 마지막 별하나
	...
	...
	...
}
```

`early_trap_pf_init` 함수가 호출 되었을 때, `set_intr_gate` 매크로는 페이지 폴트를 위한 핸들러와 함께 `IDT` 를 채우는 `_set_gate` 매크로의 호출로 확장 될 것이다. 이제 `page_fault` 의 구현을 살펴보자. `page_fault` 핸들러는 모든 예외 처리 핸들러가 있는 [arch/x86/kernel/entry_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/entry/entry_64.S) 어세블리 소스 코드 파일에 구현되어 있다. 살펴 보도록 하자.:

```assembly
trace_idtentry page_fault do_page_fault has_error_code=1
```

우리는 이전 [파트](https://github.com/daeseokyoun/linux-insides/blob/korean-trans/interrupts/interrupts-3.md) 에서 어떻게 `#DB` 와 `#BP` 핸들러가 정의되었는지 보았다. 그것들은 `idtentry` 매크로로 정의되었지만, 여기서는 `trace_idtentry` 를 볼 수 있을 것이다. 이 매크로는 같은 소스 파일에 정의되어 있으며 `CONFIG_TRACING` 커널 구성 옵션에 의존적이다.:

```assembly
#ifdef CONFIG_TRACING
.macro trace_idtentry sym do_sym has_error_code:req
idtentry trace(\sym) trace(\do_sym) has_error_code=\has_error_code
idtentry \sym \do_sym has_error_code=\has_error_code
.endm
#else
.macro trace_idtentry sym do_sym has_error_code:req
idtentry \sym \do_sym has_error_code=\has_error_code
.endm
#endif
```

우리는 예외 [추적](https://en.wikipedia.org/wiki/Tracing_%28software%29)에 관련해서는 살펴보지 않을 것이다. 만약, `CONFIG_TRACING` 가 설정되어 있지 않다면, 우리는 `trace_idtentry` 매크로는 단지 일반적으로 `idtentry` 매크로로 확장될 것이다. 우리는 이미 이전 [파트](https://github.com/daeseokyoun/linux-insides/blob/korean-trans/interrupts/interrupts-3.md) 에서 `idtentry` 매크로의 구현을 보았기에 `page_fault` 예외 핸들러 부터 살펴 보기로 한다.

`idtentry` 선언을 보게 되면, `page_fault` 의 핸들러는 [arch/x86/mm/fault.c](https://github.com/torvalds/linux/blob/master/arch/x86/mm/fault.c) 에 구현된 `do_page_fault` 함수이고 모든 예외처리 핸들러가 그러하듯 2 개의 인자를 받는다.:

* `regs` - `pt_regs` 구조체는 인터럽트된 프로세스의 상태를 갖고 있다.
* `error_code` - 페이지 폴트 예외의 에러 코드

이 함수의 내부를 살펴 보자. 맨 먼저 이 함수는 [cr2](https://en.wikipedia.org/wiki/Control_register) 컨트롤 레지스터의 내용을 읽는다.:

```C
dotraplinkage void notrace
do_page_fault(struct pt_regs *regs, unsigned long error_code)
{
	unsigned long address = read_cr2();
	...
	...
	...
}
```

이 레지스터는 `page fault`를 유발시켰던 선형 주소(linear address)를 갖고 있다. 다음 단계는 [include/linux/context_tracking.h](https://github.com/torvalds/linux/blob/master/include/context_tracking.h) 에 있는 `exception_enter` 함수를 호출한다. 여기서 `exception_enter` 와 `exception_exit` 가 있는데, 이들은 프로세서가 사용자 영역에서 수행 중에 타이머 틱(timer tick) 에 의존적인 부분을 제거하기 위해 [RCU](https://en.wikipedia.org/wiki/Read-copy-update) 에 의해 사용되어 지는 리눅스 커널의 문맥 추적 서브시스템(context tracking subsystem)에 존재하는 것이다. 거의 모든 예외 처리 핸들러는 아래과 같은 형태로 작성되어져 있을 것이다.:

```C
enum ctx_state prev_state;
prev_state = exception_enter();
...
... // exception handler here
...
exception_exit(prev_state);
```

`exception_enter` 함수는 `context_tracking_is_enabled` 함수로 `context tracking(문맥 추적)` 기능이 활성화 되어 있는지 확인하고, 만약 그것이 활성화 상태라면, 우리는 `this_cpu_read`(`this_cpu_*` 에 관하여 더 자세히 알고 싶다면, 이 [문서](https://github.com/torvalds/linux/blob/master/Documentation/this_cpu_ops.txt)를 읽어보기 바란다.) 함수를 통해 이전 문맥을 얻을 수 있다. 이후에 프로세서가 사용자 영역을 빠져나와 커널 영역으로 진입했다는 문맥 추적을 알려주는 `context_tracking_user_exit` 함수가 호출된다.:

```C
static inline enum ctx_state exception_enter(void)
{
        enum ctx_state prev_ctx;

        if (!context_tracking_is_enabled())
                return 0;

        prev_ctx = this_cpu_read(context_tracking.state);
        context_tracking_user_exit();

        return prev_ctx;
}
```

그 상태는 아래의 값 중 하나가 될 수 있다.:

```C
enum ctx_state {
  IN_KERNEL = 0,
	IN_USER,
} state;
```

그리고 마지막에는 이전 문맥(context)를 반환한다. `exception_enter` 와 `exception_exit` 사이에 우리는 실제 페이지 폴트 핸들러를 호출한다.:

```C
__do_page_fault(regs, error_code, address);
```

`__do_page_fault` 함수는 `do_page_fault` 함수와 같은 파일인 [arch/x86/mm/fault.c](https://github.com/torvalds/linux/blob/master/arch/x86/mm/fault.c)에 구현되어 있다. `__do_page_fault` 함수 시작은 [kmemcheck](https://www.kernel.org/doc/Documentation/kmemcheck.txt) 체커의 상태를 확인한다. `kmemcheck` 는 초기화 되지 않는 메모리의 어떤 사용에 대한 경고를 한다. 우리는 페이지 폴트가 kmemcheck 에 의해 야기되었을 수도 있기 때문에 확인할 필요가 있다.:

```C
if (kmemcheck_active(regs))
		kmemcheck_hide(regs);
	prefetchw(&mm->mmap_sem);
```

그다음에 우리는 전용 [cache line](https://en.wikipedia.org/wiki/CPU_cache) 을 얻기 위해 [X86_FEATURE_3DNOW](https://en.wikipedia.org/?title=3DNow!) 를 갖고 오는 같은 [이름](http://www.felixcloutier.com/x86/PREFETCHW.html)을 가지는 명령어를 수행하는 `prefetchw` 함수의 호출을 볼 수 있다. prefetching 의 주요 목적은 메모리 접근 지연을 줄이는 것이다. 다음 단계는 다음과 같은 조건으로 커널 영역에서 페이지 폴트가 난 것이 아니라는 것을 확인한다.:

```C
if (unlikely(fault_in_kernel_space(address))) {
...
...
...
}
```

`fault_in_kernel_space` 는 아래와 같이:

```C
static int fault_in_kernel_space(unsigned long address)
{
        return address >= TASK_SIZE_MAX;
}
```

`TASK_SIZE_MAX` 매크로는 아래와 같이 확장된다.:

```C
#define TASK_SIZE_MAX   ((1UL << 47) - PAGE_SIZE)
```

또는 `0x00007ffffffff000`. 여기서 `unlikely` 매크로에 주목하자. 리눅스 커널에는 두개의 매크로가 있다.:

```C
#define likely(x)      __builtin_expect(!!(x), 1)
#define unlikely(x)    __builtin_expect(!!(x), 0)
```

당신은 리눅스 커널 코드에서 이 매크로들을 종종 볼 수 있을 것이다. 이 매크로들의 주요 목적은 최적화이다. 이 상태는 코드의 상태를 확인할 때 필요하고 우리는 그것이 극히 드물에 `true` 이거나 `false`라는 것을 알고 있다는 것이다. 이 매크로들을 이용해서 컴파일러에 이러한 사실을 알려 줄 수 있는 것이다. 예를 들어,

```C
static int proc_root_readdir(struct file *file, struct dir_context *ctx)
{
        if (ctx->pos < FIRST_PROCESS_ENTRY) {
                int error = proc_readdir(file, ctx);
                if (unlikely(error <= 0))
                        return error;
...
...
...
}
```

여기서 우리는 리눅스 [VFS](https://en.wikipedia.org/wiki/Virtual_file_system) 가 `root` 디렉토리 내용을 읽을 필요가 있을 때, 호출 될 수 있는 `proc_root_readdir` 함수를 볼 수 있다. 만약 조건이 `unlikely` 와 함께 쓰였다면, 컴파일러는 `false`를 브랜치(branch) 하는 코드 바로 다음에 넣을 수 있다. 이제 우리의 주소 확인하는 부분으로 돌아가보자. 주어진 주소와 `0x00007ffffffff000` 사이의 비교는 우리가 알고 있듯이, 페이지 폴트가 커널 모드 혹은 사용자 모드에서 일어난 것인지 확인하는 것이다. 이런 확인 후에 `__do_page_fault` 함수는 페이지 폴트 예외가 야기된 문제를 이해하도록 시도하고 알맞은 루틴의 주소를 넘겨 줄 것이다. 그것은 `kmemcheck` 폴트, 가짜로 일어난 폴트, [kprobes](https://www.kernel.org/doc/Documentation/kprobes.txt) 폴트 등이 될 수 있다. 이 파트에서는 페이지 폴트 예외 처리 핸들러의 구체적인 구현을 살펴 보지 않을 것이다. 이유는 우리가 리눅스 커널에서 제공되는 많은 개념들을 알 필요가 있기 때문이다. 하지만 이 부분은 [메모리 관리](https://github.com/daeseokyoun/linux-insides/tree/korean-trans/mm) 에서 살펴 보도록 하자.

start_kernel로 복귀
--------------------------------------------------------------------------------

다른 커널 서브 시스템으로 부터 `setup_arch` 함수 내에 `early_trap_pf_init` 함수 후에 불리는 많은 다른 함수들이 존재한다. 하지만 인터럽트와 예외 처리에 관련된 것들은 하나도 없다. 그래서, 우리는 [init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c#L492) 에 있는 `start_kernel`로 돌아갈 필요가 있다. `setup_arch` 함수 이후에 첫번째로 하는 일은 [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/tree/master/arch/x86/kernel/traps.c) 에 있는 `trap_init` 함수이다. 이 함수는 남아 있는 예외 핸들러들의 초기화를 진행한다.(우리는 이미 `#DB` - debug exception, `#BP` - breakpoint exception, `#PF` - page fault exception 이 3 가지의 핸들러를 설정했다는 것을 기억하자.) `trap_init` 함수는 [Extended Industry Standard Architecture](https://en.wikipedia.org/wiki/Extended_Industry_Standard_Architecture) 를 확인하는 것 부터 시작한다.:

```C
#ifdef CONFIG_EISA
        void __iomem *p = early_ioremap(0x0FFFD9, 4);

        if (readl(p) == 'E' + ('I'<<8) + ('S'<<16) + ('A'<<24))
                EISA_bus = 1;
        early_iounmap(p, 4);
#endif
```

이것은 `EISA` 지원을 선택하는 `CONFIG_EISA` 커널 구성 파라미터에 의존적이라는 것을 기억하자. 여기서 우리는 `I/O` 메모리를 페이지 테이블로 맵핑하기 위해 `early_ioremap` 함수를 사용한다. 우리는 맵핑된 영역에서 첫 `4` 바이트를 읽기 위해 `readl` 함수를 사용하고 만약 `EISA` 라는 문자열과 같다면 우리는 `EISA_bus` 를 1로 설정한다. 마지막에는 맵핑했던 영역을 풀어준다. `early_ioremap` 에 조금 더 자세한 설명은 [고정 맵핑 주소와 ioremap](https://github.com/daeseokyoun/linux-insides/blob/korean-trans/mm/linux-mm-2.md) 에서 볼 수 있다.

이 다음에 우리는 다른 인터럽트 게이트를 갖고 `Interrupt Descriptor Table` 를 채운다. 이 모든 것 중에 첫번째는 `#DE` 또는 `Divide Error` 와 `#NMI` 또는 `Non-maskable Interrupt`를 설정하는 것이다.:

```C
set_intr_gate(X86_TRAP_DE, divide_error);
set_intr_gate_ist(X86_TRAP_NMI, &nmi, NMI_STACK);
```

우리는 `#DE` 예외를 위해 인터럽트 게이트를 설정하는 `set_intr_gate` 매크로를 사용하고 `#NMI`를 위해서는 `set_intr_gate_ist` 매크로를 사용한다. 당신은 우리가 이미 페이지 폴트 핸들러, 디버깅 핸들러 등을 위한 인터럽트 게이트를 설정했을 때 이 매크로를 사용했다는 것을 기억할 수 있을 것이다. 이 다음에는 아래에 기술할 예외들을 위한 예외 게이트를 설정한다.:

```C
set_system_intr_gate(X86_TRAP_OF, &overflow);
set_intr_gate(X86_TRAP_BR, bounds);
set_intr_gate(X86_TRAP_UD, invalid_op);
set_intr_gate(X86_TRAP_NM, device_not_available);
```

간략 설명을 하면,:

* `#OF` 또는 `Overflow` 예외. 이 예외는 특별한 [INTO](http://x86.renejeschke.de/html/file_module_x86_id_142.html) 명령어가 실행되었을 때 오버플로우 트랩이 발생했다는 것을 말한다.
* `#BR` 또는 `BOUND Range exceeded` 예외. 이 예외는 [BOUND](http://pdos.csail.mit.edu/6.828/2005/readings/i386/BOUND.htm) 명령이 실행되었을 때,`BOUND-range-exceed` 폴트가 발생하면 알려준다.
* `#UD` 또는 `Invalid Opcode` 예외. 프로세서가 유요하지 않거나 예약된 [opcode](https://en.wikipedia.org/?title=Opcode) 실행 및 유요하지 않는 피연산자와 함께 명령어를 수행하려고 하면 발생한다.
* `#NM` 또는 `Device Not Available` 예외.[control register](https://en.wikipedia.org/wiki/Control_register#CR0) `cr0`에서 `EM` 플래그가 설정되어 있는 동안에 프로세서가 `x87 FPU` floating point 명령어를 실행하려고 하면 발생한다.

다음 단계는 `#DF` 또는 `Double fault` 예외를 위한 인터럽트 게이트를 설정한다.:

```C
set_intr_gate_ist(X86_TRAP_DF, &double_fault, DOUBLEFAULT_STACK);
```

이 예외는 프로세서가 이전 예외를 위한 예외 처리 핸들러를 호출하는 동안에 두번째 예외가 검출되면 발생한다. 대게는 프로세서가 예외 핸들러를 호출하려고 하는 동안에 다른 예외가 검출이 되고, 두 개의 예외를 순차적으로 실행할 수 있다. 만약 프로세서가 순차적으로 처리하지 못한다고 하면, 그것은 double-fault 혹은 `#DF` 예외를 발생하게 된다.

인터럽트 게이트의 설정을 보자.:

```C
set_intr_gate(X86_TRAP_OLD_MF, &coprocessor_segment_overrun);
set_intr_gate(X86_TRAP_TS, &invalid_TSS);
set_intr_gate(X86_TRAP_NP, &segment_not_present);
set_intr_gate_ist(X86_TRAP_SS, &stack_segment, STACKFAULT_STACK);
set_intr_gate(X86_TRAP_GP, &general_protection);
set_intr_gate(X86_TRAP_SPURIOUS, &spurious_interrupt_bug);
set_intr_gate(X86_TRAP_MF, &coprocessor_error);
set_intr_gate(X86_TRAP_AC, &alignment_check);
```

여기서 우리는 다음과 같은 예외 핸들러를 위한 설정을 볼 수 있다.

* `#CSO` or `Coprocessor Segment Overrun` - this exception indicates that math [coprocessor](https://en.wikipedia.org/wiki/Coprocessor) of an old processor detected a page or segment violation. Modern processors do not generate this exception
* `#TS` or `Invalid TSS` exception - indicates that there was an error related to the [Task State Segment](https://en.wikipedia.org/wiki/Task_state_segment).
* `#NP` or `Segment Not Present` exception indicates that the `present flag` of a segment or gate descriptor is clear during attempt to load one of `cs`, `ds`, `es`, `fs`, or `gs` register.
* `#SS` or `Stack Fault` exception indicates one of the stack related conditions was detected, for example a not-present stack segment is detected when attempting to load the `ss` register.
* `#GP` or `General Protection` exception indicates that the processor detected one of a class of protection violations called general-protection violations. There are many different conditions that can cause general-protection exception. For example loading the `ss`, `ds`, `es`, `fs`, or `gs` register with a segment selector for a system segment, writing to a code segment or a read-only data segment, referencing an entry in the `Interrupt Descriptor Table` (following an interrupt or exception) that is not an interrupt, trap, or task gate and many many more.
* `Spurious Interrupt` - a hardware interrupt that is unwanted.
* `#MF` or `x87 FPU Floating-Point Error` exception caused when the [x87 FPU](https://en.wikipedia.org/wiki/X86_instruction_listings#x87_floating-point_instructions) has detected a floating point error.
* `#AC` or `Alignment Check` exception Indicates that the processor detected an unaligned memory operand when alignment checking was enabled.

After that we setup this exception gates, we can see setup of the `Machine-Check` exception:

```C
#ifdef CONFIG_X86_MCE
	set_intr_gate_ist(X86_TRAP_MC, &machine_check, MCE_STACK);
#endif
```

Note that it depends on the `CONFIG_X86_MCE` kernel configuration option and indicates that the processor detected an internal [machine error](https://en.wikipedia.org/wiki/Machine-check_exception) or a bus error, or that an external agent detected a bus error. The next exception gate is for the [SIMD](https://en.wikipedia.org/?title=SIMD) Floating-Point exception:

```C
set_intr_gate(X86_TRAP_XF, &simd_coprocessor_error);
```

which indicates the processor has detected an `SSE` or `SSE2` or `SSE3` SIMD floating-point exception. There are six classes of numeric exception conditions that can occur while executing an SIMD floating-point instruction:

* Invalid operation
* Divide-by-zero
* Denormal operand
* Numeric overflow
* Numeric underflow
* Inexact result (Precision)

In the next step we fill the `used_vectors` array which defined in the [arch/x86/include/asm/desc.h](https://github.com/torvalds/linux/tree/master/arch/x86/include/asm/desc.h) header file and represents `bitmap`:

```C
DECLARE_BITMAP(used_vectors, NR_VECTORS);
```

of the first `32` interrupts (more about bitmaps in the Linux kernel you can read in the part which describes [cpumasks and bitmaps](http://0xax.gitbooks.io/linux-insides/content/Concepts/cpumask.html))

```C
for (i = 0; i < FIRST_EXTERNAL_VECTOR; i++)
	set_bit(i, used_vectors)
```

where `FIRST_EXTERNAL_VECTOR` is:

```C
#define FIRST_EXTERNAL_VECTOR           0x20
```

After this we setup the interrupt gate for the `ia32_syscall` and add `0x80` to the `used_vectors` bitmap:

```C
#ifdef CONFIG_IA32_EMULATION
        set_system_intr_gate(IA32_SYSCALL_VECTOR, ia32_syscall);
        set_bit(IA32_SYSCALL_VECTOR, used_vectors);
#endif
```

There is `CONFIG_IA32_EMULATION` kernel configuration option on `x86_64` Linux kernels. This option provides ability to execute 32-bit processes in compatibility-mode. In the next parts we will see how it works, in the meantime we need only to know that there is yet another interrupt gate in the `IDT` with the vector number `0x80`. In the next step we maps `IDT` to the fixmap area:

```C
__set_fixmap(FIX_RO_IDT, __pa_symbol(idt_table), PAGE_KERNEL_RO);
idt_descr.address = fix_to_virt(FIX_RO_IDT);
```

and write its address to the `idt_descr.address` (more about fix-mapped addresses you can read in the second part of the [Linux kernel memory management](http://0xax.gitbooks.io/linux-insides/content/mm/linux-mm-2.html) chapter). After this we can see the call of the `cpu_init` function that defined in the [arch/x86/kernel/cpu/common.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/cpu/common.c). This function makes initialization of the all `per-cpu` state. In the beginning of the `cpu_init` we do the following things: First of all we wait while current cpu is initialized and than we call the `cr4_init_shadow` function which stores shadow copy of the `cr4` control register for the current cpu and load CPU microcode if need with the following function calls:

```C
wait_for_master_cpu(cpu);
cr4_init_shadow();
load_ucode_ap();
```

Next we get the `Task State Segment` for the current cpu and `orig_ist` structure which represents origin `Interrupt Stack Table` values with the:

```C
t = &per_cpu(cpu_tss, cpu);
oist = &per_cpu(orig_ist, cpu);
```

As we got values of the `Task State Segment` and `Interrupt Stack Table` for the current processor, we clear following bits in the `cr4` control register:

```C
cr4_clear_bits(X86_CR4_VME|X86_CR4_PVI|X86_CR4_TSD|X86_CR4_DE);
```

with this we disable `vm86` extension, virtual interrupts, timestamp ([RDTSC](https://en.wikipedia.org/wiki/Time_Stamp_Counter) can only be executed with the highest privilege) and debug extension. After this we reload the `Global Descriptor Table` and `Interrupt Descriptor table` with the:

```C
	switch_to_new_gdt(cpu);
	loadsegment(fs, 0);
	load_current_idt();
```

After this we setup array of the Thread-Local Storage Descriptors, configure [NX](https://en.wikipedia.org/wiki/NX_bit) and load CPU microcode. Now is time to setup and load `per-cpu` Task State Segments. We are going in a loop through the all exception stack which is `N_EXCEPTION_STACKS` or `4` and fill it with `Interrupt Stack Tables`:

```C
	if (!oist->ist[0]) {
		char *estacks = per_cpu(exception_stacks, cpu);

		for (v = 0; v < N_EXCEPTION_STACKS; v++) {
			estacks += exception_stack_sizes[v];
			oist->ist[v] = t->x86_tss.ist[v] =
					(unsigned long)estacks;
			if (v == DEBUG_STACK-1)
				per_cpu(debug_stack_addr, cpu) = (unsigned long)estacks;
		}
	}
```

As we have filled `Task State Segments` with the `Interrupt Stack Tables` we can set `TSS` descriptor for the current processor and load it with the:

```C
set_tss_desc(cpu, t);
load_TR_desc();
```

where `set_tss_desc` macro from the [arch/x86/include/asm/desc.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/desc.h) writes given  descriptor to the `Global Descriptor Table` of the given processor:

```C
#define set_tss_desc(cpu, addr) __set_tss_desc(cpu, GDT_ENTRY_TSS, addr)
static inline void __set_tss_desc(unsigned cpu, unsigned int entry, void *addr)
{
        struct desc_struct *d = get_cpu_gdt_table(cpu);
        tss_desc tss;
        set_tssldt_descriptor(&tss, (unsigned long)addr, DESC_TSS,
                              IO_BITMAP_OFFSET + IO_BITMAP_BYTES +
                              sizeof(unsigned long) - 1);
        write_gdt_entry(d, entry, &tss, DESC_TSS);
}
```

and `load_TR_desc` macro expands to the `ltr` or `Load Task Register` instruction:

```C
#define load_TR_desc()                          native_load_tr_desc()
static inline void native_load_tr_desc(void)
{
        asm volatile("ltr %w0"::"q" (GDT_ENTRY_TSS*8));
}
```

In the end of the `trap_init` function we can see the following code:

```C
set_intr_gate_ist(X86_TRAP_DB, &debug, DEBUG_STACK);
set_system_intr_gate_ist(X86_TRAP_BP, &int3, DEBUG_STACK);
...
...
...
#ifdef CONFIG_X86_64
        memcpy(&nmi_idt_table, &idt_table, IDT_ENTRIES * 16);
        set_nmi_gate(X86_TRAP_DB, &debug);
        set_nmi_gate(X86_TRAP_BP, &int3);
#endif
```

Here we copy `idt_table` to the `nmi_dit_table` and setup exception handlers for the `#DB` or `Debug exception` and `#BR` or `Breakpoint exception`. You can remember that we already set these interrupt gates in the previous [part](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-3.html), so why do we need to setup it again? We setup it again because when we initialized it before in the `early_trap_init` function, the `Task State Segment` was not ready yet, but now it is ready after the call of the `cpu_init` function.

That's all. Soon we will consider all handlers of these interrupts/exceptions.

Conclusion
--------------------------------------------------------------------------------

It is the end of the fourth part about interrupts and interrupt handling in the Linux kernel. We saw the initialization of the [Task State Segment](https://en.wikipedia.org/wiki/Task_state_segment) in this part and initialization of the different interrupt handlers as `Divide Error`, `Page Fault` exception and etc. You can note that we saw just initialization stuff, and will dive into details about handlers for these exceptions. In the next part we will start to do it.

If you have any questions or suggestions write me a comment or ping me at [twitter](https://twitter.com/0xAX).

**Please note that English is not my first language, And I am really sorry for any inconvenience. If you find any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-insides).**

Links
--------------------------------------------------------------------------------

* [page fault](https://en.wikipedia.org/wiki/Page_fault)
* [Interrupt Descriptor Table](https://en.wikipedia.org/wiki/Interrupt_descriptor_table)
* [Tracing](https://en.wikipedia.org/wiki/Tracing_%28software%29)
* [cr2](https://en.wikipedia.org/wiki/Control_register)
* [RCU](https://en.wikipedia.org/wiki/Read-copy-update)
* [this_cpu_* operations](https://github.com/torvalds/linux/blob/master/Documentation/this_cpu_ops.txt)
* [kmemcheck](https://www.kernel.org/doc/Documentation/kmemcheck.txt)
* [prefetchw](http://www.felixcloutier.com/x86/PREFETCHW.html)
* [3DNow](https://en.wikipedia.org/?title=3DNow!)
* [CPU caches](https://en.wikipedia.org/wiki/CPU_cache)
* [VFS](https://en.wikipedia.org/wiki/Virtual_file_system)
* [Linux kernel memory management](http://0xax.gitbooks.io/linux-insides/content/mm/index.html)
* [Fix-Mapped Addresses and ioremap](http://0xax.gitbooks.io/linux-insides/content/mm/linux-mm-2.html)
* [Extended Industry Standard Architecture](https://en.wikipedia.org/wiki/Extended_Industry_Standard_Architecture)
* [INT isntruction](https://en.wikipedia.org/wiki/INT_%28x86_instruction%29)
* [INTO](http://x86.renejeschke.de/html/file_module_x86_id_142.html)
* [BOUND](http://pdos.csail.mit.edu/6.828/2005/readings/i386/BOUND.htm)
* [opcode](https://en.wikipedia.org/?title=Opcode)
* [control register](https://en.wikipedia.org/wiki/Control_register#CR0)
* [x87 FPU](https://en.wikipedia.org/wiki/X86_instruction_listings#x87_floating-point_instructions)
* [MCE exception](https://en.wikipedia.org/wiki/Machine-check_exception)
* [SIMD](https://en.wikipedia.org/?title=SIMD)
* [cpumasks and bitmaps](http://0xax.gitbooks.io/linux-insides/content/Concepts/cpumask.html)
* [NX](https://en.wikipedia.org/wiki/NX_bit)
* [Task State Segment](https://en.wikipedia.org/wiki/Task_state_segment)
* [Previous part](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-3.html)
