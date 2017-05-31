커널 초기화. Part 8.
================================================================================

스케줄러 초기화
================================================================================

리눅스 커널 초기화 과정의 8 번째 [파트](https://github.com/daeseokyoun/linux-insides/blob/master/Initialization/README.md)를 시작한다. 이전 [파트](https://github.com/daeseokyoun/linux-insides/blob/master/Initialization/linux-initialization-7.md) 에서 `setup_nr_cpu_ids` 함수까지 하고 마무리 지었었다. 현재 파트에서 주요 쟁점은 [scheduler](http://en.wikipedia.org/wiki/Scheduling_%28computing%29)의 초기화이다. 하지만 스케줄러 초기화 과정을 배우기 전에, 몇가지 알고 넘어가야 하는 것들이 있다. [init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c) 에서 다음 단계는 `setup_per_cpu_areas` 함수이다. 이 함수는 `percpu` 변수를 위한 공간을 설정한다. 이것을 조금 더 자세히 알고 싶다면 [Per-CPU](https://github.com/daeseokyoun/linux-insides/blob/master/Concepts/per-cpu.md) 관련 챕터를 참조하자. `percpu` 공간이 설정된다면, 다음 단계는 `smp_prepare_boot_cpu` 함수 호출이다. 이 함수는 [SMP](http://en.wikipedia.org/wiki/Symmetric_multiprocessing) 를 위한 몇몇의 준비 작업을 한다.:

```C
static inline void smp_prepare_boot_cpu(void)
{
         smp_ops.smp_prepare_boot_cpu();
}
```

이 함수는 `smp_prepare_boot_cpu` 를 호출 하고, 이 함수는 다시 `native_smp_prepare_boot_cpu` 를 호출한다. (`smp_ops` 에 관련된 사항을 보려면 `SMP` 챕터를 확인하자.):

```C
void __init native_smp_prepare_boot_cpu(void)
{
        int me = smp_processor_id();
        switch_to_new_gdt(me);
        cpumask_set_cpu(me, cpu_callout_mask);
        per_cpu(cpu_state, me) = CPU_ONLINE;
}
```

`native_smp_prepare_boot_cpu` 함수는 먼저, `smp_processor_id` 함수로 현재 CPU id 를 얻는다.(현재 부트스트랩 프로세서의 `id`는 0 이다.) 여기서는 `smp_processor_id` 함수가 어떻게 동작하는지 설명하지 않을 것이다. 이유는 이미 [커널 엔트리 포인트](https://github.com/daeseokyoun/linux-insides/blob/master/Initialization/linux-initialization-4.md) 에서 살펴 보았기 때문이다. 프로세서 `id` 번호를 얻었다면, `switch_to_new_gdt` 함수로 주어진 CPU 를 위한 [Global Descriptor Table-GDT](http://en.wikipedia.org/wiki/Global_Descriptor_Table) 를 재 로드한다.:

```C
void switch_to_new_gdt(int cpu)
{
        struct desc_ptr gdt_descr;

        gdt_descr.address = (long)get_cpu_gdt_table(cpu);
        gdt_descr.size = GDT_SIZE - 1;
        load_gdt(&gdt_descr);
        load_percpu_segment(cpu);
}
```

여기서 `gdt_descr` 변수는 `GDT` 디스크립터의 포인터를 나타낸다.(우리는 이미 [초기 인터럽트와 예외 처리](https://github.com/daeseokyoun/linux-insides/blob/master/Initialization/linux-initialization-2.md) 에서 살펴 보았다.) 우리는 `GDT` 디스크립터의 크기와 주소를 얻는다. 여기서 `GDT_SIZE` 는 `256` 이다.:

```C
#define GDT_SIZE (GDT_ENTRIES * 8)
```

그리고 `get_cpu_gdt_table` 함수로 부터 디스크립터의 주소를 얻을 것이다.:

```C
static inline struct desc_struct *get_cpu_gdt_table(unsigned int cpu)
{
        return per_cpu(gdt_page, cpu).gdt;
}
```

`get_cpu_gdt_table` 함수는 주어진 CPU 번호를 위한 `gdt_page` 인 percpu 변수를 얻기 위해 `per_cpu` 매크로를 사용한다.(우리의 경우에는, 부트스트랩 프로세서의 `id`는 0 이다.) 그렇다면 이런 질문이 생길 수 있다: 그래서 만약 `gdt_page` percpu 변수를 접근 할 수 있다면, 어디서 이 선언을 찾을 수 있을까? 실제 우리는 이미 그것을 이 책을 통해서 살펴 봤다. 만약 첫 [파트](https://github.com/daeseokyoun/linux-insides/blob/master/Initialization/linux-initialization-1.md)를 봤다면, 당신은 [arch/x86/kernel/head_64.S](https://github.com/0xAX/linux/blob/master/arch/x86/kernel/head_64.S) 에서 `gdt_page` 의 선언을 봤었을 것이다.:

```assembly
early_gdt_descr:
	.word	GDT_ENTRIES*8-1
early_gdt_descr_base:
	.quad	INIT_PER_CPU_VAR(gdt_page)
```

그리고 만약 우리가 [linker](https://github.com/0xAX/linux/blob/master/arch/x86/kernel/vmlinux.lds.S) 파일을 봤다면, `__per_cpu_load` 심볼 바로 뒤에 위치했다는 것도 알수 있다.:

```C
#define INIT_PER_CPU(x) init_per_cpu__##x = x + __per_cpu_load
INIT_PER_CPU(gdt_page);
```

그리고 [arch/x86/kernel/cpu/common.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/cpu/common.c#L94)에서 `gdt_page`를 채웠다.:

```C
DEFINE_PER_CPU_PAGE_ALIGNED(struct gdt_page, gdt_page) = { .gdt = {
#ifdef CONFIG_X86_64
	[GDT_ENTRY_KERNEL32_CS]		= GDT_ENTRY_INIT(0xc09b, 0, 0xfffff),
	[GDT_ENTRY_KERNEL_CS]		= GDT_ENTRY_INIT(0xa09b, 0, 0xfffff),
	[GDT_ENTRY_KERNEL_DS]		= GDT_ENTRY_INIT(0xc093, 0, 0xfffff),
	[GDT_ENTRY_DEFAULT_USER32_CS]	= GDT_ENTRY_INIT(0xc0fb, 0, 0xfffff),
	[GDT_ENTRY_DEFAULT_USER_DS]	= GDT_ENTRY_INIT(0xc0f3, 0, 0xfffff),
	[GDT_ENTRY_DEFAULT_USER_CS]	= GDT_ENTRY_INIT(0xa0fb, 0, 0xfffff),
    ...
    ...
    ...
```

`percpu` 변수에 대해 더 자세히 알고 싶다면, [Per-CPU variables](https://github.com/daeseokyoun/linux-insides/blob/master/Concepts/per-cpu.md) 파트를 살펴 보자. `GDT` 디스크립터의 크기와 주소를 얻고 나면, 단지 `lgdt` 명령어를 실행하는 `load_gdt` 를 통해 `GDT` 를 재 로드하고 다음 함수를 통해 `percpu_segment` 를 로드 한다.:

```C
void load_percpu_segment(int cpu) {
    loadsegment(gs, 0);
    wrmsrl(MSR_GS_BASE, (unsigned long)per_cpu(irq_stack_union.gs_base, cpu));
    load_stack_canary_segment();
}
```

`percpu` 영역의 시작 주소는 `gs` 레지스터를 반드시 포함해야 해야 하기 때문에, `gs`를 `loadsegment` 매크로에게 넘겨주어 실행한다. 다음 단계에서는 [IRQ](http://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29) 스택이 있다면 그 시작 주소를 써주고 스택 [canary](http://en.wikipedia.org/wiki/Buffer_overflow_protection)도 설정(이것은 `x86_32` 를 위한 것이다.)한다. 새로운 `GDT`를 로드하고 나서, 현재 CPU 의 `cpu_callout_mask` 비트맵을 채우고, 현재 프로세서를 위해 `cpu_state` percpu 변수에서 cpu 상태를 online - `CPU_ONLINE` - 으로 설정한다.:

```C
cpumask_set_cpu(me, cpu_callout_mask);
per_cpu(cpu_state, me) = CPU_ONLINE;
```

그러면, `cpu_callout_mask` 비트맵은 무엇이냐... 우리는 부트스트랩 프로세서를 초기화 했기에 멀리 프로세서 시스템에서 다른 프로세서들은 `secondary processors` 로 알려져있다. 리눅스 커널은 아래 두 비트맵 마스크를 사용한다.:

* `cpu_callout_mask`
* `cpu_callin_mask`

부트스트랩 프로세서가 초기화되면, 그것은 보조 프로세서들을 초기화 할 수 있도록 표시하기 위해 `cpu_callout_mask` 를 업데이트 한다. 모든 다른 보조 프로세서(secondary processors)들은 초기화 되기 전에 몇가지 일들을 해야 하고 `cpu_callout_mask` 에 부트스트랩 프로세서 비트가 설정되어 있는지 확인한다. 단지 보조 프로세서와 함께 부트스트랩 프로세서의 `cpu_callout_mask` 를 채운다면, 그것의 초기화의 나머지를 진행할 수 있다. 특정 프로세서의 초기화를 마무리 한뒤에는, 이 프로세서는 남은 보조 프로세서들 중 하나를 초기화를 위해 같은 과정으로 반복한다. 이와 관련된 세부 사항은 `SMP` 관련된 챕터를 참고 바란다.

끝이다. 이제 `SMP` 부트 준비를 마쳤다.

zonelists 만들기
-----------------------------------------------------------------------

다음 단계에서는 `build_all_zonelists` 함수의 호출을 볼 수 있다. 이 함수는 할당이 어떤 zone 으로 부터 받아야 하는지 zone 의 우선순위를 설정한다. 먼저 zone 들과 우선순위에 대해 먼저 이해를 할 필요가 있다. 리눅스 커널에서 물리적 메모리를 어떻게 처리하는지 알아보자. 물리적 메모리는 `nodes` 라는 이름으로 뱅크들로 분리되어 있다. 만약 당신이 `NUMA` 를 지원하는 하드웨어가 없다면, 단지 하나의 node 만 있다고 생각하고 보자.:

```
$ cat /sys/devices/system/node/node0/numastat
numa_hit 72452442
numa_miss 0
numa_foreign 0
interleave_hit 12925
local_node 72452442
other_node 0
```

모든 `node` 는 리눅스 커널에서 `struct pglist_data` 로 표현이 된다. 각 node 는 `zones` 이라는 특별한 몇 개의 블럭으로 나뉘어져 있다. 모든 zone 은 리눅스 커널에서 `zone struct` 으로 표현이 되고 아래 중 하나의 타입을 가진다.:

* `ZONE_DMA` - 0-16M;
* `ZONE_DMA32` - 4G 아래에 있는 DMA 영역 만을 접근하여 사용하는 32 비트 장치를 위해 사용
* `ZONE_NORMAL` - `x86_64` 에서 4 GB 에서 부터 모든 메모리
* `ZONE_HIGHMEM` - `x86_64` 에서는 필요 없음
* `ZONE_MOVABLE` - 이동 가능한 페이지들을 포함하는 zone

이것들은 `zone_type` enum 타입으로 표현된다. 아래와 같이 zone에 대한 정보를 얻을 수 있다.:

```
$ cat /proc/zoneinfo
Node 0, zone      DMA
  pages free     3975
        min      3
        low      3
        ...
        ...
Node 0, zone    DMA32
  pages free     694163
        min      875
        low      1093
        ...
        ...
Node 0, zone   Normal
  pages free     2529995
        min      3146
        low      3932
        ...
        ...
```

위에서 언급했듯이 모든 노드들은 메모리내에 `pglist_data` 나 `pg_data_t` 구조체로 기술된다. 이 구조체는 [include/linux/mmzone.h](https://github.com/torvalds/linux/blob/master/include/linux/mmzone.h) 에 선언되어 있다. [mm/page_alloc.c](https://github.com/torvalds/linux/blob/master/mm/page_alloc.c)에 있는 `build_all_zonelists` 함수는 선택된 `zone`이나 `node`에서 할당 요청을 처리하지 못했을 때 가능한 zone 이나 node 들을 지정하도록 하는 (순서가 있는) `zonelist` 를 구성한다. `NUMA`나 멀티 프로세서 시스템에 대해 더 기술 하기 위해 특별한 파트를 구성할 것이다.

스케줄러 초기화 전에 남은 일들
--------------------------------------------------------------------------------

리눅스 커널 스케줄러 초기화 과정을 살펴보기 전에 몇가지 일들을 먼저 진행해야 한다. 첫 번째로는 [mm/page_alloc.c](https://github.com/torvalds/linux/blob/master/mm/page_alloc.c) 에 구현된 `page_alloc_init` 함수이다. 이 함수는 꽤 쉬워 보인다.:

```C
void __init page_alloc_init(void)
{
        hotcpu_notifier(page_alloc_cpu_notify, 0);
}
```

이 함수는 `CPU` [hotplug](https://www.kernel.org/doc/Documentation/cpu-hotplug.txt)를 위한 핸들러를 초기화 한다. 물론 `hotcpu_notifier` 함수는 `CONFIG_HOTPLUG_CPU` 구성 옵션에 의존적이고, 만약 이 옵션이 설정되어 있다면, hotplug cpu 핸들러 (우리의 경우에는 `page_alloc_cpu_notify` 이다.)를 초기하는 `register_cpu_notifier` 함수를 호출하는 `cpu_notifier` 함수를 호출한다.

이 호출 다음에 우리는 초기화 화면 출력에서 커널 명령 라인을 아래와 같이 확인 할 수 있다.:

![kernel command line](http://oi58.tinypic.com/2m7vz10.jpg)

그리고 리눅스 커널 명령 라인을 처리하는 `parse_early_param` 와 `parse_args` 같은 함수들을 사용될 수 있다. `parse_early_param` 함수는 6 번째 [파트](https://github.com/daeseokyoun/linux-insides/blob/master/Initialization/linux-initialization-6.md) 에서 이미 살펴 보았는데, 왜 다시 이 함수가 호출되는 것인가? 답은 간단하다.: 우리는 이 함수는 아키텍처 특화된 코드내에서 호출했지만(우리의 경우, `x86_64`) 모든 함수가 이 함수를 호출하는 것은 아니다. 그리고 우리는 파싱하기 위해 두 번째 함수인 `parse_args` 를 호출할 필요가 있고 초기 명령 라인이 아닌 인자들을 처리할 수 있다.

다음 단계에서 [kernel/jump_label.c](https://github.com/torvalds/linux/blob/master/kernel/jump_label.c) 소스 코드에 있는 `jump_label_init` 함수의 호출을 볼 수 있다. 그리고 [jump label](https://lwn.net/Articles/412072/)을 초기화 한다.

이 다음에는 [printk](http://www.makelinux.net/books/lkd2/ch18lev1sec3) 로그 버퍼을 설정하는 `setup_log_buf` 함수를 볼 수 있다. 우리는 이미 이 함수를 리눅스 초기화 과정 챕터 7 번째 [파트](https://github.com/daeseokyoun/linux-insides/blob/master/Initialization/linux-initialization-6.md)에서 보았다.

PID 해쉬 초기화
--------------------------------------------------------------------------------

다음 살펴 볼 함수는 `pidhash_init` 함수이다. 아시다시피 각 프로세스는 `process identification number` 나 `PID` 라 불리는 고유한 번호를 할당 받는다. 각 fork 나 clone 으로 생성된 프로세스는 커널로 부터 자동적으로 새로운 고유의 `PID` 를 할당 받는다. `PID` 들의 관리는 두 개의 특별한 자료 구조들인 `struct pid` 와 `struct upid`로 이루진다. 첫 번째 구조체는 커널에서 `PID` 에 관련된 정보를 나타낸다. 두 번째 구조체는 특정 네임스페이스 내에서 보여지는 정보들을 나타낸다. 모든 `PID` 인스턴스는 특별한 해쉬 테이블에 저장된다.:

```C
static struct hlist_head *pid_hash;
```

이 해쉬 테이블은 숫자의 `PID` 값에 속하는 Pid 인스턴스를 찾는데 사용된다. 그래서, `pidhash_init`는 이 해쉬 테이블을 초기화 한다. `pidhash_init` 함수의 시작에서 `alloc_large_system_hash` 함수 호출을 볼 수 있다.:

```C
pid_hash = alloc_large_system_hash("PID", sizeof(*pid_hash), 0, 18,
                                   HASH_EARLY | HASH_SMALL,
                                   &pidhash_shift, NULL,
                                   0, 4096);
```

`pid_hash` 의 요소의 수는 `RAM` 구성에 의존적이지만, 그것은 `2^4` 개에서 `2^12` 사이의 값이다. `pidhash_init` 함수는 그 크기를 계산하여 요구되는 저장소(우리의 경우에는 `hlist` 이다. - [doubly linked list](http://0xax.gitbooks.io/linux-insides/content/DataStructures/dlist.html) 와 같지만, [struct hlist_head](https://github.com/torvalds/linux/blob/master/include/linux/types.h) 대신에 하나의 포인터를 가진다.)를 할당한다. `alloc_large_system_hash` 함수는 만약 `HASH_EARLY` 플래그를 (우리의 경우와 같음) 넘겼다면, `memblock_virt_alloc_nopanic` 함수를 사용하고 넘기지 않았다면 `__vmalloc`를 이용해 큰 시스템 해쉬 테이블을 할당한다.

우리는 `dmesg` 출력에서 아래와 같은 결과를 볼 수있다.:

```
$ dmesg | grep hash
[    0.000000] PID hash table entries: 4096 (order: 3, 32768 bytes)
...
...
...
```

이게 전부다. 스케줄러 초기화 전에 몇가지 더 남은 것들은 다음과 같은 함수들이다.: `vfs_caches_init_early` 함수는 [virtual file system](http://en.wikipedia.org/wiki/Virtual_file_system) 의 초기(early) 초기화를 한다.(이와 관련된 자세한 내용은 가상 파일 시스템 챕터에서 기술 할 것이다.), `sort_main_extable` 함수는 커널 빌트인된 `__start___ex_table` 와 `__stop___ex_table` 사이에 있는 예외 테이블 엔트리들을 정렬한다., 그리고 `trap_init` 함수는 트랩 핸들러를 초기화 한다.(마지막 두 개의 함수들은 인터럽트와 관련된 몇몇 챕터에서 다루어 볼 것이다.)

스케줄러 초기호 전에 마지막 단계는 [init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c) 소스 코드에 있는 `mm_init` 함수를 통해 메모리 관리자의 초기화이다. `mm_init` 함수는 리눅스 커널 메모리 관리를 위한 초기화를 진행한다.:

```C
page_ext_init_flatmem();
mem_init();
kmem_cache_init();
percpu_init_late();
pgtable_init();
vmalloc_init();
```

첫 함수는 `CONFIG_SPARSEMEM` 커널 구성 옵션에 의존적인 `page_ext_init_flatmem` 이고, 이 함수는 매 페이지 핸들링마다 확장된 데이터를 초기화 한다. `mem_init` 함수는 `bootmem` 에서 사용된 모든 메모리를 해제한다. `kmem_cache_init` 함수는 커널 캐쉬를 초기화하고, `percpu_init_late` 함수는 [slub](http://en.wikipedia.org/wiki/SLUB_%28software%29)이 사용되기 전에 할당된 것들을 교체하는 작업을 한다. `pgtable_init` 는 `page->ptl` 커널 캐쉬를 초기화한다. `vmalloc_init` 함수는 `vmalloc` 를 초기화한다. **NOTE** 우리는 방금 언급한 함수들을 더 자세히 살펴 보진 않을 것이다. 하지만 [리눅스 커널 메모리 관리](https://github.com/daeseokyoun/linux-insides/blob/master/mm/README.md) 챕터에서 진행할 것이다.

이제 `scheduler` 초기화를 살펴 보자.

스케줄러 초기화
--------------------------------------------------------------------------------

이제서야 이 파트의 주 목적 - 태스크 스케줄러의 초기화 - 으로 진입했다. 여기서는 스케줄러의 모든 내용을 다루진 않을 것이고, 스케줄러를 위한 것은 따로 특별한 챕터를 만들 것이다. 좋다. 다음 호출되는 함수는 [kernel/sched/core.c](https://github.com/torvalds/linux/blob/master/kernel/sched/core.c) 소스 파일에 구현된 `sched_init` 함수이고, 함수이름에서도 알 수 있듯이 스케줄러를 초기화한다. 어떻게 스케줄러가 초기화 되는지 이해 해보도록 하자. `sched_init` 함수는 아래의 코드에서 부터 시작한다.:

```C
#ifdef CONFIG_FAIR_GROUP_SCHED
         alloc_size += 2 * nr_cpu_ids * sizeof(void **);
#endif
#ifdef CONFIG_RT_GROUP_SCHED
         alloc_size += 2 * nr_cpu_ids * sizeof(void **);
#endif
```

처음에 아래와 같은 구성 옵션들을 볼 수 있다.:

* `CONFIG_FAIR_GROUP_SCHED`
* `CONFIG_RT_GROUP_SCHED`

이 옵션 모두는 두 가지의 다른 스케줄러 모델을 제공한다. [문서](https://www.kernel.org/doc/Documentation/scheduler/sched-design-CFS.txt) 에서 읽어 볼 수 있듯이, 현재 스케줄러 인 `CFS`(또는 `Completely Fair Scheduler`)는 간단한 개념을 사용한다. 그것은 각 프로세스가 `1/n` 만큼의 프로세서 시간을 가지는 이상적인 멀티 태스킹 프로세서 인 것처럼 처리하도록 하는 모델이다. 여기서 `n` 은 현재 수행 가능한 프로세스들의 개수이다. 스케줄러는 특별한 규칙들을 사용한다. 이 규칙들은 `scheduling policy(스케줄링 정책)` 이라 불리는 새로운 프로세스가 선택되었을 때 얼마나 실행되고 언제 실행 될지를 결정한다. CFS (Completely Faire Scheduler) 는 `normal` 혹은 `non-real-time` 스케줄링 정책들을 지원한다.: `SCHED_NORMAL`, `SCHED_BATCH` 그리고 `SCHED_IDLE`. `SCHED_NORMAL` 는 가장 일반적인 응용 프로그램을 지원하고, [nice](http://en.wikipedia.org/wiki/Nice_%28Unix%29) 값에 의해 대부분 각 프로세스가 cpu 자원을 소비하는 양(?)을 결정한다. `SCHED_BATCH` 는 100% non-interactive(인터엑티브 하지 않는) 태스크를 위해 사용된다. 그리고 `SCHED_IDLE`는 프로세서에서 더이상 실행할 태스크가 없는 경우에 수행되는 태스크를 수행한다. `real-time` 정책은 time-critical 한 응용 프로그램을 지원하기 위해 지원된다.: `SCHED_FIFO` 와 `SCHED_RR`. 리눅스 커널 스케줄러에 관련해서 읽다보면, 스케줄러는 모듈이라는 것을 알수 있을 것이다. 이것의 의미는 서로 다른 프로세스를 스케줄하기 위해 다른 알고리즘을 지원한다는 것이다. 대게는 이 모듈의 형태를 `스케줄러 클래스` 라 부른다. 이 모듈들은 스케줄러 정책의 상세를 캡슐화하고 그것들에 대해 너무 많이 알지 못해도 스케줄러 코어에 의해 처리될 수 있다는 것이다.

이제 코드로 돌아와서 `CONFIG_FAIR_GROUP_SCHED` 와 `CONFIG_RT_GROUP_SCHED` 구성 옵션들을 살펴 보자. 스케줄러는 분리된 태스크에서 수행된다. 이 옵션들은 그룹 태스크를 스케줄하는 것을 허용한다. ([CFS group scheduling](http://lwn.net/Articles/240474/) 를 읽어보자.) 우리는 프로세서의 개수로 계산된 값을 가지는 `alloc_size` 로 `sched_entity` 구조체를 위해 메모리를 할당하고 `cfs_rq` 는 `kzalloc` 으로 `2 * nr_cpu_ids * sizeof(void **)` 의 크기 만큼 할당된 메모리의 주소를 넣는다.:

```C
ptr = (unsigned long)kzalloc(alloc_size, GFP_NOWAIT);

#ifdef CONFIG_FAIR_GROUP_SCHED
        root_task_group.se = (struct sched_entity **)ptr;
        ptr += nr_cpu_ids * sizeof(void **);

        root_task_group.cfs_rq = (struct cfs_rq **)ptr;
        ptr += nr_cpu_ids * sizeof(void **);
#endif

```

`sched_entity` 는 [include/linux/sched.h](https://github.com/torvalds/linux/blob/master/include/linux/sched.h) 선언된 구조체이고 프로세스 스케줄링 관리를 위해 사용된다. `cfg_rq` 는 [run queue-수행 큐](http://en.wikipedia.org/wiki/Run_queue) 를 표현한다. 우리는 이 수행 큐(run queue)와 `root_task_group` 의 스케줄러 엔터티를 위해 `alloc_size` 크기 만큼 공간을 할당했다. `root_task_group` 는 태스크 그룹에 관련된 정보를 갖고 있는 [kernel/sched/sched.h](https://github.com/torvalds/linux/blob/master/kernel/sched/sched.h)에 선어된 `task_group` 의 인스턴스이다.:

```C
struct task_group {
    ...
    ...
    struct sched_entity **se;
    struct cfs_rq **cfs_rq;
    ...
    ...
}
```

루트 태스크 그룹(root task group)은 시스템에 있는 모드 태스크를 갖고 있는 태스크 그룹이다. 루트 태스크 그룹 스케줄러 엔터티와 수행큐를 위해 공간할당하였다면, 모든 가능한 CPU 들(`cpu_possible_mask` 비트맵)을 갖고 루프를 돌면서 `load_balance_mask` 와 `percpu` 변수를 위해 `kzalloc_node` 함수를 통해 특정한 메모리 노드로 부터 0으로 초기화된 메모리를 할당한다.:

```C
DECLARE_PER_CPU(cpumask_var_t, load_balance_mask);
```

여기서 `cpumask_var_t` 는 한가지 다른 점이 있는 `cpumask_t` 이다.: `cpumask_var_t` 는 `cpumask_t` 가 항상 `NR_CPUS` 비트를 갖고 있을 때, `nr_cpu_ids` 비트만 설정된다. (`cpumask` 에 관해서는 [CPU masks](https://github.com/daeseokyoun/linux-insides/blob/master/Concepts/cpumask.md)를 보도록 하자.) 아래의 코드를 보자.:

```C
#ifdef CONFIG_CPUMASK_OFFSTACK
    for_each_possible_cpu(i) {
        per_cpu(load_balance_mask, i) = (cpumask_var_t)kzalloc_node(
                cpumask_size(), GFP_KERNEL, cpu_to_node(i));
    }
#endif
```

이 코드는 `CONFIG_CPUMASK_OFFSTACK` 구성 옵션에 의존적이다. 이 구성 옵션은 `cpumask` 을 위해 스택에 그것을 넣지 말고 동적 메모리 할당을 사용하라는 의미이다. 모든 그룹들은 CPU 시간의 양에 의존적이어야 한다. 아래 두 함수의 호출을 보자.:

```C
init_rt_bandwidth(&def_rt_bandwidth,
                  global_rt_period(), global_rt_runtime());
init_dl_bandwidth(&def_dl_bandwidth,
                  global_rt_period(), global_rt_runtime());
```

우리는 `SCHED_DEADLINE` 으로 real-time 태스크를 위한 대역폭(bandwidth) 관리를 초기화한다. 이 함수들은 시스템의 최대 `deadline` 대역폭에 관련된 정보를 저장하는 `rt_bandwidth` 와 `dl_bandwidth`를 초기화한다. 예를 들어 `init_rt_bandwidth` 함수의 구현을 살펴보자.:

```C
void init_rt_bandwidth(struct rt_bandwidth *rt_b, u64 period, u64 runtime)
{
        rt_b->rt_period = ns_to_ktime(period);
        rt_b->rt_runtime = runtime;

        raw_spin_lock_init(&rt_b->rt_runtime_lock);

        hrtimer_init(&rt_b->rt_period_timer,
                     CLOCK_MONOTONIC, HRTIMER_MODE_REL);
        rt_b->rt_period_timer.function = sched_rt_period_timer;
}
```

이 함수는 3개의 인자를 받는다.:

* 한 기간 내에 할당되고 사용된 양의 정보를 포함하는 `rt_bandwidth` 구조체의 주소
* `period` - 실시간 태스크의 대역폭을 강제하는 `us` 단위의 기간
* `runtime` - 우리가 태스크 수행을 허기하는 기간의 일부(`us`)

`period` 와 `runtime` 에 따라 그 결과를 `global_rt_period` 와 `global_rt_runtime` 함수로 전달한다. 각 함수는 기본값으로 `1` 초와 `0.95` 초이다. `rt_bandwidth` 구조체는 [kernel/sched/sched.h](https://github.com/torvalds/linux/blob/master/kernel/sched/sched.h) 에 선언되어 있고 아래와 같이 되어 있다.:

```C
struct rt_bandwidth {
        raw_spinlock_t          rt_runtime_lock;
        ktime_t                 rt_period;
        u64                     rt_runtime;
        struct hrtimer          rt_period_timer;
};
```

보시다 시피, 그것은 `runtime` 와 `period` 를 포함할 뿐만 아니라 아래 두개의 항목을 더 갖고 있다.:

* `rt_runtime_lock` - `rt_time` 보호를 위한 [spinlock](http://en.wikipedia.org/wiki/Spinlock)
* `rt_period_timer` - 실시간 태스크를 위한 [high-resolution 커널 타이머](https://www.kernel.org/doc/Documentation/timers/hrtimers.txt)

그래서 `init_rt_bandwidth` 에서 `rt_bandwidth` 의 period 와 runtime을 주어진 인자들로 초기화하고, spinlock 과 high-resolution 타이머를 초기화 한다. 다음 단계에서는, [SMP](http://en.wikipedia.org/wiki/Symmetric_multiprocessing) 의 활성화에 의존적인, 루트 도메인을 초기화 한다.:

```C
#ifdef CONFIG_SMP
	init_defrootdomain();
#endif
```

실시간 스케줄러는 스케줄링 결정을 위해 글로벌 자원이 요구된다. 하지만 불행하게도 CPU 개수가 증가할 수록 확장성의 병목현상이 나타난다. 루트 도메인의 개념은 확장성을 향상시키기 위해 소개 되었다. 리눅스 커널은 CPU 의 셋을 할당하기 위해 특별한 매커니즘을 제공하고 `cpuset` 이라 부른다. 만약 `cpuset` 이 다른 `cpuset` CPU 들과 겹치지 않는다면, 그것은 `exclusive(배타적) cpuset` 이다. 각 배타적 cpuset 은 다른 cpuset 들이나 CPU 들로 부터 분리된 CPU 의 `root domain`이나 고립된 도메인으로 정의된다. 하나의 `root_domain` 은 리눅스 커널의 [kernel/sched/sched.h](https://github.com/torvalds/linux/blob/master/kernel/sched/sched.h)에 선언된 `struct root_domain` 에 의해 표현되고 그것이 필요한 주된 목적은
번역 변수의 범위를 도메인 범위로 좁히고, 루트 도메인 범위 내에서 모든 실시간 스케줄 결정을 하기 위함이다. 여기까지가 실시간 스케줄러에 관련된 내용이고, 향후 관련 챕터에서 더 자세히 살펴보자.

`root domain` 초기화 이후에, 루트 태스크 그룹의 실시간 태스크들을 위한 대역폭을 초기화한다.:

```C
#ifdef CONFIG_RT_GROUP_SCHED
	init_rt_bandwidth(&root_task_group.rt_bandwidth,
			global_rt_period(), global_rt_runtime());
#endif
```

다음 단계에서, `CONFIG_CGROUP_SCHED` 커널 구성 옵션에 의존적인 루트 태스크 그룹의 리스트인 `siblings` 와 `children` 를 초기화한다. 관련 문서에서 봤다면, `CONFIG_CGROUP_SCHED` 의 정의는 :

```
This option allows you to create arbitrary task groups using the "cgroup" pseudo
filesystem and control the cpu bandwidth allocated to each such task group.

이 옵션은 "cgroup" 의사(pseudo) 파일 시스템을 사용하여 임시의 태스크 그룹을 만드는 것을 허용하고
각 태스크 그룹에게 할당된 cpu 대역폭을 제어할 수 있다.
```

리스트들의 초기화가 마무리 되면, `autogroup_init` 함수의 호출을 볼 수 있다.:

```C
#ifdef CONFIG_CGROUP_SCHED
         list_add(&root_task_group.list, &task_groups);
         INIT_LIST_HEAD(&root_task_group.children);
         INIT_LIST_HEAD(&root_task_group.siblings);
         autogroup_init(&init_task);
#endif
```

이 함수는 자동적인 프로세스 그룹 스케줄링을 초기화한다.

이후에는 모든 `가능한` cpu 를 방문하면서(`possible` CPU들은 시스템에서 이용가능한 `cpu_possible_mask` 비트맵에 저장되어 있다.) 각 가능한 CPU 를 위한 `runqueue(수행큐)` 를 초기화한다.:

```C
for_each_possible_cpu(i) {
    struct rq *rq;
    ...
    ...
    ...
```

각 프로세서는 자신만의 락(locking)과 그 프로세서를 위한 runqueue를 갖고 있다. 모든 수행가능한 태스크는 active 배열에 저장되고 우선순위에 따라 인덱싱된다. 프로세스가 그 시간 슬라이스를 소진하면, 그 프로세스는 expired 배열로 이동한다. 이 모든 배열들은 `runqueue` 의 이름을 갖는 특별한 구조체에 저장된다. 글로벌 lock 과 runqueue 는 없기 때문에 모든 가능한 CPU 들을 다 방문하여 runqueue를 초기화 해야 한다. `runqueue` 는 리눅스 커널에서 [kernel/sched/sched.h](https://github.com/torvalds/linux/blob/master/kernel/sched/sched.h)에 선언된 `rq` 구조체로 표현된다.

```C
rq = cpu_rq(i);
raw_spin_lock_init(&rq->lock);
rq->nr_running = 0;
rq->calc_load_active = 0;
rq->calc_load_update = jiffies + LOAD_FREQ;
init_cfs_rq(&rq->cfs);
init_rt_rq(&rq->rt);
init_dl_rq(&rq->dl);
rq->rt.rt_runtime = def_rt_bandwidth.rt_runtime;
```

여기서 우리는 `runqueue` percpu 변수를 반환하는 `cpu_rq` 매크로를 이용하여 모든 CPU 를 위한 runqueue 를 얻고 runqueue lock, 수행 중인 태스크의 수, CPU 부하의 계산을 위해 사용되는  `calc_load` 관련 항목들(`calc_load_active` 와 `calc_load_update`) 그리고 CFS 를 위한 초기화, runqueue에서 실시간 그리고 데드라인과 관련된 항목들을 초기화한다. 이 초기화 이후에 `cpu_load` 배열을 0으로 초기화 하고 last_load_update_tick 을 시스템 부팅 이후로 지나온 시간 tick(틱-cycles) 수를 결정하는 `jiffies` 변수로 부터 업데이트 한다.:

```C
for (j = 0; j < CPU_LOAD_IDX_MAX; j++)
    rq->cpu_load[j] = 0;

rq->last_load_update_tick = jiffies;
```

`cpu_load` 는 과거에서 runqueue 의 부하를 계속 확인한다. 현재로는 `CPU_LOAD_IDX_MAX` 의 값은 5이다. 다음 단계는 [SMP](http://en.wikipedia.org/wiki/Symmetric_multiprocessing) 와 연관된 항목인 `runqueue` 를 채운다. 하지만 이 파트에서는 다루지 않을 것이다. 그리고 이 루프의 마지막에는 주어진 `runqueue`를 위한 high-resolution 타이머 초기화를 하고 `iowait` (스케줄러와 관련된 다른 파트에서 살펴보자.) 값을 설정한다.:

```C
init_rq_hrtick(rq);
atomic_set(&rq->nr_iowait, 0);
```

이제 `for_each_possible_cpu` 루프를 빠져 나왔고, 다음으로 `init` 태스크를 위해 `set_load_weight` 함수로 load weight(부하 가중치)를 설정할 것이다. 프로세스의 `Weight`(가중치)는 프로세스의 static priority(고정 우선순위) + scheduling class(스케줄링 클래스) 의 결과값인 동적 우선순위를 통해 계산된다. 이후에 `init` 프로세스의 메모리 디스크립터의 메모리 사용 카운터를 증가시키고 current 프로세스를 위해 스케줄러 클래스를 설정한다.:

```C
atomic_inc(&init_mm.mm_count);
current->sched_class = &fair_sched_class;
```

"current" 프로세스를 (첫 프로세스는 `init`이다) `idle`하게 만들고 `calc_load_update` 를 5 초 간격의 값으로 업데이트 한다.:

```C
init_idle(current, smp_processor_id());
calc_load_update = jiffies + LOAD_FREQ;
```

그래서, 다른 후보 프로세스가 없으니 `init` 프로세스는 수행될 것이다.(시스템의 첫번째 프로세스이다.) 마지막에는 단지 `scheduler_running` 값을 `1`로 설정한다.:

```C
scheduler_running = 1;
```

리눅스 커널 스케줄러가 초기화 되었다. 물론, 몇가지 자세한 사항이나 설명이 그냥 넘어가긴 했지만, 다른 챕터에서 조금 더 자세히 살펴 보기로 하자.

결론
--------------------------------------------------------------------------------

리눅스 커널 초기화 과정의 8 번째 파트가 마무리되었다. 이 파트에서는 스케줄러의 초기화 과정을 살펴 보았고, 다음 파트에서 리눅스 커널 초기화 과정을 계속 살펴 볼 것이다. 계속 살펴 볼 내용은, [RCU](http://en.wikipedia.org/wiki/Read-copy-update) 와 많은 다른 초기화 과정이 될 것이다.

어떤 질문이나 제안이 있다면, twitter [0xAX](https://twitter.com/0xAX), [email](anotherworldofworld@gmail.com) 또는 [issue](https://github.com/0xAX/linux-insides/issues/new) 를 만들어 주길 바란다.

**나는 영어권의 사람이 아니고 이런 것에 대해 매우 미안해 하고 있다. 만약 어떤 실수를 발견한다면, 나에게 PR을 [linux-insides](https://github.com/0xAX/linux-internals)을 보내줘**

Links
--------------------------------------------------------------------------------

* [CPU masks](http://0xax.gitbooks.io/linux-insides/content/Concepts/cpumask.html)
* [high-resolution kernel timer](https://www.kernel.org/doc/Documentation/timers/hrtimers.txt)
* [spinlock](http://en.wikipedia.org/wiki/Spinlock)
* [Run queue](http://en.wikipedia.org/wiki/Run_queue)
* [Linux kernel memory manager](http://0xax.gitbooks.io/linux-insides/content/mm/index.html)
* [slub](http://en.wikipedia.org/wiki/SLUB_%28software%29)
* [virtual file system](http://en.wikipedia.org/wiki/Virtual_file_system)
* [Linux kernel hotplug documentation](https://www.kernel.org/doc/Documentation/cpu-hotplug.txt)
* [IRQ](http://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29)
* [Global Descriptor Table](http://en.wikipedia.org/wiki/Global_Descriptor_Table)
* [Per-CPU variables](http://0xax.gitbooks.io/linux-insides/content/Concepts/per-cpu.html)
* [SMP](http://en.wikipedia.org/wiki/Symmetric_multiprocessing)
* [RCU](http://en.wikipedia.org/wiki/Read-copy-update)
* [CFS Scheduler documentation](https://www.kernel.org/doc/Documentation/scheduler/sched-design-CFS.txt)
* [Real-Time group scheduling](https://www.kernel.org/doc/Documentation/scheduler/sched-rt-group.txt)
* [Previous part](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-7.html)
* [boot_cpu_init](http://jake.dothome.co.kr/boot_cpu_init/)
* [실시간 리눅스 커널 스케줄러](http://www.linuxjournal.com/magazine/real-time-linux-kernel-scheduler?page=0,2)
