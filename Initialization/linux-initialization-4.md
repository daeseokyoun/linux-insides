커널 초기화. Part 4.
================================================================================

커널 엔트리 포인트
================================================================================

[init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c) 소스 파일에 있는 `start_kernel` 함수 호출 바로 직전에 수행하는 모든 초기화 작업을 확인했을 것이다. `start_kernel` 는 일반적인 엔트리 포인트(모드 아키텍쳐가 부르는)이고 비록 `arch/` 디렉토리의 호출이 있지만 아키텍처에 독립적인 커널 코드이다. 이미 `start_kernel` 함수를 봤다면, 이 함수가 얼마나 많은 내용을 담고 있는지 알 수 있을 것이다. 이 함수는 약 `86` 개의 함수 호출을 갖고 있다. 그렇다. 이 함수는 매우 크고, 이 파트에서 이 함수에서 일어나는 모든 일들을 다룰 수는 없을 것이다. 현재 파트에서는 단지 시작에 불과하다. [커널 초기화 과정](https://github.com/daeseokyoun/linux-insides/blob/master/Initialization/README.md) 을 보면, 다음 파트에서 어떤 내용을 다룰지 대략적으로 알 수 있다.

`start_kernel` 함수의 주된 목적은 커널 초기화 과정을 마무리하고 `init` 프로세스를 실행하는 것이다. 첫 프로세스가 시작되기 전에 `start_kernel` 은 많은 일들을 처리해야 한다: [lock validator](https://www.kernel.org/doc/Documentation/locking/lockdep-design.txt) 활성화, 프로세서 ID 초기화, 초기 [cgroups](http://en.wikipedia.org/wiki/Cgroups) 서브 시스템 활성화, per-cpu 영역 설정, [vfs](http://en.wikipedia.org/wiki/Virtual_file_system) 내에 서로 다른 캐쉬들을 초기화, 메모리 관리 시스템, RCU, vmalloc, 스케쥴러, IRQ 들, ACPI 그리고 많은 것을 초기화 한다. 이 모든 단계 바로 뒤에는 이 파트에 마지막에서 첫 `init` 프로세스의 실행으로 볼 수 있다. 정말 많은 커널 코드가 우리를 기다리고 있다. 시작해보자.

**NOTE: 이 파트를 포함한 `리눅스 커널 초기화 과정` 내에서는 디버깅과 관련된 내용을 포함하지 않는다. 디버깅 관련사항은 디버깅 챕터에서 따로 살펴 볼 것이다.**

함수 속성
---------------------------------------------------------------------------------

[init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c) 에 `start_kernel`은 구현되어 있다. 이 함수는 `__init` 속성과 함께 선언이 되어 있고, 다른 파트에서 이미 봐서 알고 있을 수 있겠지만, 커널 초기화 동안에 필요했던 함수들은 이 속성과 함께 선언되어 있다.

```C
#define __init      __section(.init.text) __cold notrace__ // TODO 맨뒤 언더바 두개
```

초기화 과정이 끝나면, 커널은 `free_initmem` 함수 호출을 통해 이 섹션들을 해제한다. `__init` 은 두 개의 속성 `__cold` 와 `notrace` 이 함께 정의 되어 있다. 첫번째 `cold` 속성의 목적은 이 함수는 거의 사용되지 않는 함수로 설정하는 것이고 컴파일러에게 크기를 위한 최적화를 하도록 한다. 두번째 `notrace` 속성은 아래와 같이 선언되어 있다:

```C
#define notrace __attribute__((no_instrument_function))
```

`no_instrument_function` 은 컴파일러에게 이 속성을 사용한 함수에서 프로파일링을 위한 추가 코드를 생성하지 않도록 한다.

`start_kernel` 함수의 선언을 보면, 또 다른 `__visible` 속성을 볼 수 있을 것이다:

```
#define __visible __attribute__((externally_visible))
```

`externally_visible` 속성은 컴파일러에게 함수나 변수에게 `unusable` 로 마킹하는 것을 방지 하도록 한다. 다른 매크로 속성들을 확인하고 싶다면, [include/linux/init.h](https://github.com/torvalds/linux/blob/master/include/linux/init.h) 을 보시길 바란다.

start_kernel 의 첫 단계
--------------------------------------------------------------------------------

`start_kernel` 함수 첫 부분에 두 개의 변수 선언을 볼 수 있다:

```C
char *command_line;
char *after_dashes;
```

첫 번째 변수는 커널 명령 라인의 포인터을 위한 포인터이고, 두 번째는 `name=value` 의 형태로 주어지는 인자들의 문자열을 파싱하고 특정 키워드를 찾아 알맞은 핸들러를 수행하도록 하는 `parse_args` 함수의 결과를 저장하기 위한 것이다. 지금은 이 변수들을 자세히 살펴 보진 않을 것이고, 다음 파트에서 보도록 하자. 다음 단계는 `set_task_stack_end_magic` 함수의 호출이다. 이 함수는 `init_task` 의 주소를 받아서 canary 값인 `STACK_END_MAGIC` (`0x57AC6E9D`) 을 설정한다. `init_task` 은 초기 태스크 구조체를 나타낸다.:

```C
struct task_struct init_task = INIT_TASK(init_task);
```

`task_struct` 는 하나의 프로세스에 관련된 모든 정보를 저장한다. 나는 너무 설명이 많이 필요한 이 구조체 대해서 설명하지 않은 것이다. [include/linux/sched.h](https://github.com/torvalds/linux/blob/master/include/linux/sched.h#L1278) 에서 선언을 찾아 볼 수 있다. 보면 알겠지만, `task_struct` 에는 100 개가 넘는 필드가 있다! 비록 `task_struct` 의 설명을 이 책에서 얻을 수는 없겠지만, 우리는 리눅스 커널에서 `process(프로세스)` 를 기술하면서 기본적인 구조체이기 때문에 매우 많이 사용될 것이고, 사용되는 과정에서 각각에 설명이 필요한 필드들은 설명을 진행할 것이다.

`init_task` 의 선언을 보면, `INIT_TASK` 매크로에 의해 초기화 된다는 것을 알 수 있다. 이 매크로는 [include/linux/init_task.h](https://github.com/torvalds/linux/blob/master/include/linux/init_task.h) 에 있고, 첫 프로세스를 위한 값들을 `init_task`에 채워넣는다. 예를 들어, 그것은 아래와 같이 설정한다:

* init 프로세스 상태를 0 또는 `runnable` 로 한다. 수행가능한 상태(`runnable`) 프로세스는 단지 CPU 에서 수행되기를 기다리는 상태 이다.
* init 프로세스 플래그 - `PF_KTHREAD` 로 한다. 이는 커널 쓰레드라는 의미 이다.
* 수행가능한 상태 태스크들의 리스트
* 프로세스 주소 공간
* init 프로세스 스택은 `thread_info` 와 프로세스 스택을 갖고 있는 `thread_union` 에 `&init_thread_info` 를 할당하여 정한다.:

```C
union thread_union {
	  struct thread_info thread_info;
    unsigned long stack[THREAD_SIZE/sizeof(long)];
};
```

모든 프로세스는 각자의 스택을 가지고 그 크기는 16 킬로바이트 이거나 4 개의 페이지 프레임을 가진다. 물론 `x86_64` 기준이다. 우리는 `unsigned long` 의 배열으로 선언된 것을 볼 수 있다. 다음 항목으로는 `thread_info` 가 있고 아래와 같이 선언되어 있다:

```C
struct thread_info {
        struct task_struct      *task;
        struct exec_domain      *exec_domain;
        __u32                   flags;
        __u32                   status;
        __u32                   cpu;
        int                     saved_preempt_count;
        mm_segment_t            addr_limit;
        struct restart_block    restart_block;
        void __user             *sysenter_return;* // TODO 마지막 별 지우기
        unsigned int            sig_on_uaccess_error:1;
        unsigned int            uaccess_err:1;
};
```

이 구조체는 52 바이트를 차지 한다. `thread_info` 구조체는 쓰레드의 아키텍처 의존적인 정보를 포함한다. 우리는 `x86_64` 에서 스택은 아래로 자라고 `thread_union.thread_info` 는 스택의 맨 바닥에 저장되어 있다는 것을 알 것이다. 그래서 프로세스 스택은 16 KB 이고, `thread_info` 는 스택의 맨 밑바닥에 있다. 남은 쓰레드의 크기는 `16 킬로바이트 - 52 바이트 = 16332 bytes` 가 될 것이다. 아시다 시피 `thread_union` 의 타입인 [union](http://en.wikipedia.org/wiki/Union_type) 은 구조체가 아니기 때문에 `thread_info` 와 스택은 같은 메모리 공간을 공유할 것이다.

개략적으로 아래와 같이 표현될 수 있다.:

```C
+-----------------------+
|                       |
|                       |
|        stack          |
|                       |
|_______________________|
|          |            |
|          |            |
|          |            |
|__________↓____________|             +--------------------+
|                       |             |                    |
|      thread_info      |<----------->|     task_struct    |
|                       |             |                    |
+-----------------------+             +--------------------+
```

http://www.quora.com/In-Linux-kernel-Why-thread_info-structure-and-the-kernel-stack-of-a-process-binds-in-union-construct

그래서 `INIT_TASK` 매크로는 `task_struct 의` 많은 필드들을 채운다. 내가 위에서 언급했듯이 모든 필드들을 살펴보진 않을 것이고 차차 나오게 되면 그때 하나씩 보도록 하자.

이제 `set_task_stack_end_magic` 함수로 돌아가 보자. 이 함수는 [kernel/fork.c](https://github.com/torvalds/linux/blob/master/kernel/fork.c#L297) 에 선언되어 있고 [canary](http://en.wikipedia.org/wiki/Stack_buffer_overflow) 값을 스택 오버플로우를 막기 위해 `init` 프로세스 스택에 설정한다.

```C
void set_task_stack_end_magic(struct task_struct *tsk)
{
	unsigned long *stackend;
	stackend = end_of_stack(tsk);
	*stackend = STACK_END_MAGIC; /* for overflow detection */* // TODO 마지막 별
}
```

이 함수의 구현은 간단하다. `set_task_stack_end_magic` 함수는 주어진 `task_struct` 타입의 변수를 가지고 `end_of_stack` 함수를 통해 스택의 끝을 얻는다. 초기 (`x86_64` 이외의 모든 아키텍쳐) 스택은 `thread_info` 구조체에 위치한다. 그래서 프로세스 스택의 끝은 `CONFIG_STACK_GROWSUP` 구성 옵션에 의존적이다. `x86_64` 아키텍쳐를 배우고 있으니까, 스택은 아래로 자란다. 그래서 프로세스 스택의 끝은 아래와 같이 된다:

```C
(unsigned long *)(task_thread_info(p) + 1);
```

`task_thread_info` 는 단지 `INIT_TASK` 매크로에 의해 채워진 스택을 반환한다.:

```C
#define task_thread_info(task)  ((struct thread_info *)*(task)->stack) // TODO 타입뒤에 두번째 별 지우기
```

리눅스 커널 `v4.9-rc1` 릴리즈에서는, `thread_info` 구조체는 아마도 단지 플래그들과 리눅스 커널에서 쓰레드를 표현하는 `task_struct` 구조체에 있는 스택 포인터만 포함 할 것이다. 그것은 `x86_64` 에서는 기본적으로 활성화 되어 있는  `CONFIG_THREAD_INFO_IN_TASK` 커널 구성 옵션에 의존적이다. 당신은 [init/Kconfig](https://github.com/torvalds/linux/blob/master/init/Kconfig) 를 확인해 보면 관련 내용을 확인 할 수 있을 것이다.:

```
config THREAD_INFO_IN_TASK
	bool
	help
	  Select this to move thread_info off the stack into task_struct.  To
	  make this work, an arch will need to remove all thread_info fields
	  except flags and fix any runtime bugs.

	  One subtle change that will be needed is to use try_get_task_stack()
	  and put_task_stack() in save_thread_stack_tsk() and get_wchan().
  [번역] 이 옵션의 선택으로 thread_info 를 task_struct 로 옮기고, thread_union 에
	스택만 남긴다. 이것이 가능하려면, arch 는 플래그를 제외한 thread_info 관련 항목을
	모두 지우고 runtime 버그를 수정해야 한다.
```

그리고 [arch/x86/Kconfig](https://github.com/torvalds/linux/blob/master/arch/x86/Kconfig):

```
config X86
	def_bool y
        ...
        ...
        ...
        select THREAD_INFO_IN_TASK
        ...
        ...
        ...
```

이러면, 주어진 `task_struct` 구조체로 부터 쓰레드 스택의 끝을 바로 얻을 수 있다.:

```C
#ifdef CONFIG_THREAD_INFO_IN_TASK
static inline unsigned long *end_of_stack(const struct task_struct *task)* // TODO 마지막 별
{
	return task->stack;
}
#endif
```

init 프로세스 스택의 끝을 얻었으니, 거기에 `STACK_END_MAGIC` 값을 쓴다. `canary` 가 설정되면, 우리는 아래와 같이 확인 가능하다.:

```C
if (*end_of_stack(task) != STACK_END_MAGIC) {* // TODO 마지막 별
        //
        // handle stack overflow here
	//
}
```

`set_task_stack_end_magic` 가 완료되면, 다음 호출 함수는 `smp_setup_processor_id` 이다. 이 함수는 `x86_64` 에서는 비어 있다.:

```C
void __init __weak smp_setup_processor_id(void)
{
}
```

모든 아키텍쳐에서 필요한 사항은 아니고, 몇몇의 [s390](http://en.wikipedia.org/wiki/IBM_ESA/390) 와 [arm64](http://en.wikipedia.org/wiki/ARM_architecture#64.2F32-bit_architecture)에서 필요하다.

`start_kernel` 에서 다음 함수는 `debug_objects_early_init` 이다. 이 함수의 구현은 `lockdep_init` 와 거의 유사하지만, 오브젝트 디버깅을 위해 해쉬들을 초기화 한다. 이 챕터에서는 디버깅 목적을 위한 내용을 다루지 않을 것이다.

`debug_object_early_init` 호출 다음에 `-fstack-protector` gcc 옵션을 위해 canary 값을 `task_struct->canary` 에 채우는 `boot_init_stack_canary` 함수를 호출한다. 이 함수는 `CONFIG_CC_STACKPROTECTOR` 구성 옵션에 의존적이며, 만약 이 옵션이 비활성화 되어 있다면, `boot_init_stack_canary` 함수는 아무것도 하지 않을 것이며, 반대로 활성화 되어 있다면, [TSC](http://en.wikipedia.org/wiki/Time_Stamp_Counter) 와 랜덤 풀을 기반으로 난수값을 생성한다.:

```C
get_random_bytes(&canary, sizeof(canary));
tsc = __native_read_tsc();
canary += tsc + (tsc << 32UL);
```

난수를 얻은 다음에, 우리는 `task_struct` 구조체의 `stack_canary` 항목을 그 값으로 채운다.:

```C
current->stack_canary = canary;
```

그리고 이 값을 IRQ 스택의 맨 위에 쓴다.

```C
this_cpu_write(irq_stack_union.stack_canary, canary); // read below about this_cpu_write
```

이 부분도 자세히 다루지 않을 것이고, [IRQs](http://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29) 관련된 파트에서 더 살펴 보자. canary 값이 설정되면, [include/linux/percpu-defs.h](https://github.com/torvalds/linux/blob/master/include/linux/percpu-defs.h) 에 선언된 대로 `arch_local_irq_disable` 함수의 호출을 하는 `local_irq_disable` 함수를 통해 현재 인터럽트를 금지하도록 한다.:

```C
static inline notrace void arch_local_irq_enable(void)
{
        native_irq_enable();
}
```

`native_irq_enable` 함수는 `x86_64` 에서 `cli` 명령어를 수행한다. 인터럽트를 금지함으로써 우리는 CPU 비트맵에서 주어진 ID 로 현재 CPU를 등록할 수 있다.

첫번째 프로세서 활성화
---------------------------------------------------------------------------------

이제 봐야 하는 함수는 `boot_cpu_init` 이다. 이 함수는 부트스트랩 프로세서를 위해 다양한 CPU 마스크를 초기화 한다. 제일 처음으로는 아래와 같은 호출로 부트스트랩 프토세서의 ID 를 얻어온다:

```C
int cpu = smp_processor_id();
```

지금은 그냥 0 이다. 만약 `CONFIG_DEBUG_PREEMPT` 구성 옵션이 비활성화 상태라면, `smp_processor_id` 함수는 `raw_smp_processor_id` 의 호출을 하고, 그 호출은 아래와 같다:

```C
#define raw_smp_processor_id() (this_cpu_read(cpu_number))
```

`this_cpu_read` 와 같은 다음 많은 시리즈(`this_cpu_write`, `this_cpu_add` 과 같은)의 함수가 [include/linux/percpu-defs.h](https://github.com/torvalds/linux/blob/master/include/linux/percpu-defs.h) 에 선언되어 있고, `this_cpu` 수행을 표현한다. 이 모든 수행은 현재 프로세서와 연결된 [per-cpu](http://0xax.gitbooks.io/linux-insides/content/Theory/per-cpu.html) 변수들을 최적화된 접근 방법을 제공한다. 현재 우리의 상황에 `this_cpu_read`는 아래와 같다:

```
__pcpu_size_call_return(this_cpu_read_, pcp)
```

우리는 `raw_smp_processor_id` 매크로를 통해 `this_cpu_read` 를 호출하고 `pcp` 로 `cpu_number` 를 넘겨주었다. 이제 `__pcpu_size_call_return` 구현을 살펴보자.:

```C
#define __pcpu_size_call_return(stem, variable)                         \
({                                                                      \
        typeof(variable) pscr_ret__;                                    \
        __verify_pcpu_ptr(&(variable));                                 \
        switch(sizeof(variable)) {                                      \
        case 1: pscr_ret__ = stem##1(variable); break;                  \
        case 2: pscr_ret__ = stem##2(variable); break;                  \
        case 4: pscr_ret__ = stem##4(variable); break;                  \
        case 8: pscr_ret__ = stem##8(variable); break;                  \
        default:                                                        \
                __bad_size_call_parameter(); break;                     \
        }                                                               \
        pscr_ret__;                                                     \
})
```

그렇다, 조금 이상해 보일 수는 있지만 쉬운 내용이다. 첫째로 `pscr_ret__` 변수를 `int` 타입으로 선언하는 것을 볼 수 있다. 왜 int 인가? `variable` 는 `cpu_number` 이고 그것은 per-cpu 로써 int 로 선언되어 있다:

```C
DECLARE_PER_CPU_READ_MOSTLY(int, cpu_number);
```

다음 단계는 `cpu_number` 주소와 함께 `__verify_pcpu_ptr` 의 호출이다. `__veryf_pcpu_ptr` 은 주어진 인자가 per-cpu 포인터 인지 검증한다. 그 다음에 `variable` 의 크기에 의존적으로 `pscr_ret__`를 설정한다. `cpu_number` 변수는 `int` 이니, 그 크기는 4 바이트 일 것이다. 그것은 `pscr_ret__` 로 부터 `this_cpu_read_4(cpu_number)` 를 얻을 수 있다는 의미이다. `__pcpu_size_call_return` 의 마지막에는 단지 그것을 호출하는 것으로 끝을 낸다.:

```C
#define this_cpu_read_4(pcp)       percpu_from_op("mov", pcp)
```

`this_cpu_read_4` 는 `percpu_from_op` 를 호출하고 `mov` 명령어와 per-cpu 변수를 넘겨준다. 결국 `percpu_from_op` 는 아래와 같이 인라인 어셈블리 호출로 변환된다.:

```C
asm("movl %%gs:%1,%0" : "=r" (pfo_ret__) : "m" (cpu_number))
```

Let's try to understand how it works and what it does. The `gs` segment register contains the base of per-cpu area. Here we just copy `common_cpu` which is in memory to the `pfo_ret__` with the `movl` instruction. Or with another words:

```C
this_cpu_read(common_cpu)
```

is the same as:

```C
movl %gs:$common_cpu, $pfo_ret__
```

As we didn't setup per-cpu area, we have only one - for the current running CPU, we will get `zero` as a result of the `smp_processor_id`.

As we got the current processor id, `boot_cpu_init` sets the given CPU online, active, present and possible with the:

```C
set_cpu_online(cpu, true);
set_cpu_active(cpu, true);
set_cpu_present(cpu, true);
set_cpu_possible(cpu, true);
```

All of these functions use the concept - `cpumask`. `cpu_possible` is a set of CPU ID's which can be plugged in at any time during the life of that system boot. `cpu_present` represents which CPUs are currently plugged in. `cpu_online` represents subset of the `cpu_present` and indicates CPUs which are available for scheduling. These masks depend on the `CONFIG_HOTPLUG_CPU` configuration option and if this option is disabled `possible == present` and `active == online`. Implementation of the all of these functions are very similar. Every function checks the second parameter. If it is `true`, it calls `cpumask_set_cpu` or `cpumask_clear_cpu` otherwise.

For example let's look at `set_cpu_possible`. As we passed `true` as the second parameter, the:

```C
cpumask_set_cpu(cpu, to_cpumask(cpu_possible_bits));
```

will be called. First of all let's try to understand the `to_cpumask` macro. This macro casts a bitmap to a `struct cpumask *`. CPU masks provide a bitmap suitable for representing the set of CPU's in a system, one bit position per CPU number. CPU mask presented by the `cpu_mask` structure:

```C
typedef struct cpumask { DECLARE_BITMAP(bits, NR_CPUS); } cpumask_t;
```

which is just bitmap declared with the `DECLARE_BITMAP` macro:

```C
#define DECLARE_BITMAP(name, bits) unsigned long name[BITS_TO_LONGS(bits)]
```

As we can see from its definition, the `DECLARE_BITMAP` macro expands to the array of `unsigned long`. Now let's look at how the `to_cpumask` macro is implemented:

```C
#define to_cpumask(bitmap)                                              \
        ((struct cpumask *)(1 ? (bitmap)                                \
                            : (void *)sizeof(__check_is_bitmap(bitmap))))
```

I don't know about you, but it looked really weird for me at the first time. We can see a ternary operator here which is `true` every time, but why the `__check_is_bitmap` here? It's simple, let's look at it:

```C
static inline int __check_is_bitmap(const unsigned long *bitmap)
{
        return 1;
}
```

Yeah, it just returns `1` every time. Actually we need in it here only for one purpose: at compile time it checks that the given `bitmap` is a bitmap, or in other words it checks that the given `bitmap` has a type of `unsigned long *`. So we just pass `cpu_possible_bits` to the `to_cpumask` macro for converting the array of `unsigned long` to the `struct cpumask *`. Now we can call `cpumask_set_cpu` function with the `cpu` - 0 and `struct cpumask *cpu_possible_bits`. This function makes only one call of the `set_bit` function which sets the given `cpu` in the cpumask. All of these `set_cpu_*` functions work on the same principle.

If you're not sure that this `set_cpu_*` operations and `cpumask` are not clear for you, don't worry about it. You can get more info by reading the special part about it - [cpumask](http://0xax.gitbooks.io/linux-insides/content/Concepts/cpumask.html) or [documentation](https://www.kernel.org/doc/Documentation/cpu-hotplug.txt).

As we activated the bootstrap processor, it's time to go to the next function in the `start_kernel.` Now it is `page_address_init`, but this function does nothing in our case, because it executes only when all `RAM` can't be mapped directly.

Print linux banner
---------------------------------------------------------------------------------

The next call is `pr_notice`:

```C
#define pr_notice(fmt, ...) \
    printk(KERN_NOTICE pr_fmt(fmt), ##__VA_ARGS__)
```

as you can see it just expands to the `printk` call. At this moment we use `pr_notice` to print the Linux banner:

```C
pr_notice("%s", linux_banner);
```

which is just the kernel version with some additional parameters:

```
Linux version 4.0.0-rc6+ (alex@localhost) (gcc version 4.9.1 (Ubuntu 4.9.1-16ubuntu6) ) #319 SMP
```

Architecture-dependent parts of initialization
---------------------------------------------------------------------------------

The next step is architecture-specific initialization. The Linux kernel does it with the call of the `setup_arch` function. This is a very big function like `start_kernel` and we do not have time to consider all of its implementation in this part. Here we'll only start to do it and continue in the next part. As it is `architecture-specific`, we need to go again to the `arch/` directory. The `setup_arch` function defined in the [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/setup.c) source code file and takes only one argument - address of the kernel command line.

This function starts from the reserving memory block for the kernel `_text` and `_data` which starts from the `_text` symbol (you can remember it from the [arch/x86/kernel/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head_64.S#L46)) and ends before `__bss_stop`. We are using `memblock` for the reserving of memory block:

```C
memblock_reserve(__pa_symbol(_text), (unsigned long)__bss_stop - (unsigned long)_text);
```

You can read about `memblock` in the [Linux kernel memory management Part 1.](http://0xax.gitbooks.io/linux-insides/content/mm/linux-mm-1.html). As you can remember `memblock_reserve` function takes two parameters:

* base physical address of a memory block;
* size of a memory block.

We can get the base physical address of the `_text` symbol with the `__pa_symbol` macro:

```C
#define __pa_symbol(x) \
	__phys_addr_symbol(__phys_reloc_hide((unsigned long)(x)))
```

First of all it calls `__phys_reloc_hide` macro on the given parameter. The `__phys_reloc_hide` macro does nothing for `x86_64` and just returns the given parameter. Implementation of the `__phys_addr_symbol` macro is easy. It just subtracts the symbol address from the base address of the kernel text mapping base virtual address (you can remember that it is `__START_KERNEL_map`) and adds `phys_base` which is the base address of `_text`:

```C
#define __phys_addr_symbol(x) \
 ((unsigned long)(x) - __START_KERNEL_map + phys_base)
```

After we got the physical address of the `_text` symbol, `memblock_reserve` can reserve a memory block from the `_text` to the `__bss_stop - _text`.

Reserve memory for initrd
---------------------------------------------------------------------------------

In the next step after we reserved place for the kernel text and data is reserving place for the [initrd](http://en.wikipedia.org/wiki/Initrd). We will not see details about `initrd` in this post, you just may know that it is temporary root file system stored in memory and used by the kernel during its startup. The `early_reserve_initrd` function does all work. First of all this function gets the base address of the ram disk, its size and the end address with:

```C
u64 ramdisk_image = get_ramdisk_image();
u64 ramdisk_size  = get_ramdisk_size();
u64 ramdisk_end   = PAGE_ALIGN(ramdisk_image + ramdisk_size);
```

All of these parameters are taken from `boot_params`. If you have read the chapter about [Linux Kernel Booting Process](http://0xax.gitbooks.io/linux-insides/content/Booting/index.html), you must remember that we filled the `boot_params` structure during boot time. The kernel setup header contains a couple of fields which describes ramdisk, for example:

```
Field name:	ramdisk_image
Type:		write (obligatory)
Offset/size:	0x218/4
Protocol:	2.00+

  The 32-bit linear address of the initial ramdisk or ramfs.  Leave at
  zero if there is no initial ramdisk/ramfs.
```

So we can get all the information that interests us from `boot_params`. For example let's look at `get_ramdisk_image`:

```C
static u64 __init get_ramdisk_image(void)
{
        u64 ramdisk_image = boot_params.hdr.ramdisk_image;

        ramdisk_image |= (u64)boot_params.ext_ramdisk_image << 32;

        return ramdisk_image;
}
```

Here we get the address of the ramdisk from the `boot_params` and shift left it on `32`. We need to do it because as you can read in the [Documentation/x86/zero-page.txt](https://github.com/0xAX/linux/blob/master/Documentation/x86/zero-page.txt):

```
0C0/004	ALL	ext_ramdisk_image ramdisk_image high 32bits
```

So after shifting it on 32, we're getting a 64-bit address in `ramdisk_image` and we return it. `get_ramdisk_size` works on the same principle as `get_ramdisk_image`, but it used `ext_ramdisk_size` instead of `ext_ramdisk_image`. After we got ramdisk's size, base address and end address, we check that bootloader provided ramdisk with the:

```C
if (!boot_params.hdr.type_of_loader ||
    !ramdisk_image || !ramdisk_size)
	return;
```

and reserve memory block with the calculated addresses for the initial ramdisk in the end:

```C
memblock_reserve(ramdisk_image, ramdisk_end - ramdisk_image);
```

Conclusion
---------------------------------------------------------------------------------

It is the end of the fourth part about the Linux kernel initialization process. We started to dive in the kernel generic code from the `start_kernel` function in this part and stopped on the architecture-specific initialization in the `setup_arch`. In the next part we will continue with architecture-dependent initialization steps.

If you have any questions or suggestions write me a comment or ping me at [twitter](https://twitter.com/0xAX).

**Please note that English is not my first language, And I am really sorry for any inconvenience. If you find any mistakes please send me a PR to [linux-insides](https://github.com/0xAX/linux-insides).**

Links
--------------------------------------------------------------------------------

* [GCC function attributes](https://gcc.gnu.org/onlinedocs/gcc/Function-Attributes.html)
* [this_cpu operations](https://www.kernel.org/doc/Documentation/this_cpu_ops.txt)
* [cpumask](http://www.crashcourse.ca/wiki/index.php/Cpumask)
* [lock validator](https://www.kernel.org/doc/Documentation/locking/lockdep-design.txt)
* [cgroups](http://en.wikipedia.org/wiki/Cgroups)
* [stack buffer overflow](http://en.wikipedia.org/wiki/Stack_buffer_overflow)
* [IRQs](http://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29)
* [initrd](http://en.wikipedia.org/wiki/Initrd)
* [Previous part](https://github.com/0xAX/linux-insides/blob/master/Initialization/linux-initialization-3.md)
* [SSP (stack-smashing protector)](http://egloos.zum.com/studyfoss/v/5279959)
