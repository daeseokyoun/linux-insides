인터럽트와 인터럽트 핸들링. Part 2.
================================================================================

리눅스 커널에서 인터럽트와 예외 처리의 시작
--------------------------------------------------------------------------------

우리는 이전 [파트](https://github.com/daeseokyoun/linux-insides/blob/korean-trans/interrupts/interrupts-1.md) 에서 인터럽트와 예외 처리에 관해 약간의 이론을 보았고 그 파트에서 언급했듯이, 이 파트에서 리눅스 커널 소스 코드에서 인터럽트와 예외에 관련해서 더 자세히 알아보도록 하자. 이전 파트에서는 이론적인 측면에서 살펴보았고 이번 파트에서는 리눅스 커널 소스를 직접 살펴 보도록 할 것이다. 다른 챕터에서 했던 방법으로 매우 초기 부분 부터 시작해 볼 것이다. [리눅스 커널 부팅 과정](https://github.com/daeseokyoun/linux-insides/blob/korean-trans/interrupts/interrupts-1.md)에서 예제로 살펴본 초기 [코드 라인들](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L292)은 살펴 보지 않을 것이지만, 인터럽트와 예외에 관련된 초기 코드에서 시작은 할 것이다. 이 파트에서는 리눅스 커널 소스 코드에서 찾아볼 수 잇는 인터럽트외 예외에 관련된 모든 내용을 살펴 보도록 하겠다.

만약 당신이 이전 파트를 읽었다면, 리눅스 커널에서 [arch/x86/boot/pm.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/pm.c) 소스 코드에 위치하는 인터럽트와 연관된  `x86_64` 아키텍쳐 의존적인 코드가 초반에 있고, [Interrupt Descriptor Table](http://en.wikipedia.org/wiki/Interrupt_descriptor_table)를 첫 설정하는 내용이 있었다는 것을 기억할 것이다. 그것은 `setup_idt`에서 호출되는 `go_to_protected_mode` 함수에서 [protected mode](http://en.wikipedia.org/wiki/Protected_mode) 로 전환하기 바로 직전에 실행된다.:


```C
void go_to_protected_mode(void)
{
	...
	setup_idt();
	...
}
```

`setup_idt` 함수는 `go_to_protected_mode` 함수와 같은 파일에 구현되어 있으며, `NULL` 인터럽트 디스크립터 테이블의 주소를 로드한다.:

```C
static void setup_idt(void)
{
	static const struct gdt_ptr null_idt = {0, 0};
	asm volatile("lidtl %0" : : "m" (null_idt));
}
```

`gdt_ptr` 은 `Global Descriptor Table` 의 베이스 주소를 반드시 갖고 있어야하는 48 비트의 특별한 `GDTR` 을 표현한다.:

```C
struct gdt_ptr {
	u16 len;
	u32 ptr;
} __attribute__((packed));
```

물론 우리의 경우에 `gdt_ptr` 는 `GDTR` 레지스터를 나타내지는 않지만, 우리가 `Interrupt Descriptor Table` 를 설정했기에 `IDTR`을 나내낸다. 당신은 `idt_ptr` 구조체를 찾을 수 없을 것이다. 이유는 리눅스 커널에서 갖고 있다면 `gdt_ptr` 과 동일하지만 이름만 다른 것이다. 그래서, 당신은 단지 이름만 다른 같은 두개의 구조체를 갖고 있을 필요가 없다는 것을 이해할 것이다. 당신은 엔트리를 `Interrupt Descriptor Table` 에 채우지 않는다는 것을 알 수 있다. 이유는 이 시에서는 어떤 인터럽트나 예외가 일어나지 않는 초기 시점이기 때문에 처리할 필요가 없는 것이다. 그래서 여기서 `IDT`를 `NULL`로 채워 지는 것이다.

[Interrupt descriptor table](http://en.wikipedia.org/wiki/Interrupt_descriptor_table), [Global Descriptor Table](http://en.wikipedia.org/wiki/GDT) 그리고 다른 몇몇의 설정 이후에 [arch/x86/boot/pmjump.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/pmjump.S) 에 있는 [protected mode](http://en.wikipedia.org/wiki/Protected_mode) 로 점프한다. 보호모드로 전환에 대한 보다 자세한 내용은 [여기](https://github.com/daeseokyoun/linux-insides/blob/korean-trans/Booting/linux-bootstrap-3.md) 에서 살펴보시길 바란다.

우리는 초기 보호모드로 전환하는 엔트리인 `boot_params.hdr.code32_start` 내에 있다는 것을 초기 부분에서 알아보았고 당신은 [arch/x86/boot/pm.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/pm.c) 의 마지막 부분에 `protected_mode_jump` 수로 `boot_params` 와 보호 모드의 엔트리를 전달했다는 것을 볼 수 있다.:


```C
protected_mode_jump(boot_params.hdr.code32_start,
			    (u32)&boot_params + (ds() << 4));
```

`protected_mode_jump` 함수는 [arch/x86/boot/pmjump.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/pmjump.S) 에 구현되어 있고 여기 두 인자는 [8086](http://en.wikipedia.org/wiki/Intel_8086) 호출  [규약](http://en.wikipedia.org/wiki/X86_calling_conventions#List_of_x86_calling_conventions)에 의해 `ax` 와 `dx` 레지스터들로 부터 을 수 있다.:

```assembly
GLOBAL(protected_mode_jump)
	...
	...
	...
	.byte	0x66, 0xea		# ljmpl opcode
2:	.long	in_pm32			# offset
	.word	__BOOT_CS		# segment
...
...
...
ENDPROC(protected_mode_jump)
```

`in_pm32` 는 32 비트 엔트리 포인트로 점프를 한다.:

```assembly
GLOBAL(in_pm32)
	...
	...
	jmpl	*%eax // %eax contains address of the `startup_32`
	...
	...
ENDPROC(in_pm32)
```

32 비트 엔트리 포인트는 비록 파일 이름에 `_64` 가 있지만, [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_64.S) 어셈블리 파일에 있다. 우리는 `arch/x86/boot/compressed` 디렉토리내에 비슷한 파일이 있다는 것을 볼 수 있다.:

* `arch/x86/boot/compressed/head_32.S`.
* `arch/x86/boot/compressed/head_64.S`;

하지만 32 비트 모드 엔트리 포인트는 우리의 경우 2 번째 파일에 있다. 첫번째 파일은 `x86_64` 를 위해 빌드조차 되지 않는다. [arch/x86/boot/compressed/Makefile](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/Makefile) 파일을 보자.:

```
vmlinux-objs-y := $(obj)/vmlinux.lds $(obj)/head_$(BITS).o $(obj)/misc.o \
...
...
```

우리는 여기서 `head_*` 가 아키텍처 의존적인 `$(BITS)` 변수에 의존임을 알 수 있다. 당신은 이것을 [arch/x86/Makefile](https://github.com/torvalds/linux/blob/master/arch/x86/Makefile) 에서 찾아 볼 수 있다.:

```
ifeq ($(CONFIG_X86_32),y)
...
	BITS := 32
else
	BITS := 64
	...
endif
```

이제 우리는 [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_64.S)로 부터 `startup_32` 로 점프 했지만 아직은 인터트와 연관된 어떤 것도 찾을 수 없을 것이다. `startup_32` 는 [long mode](http://en.wikipedia.org/wiki/Long_mode) 로 전환하기 전에 준비하는 코드를 포함하고 바로 점프한다. `long mode` 엔트리는 [arch/x86/boot/compressed/misc.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/misc.c) 에 위치한다. 커널이 압해제 된 이후에, [NX](http://en.wikipedia.org/wiki/NX_bit) 비트 확인, `Extended Feature Enable Register` 설정, 그리고 `lgdt` 명령어로 초기 `Global Descriptor Table` 업데이트와 아래의 코드로 `gs`레지스터를 설정할 필요가 있다.:

```assembly
movl	$MSR_GS_BASE,%ecx
movl	initial_gs(%rip),%eax
movl	initial_gs+4(%rip),%edx
wrmsr
```

우리는 이미 이전 [파트](https://github.com/daeseokyoun/linux-insides/blob/korean-trans/interrupts/interrupts-1.md) 에서 이 코드를 살펴 보았다. 우리가 이 모든것의 처음으로 초점을 맞춰야 하는 것은 마지막에 있는 `wrmsr` 명령어 이다. 이 명령어는 `edx:eax` 레지스터들로 부터 데이터를 읽어 `ecx` 레지스터에 의해 정의되는 [model specific register](http://en.wikipedia.org/wiki/Model-specific_register) 에 쓴다. 우리는 `ecx` 가 [arch/x86/include/uapi/asm/msr-index.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/uapi/asm/msr-index.h) 에 선언되어 있는 `$MSR_GS_BASE` 를 갖고 있다.:

```C
#define MSR_GS_BASE             0xc0000101
```

이것으로 부터 우리는 `MSR_GS_BASE` 가 `model specific register` 의 수를 정의한다는 것을 이해 했다. `cs`, `ds`, `es`, 그리고 `ss` 레지스터들은 64 비트 모드에서는 사용되지 않기 때문에, 이런 필드은 무시된다. 하지만 우리는 `fs` 와 `gs` 레지스터들을 통해 접근할 수 있다. 이 모델 정의 레지스터(model specific register) 는 이런 세그먼트 레지스터들과 같은 숨겨진 부분들을 위한 `back door`를 제공하고 `fs` 와 `gs` 에 의해 접근되는 세그먼트 레지스터를 위한 64 비트 베이스 주소를 사용할 수 있도록 허가한다. 그래서 `MSR_GS_BASE` 는 숨겨진 파트이고 이 파트는 `GS.base` 필드에 맵핑되어 있다. `initial_gs` 를 살펴 보자.:

```assembly
GLOBAL(initial_gs)
	.quad	INIT_PER_CPU_VAR(irq_stack_union)
```

우리는 `irq_stack_union` 심볼을 주어진 심볼을 `init_per_cpu__` 접두사를 붙이는 `INIT_PER_CPU_VAR` 매크로의 인자로 넘겨 준다. 우리의 경우에서 `init_per_cpu__irq_stack_union` 심을 얻을 수 있다. [linker](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/vmlinux.lds.S) 스크립트를 한번 보자. 아래와 같은 정의를 볼 수 있다.:

```
#define INIT_PER_CPU(x) init_per_cpu__##x = x + __per_cpu_load
INIT_PER_CPU(irq_stack_union);
```

그것은 우리에게 `init_per_cpu__irq_stack_union` 의 주소가 `irq_stack_union + __per_cpu_load` 이라는 것을 알려준다. 이제 우리는 `init_per_cpu__irq_stack_union` 와 `__per_cpu_load` 가 무엇을 의하고 어디에 있는지 알 필요가 있다. `irq_stack_union` 는 `init_per_cpu_var` 매크로를 확장한 `DECLARE_INIT_PER_CPU` 와 함께 [arch/x86/include/asm/processor.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/processor.h) 에 선언되어 있다.:

```C
DECLARE_INIT_PER_CPU(irq_stack_union);

#define DECLARE_INIT_PER_CPU(var) \
       extern typeof(per_cpu_var(var)) init_per_cpu_var(var)

#define init_per_cpu_var(var)  init_per_cpu__##var
```

만약 모든 매크로를 확장시켜 보면, 우리는 `INIT_PER_CPU` 매크로를 수행했을 때와 같은 `init_per_cpu__irq_stack_union` 를 얻을 수 있을 것이다. 하지만 이 것은 단지 심볼이 아니라, 변수임을 기억하자. `typeof(per_cpu_var(var))` 를 주목해보자. 여기서 `var` 는 `irq_stack_union` 고 `per_cpu_var` 매크로는 [arch/x86/include/asm/percpu.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/percpu.h) 에 정의되어 있다.:

```C
#define PER_CPU_VAR(var)        %__percpu_seg:var__ // TODO 마지막 언더바 두개
```

where:

```C
#ifdef CONFIG_X86_64
    #define __percpu_seg gs__ // TODO 마지막 언더바 두개
endif
```

그래서, 우리는 `gs:irq_stack_union` 로 접근하여 `irq_union` 의 타입을 얻을 수 있다. 좋다, 우리는 첫번째 변수를 선언하고 그것의 주소를 알고 있다. 이제 `__per_cpu_load` 심볼을 살펴보자. 이 심볼 뒤에 따르는 두어개의 `per-cpu` 변수들이 있다. `__per_cpu_load` 는 [include/asm-generic/sections.h](https://github.com/torvalds/linux/blob/master/include/asm-generic-sections.h)에 정의되어 있다.:

```C
extern char __per_cpu_load[], __per_cpu_start[], __per_cpu_end[];
```

그리고 데이터 영역에 `per-cpu` 변수들의 베이스 주소들이 있다. 그래서, 우리는 `irq_stack_union`, `__per_cpu_load` 의 주소를 알고 있고 `init_per_cpu__irq_stack_union`는 반드시 `__per_cpu_load` 바로 다음에 위치해야 하는 것도 알고 있다. [System.map](http://en.wikipedia.org/wiki/System.map) 에서 관련된 사항을 볼 수 있다.:

```
...
...
...
ffffffff819ed000 D __init_begin
ffffffff819ed000 D __per_cpu_load
ffffffff819ed000 A init_per_cpu__irq_stack_union
...
...
...
```

이제 우리는 `initial_gs` 에 관해 알아보았고, 아래의 코드를 한번 보자.:

```assembly
movl	$MSR_GS_BASE,%ecx
movl	initial_gs(%rip),%eax
movl	initial_gs+4(%rip),%edx
wrmsr
```

여기서 우리는 `MSR_GS_BASE` 를 `모델 정의 레지스터`로 정의하고, `initial_gs` 의 64 비트 주소를 `edx:eax` 에 넣고 `wrmsr` 명령어를 실행하여 인터럽트 스택의 밑바닥이 될 `init_per_cpu__irq_stack_union` 의 베이스 주소를 `gs` 레지스터에 채워넣는다. 이 다음에 우리는 C 코드 인 [arch/x86/kernel/head64.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head64.c) 에 있는 `x86_64_start_kernel`로 점프한다. `x86_64_start_kernel` 함수에서 우리는 일반적이고 아케텍처에 독립적인 커널 코드로 진입하기 전에 마지막 준비를 하는데, 그 준비 작업중 하나가 `early_idt_handlers` 또는 인터럽트 핸들러 엔트리들로 초기 `Interrupt Descriptor Table` 를 채우는 것을 한다. 기억하겠지만, [Early interrupt and exception handling](https://github.com/daeseokyoun/linux-insides/blob/korean-trans/Initialization/linux-initialization-2.md) 를 읽어 봤다면, 아래의 코드를 기억할 것이다.:

```C
for (i = 0; i < NUM_EXCEPTION_VECTORS; i++)
	set_intr_gate(i, early_idt_handlers[i]);

load_idt((const struct desc_ptr *)&idt_descr);
```

`초기 인터럽트와 예외 처리` 파트를 작성한 시점의 리눅스 커널 버전은 `3.18`이었다. 현재 이 챕터를 쓰는 시점에서 리눅스 커널은 `4.1.0-rc6+` 버전이 되었고 `Andy Lutomirski`는 [패치](https://lkml.org/lkml/2015/6/2/106) 를 보내 `early_idt_handlers` 를 위한 행동에 변화를 주었고 메인라인 커널에 포함되었다. **Note** 현재 이 파트를 쓰는 동안에 [그 패치](https://github.com/torvalds/linux/commit/425be5679fd292a3c36cb1fe423086708a99f11a)가 리눅스 커널 소스 코드에 적용이 완료되었다. 이제 같은 부분의 코드가 어떻게 변경되었는지 보자.:

```C
for (i = 0; i < NUM_EXCEPTION_VECTORS; i++)
	set_intr_gate(i, early_idt_handler_array[i]);

load_idt((const struct desc_ptr *)&idt_descr);
```

여기서는 단지 인터럽트 핸들의 엔트리 포인트의 배열 이름만 차이가 있다는 것을 볼 수 있다. 이제 그것은 `early_idt_handler_array` 이다.:

```C
extern const char early_idt_handler_array[NUM_EXCEPTION_VECTORS][EARLY_IDT_HANDLER_SIZE];
```

`NUM_EXCEPTION_VECTORS` 와 `EARLY_IDT_HANDLER_SIZE`는 아래와 같이 선언되었다:

```C
#define NUM_EXCEPTION_VECTORS 32
#define EARLY_IDT_HANDLER_SIZE 9
```

그래서, `early_idt_handler_array` 는 인터럽트 핸들러 엔트리 포인트의 배열이고 각 9 바이트 마다 하나의 엔트리 포인트를 가지게 된다. 이전 `early_idt_handlers` 는 [arch/x86/kernel/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head_64.S) 에 선언되어 있었다. `early_idt_handler_array` 도 같은 소스 파일에 선언되어 있다.:

```assembly
ENTRY(early_idt_handler_array)
...
...
...
ENDPROC(early_idt_handler_common)
```

위의 코드는 `.rept NUM_EXCEPTION_VECTORS`로 `early_idt_handler_array` 를 채우고 `early_make_pgtable` 인터럽트 핸들러의 엔트리 구현([초기 인터럽트와 예외처리](https://github.com/daeseokyoun/linux-insides/blob/korean-trans/Initialization/linux-initialization-2.md)에서 확인 가능하다.)을 포함시킨다. 이제 `x86_64` 아키텍처 의존적인 코드의 막바지에 왔고 다음 파트에서는 일반적인 커널 코드가 소개 될 것이다. 물론, 당신은 이미 `setup_arch` 와 다른 곳에서 아키텍처 의존적인 코드로 돌아갈 것이지만 이것이 초기 `x86_64` 코드의 마지막 부분이 될 것이다.

Setting stack canary for the interrupt stack
-------------------------------------------------------------------------------

The next stop after the [arch/x86/kernel/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head_64.S) is the biggest `start_kernel` function from the [init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c). If you've read the previous [chapter](http://0xax.gitbooks.io/linux-insides/content/Initialization/index.html) about the Linux kernel initialization process, you must remember it. This function does all initialization stuff before kernel will launch first `init` process with the [pid](https://en.wikipedia.org/wiki/Process_identifier) - `1`. The first thing that is related to the interrupts and exceptions handling is the call of the `boot_init_stack_canary` function.

This function sets the [canary](http://en.wikipedia.org/wiki/Stack_buffer_overflow#Stack_canaries) value to protect interrupt stack overflow. We already saw a little some details about implementation of the `boot_init_stack_canary` in the previous part and now let's take a closer look on it. You can find implementation of this function in the [arch/x86/include/asm/stackprotector.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/stackprotector.h) and its depends on the `CONFIG_CC_STACKPROTECTOR` kernel configuration option. If this option is not set this function will not do anything:

```C
#ifdef CONFIG_CC_STACKPROTECTOR
...
...
...
#else
static inline void boot_init_stack_canary(void)
{
}
#endif
```

If the `CONFIG_CC_STACKPROTECTOR` kernel configuration option is set, the `boot_init_stack_canary` function starts from the check stat `irq_stack_union` that represents [per-cpu](http://0xax.gitbooks.io/linux-insides/content/Concepts/per-cpu.html) interrupt stack has offset equal to forty bytes from the `stack_canary` value:

```C
#ifdef CONFIG_X86_64
        BUILD_BUG_ON(offsetof(union irq_stack_union, stack_canary) != 40);
#endif
```

As we can read in the previous [part](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-1.html) the `irq_stack_union` represented by the following union:

```C
union irq_stack_union {
	char irq_stack[IRQ_STACK_SIZE];

    struct {
		char gs_base[40];
		unsigned long stack_canary;
	};
};
```

which defined in the [arch/x86/include/asm/processor.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/processor.h). We know that [union](http://en.wikipedia.org/wiki/Union_type) in the [C](http://en.wikipedia.org/wiki/C_%28programming_language%29) programming language is a data structure which stores only one field in a memory. We can see here that structure has first field - `gs_base` which is 40 bytes size and represents bottom of the `irq_stack`. So, after this our check with the `BUILD_BUG_ON` macro should end successfully. (you can read the first part about Linux kernel initialization [process](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-1.html) if you're interesting about the `BUILD_BUG_ON` macro).

After this we calculate new `canary` value based on the random number and [Time Stamp Counter](http://en.wikipedia.org/wiki/Time_Stamp_Counter):

```C
get_random_bytes(&canary, sizeof(canary));
tsc = __native_read_tsc();
canary += tsc + (tsc << 32UL);
```

and write `canary` value to the `irq_stack_union` with the `this_cpu_write` macro:

```C
this_cpu_write(irq_stack_union.stack_canary, canary);
```

more about `this_cpu_*` operation you can read in the [Linux kernel documentation](https://github.com/torvalds/linux/blob/master/Documentation/this_cpu_ops.txt).

Disabling/Enabling local interrupts
--------------------------------------------------------------------------------

The next step in the [init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c) which is related to the interrupts and interrupts handling after we have set the `canary` value to the interrupt stack - is the call of the `local_irq_disable` macro.

This macro defined in the [include/linux/irqflags.h](https://github.com/torvalds/linux/blob/master/include/linux/irqflags.h) header file and as you can understand, we can disable interrupts for the CPU with the call of this macro. Let's look on its implementation. First of all note that it depends on the `CONFIG_TRACE_IRQFLAGS_SUPPORT` kernel configuration option:

```C
#ifdef CONFIG_TRACE_IRQFLAGS_SUPPORT
...
#define local_irq_disable() \
         do { raw_local_irq_disable(); trace_hardirqs_off(); } while (0)
...
#else
...
#define local_irq_disable()     do { raw_local_irq_disable(); } while (0)
...
#endif
```

They are both similar and as you can see have only one difference: the `local_irq_disable` macro contains call of the `trace_hardirqs_off` when `CONFIG_TRACE_IRQFLAGS_SUPPORT` is enabled. There is special feature in the [lockdep](http://lwn.net/Articles/321663/) subsystem - `irq-flags tracing` for tracing `hardirq` and `softirq` state. In our case `lockdep` subsystem can give us interesting information about hard/soft irqs on/off events which are occurs in the system. The `trace_hardirqs_off` function defined in the [kernel/locking/lockdep.c](https://github.com/torvalds/linux/blob/master/kernel/locking/lockdep.c):

```C
void trace_hardirqs_off(void)
{
         trace_hardirqs_off_caller(CALLER_ADDR0);
}
EXPORT_SYMBOL(trace_hardirqs_off);
```

and just calls `trace_hardirqs_off_caller` function. The `trace_hardirqs_off_caller` checks the `hardirqs_enabled` field of the current process and increases the `redundant_hardirqs_off` if call of the `local_irq_disable` was redundant or the `hardirqs_off_events` if it was not. These two fields and other `lockdep` statistic related fields are defined in the [kernel/locking/lockdep_insides.h](https://github.com/torvalds/linux/blob/master/kernel/locking/lockdep_insides.h) and located in the `lockdep_stats` structure:

```C
struct lockdep_stats {
...
...
...
int     softirqs_off_events;
int     redundant_softirqs_off;
...
...
...
}
```

If you will set `CONFIG_DEBUG_LOCKDEP` kernel configuration option, the `lockdep_stats_debug_show` function will write all tracing information to the `/proc/lockdep`:

```C
static void lockdep_stats_debug_show(struct seq_file *m)
{
#ifdef CONFIG_DEBUG_LOCKDEP
	unsigned long long hi1 = debug_atomic_read(hardirqs_on_events),
	                         hi2 = debug_atomic_read(hardirqs_off_events),
							 hr1 = debug_atomic_read(redundant_hardirqs_on),
    ...
	...
	...
    seq_printf(m, " hardirq on events:             %11llu\n", hi1);
    seq_printf(m, " hardirq off events:            %11llu\n", hi2);
    seq_printf(m, " redundant hardirq ons:         %11llu\n", hr1);
#endif
}
```

and you can see its result with the:

```
$ sudo cat /proc/lockdep
 hardirq on events:             12838248974
 hardirq off events:            12838248979
 redundant hardirq ons:               67792
 redundant hardirq offs:         3836339146
 softirq on events:                38002159
 softirq off events:               38002187
 redundant softirq ons:                   0
 redundant softirq offs:                  0
```

Ok, now we know a little about tracing, but more info will be in the separate part about `lockdep` and `tracing`. You can see that the both `local_disable_irq` macros have the same part - `raw_local_irq_disable`. This macro defined in the [arch/x86/include/asm/irqflags.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/irqflags.h) and expands to the call of the:

```C
static inline void native_irq_disable(void)
{
        asm volatile("cli": : :"memory");
}
```

And you already must remember that `cli` instruction clears the [IF](http://en.wikipedia.org/wiki/Interrupt_flag) flag which determines ability of a processor to handle an interrupt or an exception. Besides the `local_irq_disable`, as you already can know there is an inverse macro - `local_irq_enable`. This macro has the same tracing mechanism and very similar on the `local_irq_enable`, but as you can understand from its name, it enables interrupts with the `sti` instruction:

```C
static inline void native_irq_enable(void)
{
        asm volatile("sti": : :"memory");
}
```

Now we know how `local_irq_disable` and `local_irq_enable` work. It was the first call of the `local_irq_disable` macro, but we will meet these macros many times in the Linux kernel source code. But for now we are in the `start_kernel` function from the [init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c) and we just disabled `local` interrupts. Why local and why we did it? Previously kernel provided a method to disable interrupts on all processors and it was called `cli`. This function was [removed](https://lwn.net/Articles/291956/) and now we have `local_irq_{enabled,disable}` to disable or enable interrupts on the current processor. After we've disabled the interrupts with the `local_irq_disable` macro, we set the:

```C
early_boot_irqs_disabled = true;
```

The `early_boot_irqs_disabled` variable defined in the [include/linux/kernel.h](https://github.com/torvalds/linux/blob/master/include/linux/kernel.h):

```C
extern bool early_boot_irqs_disabled;
```

and used in the different places. For example it used in the `smp_call_function_many` function from the [kernel/smp.c](https://github.com/torvalds/linux/blob/master/kernel/smp.c) for the checking possible deadlock when interrupts are disabled:

```C
WARN_ON_ONCE(cpu_online(this_cpu) && irqs_disabled()
                     && !oops_in_progress && !early_boot_irqs_disabled);
```

Early trap initialization during kernel initialization
--------------------------------------------------------------------------------

The next functions after the `local_disable_irq` are `boot_cpu_init` and `page_address_init`, but they are not related to the interrupts and exceptions (more about this functions you can read in the chapter about Linux kernel [initialization process](http://0xax.gitbooks.io/linux-insides/content/Initialization/index.html)). The next is the `setup_arch` function. As you can remember this function located in the [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel.setup.c) source code file and makes initialization of many different architecture-dependent [stuff](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-4.html). The first interrupts related function which we can see in the `setup_arch` is the - `early_trap_init` function. This function defined in the [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/traps.c) and fills `Interrupt Descriptor Table` with the couple of entries:

```C
void __init early_trap_init(void)
{
        set_intr_gate_ist(X86_TRAP_DB, &debug, DEBUG_STACK);
        set_system_intr_gate_ist(X86_TRAP_BP, &int3, DEBUG_STACK);
#ifdef CONFIG_X86_32
        set_intr_gate(X86_TRAP_PF, page_fault);
#endif
        load_idt(&idt_descr);
}
```

Here we can see calls of three different functions:

* `set_intr_gate_ist`
* `set_system_intr_gate_ist`
* `set_intr_gate`

All of these functions defined in the [arch/x86/include/asm/desc.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/desc.h) and do the similar thing but not the same. The first `set_intr_gate_ist` function inserts new an interrupt gate in the `IDT`. Let's look on its implementation:

```C
static inline void set_intr_gate_ist(int n, void *addr, unsigned ist)
{
        BUG_ON((unsigned)n > 0xFF);
        _set_gate(n, GATE_INTERRUPT, addr, 0, ist, __KERNEL_CS);
}
```

First of all we can see the check that `n` which is [vector number](http://en.wikipedia.org/wiki/Interrupt_vector_table) of the interrupt is not greater than `0xff` or 255. We need to check it because we remember from the previous [part](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-1.html) that vector number of an interrupt must be between `0` and `255`. In the next step we can see the call of the `_set_gate` function that sets a given interrupt gate to the `IDT` table:

```C
static inline void _set_gate(int gate, unsigned type, void *addr,
                             unsigned dpl, unsigned ist, unsigned seg)
{
        gate_desc s;

        pack_gate(&s, type, (unsigned long)addr, dpl, ist, seg);
        write_idt_entry(idt_table, gate, &s);
        write_trace_idt_entry(gate, &s);
}
```

Here we start from the `pack_gate` function which takes clean `IDT` entry represented by the `gate_desc` structure and fills it with the base address and limit, [Interrupt Stack Table](https://www.kernel.org/doc/Documentation/x86/x86_64/kernel-stacks), [Privilege level](http://en.wikipedia.org/wiki/Privilege_level), type of an interrupt which can be one of the following values:

* `GATE_INTERRUPT`
* `GATE_TRAP`
* `GATE_CALL`
* `GATE_TASK`

and set the present bit for the given `IDT` entry:

```C
static inline void pack_gate(gate_desc *gate, unsigned type, unsigned long func,
                             unsigned dpl, unsigned ist, unsigned seg)
{
        gate->offset_low        = PTR_LOW(func);
        gate->segment           = __KERNEL_CS;
        gate->ist               = ist;
        gate->p                 = 1;
        gate->dpl               = dpl;
        gate->zero0             = 0;
        gate->zero1             = 0;
        gate->type              = type;
        gate->offset_middle     = PTR_MIDDLE(func);
        gate->offset_high       = PTR_HIGH(func);
}
```

After this we write just filled interrupt gate to the `IDT` with the `write_idt_entry` macro which expands to the `native_write_idt_entry` and just copy the interrupt gate to the `idt_table` table by the given index:

```C
#define write_idt_entry(dt, entry, g)           native_write_idt_entry(dt, entry, g)

static inline void native_write_idt_entry(gate_desc *idt, int entry, const gate_desc *gate)
{
        memcpy(&idt[entry], gate, sizeof(*gate));
}
```

where `idt_table` is just array of `gate_desc`:

```C
extern gate_desc idt_table[];
```

That's all. The second `set_system_intr_gate_ist` function has only one difference from the `set_intr_gate_ist`:

```C
static inline void set_system_intr_gate_ist(int n, void *addr, unsigned ist)
{
        BUG_ON((unsigned)n > 0xFF);
        _set_gate(n, GATE_INTERRUPT, addr, 0x3, ist, __KERNEL_CS);
}
```

Do you see it? Look on the fourth parameter of the `_set_gate`. It is `0x3`. In the `set_intr_gate` it was `0x0`. We know that this parameter represent `DPL` or privilege level. We also know that `0` is the highest privilege level and `3` is the lowest.Now we know how `set_system_intr_gate_ist`, `set_intr_gate_ist`, `set_intr_gate` are work and we can return to the `early_trap_init` function. Let's look on it again:

```C
set_intr_gate_ist(X86_TRAP_DB, &debug, DEBUG_STACK);
set_system_intr_gate_ist(X86_TRAP_BP, &int3, DEBUG_STACK);
```

We set two `IDT` entries for the `#DB` interrupt and `int3`. These functions takes the same set of parameters:

* vector number of an interrupt;
* address of an interrupt handler;
* interrupt stack table index.

That's all. More about interrupts and handlers you will know in the next parts.

Conclusion
--------------------------------------------------------------------------------

It is the end of the second part about interrupts and interrupt handling in the Linux kernel. We saw the some theory in the previous part and started to dive into interrupts and exceptions handling in the current part. We have started from the earliest parts in the Linux kernel source code which are related to the interrupts. In the next part we will continue to dive into this interesting theme and will know more about interrupt handling process.

If you have any questions or suggestions write me a comment or ping me at [twitter](https://twitter.com/0xAX).

**Please note that English is not my first language, And I am really sorry for any inconvenience. If you find any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-insides).**

Links
--------------------------------------------------------------------------------

* [IDT](http://en.wikipedia.org/wiki/Interrupt_descriptor_table)
* [Protected mode](http://en.wikipedia.org/wiki/Protected_mode)
* [List of x86 calling conventions](http://en.wikipedia.org/wiki/X86_calling_conventions#List_of_x86_calling_conventions)
* [8086](http://en.wikipedia.org/wiki/Intel_8086)
* [Long mode](http://en.wikipedia.org/wiki/Long_mode)
* [NX](http://en.wikipedia.org/wiki/NX_bit)
* [Extended Feature Enable Register](http://en.wikipedia.org/wiki/Control_register#Additional_Control_registers_in_x86-64_series)
* [Model-specific register](http://en.wikipedia.org/wiki/Model-specific_register)
* [Process identifier](https://en.wikipedia.org/wiki/Process_identifier)
* [lockdep](http://lwn.net/Articles/321663/)
* [irqflags tracing](https://www.kernel.org/doc/Documentation/irqflags-tracing.txt)
* [IF](http://en.wikipedia.org/wiki/Interrupt_flag)
* [Stack canary](http://en.wikipedia.org/wiki/Stack_buffer_overflow#Stack_canaries)
* [Union type](http://en.wikipedia.org/wiki/Union_type)
* [this_cpu_* operations](https://github.com/torvalds/linux/blob/master/Documentation/this_cpu_ops.txt)
* [vector number](http://en.wikipedia.org/wiki/Interrupt_vector_table)
* [Interrupt Stack Table](https://www.kernel.org/doc/Documentation/x86/x86_64/kernel-stacks)
* [Privilege level](http://en.wikipedia.org/wiki/Privilege_level)
* [Previous part](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-1.html)
* [8086 호출 규약](https://ko.wikipedia.org/wiki/X86_%ED%98%B8%EC%B6%9C_%EA%B7%9C%EC%95%BD#.ED.98.B8.EC.B6.9C.EC.9E.90.2F.ED.94.BC.ED.98.B8.EC.B6.9C.EC.9E.90_.EC.A0.95.EB.A6.AC)
