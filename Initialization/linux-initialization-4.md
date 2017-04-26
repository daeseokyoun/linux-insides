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

and occupies 52 bytes. The `thread_info` structure contains architecture-specific information on the thread. We know that on `x86_64` the stack grows down and `thread_union.thread_info` is stored at the bottom of the stack in our case. So the process stack is 16 kilobytes and `thread_info` is at the bottom. The remaining thread_size will be `16 kilobytes - 62 bytes = 16332 bytes`. Note that `thread_union` represented as the [union](http://en.wikipedia.org/wiki/Union_type) and not structure, it means that `thread_info` and stack share the memory space.
이 구조체는 52 바이트를 차지 한다. `thread_info` 구조체는 쓰레드의 아키텍처 의존적인 정보를 포함한다. 우리는 ``

Schematically it can be represented as follows:

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

So the `INIT_TASK` macro fills these `task_struct's` fields and many many more. As I already wrote above, I will not describe all the fields and values in the `INIT_TASK` macro but we will see them soon.

Now let's go back to the `set_task_stack_end_magic` function. This function defined in the [kernel/fork.c](https://github.com/torvalds/linux/blob/master/kernel/fork.c#L297) and sets a [canary](http://en.wikipedia.org/wiki/Stack_buffer_overflow) to the `init` process stack to prevent stack overflow.

```C
void set_task_stack_end_magic(struct task_struct *tsk)
{
	unsigned long *stackend;
	stackend = end_of_stack(tsk);
	*stackend = STACK_END_MAGIC; /* for overflow detection */
}
```

Its implementation is simple. `set_task_stack_end_magic` gets the end of the stack for the given `task_struct` with the `end_of_stack` function. Earlier (and now for all architectures besides `x86_64`) stack was located in the `thread_info` structure. So the end of a process stack depends on the `CONFIG_STACK_GROWSUP` configuration option. As we learn in `x86_64` architecture, the stack grows down. So the end of the process stack will be:

```C
(unsigned long *)(task_thread_info(p) + 1);
```

where `task_thread_info` just returns the stack which we filled with the `INIT_TASK` macro:

```C
#define task_thread_info(task)  ((struct thread_info *)(task)->stack)
```

From the Linux kernel `v4.9-rc1` release, `thread_info` structure may contains only flags and stack pointer resides in `task_struct` structure which represents a thread in the Linux kernel. This depends on `CONFIG_THREAD_INFO_IN_TASK` kernel configuration option which is enabled by default for `x86_64`. You can be sure in this if you will look in the [init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c) configuration build file:

```
config THREAD_INFO_IN_TASK
	bool
	help
	  Select this to move thread_info off the stack into task_struct.  To
	  make this work, an arch will need to remove all thread_info fields
	  except flags and fix any runtime bugs.

	  One subtle change that will be needed is to use try_get_task_stack()
	  and put_task_stack() in save_thread_stack_tsk() and get_wchan().
```

and [arch/x86/Kconfig](https://github.com/torvalds/linux/blob/master/arch/x86/Kconfig):

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

So, in this way we may just get end of a thread stack from the given `task_struct` structure:

```C
#ifdef CONFIG_THREAD_INFO_IN_TASK
static inline unsigned long *end_of_stack(const struct task_struct *task)
{
	return task->stack;
}
#endif
```

As we got the end of the init process stack, we write `STACK_END_MAGIC` there. After `canary` is set, we can check it like this:

```C
if (*end_of_stack(task) != STACK_END_MAGIC) {
        //
        // handle stack overflow here
	//
}
```

The next function after the `set_task_stack_end_magic` is `smp_setup_processor_id`. This function has an empty body for `x86_64`:

```C
void __init __weak smp_setup_processor_id(void)
{
}
```

as it not implemented for all architectures, but some such as [s390](http://en.wikipedia.org/wiki/IBM_ESA/390) and [arm64](http://en.wikipedia.org/wiki/ARM_architecture#64.2F32-bit_architecture).

The next function in `start_kernel` is `debug_objects_early_init`. Implementation of this function is almost the same as `lockdep_init`, but fills hashes for object debugging. As I wrote above, we will not see the explanation of this and other functions which are for debugging purposes in this chapter.

After the `debug_object_early_init` function we can see the call of the `boot_init_stack_canary` function which fills `task_struct->canary` with the canary value for the `-fstack-protector` gcc feature. This function depends on the `CONFIG_CC_STACKPROTECTOR` configuration option and if this option is disabled, `boot_init_stack_canary` does nothing, otherwise it generates random numbers based on random pool and the [TSC](http://en.wikipedia.org/wiki/Time_Stamp_Counter):

```C
get_random_bytes(&canary, sizeof(canary));
tsc = __native_read_tsc();
canary += tsc + (tsc << 32UL);
```

After we got a random number, we fill the `stack_canary` field of `task_struct` with it:

```C
current->stack_canary = canary;
```

and write this value to the top of the IRQ stack with the:

```C
this_cpu_write(irq_stack_union.stack_canary, canary); // read below about this_cpu_write
```

Again, we will not dive into details here, we will cover it in the part about [IRQs](http://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29). As canary is set, we disable local and early boot IRQs and register the bootstrap CPU in the CPU maps. We disable local IRQs (interrupts for current CPU) with the `local_irq_disable` macro which expands to the call of the `arch_local_irq_disable` function from [include/linux/percpu-defs.h](https://github.com/torvalds/linux/blob/master/include/linux/percpu-defs.h):

```C
static inline notrace void arch_local_irq_enable(void)
{
        native_irq_enable();
}
```

Where `native_irq_enable` is `cli` instruction for `x86_64`. As interrupts are disabled we can register the current CPU with the given ID in the CPU bitmap.

The first processor activation
---------------------------------------------------------------------------------

The current function from the `start_kernel` is `boot_cpu_init`. This function initializes various CPU masks for the bootstrap processor. First of all it gets the bootstrap processor id with a call to:

```C
int cpu = smp_processor_id();
```

For now it is just zero. If the `CONFIG_DEBUG_PREEMPT` configuration option is disabled, `smp_processor_id` just expands to the call of `raw_smp_processor_id` which expands to the:

```C
#define raw_smp_processor_id() (this_cpu_read(cpu_number))
```

`this_cpu_read` as many other function like this (`this_cpu_write`, `this_cpu_add` and etc...) defined in the [include/linux/percpu-defs.h](https://github.com/torvalds/linux/blob/master/include/linux/percpu-defs.h) and presents `this_cpu` operation. These operations provide a way of optimizing access to the [per-cpu](http://0xax.gitbooks.io/linux-insides/content/Theory/per-cpu.html) variables which are associated with the current processor. In our case it is `this_cpu_read`:

```
__pcpu_size_call_return(this_cpu_read_, pcp)
```

Remember that we have passed `cpu_number` as `pcp` to the `this_cpu_read` from the `raw_smp_processor_id`. Now let's look at the `__pcpu_size_call_return` implementation:

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

Yes, it looks a little strange but it's easy. First of all we can see the definition of the `pscr_ret__` variable with the `int` type. Why int? Ok, `variable` is `common_cpu` and it was declared as per-cpu int variable:

```C
DECLARE_PER_CPU_READ_MOSTLY(int, cpu_number);
```

In the next step we call `__verify_pcpu_ptr` with the address of `cpu_number`. `__veryf_pcpu_ptr` used to verify that the given parameter is a per-cpu pointer. After that we set `pscr_ret__` value which depends on the size of the variable. Our `common_cpu` variable is `int`, so it 4 bytes in size. It means that we will get `this_cpu_read_4(common_cpu)` in `pscr_ret__`. In the end of the `__pcpu_size_call_return` we just call it. `this_cpu_read_4` is a macro:

```C
#define this_cpu_read_4(pcp)       percpu_from_op("mov", pcp)
```

which calls `percpu_from_op` and pass `mov` instruction and per-cpu variable there. `percpu_from_op` will expand to the inline assembly call:

```C
asm("movl %%gs:%1,%0" : "=r" (pfo_ret__) : "m" (common_cpu))
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
