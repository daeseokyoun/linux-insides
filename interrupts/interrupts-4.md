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
                        0, 0, __KERNEL_CS);                     \
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
	X86_TRAP_PF,            /* 14, Page Fault */
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

* `#CSO` 또는 `Coprocessor Segment Overrun` - 이 예외는 예전 프로세서의 수치 연산 보조 프로세서가 페이지 또는 세그먼트 위반을 감지했음을 나타낸다. 현대의 프로세서들은 이런 예외는 발생시키지 않는다.
* `#TS` 또는 `Invalid TSS` 예외 - [Task State5^ Segment](https://en.wikipedia.org/wiki/Task_state_segment) 현관된 에러가 있었다는 것을 나타낸다.
* `#NP` 또는 `Segment Not Present` 예외 - `cs`, `ds`, `es`, `fs`, 또는 `gs` 레지스터 중 하나가 로드가 시도되는 과정에서 세그먼트나 게이트 디스크립터의 `present flag` 가 클리어 되었음을 나타낸다.
* `#SS` 또는 `Stack Fault` 예외 - 스택과 관련된 상태가 감지되었을때 발생한다, 예를 들면 `ss` 레지스터가 로드되려 할때, 존재하지 않는 스택 세그먼트가 감지되었을 때 발생한다.
* `#GP` 또는 `General Protection` 예외 - 일반-보호(general-protection) 위반이라 불리는 보호 계열 중 하나가 감지되었음을 나타낸다. 일반 보호 예외에는 다양한 상태들이 존재한다. 예를 들어 시스템 세그먼크를 위한 세그먼트 셀렉터와 함께 `ss`, `ds`, `es`, `fs`, 또는 `gs` 레지스터를 로딩, 읽기 전용 데이터 세그먼트에 쓰기, 인터럽트가 아닌 상태에서 `Interrupt Descriptor Table` 내에 엔트리를 참조 등이 있다.
* `Spurious Interrupt` - 원치 않는 하드웨어 인터럽트
* `#MF` 또는 `x87 FPU Floating-Point Error` 예외 - [x87 FPU](https://en.wikipedia.org/wiki/X86_instruction_listings#x87_floating-point_instructions) 에서 floating point 에러가 검출되면 발생
* `#AC` 또는 `Alignment Check` 예외 - 정렬 확인(alignment checking) 이 활성화 되어 있을때 정렬되지 않는 메모리 피 연산자가 검출됨을 나타낸다.

이 예외 게이트들을 설정하고 나면, 우리는 `Machine-Check` 예외를 설정한다.

```C
#ifdef CONFIG_X86_MCE
	set_intr_gate_ist(X86_TRAP_MC, &machine_check, MCE_STACK);
#endif
```

`CONFIG_X86_MCE` 커널 구성 옵션에 의존적이라는 것과 내부 [machine error](https://en.wikipedia.org/wiki/Machine-check_exception) 또는 버스 에러(외부 에이전트가 버스 에러를 검출)가 검출하는 것을 알린다. 다음 예외 게이트는 [SIMD](https://en.wikipedia.org/?title=SIMD) floating-point 예외이다.:

```C
set_intr_gate(X86_TRAP_XF, &simd_coprocessor_error);
```

프로세서가 `SSE` or `SSE2` or `SSE3` SIMD floating-point 예외가 검출함을 나타낸다. 여기서 SIMD floating-point 명령어를 수행하는 동안에 발생할 수 있는 숫자의 예외 상태를 가리키는 6개의 클래스가 있다.:

* Invalid operation (유효하지 않는 연산)
* Divide-by-zero (0 으로 나누기)
* Denormal operand (비정상적 피연산자)
* Numeric overflow (숫자 오버플로우)
* Numeric underflow (숫자 언더플로우)
* Inexact result (Precision) (부정확한 결과)

다음 단계는 [arch/x86/include/asm/desc.h](https://github.com/torvalds/linux/tree/master/arch/x86/include/asm/desc.h) 헤더 파일에 정의된 `used_vectors` 배열을 채우는데 이는 `bitmap` 의 형태로 관리된다.:

```C
DECLARE_BITMAP(used_vectors, NR_VECTORS);
```

첫 `32` 개의 인터럽트이다. (리눅스 커널에서 이 비트맵에 관한 자세한 내용은, [cpumask and bitmaps](https://github.com/daeseokyoun/linux-insides/blob/korean-trans/Concepts/cpumask.md) 에서 볼수 있다.)

```C
for (i = 0; i < FIRST_EXTERNAL_VECTOR; i++)
	set_bit(i, used_vectors)
```

`FIRST_EXTERNAL_VECTOR` 은 아래의 값과 같다.:

```C
#define FIRST_EXTERNAL_VECTOR           0x20
```

이 다음에는 `ia32_syscall`을 위한 인터럽트 게이트를 설정하고 `used_vectors` 비트맵에 `0x80` 을 더한다.:

```C
#ifdef CONFIG_IA32_EMULATION
        set_system_intr_gate(IA32_SYSCALL_VECTOR, ia32_syscall);
        set_bit(IA32_SYSCALL_VECTOR, used_vectors);
#endif
```

여기서 `CONFIG_IA32_EMULATION` 커널 구성 옵션은 `x86_64` 리눅스 커널에 있는 것이다. 이 옵션은 호완 모드로써 32 비트 프로세스들을 실행할 수 있는 기능을 제공한다. 다음 파트에서 이 것이 어떻게 동작하는지 볼 것이며, 그 동안 우리는 단지 `IDT` 에 벡터 번호 `0x80` 으로 다른 인터럽트 게이트가 있다는 것만 알면된다. 다음 단계에서 우리는 `IDT`를 fixmap 영역으로 맵핑한다.:

```C
__set_fixmap(FIX_RO_IDT, __pa_symbol(idt_table), PAGE_KERNEL_RO);
idt_descr.address = fix_to_virt(FIX_RO_IDT);
```

그리고 그것의 주소를 `idt_descr.address` 에 넣는다.(fix-mapped 주소에 관련된 내용을 더 읽고 싶다면, [리눅스 커널 메모리 관리](https://github.com/daeseokyoun/linux-insides/blob/korean-trans/mm/linux-mm-2.md)의 두번째 파트를 읽어보길 바란다.) 이 다음에는 [arch/x86/kernel/cpu/common.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/cpu/common.c) 에 구현되어 있는 `cpu_init` 함수의 호출을 볼 수 있다. 이 함수는 모든 `per-cpu` 상태의 초기화를 진행한다. `cpu_init` 의 초반에는 다음과 같은 일을 한다: 무엇보다도 먼저 현재 cpu 가 초기화 되는 동안 기다려야 한다. 그다음에 현재 CPU 를 위한 `cr4` 제어 레지스터의 복사본을 저장하는 `cr4_init_shadow` 함수를 호출한다. 그런 다음 이후에 호출되는 함수들이 필요하다면 CPU 마이크로 코드를 로드한다.:

```C
wait_for_master_cpu(cpu);
cr4_init_shadow();
load_ucode_ap();
```

다음 우리는 현재 cpu 를 위한 `Task State Segment`와 `Interrupt Stack Table` 값을 표현하는 `orig_ist` 를 얻는다.:

```C
t = &per_cpu(cpu_tss, cpu);
oist = &per_cpu(orig_ist, cpu);
```

현재 프로세서를 위한 `Task State Segment` 와 `Interrupt Stack Table` 의 값을 얻고 나서, 우리는 `cr4` 제어 레지스터에 있는 특정 비트를 클리어 해줘야한다.:

```C
cr4_clear_bits(X86_CR4_VME|X86_CR4_PVI|X86_CR4_TSD|X86_CR4_DE);
```

이 클리어하는 비트들은 `vm86` 확장, 가상 인터럽트, 타임스탬프 ([RDTSC](https://en.wikipedia.org/wiki/Time_Stamp_Counter)는 가장 높은 특권에서만 실행될 수 있다.) 그리고 디버그 확장을 위한 것이다. 이 다음에는 `Global Descriptor Table` 와 `Interrupt Descriptor table`를 재 로드한다.:

```C
	switch_to_new_gdt(cpu);
	loadsegment(fs, 0);
	load_current_idt();
```

그런 다음 쓰레드-지역 저장소 기술자(Thread-Local Storage Descriptor) 의 배열을 설정, [NX](https://en.wikipedia.org/wiki/NX_bit) 구성, CPU 마이크로 코드 로드. 이제 `per-cpu` 태스크 상태 세그먼트를 설정하고 로드할 차례이다. 우리는 모든 예외 스택을 순회하며 (`N_EXCEPTION_STACKS` == `4`) `Interrupt Stack Tables`를 채울 것이다.:

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

이제 `Task State Segments` 를 `Interrupt Stack Tables` 으로 채우고 나면, 현재 프로세서를 위한 `TSS` 기술자(Descriptor)를 설정하고 로드 할 수 있다.:

```C
set_tss_desc(cpu, t);
load_TR_desc();
```

[arch/x86/include/asm/desc.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/desc.h)에 정의되어 있는 `set_tss_desc` 매크로는 주어진 디스크립터를 주어진 프로세서의 `Global Descriptor Table` 에 쓴다.:


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

그리고 `load_TR_desc` 매크로 호출로 `ltr` 또는 `Load Task Register` 명령어러를 수행한다.:

```C
#define load_TR_desc()                          native_load_tr_desc()
static inline void native_load_tr_desc(void)
{
        asm volatile("ltr %w0"::"q" (GDT_ENTRY_TSS*8));
}
```

`trap_init` 함수 마지막 부분에는 아래와 같은 코드들을 볼 수 있다.:

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

여기서 우리는 `idt_table` 를 `nmi_dit_table` 로 복사하고 `#DB`(`Debug exception`) 와 `#BR`(`Breakpoint exception`) 를 위한 예외 핸들러를 설정한다. 당신은 우리가 이미 이전 [파트](https://github.com/daeseokyoun/linux-insides/blob/korean-trans/interrupts/interrupts-3.md)에서 이런 인터럽트 게이트들을 설정하는 것을 보았을 텐데, 왜 이런일을 또 하는 것일까? `early_trap_init` 함수 전에 그것을 초기화 할때, `Task State Segment` 가 아직 준비가 되어 있지 않았기 때문에, 다시 설정하여 `cpu_init` 함수의 호출이후에는 완전히 준비가 되게 한다.

끝이다. 이제 우리는 인터럽트/예외의 모든 핸들러를 고려해 볼 것이다.

결론
--------------------------------------------------------------------------------

리눅스 커널의 인터럽트와 인터럽트 핸들러 관련하여 4번째 파트가 끝이 났다. 우리는 이 파트에서 [Task State Segment](https://en.wikipedia.org/wiki/Task_state_segment) 의 초기화를 보았고 `Divide Error`, `Page Fault` 예외 등의 인터럽트 핸들러의 초기화도 보았다. 당신은 단지 초기화 관련해서 본 것이고 이 예외들을 위한 핸들러들의 자세한 사항은 다음 파트에서 시작해볼 것이다.

어떤 질문이나 제안이 있다면, twitter [0xAX](https://twitter.com/0xAX), [email](anotherworldofworld@gmail.com) 또는 [issue](https://github.com/0xAX/linux-insides/issues/new) 를 만들어 주길 바란다.

**나는 영어권의 사람이 아니고 이런 것에 대해 매우 미안해 하고 있다. 만약 어떤 실수를 발견한다면, 나에게 PR을 [linux-insides](https://github.com/0xAX/linux-internals)을 보내줘**

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
