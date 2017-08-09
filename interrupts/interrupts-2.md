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
#define PER_CPU_VAR(var)        %__percpu_seg:var
```

where:

```C
#ifdef CONFIG_X86_64
    #define __percpu_seg gs
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

인터럽트 스택을 위한 스택 카나리(canary) 설정
-------------------------------------------------------------------------------

[arch/x86/kernel/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head_64.S) 다음으로 [init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c)에서 가장 큰 부분인 `start_kernel` 을 살펴 보자. 만약 이미 리눅스 초기화 과정에 관련된 챕터에서 읽어보았다면, 당신은 반드시 기억해야 합니다. 이 함수는 첫 [pid](https://en.wikipedia.org/wiki/Process_identifier) 로 `1` 값을 가지는 `init` 프로세스가 수행되기 전에 커널의 거의 모든 초기화를 진행한다. 인터럽트와 예외 처리와 관련된 첫번째는 `boot_init_stack_canary` 함수 호출이다.

이 함수는 [canary](http://en.wikipedia.org/wiki/Stack_buffer_overflow#Stack_canaries) 값을 인터럽트 스택 오버플로우를 막기 위해 설정한다. 우리는 이미 이전 파트에서 `boot_init_stack_canary` 구현에 관련된 약간의 상세를 보았고 이제는 더 자세히 봐야 할 때이다. 이 함수의 구현은 [arch/x86/include/asm/stackprotector.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/stackprotector.h) 에서 찾을 수 있고 `CONFIG_CC_STACKPROTECTOR` 구성 옵션에 의존적이다. 만약 이 옵션이 설정되어 있지 않다면, 이 함수는 아무 것도 하지 않을 것이다.:

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

만약 `CONFIG_CC_STACKPROTECTOR` 커널 구성 옵션이 설정되어 있다면, `boot_init_stack_canary` 함수는 [per-cpu](https://github.com/daeseokyoun/linux-insides/blob/korean-trans/Concepts/per-cpu.md) 를 나타내는 `irq_stack_union` 내에 있는 `stack_canary` 가 40 바이트 오프셋만큼 떨어져있는지 확인하는 것부터 시작한다.:

```C
#ifdef CONFIG_X86_64
        BUILD_BUG_ON(offsetof(union irq_stack_union, stack_canary) != 40);
#endif
```

이전 [파트](https://github.com/daeseokyoun/linux-insides/blob/korean-trans/interrupts/interrupts-1.md) 에서 보면 `irq_stack_union` 은 아래와 같은 유니온(union) 타입으로 만들어져 있다.:

```C
union irq_stack_union {
	char irq_stack[IRQ_STACK_SIZE];

    struct {
		char gs_base[40];
		unsigned long stack_canary;
	};
};
```

이 자료구조는 [arch/x86/include/asm/processor.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/processor.h)에 선언되어 있다. 우리는 [C](http://en.wikipedia.org/wiki/C_%28programming_language%29) 프로그래밍 언어에서 [union](http://en.wikipedia.org/wiki/Union_type) 은 메모리 공간에 한가지의 요소만 저장 할 수 있도록 하는 자료 구조라는 것을 안다. 이 구조체의 첫번째 항목으로 `gs_base` 를 볼 수 있다. 이것은 40 바이트의 크기를 가지고 `irq_stack` 의 밑바닥을 표현한다. 그래서 `BUILD_BUG_ON` 매크로와 확인하는 내용은 성공적으로 마무리가 될 것이다. (리눅스 초기화 첫번째 [과정](https://github.com/daeseokyoun/linux-insides/blob/korean-trans/Initialization/linux-initialization-1.md) 에서 `BUILD_BUG_ON` 매크로에 대해 더 자세히 볼 수 있을 것이다.).

이 확인 다음에는 새로운 `canary` 값을 랜덤 숫자와 [Time Stamp Counter](http://en.wikipedia.org/wiki/Time_Stamp_Counter) 에 기반하여 계산한다.:

```C
get_random_bytes(&canary, sizeof(canary));
tsc = __native_read_tsc();
canary += tsc + (tsc << 32UL);
```

그리고 `canary` 값을 `this_cpu_write` 매크로를 통해 `irq_stack_union`에 써주게 된다.

```C
this_cpu_write(irq_stack_union.stack_canary, canary);
```

`this_cpu_*` 수행 관련해서 더 많은 내용을 읽고 싶다면, [리눅스 커널 문서](https://github.com/torvalds/linux/blob/master/Documentation/this_cpu_ops.txt)를 참조하길 바란다.

비활성/활성(Disabling/Enabling) 지역 인터럽트
--------------------------------------------------------------------------------

인터럽트와 인터럽트 처리와 관련된 [init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c) 부분에서 인터럽트 스택에 `canary` 값을 설정한 후 다음 단계는 `local_irq_disable` 매크로의 호출이다.

이 매크로는 [include/linux/irqflags.h](https://github.com/torvalds/linux/blob/master/include/linux/irqflags.h) 헤더 파일에 선언되어 있으며, 당신이 이해 했듯이, 우리는 이 매크로의 호출로 CPU 를 위한 인터럽트를 비활성화 할 수 있다. 이것의 구현을 살펴보자. 첫번째로 `CONFIG_TRACE_IRQFLAGS_SUPPORT` 커널 구성 옵션에 의존적이라는 것을 알아두자.:

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

이 두 매크로는 비슷하지만 단지 하나의 차이가 있다: `local_irq_disable` 매크로가 `CONFIG_TRACE_IRQFLAGS_SUPPORT` 옵션이 설정되면 `trace_hardirqs_off` 의 호출을 포함한다. 여기에 [lockdep](http://lwn.net/Articles/321663/) 서브시스템의 특별한 항목이 있다. `irq-flags tracing` 는 `hardirq` 와 `softirq` 상태를 추적하기 위해 사용된다. 우리의 경우, `lockdep` 서브시스템은 시스템에서 발생하는 hard/soft irq 들의 on/off 이벤트 정보를 우리에게 제공한다. `trace_hardirqs_off` 함수는 [kernel/locking/lockdep.c](https://github.com/torvalds/linux/blob/master/kernel/locking/lockdep.c) 에 구현되어 있다.:

```C
void trace_hardirqs_off(void)
{
         trace_hardirqs_off_caller(CALLER_ADDR0);
}
EXPORT_SYMBOL(trace_hardirqs_off);
```

그리고 단지 `trace_hardirqs_off_caller` 함수를 호출한다. `trace_hardirqs_off_caller` 함수는 현재 프로세스틔 `hardirqs_enabled` 항목을 확인하고 `local_irq_disable` 의 호출이 불필요한 경우에는 `redundant_hardirqs_off` 만을 증가시키고 아니라면 `hardirqs_off_events` 를 증가시킨다. 여기에 [kernel/locking/lockdep_insides.h](https://github.com/torvalds/linux/blob/master/kernel/locking/lockdep_insides.h)에 정의된 `lockdep_stats` 구조체에 정의된 `lockdep` 의 두 개의 통계적 항목있다.:

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

만약 당신이 `CONFIG_DEBUG_LOCKDEP` 커널 구성 옵션을 설정했다면, `lockdep_stats_debug_show` 함수는 모든 추적 정보를 `/proc/lockdep` 에 쓸것이다.:

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

그리고 당신은 그것의 결과를 볼 수 있다.:

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

좋다, 이제 우리는 추적(tracing)에 관해 조금 알아보았다, 하지만 더 많은 정보는 `lockdep` 와 `tracing` 관련된 다른 챕터에서 볼 수 있을 것이다. 여기서 `local_disable_irq` 매크로와 같은 파트에서 `raw_local_irq_disable`도 볼 수 있을 것이다. 이 매크로는 [arch/x86/include/asm/irqflags.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/irqflags.h) 에 선언되어 있고 아래의 호출로 확장된다.:

```C
static inline void native_irq_disable(void)
{
        asm volatile("cli": : :"memory");
}
```

그리고 당신은 인터럽트나 예외를 처리하기 위한 프로세서의 기능을 결정하는 [IF](http://en.wikipedia.org/wiki/Interrupt_flag) 플래그를 클리어하는 `cli` 명령어를 기억해야 한다. 또한 이 것과 상반되는 `local_irq_enable` 라는 매크로도 있다는 것을 알 수 있다. 이 매크로는 같은 추적 매커너즘을 가지고 있고, 이름에서도 알 수 있듯이 `sti` 명령어로 인터럽트들의 처리를 활성화 한다.:

```C
static inline void native_irq_enable(void)
{
        asm volatile("sti": : :"memory");
}
```

이제 우리는 `local_irq_disable` 와 `local_irq_enable`가 어떻게 동작하는지 알아보았다. `local_irq_disable` 매크로의 첫번째 호출이었지만 우리는 리눅스 커널 소스 코드에서 이런 매크로들을 자주 만나게 될 것이다. 하지만 지금은 [init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c) 소스 파일의 `start_kernel` 함수를 살펴 보고 있고 우리는 방금 `local` 인터럽트들을 비활성화 했다. 왜 지역(local) 이어야 하고 그것을 비활성화 했는가"? 이전 커널은 `cli` 명령을 사용하여 모든 프로세서들의 인터럽트를 비활성화하는 함수를 제공했다. 그 함수는 [제거](https://lwn.net/Articles/291956/) 되었고 이제는 현재 프로세서의 인터럽트를 비활성화/활성화를 하기 위해 `local_irq_{enabled,disable}`를 사용한다. `local_irq_disable` 매크로를 이용해서 인터럽트들을 비활성화 한 후에, 우리는 아래 값을 설정한다:

```C
early_boot_irqs_disabled = true;
```

`early_boot_irqs_disabled` 변수는 [include/linux/kernel.h](https://github.com/torvalds/linux/blob/master/include/linux/kernel.h) 에 선언되어 있다.:

```C
extern bool early_boot_irqs_disabled;
```

그리고 다른 곳에서 사용된다. 예를 들어 이 변수는 인터럽트가 비활성화 되는 시점에 데드락에 빠지지 않는지 확인하기 위해 [kernel/smp.c](https://github.com/torvalds/linux/blob/master/kernel/smp.c)에 구현된 `smp_call_function_many` 함수에서 사용된다.:

```C
WARN_ON_ONCE(cpu_online(this_cpu) && irqs_disabled()
                     && !oops_in_progress && !early_boot_irqs_disabled);
```

커널 초기화 중에 초기 트랩 초기화
--------------------------------------------------------------------------------

`local_disable_irq` 호출 이후에 볼 다음 함수는 `boot_cpu_init` 와 `page_address_init` 이다, 하지만 이것들은 인터럽트와 예외와는 연관되어 있지 않다.(이 함수들에 대해 더 알고 싶다면 리눅스 커널 초기화 과정에서 읽어보길 바란다.) 그 다음은 `setup_arch` 함수이다. 이 함수는 [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel.setup.c) 소스 코드 파일에 위치하고 다른 많은 아키텍처 의존적인 [것들](https://github.com/daeseokyoun/linux-insides/blob/korean-trans/Initialization/linux-initialization-4.md)을 초기화 한다. 우리가 지금 보고 있는 `setup_arch` 함수에서 인터럽트와 연관된 첫번째로 함수는 `early_trap_init` 이다. 이 함수는 [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/traps.c) 에 구현되어 있고, 몇개의 엔트리들을 `Interrupt Descriptor Table` 에 채운다.:

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

여기서 우리는 3 개의 다른 함수들을 볼 수 있다.

* `set_intr_gate_ist`
* `set_system_intr_gate_ist`
* `set_intr_gate`

이 모든 함수들은 [arch/x86/include/asm/desc.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/desc.h) 에 구현되어 있고 같진 않지만 비슷한 일을 한다. 첫 `set_intr_gate_ist` 함수는 새로운 인터럽트 게이트를 `IDT`에 삽입한다. 이것의 구현을 보자.:

```C
static inline void set_intr_gate_ist(int n, void *addr, unsigned ist)
{
        BUG_ON((unsigned)n > 0xFF);
        _set_gate(n, GATE_INTERRUPT, addr, 0, ist, __KERNEL_CS);
}
```

이 함수에서 맨 처음 하는 것은 인터럽트 [벡터 번호](http://en.wikipedia.org/wiki/Interrupt_vector_table)가 `0xff` (255)를 넘지 않는지 확인하는 것이다. 이전 [파트](https://github.com/daeseokyoun/linux-insides/blob/korean-trans/interrupts/interrupts-1.md) 에서 봤듯이 인터럽트 벡터 번호는 반드시 `0`와 `255` 사이에 있어야 한다. 다음 볼 함수는 `_set_gate` 이다. 이 함수는 주어진 인터럽트 게이트를 `IDT` 테이블에 설정하는 일을 한다.:

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

여기서 우리는 `gate_desc` 구조체의 의해 표현되는 깨끗한 `IDT` 엔트리를 가져오는 `pack_gate` 함수부터 시작하고 그것을 베이스 주소와 제한(limit), [인터럽트 스택 테이블](https://www.kernel.org/doc/Documentation/x86/x86_64/kernel-stacks),[특권 레벨](http://en.wikipedia.org/wiki/Privilege_level), 아래의 값들중 하나가 사용될 수 있는 인텁럽트 타입으로 채운다.:

* `GATE_INTERRUPT`벨
* `GATE_TRAP`
* `GATE_CALL`
* `GATE_TASK`

그리고 주어진 `IDT` 엔트리를 위해 `present` 비트를 설정한다.:

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

이 다음에는 채워진 인터럽트 게이트를 `native_write_idt_entry` 호출로 확장되는 `write_idt_entry` 매크로를 사용해서 `IDT`에 채우고 인터럽트 게이트를 주어진 인덱스에 있는 `idt_table` 로 복사한다.:

```C
#define write_idt_entry(dt, entry, g)           native_write_idt_entry(dt, entry, g)

static inline void native_write_idt_entry(gate_desc *idt, int entry, const gate_desc *gate)
{
        memcpy(&idt[entry], gate, sizeof(*gate));
}
```

`idt_table` 은 단지 `gate_desc` 의 배열이다.:

```C
extern gate_desc idt_table[];
```

두 번째 `set_system_intr_gate_ist` 함수는 `set_intr_gate_ist`와 단 한가지의 차이점이 있다.:

```C
static inline void set_system_intr_gate_ist(int n, void *addr, unsigned ist)
{
        BUG_ON((unsigned)n > 0xFF);
        _set_gate(n, GATE_INTERRUPT, addr, 0x3, ist, __KERNEL_CS);
}
```

알아 보겠는가? `_set_gate` 함수 호출의 4 번째 인자를 보자. 그것은 `0x3`이다. `set_intr_gate` 에서는 `0x0`이었다. 우리는 이 인자가 `DPL` (특권 레벨) 이라는 것을 알고 있다. `0` 이라면 가장 높은 특권레벨이고 `3`이라면 가장 낮은 특권 레벨이다. 이제 어떻게 `set_system_intr_gate_ist`, `set_intr_gate_ist`, `set_intr_gate` 들이 동작하는지 알아보았고 `early_trap_init` 함수로 돌아가자. 다시 살펴보면,:

```C
set_intr_gate_ist(X86_TRAP_DB, &debug, DEBUG_STACK);
set_system_intr_gate_ist(X86_TRAP_BP, &int3, DEBUG_STACK);
```

우리는 두개의 `#DB` 인터럽트와 `int3`를 위해 `IDT` 엔트리들을 설정했다. 이 함수들을 다음과 같은 3개의 인자들을 받는다:

* 인터럽트의 벡터 번호
* 인터럽트 핸들러의 주소
* 인터럽트 스택 테이블의 인덱스

이제 이 파트가 마무리되었다. 인터럽트와 핸들러에 관해 다음 파트에서 더 자세히 알아보자.

결론
--------------------------------------------------------------------------------

리눅스 커널의 인터럽트와 인터럽트 처리에 관한 두 번째 파트가 끝났다. 우리는 이전 파트에서 특정 이론을 보았고 현재 파트에서 인터럽트와 예외 처리에 관해 깊게 다루어 보기 시작했다. 우리는 리눅스 커널의 인터럽트와 연관된 극초반의 부분부터 살펴보기 시작했다. 다음 파트에서는 인터럽트 핸들리 처리 루틴에 관련해서 알아볼 것이다.

어떤 질문이나 제안이 있다면, twitter [0xAX](https://twitter.com/0xAX), [email](anotherworldofworld@gmail.com) 또는 [issue](https://github.com/0xAX/linux-insides/issues/new) 를 만들어 주길 바란다.

**나는 영어권의 사람이 아니고 이런 것에 대해 매우 미안해 하고 있다. 만약 어떤 실수를 발견한다면, 나에게 PR을 [linux-insides](https://github.com/0xAX/linux-internals)을 보내줘**

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
