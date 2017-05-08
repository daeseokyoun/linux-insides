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

`this_cpu_read` 와 같은 다음 많은 시리즈(`this_cpu_write`, `this_cpu_add` 과 같은)의 함수가 [include/linux/percpu-defs.h](https://github.com/torvalds/linux/blob/master/include/linux/percpu-defs.h) 에 선언되어 있고, `this_cpu` 수행을 표현한다. 이 모든 수행은 현재 프로세서와 연결된 [per-cpu](https://github.com/daeseokyoun/linux-insides/blob/master/Theory/per-cpu.html) 변수들을 최적화된 접근 방법을 제공한다. 현재 우리의 상황에 `this_cpu_read`는 아래와 같다:

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

이것이 어떻게 동작하고 무엇을 하는지 이해해보자. `gs` 세그먼트 레지스터는 per-cpu 영역의 기본 주소를 갖고 있다. 여기서 우리는 단지 `movl` 명령어로 `cpu_number` 에서 `pfo_ret__` 로 복사를 한다. 즉, :

```C
this_cpu_read(cpu_number)
```

는 아래와 같다:

```C
movl %gs:$cpu_number, $pfo_ret__
```

아직 per-cpu 영역이 설정되지 않아, 우리는 단지 현재 수행중인 CPU 를 위한 하나만 갖고 있다. 그래서 `smp_processor_id` 의 결과는 0으로 얻어질 것이다.

우리는 현재 프로세서 ID 를 얻어, `boot_cpu_init` 에서 주어진 CPU 를 online, active, present 그리고 possible 상태로 설정한다.:

```C
set_cpu_online(cpu, true);
set_cpu_active(cpu, true);
set_cpu_present(cpu, true);
set_cpu_possible(cpu, true);
```

위의 모든 함수들은 `cpumask` 라는 개념을 사용한다. `cpu_possible` 은 시스템 부팅 중에 언제든지 끼워넣을 수 있는 CPU ID 들의 집합이다. `cpu_present`는 현재 존재하는 CPU ID 들의 집합이다. `cpu_online` 은 `cpu_present` 의 서브셋인데, 스케쥴러가 관리하는 현재 존재하는 CPU 를 가르킨다. 이 mask 들은 `CONFIG_HOTPLUG_CPU` 구성 옵션에 의존적이며, 이 옵션이 비활성화 되어 있다면, `possible == present` 와 `active == online` 의 공식이 성립한다. 이 모든 것을 위한 함수의 구현들은 매우 비슷하다. 모든 함수는 두 번째 인자를 확인한다. 만약 그것이 `true` 이면, `cpumask_set_cpu` 를 호출하고 아니라면, `cpumask_clear_cpu`를 호출한다.

예를들어 `set_cpu_possible`를 보자. 두 번째 인자로 `true`를 넘긴다면, :

```C
cpumask_set_cpu(cpu, to_cpumask(cpu_possible_bits));
```

위의 함수가 호출될 것이다. 우선 `to_cpumask` 매크로를 이해해야 한다. 이 매크로는 비트맵을 `struct cpumask *` 로 타입 캐스팅한다. CPU mask 는 시스템에서 CPU 의 설정을 비트맵으로 제공한다. 하나의 비트 위치는 CPU 번호를 의미한다. CPU mask 는 `cpu_mask` 구조체로 표현된다.:

```C
typedef struct cpumask { DECLARE_BITMAP(bits, NR_CPUS); } cpumask_t;
```로

`DECLARE_BITMAP` 매크로와 함께 비트맵을 선언한다.:

```C
#define DECLARE_BITMAP(name, bits) unsigned long name[BITS_TO_LONGS(bits)]
```

이 정의에서 볼 수 있듯이, `DECLARE_BITMAP`는 `unsigned long` 의 배열로 확장된다. 이제 `to_cpumask` 매크로가 어떻게 구현되었는지 살펴 보자:

```C
#define to_cpumask(bitmap)                                              \
        ((struct cpumask *)(1 ? (bitmap)                                \
                            : (void *)sizeof(__check_is_bitmap(bitmap))))
```

이 매크로의 구성을 보면 이상하다는 것을 알 수 있다. 무조건 `true` 를 가지는 조건의 문장이지만, 왜 `__check_is_bitmap` 가 필요한 것인가? 그것은 아주 간단하다. 아래를 보자:

```C
static inline int __check_is_bitmap(const unsigned long *bitmap*) // TODO 마지막 별지우기
{
        return 1;
}
```

`__check_is_bitmap` 함수는 무조건 `1`을 반환한다. 실제 우리는 이것을 사용하는 것은 단지 하나의 목적밖에 없다: 컴파일 시간에 그것은 주어진 `bitmap` 을 bitmap 인지 아니면 `unsigned long *` 의 타입을 가지는 `bitmap` 변수인지 확인한다. 그래서 우리는 넘어온 `cpu_possible_bits` 를 `to_cpumask` 매크로를 통해 `unsigned long` 에서 `struct cpumask *` 으로 변환한다. 이제 우리는 `cpumask_set_cpu` 함수를 `cpu` - 0 과 `struct cpumask *cpu_possible_bits` 함께 호출할 수 있다. 이 함수는 주어진 cpu mask 내에 `cpu` 가 설정하도록 하는 `set_bit` 함수를 호출한다. `set_cpu_*` 이런 종류의 함수들은 같은 원칙을 갖고 수행한다.

만약 `set_cpu_*` 수행과 `cpumask` 에 관련해서 명확하지 않는다해도 걱정말자. 당신이 추가적으로 정보를 얻을 수 있는 파트를 마련해 두었다. [cpumask](https://github.com/daeseokyoun/linux-insides/blob/master/Concepts/cpumask.md) 와 [documentation](https://www.kernel.org/doc/Documentation/cpu-hotplug.txt)이다.

우리는 부트스트랩 프로세서를 활성화했으니, `start_kernel` 내의 그 다음 함수를 살펴보자. 이제 `page_address_init` 함수이다. 하지만 이 함수는 우리가 진행하고 있는 경우에는 아무 것도 하지 않는다. 왜냐하면 그것은 모든 `RAM` 을 직접 맵핑 할 수 없는 경우에만 실행되기 때문이다.

리눅스 배너 출력
---------------------------------------------------------------------------------

다음 호출은 `pr_notice`이다:

```C
#define pr_notice(fmt, ...) \
    printk(KERN_NOTICE pr_fmt(fmt), ##__VA_ARGS__)
```

이 함수는 단지 `printk` 의 호출을 확장했다는 것을 볼 수 있다. 그리고 이것은 리눅스 배너를 출력할때 사용된다:

```C
pr_notice("%s", linux_banner);
```

추가적인 인자들과 커널 버전을 출력한다.:

```
Linux version 4.0.0-rc6+ (alex@localhost) (gcc version 4.9.1 (Ubuntu 4.9.1-16ubuntu6) ) #319 SMP
```

아키텍처 의존적인 부분의 초기화
---------------------------------------------------------------------------------

다음 단계는 아키텍처 의존적인 초기화이다. 리눅스 커널은 `setup_arch` 함수를 통해 그런 일을 한다. 이 함수는 `start_kernel` 과 같이 아주 큰 함수이고 이 파트에서 모든 구현 내용을 살펴보기에는 시간이 부족한다. 여기서는 단지 시작만하고 다른 파트에서 계속해서 살펴보도록 하자. `architecture-specific(아키텍처에 특정한)` 부분으로써, 우리는 `arch/` 디렉토리로 다시 가봐야 한다. `setup_arch` 함수는 [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/setup.c) 파일에 구현되어 있으며, 커널 명령 라인의 주소만 인자로 받는다.

이 함수는 커널의 `_text` 와 `_text` 심볼로 부터 시작하는 `_data`를 위해 메모리 블럭을 예약하는 것 부터 시작한다.(당신은 이것이 [arch/x86/kernel/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head_64.S#L46)) 에 정의되어 확인 가능하다.) 그리고 `__bss_stop` 까지가 범위이다. 우리는 `memblock` 을 이용하여 메모리 블럭을 예약한다.:

```C
memblock_reserve(__pa_symbol(_text), (unsigned long)__bss_stop - (unsigned long)_text___); // TODO 마지막 언더바 3개
```

당신은 [리눅스 커널 메모리 관리 Part 1.](https://github.com/daeseokyoun/linux-insides/blob/master/mm/linux-mm-1.html) 에서 `memblock` 관련 내용을 볼 수 있다. `memblock_reserve` 는 두개의 인자를 받는다는 것을 기억하자:

* 메모리 블럭 시작 물리 주소
* 메모리 블럭의 크기

`__pa_symbol` 매크로를 이용하여 `_text` 심볼의 베이스 물리 주소를 얻을 수 있다:

```C
#define __pa_symbol(x) \
	__phys_addr_symbol(__phys_reloc_hide__((unsigned long)(x))) // TODO hide 뒤에 언더바 2개
```

`__pa_symbol` 매크로는 `__phys_reloc_hide` 를 주어진 인자를 넘겨 호출한다. `__phys_reloc_hide` 매크로는 `x86_64` 를 위해 아무 것도 하지 않을 것이고 주어진 인자를 단지 반환할 것이다. `__phys_addr_symbol` 매크로의 구현은 아주 쉽다. 그것은 단지 심볼 주소에서 커널 텍스트 가상 주소의 시작(`__START_KERNEL_map`)과 `_text` 의 시작주소인 `phys_base` 를 더한 것을 빼준다.:

```C
#define __phys_addr_symbol(x) \
 ((unsigned long)(x) - __START_KERNEL_map__ + phys_base) // TODO map 뒤에 언더바 2개
```

`_text` 심볼의 물리 주소를 얻은 다음, `memblock_reserve` 는 `_text` 에서 `__bss_stop - _text` 크기 만큼 메모리 블럭을 예약할 수 있다.

initrd 를 위한 메모리 예약
---------------------------------------------------------------------------------

커널 텍스와 데이터를 위한 공간 예약이후에는 [initrd](http://en.wikipedia.org/wiki/Initrd)를 위한 공간의 확보되어야 한다. 우리는 `initrd` 관련된 내용을 자세히 살펴보지는 않을 것이고, 단지 이것은 커널이 시작하는 과정에서 사용되기 위해 메모리에 저장되는 임시의 루트 파일 시스템이라고 알고 있으면 된다. `early_reserve_initrd` 함수에서 이 공간 확보를 한다. 이함수의 처음 하는 일은 램디스크의 시작 주소, 크기 그리고 마지막 주소를 얻는 것이다:

```C
u64 ramdisk_image = get_ramdisk_image();
u64 ramdisk_size  = get_ramdisk_size();
u64 ramdisk_end   = PAGE_ALIGN(ramdisk_image + ramdisk_size);
```

이 모든 내용은 `boot_params` 로 부터 가져온다. 만약 [리눅스 커널 부팅 과정](https://github.com/daeseokyoun/linux-insides/blob/master/Booting/README.md) 을 읽었다면, 부팅 시간에 `boot_params` 가 어떻게 채워지는지 파악했을 것이다. 커널 설정 헤더는 램디스크와 관련된 몇몇의 항목을 갖고 있다. 예를 들면:

```
Field name:	ramdisk_image
Type:		write (obligatory)
Offset/size:	0x218/4
Protocol:	2.00+

  The 32-bit linear address of the initial ramdisk or ramfs.  Leave at
  zero if there is no initial ramdisk/ramfs.
```

그래서 우리는 `boot_params` 부터 관심있는 정보를 모두 얻을 수 잇다. 예를 들어 `get_ramdisk_image` 함수를 살펴보자:

```C
static u64 __init get_ramdisk_image(void)
{
        u64 ramdisk_image = boot_params.hdr.ramdisk_image;

        ramdisk_image |= (u64)boot_params.ext_ramdisk_image << 32;

        return ramdisk_image;
}
```

`boot_params` 로 부터 램디스크 관련 주소를 `32` 왼쪽 쉬프트 하여 얻는다. [Documentation/x86/zero-page.txt](https://github.com/0xAX/linux/blob/master/Documentation/x86/zero-page.txt) 내용을 보면 왜 이렇게 하는지 알 수 있다.:

```
0C0/004	ALL	ext_ramdisk_image ramdisk_image high 32bits
```

32 비트 쉬프트까지 하고, 64 비트 `ramdisk_image` 주소를 얻어 반환한다. `get_ramdisk_size` 함수는 `get_ramdisk_image` 와 같은 원리로 동작하지만, `ext_ramdisk_image` 대신에 `ext_ramdisk_size`를 사용한다. 램디스크의 크기, 시작 주소 그리고 마지막 주소를 얻고나서, 부트로더에서 제공된 램디스크 정보를 확인한다.:

```C
if (!boot_params.hdr.type_of_loader ||
    !ramdisk_image || !ramdisk_size)
	return;
```

마지막으로 초기 램디스크를 위해 계산된 주소들과 함께 메모리 블럭을 예약한다:

```C
memblock_reserve(ramdisk_image, ramdisk_end - ramdisk_image);
```

결론
---------------------------------------------------------------------------------

리눅스 커널 초기화 과정의 4번째 파트가 마무리되었다. 우리는 이 파트에서 `start_kernel` 함수에서 부터 커널 일반 코드를 살펴보기 시작했고 아키텍처 특화의 초기화하는 `setup_arch` 에서 끝난다. 다음 파트에서 아키텍처 의존적인 초기화 과정이 계속 진행 될 것이다.

어떤 질문이나 제안이 있다면, twitter [0xAX](https://twitter.com/0xAX), [email](anotherworldofworld@gmail.com) 또는 [issue](https://github.com/0xAX/linux-insides/issues/new) 를 만들어 주길 바란다.

**나는 영어권의 사람이 아니고 이런 것에 대해 매우 미안해 하고 있다. 만약 어떤 실수를 발견한다면, 나에게 PR을 [linux-insides](https://github.com/0xAX/linux-internals)을 보내줘**

링크
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
* [SMP 초기화](http://egloos.zum.com/studyfoss/v/5444259)
