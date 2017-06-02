커널 초기화. Part 9.
================================================================================

RCU 초기화
================================================================================

리눅스 커널 초기화 과정의 9번째이고 이전 파트에서는 [스케줄러 초기화](https://github.com/daeseokyoun/linux-insides/blob/master/Initialization/linux-initialization-8.md)를 살펴보았다. 이 파트에서는 리눅스 커널 초기화 과정을 계속 살펴볼 것이고, 주목할 것은 [RCU](http://en.wikipedia.org/wiki/Read-copy-update) 의 초기화에 관한 것이다. `sched_init` 함수 다음으로 [init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c) 소스 파일에서 불리는 함수는 `preempt_disable` 함수이다. 여기에 두 개의 매크로가 있다.:

* `preempt_disable`
* `preempt_enable`

선점(preemption)을 활성화하고 비활성화하기 위한 것이다. 운영체제에서 `preempt`이 무엇인지 먼저 이해를 해봐야 한다. 단순하게 말하면, 선점(preemption)은 우선순위가 더 높은 태스크가 현재 수행중이 태스크를 선점할 수 있는 운영체제의 능력이라고 보면 된다. 여기서 우리는 선점을 비활성화 할 필요가 있는데, 이유는 초기 부팅 시간에는 단지 `init` 프로세스만 있고 `cpu_idle` 함수가 불리기 전에는 이 프로세스를 멈출 필요가 없기 때문이다. `preempt_disable` 매크로는 [include/linux/preempt.h](https://github.com/torvalds/linux/blob/master/include/linux/preempt.h) 에 정의되어 있으며, `CONFIG_PREEMPT_COUNT` 구성 옵션에 의존적이다. 이 매크로는 아래와 같이 구현되어 있다.:

```C
#define preempt_disable() \
do { \
        preempt_count_inc(); \
        barrier(); \
} while (0)
```

만약 `CONFIG_PREEMPT_COUNT` 이 설정되어 있지 않다면, 단지 barrier() 를 호출한다.:

```C
#define preempt_disable()                       barrier()
```

위의 두 가지 경우의 매크로 구현에 하나의 차이점을 볼 수 있다. `CONFIG_PREEMPT_COUNT` 이 설정되어 있다면, `preempt_disable` 은 `preempt_count_inc` 의 호출을 포함할 것이다. 여기서 `preempt_disable`와 lock 을 잡은 횟수를 저장하는 특별한 `percpu` 변수가 있다.:

```C
DECLARE_PER_CPU(int, __preempt_count__); // TODO 마지막 언더바 두개
```

`preempt_disable` 구현에서 처음으로 하는 것은 `__preempt_count` 값을 증가시키는 것이다. `__preempt_count` 의 값을 반환하는 API 가 있는데, 그것은 `preempt_count` 함수이다. `preempt_disable` 함수 호출이 되면, 제일 먼저 아래의 코드로 확장되는 `preempt_count_inc` 매크로를 통해 선점 카운더 값을 증가시킨다.:

```C
#define preempt_count_inc() preempt_count_add(1)
#define preempt_count_add(val)  __preempt_count_add(val)
```

`preempt_count_add` 는 다시 주어진 `percpu` 변수(`__preempt_count`) 에 `1` 을 더해주는 `raw_cpu_add_4` 매크로를 호출한다. (`precpu` 에 대해 조금 더 살펴보려면 [Per-CPU 변수](https://github.com/daeseokyoun/linux-insides/blob/master/Concepts/per-cpu.md) 파트를 참조하자.) 좋다, 우리는 `__preempt_count` 를 증가시켰고 그 다음으로 `barrier` 매크로를 호출하는 것을 볼 수 있다. `barrier` 매크로는 컴파일러 최적화 과정에서 `barrier`를 호출한 지점의 이전 코드와 이후 코드를 완벽하게 분리시켜주는 역할을 한다. `x86_64` 아키텍처의 프로세서들은 독립적인 메모리 접근 수행이 순서에 상관없이 일어날 수 있다. 그렇게 때문에 컴파일러에거 이런 점을 알려주어, 선점을 막고나서 불려야 하는 코드들이 먼저 불리는 일을 막아준다. 이 매커니즘이 메모리 barrier(장벽)이다. 간단한 예제를 보자.:

```C
preempt_disable();
foo();
preempt_enable();
```

컴파일러는 아래와 같이 재배치를 할 수 있다.:

```C
preempt_disable();
preempt_enable();
foo();
```

이 경우에 `foo` 라는 함수를 선점하지 못하게 의도했지만, 선점이 가능한 상태로 된다. 그래서 우리는 `barrier` 매크로를 통해 `preempt_disable` 와 `preempt_enable` 매크로 사이에 그 함수를 잘 호출할 수 있도록 한다. 즉, 컴파일러에게 다른 구문과 `preempt_count_inc`를 바꾸지 못하게 알려주는 역할을 한다. barrier에 대해 더 자세한 설명은 [여기](http://en.wikipedia.org/wiki/Memory_barrier) 와 [여기](https://www.kernel.org/doc/Documentation/memory-barriers.txt)를 보자.

다음 단계로 아래와 같은 코드를 볼 수 있다.:

```C
if (WARN(!irqs_disabled(),
	 "Interrupts were enabled *very* early, fixing it\n"))
	local_irq_disable();
```

이것은 [IRQs](http://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29) 의 상태를 확인하고 만약 활성화 되어 있다면, 비활성화(`x86_64`에서는 `cli` 명령어로) 한다.

선점은 비활성화 되었고, 계속 진행해보자.

정수형 ID 관리의 초기화
--------------------------------------------------------------------------------

다음 단계에서 [lib/idr.c](https://github.com/torvalds/linux/blob/master/lib/idr.c) 구현된 `idr_init_cache` 함수의 호출을 볼 수 있다. `idr` 라이브러리는 리눅스 커널내에 다양한 [곳](http://lxr.free-electrons.com/ident?i=idr_find)에서 객체로 할당된 정수형 `ID`들을 관리하고 id 로 객체를 찾아볼때 사용된다.

`idr_init_cache` 함수의 구현을 살펴 보자.:

```C
void __init idr_init_cache(void)
{
        idr_layer_cache = kmem_cache_create("idr_layer_cache",
                                sizeof(struct idr_layer), 0, SLAB_PANIC, NULL);
}
```

여기서 `kmem_cache_create` 의 호출을 볼 수 있다. 우리는 이미 [init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c#L485)에서 `kmem_cache_init` 호출을 보았다. 이 함수는 `kmem_cache_alloc`을 사용하여 캐쉬를 만든다. ([리눅스 메모리 관리](https://github.com/daeseokyoun/linux-insides/blob/master/mm/README.md) 에서 캐쉬에 관해 더 자세히 볼 수 있다.) 우리의 경우에, [slab](http://en.wikipedia.org/wiki/Slab_allocation) 할당자에 의해 사용되어질 `kmem_cache_t` 을 사용하고 그것은 `kmem_cache_create`로 만들어진다. `kmem_cache_create` 함수는 5개의 인자를 넘겨 받는다.:

* 캐쉬의 이름
* 캐쉬에 저장된 객체의 크기
* 페이지 내에서 첫 객체의 오프셋
* 플래그
* 객체를 위한 생성자

그리고 그것은 정수형 ID들을 위한 `kmem_cache`를 만들 것이다. 정수 `ID`는 일반적으로 정수 ID 를 갖고 연결된 포인터를 관리한다. 우리는 [i2c](http://en.wikipedia.org/wiki/I%C2%B2C) 드라이버 서브시스템내에 정수 ID 의 사용을 볼 수 있다. 예를 들어, `i2c` 서브 시스템의 코어 코드가 있는 [drivers/i2c/i2c-core.c](https://github.com/torvalds/linux/blob/master/drivers/i2c/i2c-core.c) 에서 `i2c` 어뎁터를 위한 `ID`를 `DEFINE_IDR` 매크로로 선언한다.:

```C
static DEFINE_IDR(i2c_adapter_idr);
```

그리고 나서 `i2c` 어뎁터의 선언을 아래와 같이 사용한다.:

```C
static int __i2c_add_numbered_adapter(struct i2c_adapter *adap)
{
  int     id;
  ...
  ...
  ...
  id = idr_alloc(&i2c_adapter_idr, adap, adap->nr, adap->nr + 1, GFP_KERNEL);
  ...
  ...
  ...
}
```

그리고 `id2_adapter_idr`는 동적으로 계산된 버스 번호를 가지게 된다.

정수 ID 관리를 위해 [문서](https://lwn.net/Articles/103209/)를 참조하자.

RCU 초기화
--------------------------------------------------------------------------------

다음 단계는 `rcu_init` 함수로 [RCU](http://en.wikipedia.org/wiki/Read-copy-update)를 초기화 하는 것이고 이 구현은 아래 두 개의 커널 구성 옵션에 의존적이다.:

* `CONFIG_TINY_RCU`
* `CONFIG_TREE_RCU`

첫 번째 구성옵션의 경우에는 `rcu_init` 함수는 [kernel/rcu/tiny.c](https://github.com/torvalds/linux/blob/master/kernel/rcu/tiny.c)에 있는 것으로 이용되고, 두 번째 경우에 [kernel/rcu/tree.c](https://github.com/torvalds/linux/blob/master/kernel/rcu/tree.c)에 있는 `rcu_init` 함수를 이용한다. 여기서 우리는 `tree rcu` 의 구현을 살펴볼 것이다. 하지만, 일반적으로 `RCU`에 관련된 내용은 대부분 첫번째 구성옵션을 쓴다.

`RCU` 혹은 Read-copy-update 는 리눅스 커널에서 구현된 확장성 있는 성능이 좋은 동기화 매커니즘(scalable high-performance synchronization mechanism)이다. 리눅스 커널 초반에 동시에 응용 프로그램들을 위한 환경과 지원을 했지만, 모든 실행은 단일 글로벌 락을 사용하여 커널 내에서 동시에 실행되지 않았다. 오늘날 리눅스는 단일 글로벌 락은 없어졌고 [lock-free data structures](http://en.wikipedia.org/wiki/Concurrent_data_structure), [percpu](https://github.com/daeseokyoun/linux-insides/blob/master/Concepts/per-cpu.md) 데이터 등을 이용하여 다른 매커니즘을 제공한다. 이들 매커니즘 중에 하나가 `read-copy update` 이다. `RCU` 기술은 드물게 쓰기가 일어나는 자료구조들을 위해 설계되었다. `RCU` 의 아이디어는 간단한다. 예를 들어, 드물게 수정이 일어나는 자료 구조를 갖고 있고 누군가가 자료구조를 변경하기 원했다면, 이 자료구조의 복사본을 만들고 모든 변경이 완료되면 복사한다. 동시에 모든 다른 이 자료구조의 사용자는 그것의 예전 버전을 사용한다. 다음, 안전하게 처리하기 위해 원본의 자료 구조를 더 이상 사용하는 사용자가 없으면 그것을 수정된 복사본으로 부터 업데이트 한다.

물론 이 `RCU`의 설명은 매우 간단하게 한 것이다. `RCU` 관해 조금 더 알기 위해, 먼저 몇몇의 용어들을 배울 필요가 있다. `RCU` 에서 데이터 리더(data reader)는 [critical section - 임계영역](http://en.wikipedia.org/wiki/Critical_section) 에서 수행된다. 데이터 리더가 임계영역에서 데이터를 얻는 모든 시점에는 `rcu_read_lock` 와 `rcu_read_unlock` 를 사용하여 임계역을 진입하고 빠지게 된다. 만약 쓰레드(Thread) 가 읽기를 위한 임계 영역에 있지 않다면, 그것의 상태는 `quiescent state` 라 불린다. 모든 쓰레드가 `quiescent state` 의 상태를 갖게 되는 순간을 `grace period` 라 불린다. 만약 하나의 쓰레드가 자료 구조에 있는 하나의 항목을 제거하기 원한다면, 두 단계로 이 단계를 지원한다. 첫 번째는 `removal(삭제)` 이다 - 자동적으로 자료구조로 부터 항목을 제거한다. 하지만 실제 메모리에는 반영하지 않는다. 그런 다음 thread-writer 는 업데이트를 알리고 마무리 될 때까지 기다린다. 이 순간에는 삭제되는 항목이 thread-reader 에게는 사용가능하다. `grace period` 기간이 끝나면, 두 번째 단계는 항목 삭제를 시작하고 실제 메모리에 반영을 한다.

`RCU` 의 구현은 한 두가지가 있다. 예전에는 클래식 `RCU` 라 불렸고, 새로운 구현은 `tree RCU` 라 불린다. 이미 이해했겠지만, `CONFIG_TREE_RCU` 커널 구성 옵션이 활성화되면 `tree RCU` 가 선택된다. 다른 하나는 `CONFIG_TINY_RCU` 와 `CONFIG_SMP=n` 으로 설정되면 `tiny RCU` 이다. `RCU` 에 대해 더 자세히 알고 싶다면 동기화 관련 챕터에서 볼 수 있다. 여기서는 [kernel/rcu/tree.c](https://github.com/torvalds/linux/blob/master/kernel/rcu/tree.c) 에 구현되어 잇는 `rcu_init`를 보자.:

```C
void __init rcu_init(void)
{
         int cpu;

         rcu_bootup_announce();
         rcu_init_geometry();
         rcu_init_one(&rcu_bh_state, &rcu_bh_data);
         rcu_init_one(&rcu_sched_state, &rcu_sched_data);
         __rcu_init_preempt();
         open_softirq(RCU_SOFTIRQ, rcu_process_callbacks);

         /*
          * We don't need protection against CPU-hotplug here because
          * this is called early in boot, before either interrupts
          * or the scheduler are operational.
          */
         cpu_notifier(rcu_cpu_notify, 0);
         pm_notifier(rcu_pm_notify, 0);
         for_each_online_cpu(cpu)
                 rcu_cpu_notify(NULL, CPU_UP_PREPARE, (void *)*(long)cpu); // TODO long 앞의 별하나

         rcu_early_boot_tests();
}
```

`rcu_init` 함수의 초반에는 `cpu` 변수를 선언하고 `rcu_bootup_announce` 함수를 호출한다. `rcu_bootup_announce` 함수는 꽤나 단순한다.:

```C
static void __init rcu_bootup_announce(void)
{
        pr_info("Hierarchical RCU implementation.\n");
        rcu_bootup_announce_oddness();
}
```

이 함수는 단지 `RCU` 관련하여 `pr_info` 함수를 통해 정보를 출력하고, `pr_info` 함수를 이용하여 추가적인 정보를 출력하는 `rcu_bootup_announce_oddness` 함수를 호출한다. 현재 `CONFIG_RCU_TRACE`, `CONFIG_PROVE_RCU`, `CONFIG_RCU_FANOUT_EXACT` 등의 서로 다른 커널 구성 옵션에 의존적인 `RCU` 구성 옵션에 관련된 다른 정보를 출력하기 위해서 `rcu_bootup_announce_oddness` 함수를 이용한다. 다음 단계에서는, `rcu_init_geometry` 함수의 호출을 볼 수 있다. 이 함수는 같은 소스 파일에 구현되어 있고 CPU 개수에 의존적인 노드 트리를 계산한다. 실제 `RCU`는 정말 낮은 RCU lock 경쟁을 보장하기 위해 확장성을 제공한다. 만약 하나의 자료 구조가 다른 CPU 에서 읽혀진다면? `RCU` API 는 노드 계층 구조를 포함한 RCU 글로벌 상태를 표현하는 `rcu_state` 를 제공한다. 계층 구조는 아래 구조체로 표현된다.:

```
struct rcu_node node[NUM_RCU_NODES];
```

위의 선언의 커멘트를 아래와 같이 읽을 수 있다.:

```
The root (first level) of the hierarchy is in ->node[0] (referenced by ->level[0]), the second
level in ->node[1] through ->node[m] (->node[1] referenced by ->level[1]), and the third level
in ->node[m+1] and following (->node[m+1] referenced by ->level[2]).  The number of levels is
determined by the number of CPUs and by CONFIG_RCU_FANOUT.

Small systems will have a "hierarchy" consisting of a single rcu_node.

계층 구조에서 루트(첫 레벨)는 ->node[0] 에 있다. (->level[0] 의해 참조된), 두 번째 레벨은 ->node[1]
(->level[1] 에 의해 ->node[1] 접근) 그리고 3번째 레벨은 당연히 ->node[m+1] (->level[2] 에 참조되는 ->node[m+1]) 이다.
레벨의 수는 CPU 의 개수와 CONFIG_RCU_FANOUT 에 의해 결정된다.

작은 시스템에서 "계층"은 단일 rcu_node 로 구성된다.
```

`rcu_node` 구조체는 [kernel/rcu/tree.h](https://github.com/torvalds/linux/blob/master/kernel/rcu/tree.h)에 선언되어 있고 현재 grace period 에 관련된 정보, 즉 grace period 가 완료 혹은 완료상태가 아닌, 현재 grace period 를 진행하기 위한 다음 순서에 진행될 필요가 있는 CPU 들이나 그룹 같은 것을 포함한다. 모든 `rcu_node` 는 몇몇의 CPU 들을 위한 lock 을 포함한다. 이런 `rcu_node` 구조체는 `rcu_state` 구조체의 선형 배열내에 들어가 있고, 첫 요소인 루트를 트리 형태로 표현하여 모든 CPU 를 다룬다. 가용한 CPU 의 수에 의존적인 `NUM_RCU_NODES` 에 의해 rcu 노드의 개수를 결정한다.:

```C
#define NUM_RCU_NODES (RCU_SUM - NR_CPUS)
#define RCU_SUM (NUM_RCU_LVL_0 + NUM_RCU_LVL_1 + NUM_RCU_LVL_2 + NUM_RCU_LVL_3 + NUM_RCU_LVL_4)
```

`CONFIG_RCU_FANOUT_LEAF` 구성 옵션에 의존적인 레벨의 값을 결정한다. 예를 들어, 가장 단순한 경우에, 하나의 `rcu_node` 는 8 개의 코어를 가지는 두 개의 CPU 를 다룰 수 있다.:

```
+-----------------------------------------------------------------+
|  rcu_state                                                      |
|                 +----------------------+                        |
|                 |         root         |                        |
|                 |       rcu_node       |                        |
|                 +----------------------+                        |
|                    |                |                           |
|               +----v-----+       +--v-------+                   |
|               |          |       |          |                   |
|               | rcu_node |       | rcu_node |                   |
|               |          |       |          |                   |
|         +------------------+     +----------------+             |
|         |                  |        |             |             |
|         |                  |        |             |             |
|    +----v-----+    +-------v--+   +-v--------+  +-v--------+    |
|    |          |    |          |   |          |  |          |    |
|    | rcu_node |    | rcu_node |   | rcu_node |  | rcu_node |    |
|    |          |    |          |   |          |  |          |    |
|    +----------+    +----------+   +----------+  +----------+    |
|         |                 |             |               |       |
|         |                 |             |               |       |
|         |                 |             |               |       |
|         |                 |             |               |       |
+---------|-----------------|-------------|---------------|-------+
          |                 |             |               |
+---------v-----------------v-------------v---------------v--------+
|                 |                |               |               |
|     CPU1        |      CPU3      |      CPU5     |     CPU7      |
|                 |                |               |               |
|     CPU2        |      CPU4      |      CPU6     |     CPU8      |
|                 |                |               |               |
+------------------------------------------------------------------+
```

그래서, `rcu_init_geometry` 함수 내에서는 단지 `rcu_node` 구조체의 총 개수를 계산을 한다. 이 함수는 `force-quiescent-state` 인 첫 번째와 그 다음의 `fqs` 의 값을 `jiffies` 로 계산하는 것으로 시작한다.:

```C
d = RCU_JIFFIES_TILL_FORCE_QS + nr_cpu_ids / RCU_JIFFIES_FQS_DIV;
if (jiffies_till_first_fqs == ULONG_MAX)
        jiffies_till_first_fqs = d;
if (jiffies_till_next_fqs == ULONG_MAX)
        jiffies_till_next_fqs = d;
```

각 선언된 값들은:

```C
#define RCU_JIFFIES_TILL_FORCE_QS (1 + (HZ > 250) + (HZ > 500))
#define RCU_JIFFIES_FQS_DIV     256
```

이렇게 [jiffies](http://en.wikipedia.org/wiki/Jiffy_%28time%29) 값을 계산하고 나서 `jiffies_till_first_fqs` 와 `jiffies_till_next_fqs` 변수 값이 [ULONG_MAX](http://www.rowleydownload.co.uk/avr/documentation/index.htm?http://www.rowleydownload.co.uk/avr/documentation/ULONG_MAX.htm) 와 같은지 확인하고 계산된 값을 설정한다. 사실 이 변수들은 이전에 변경한 적이 없으므로, `ULONG_MAX` 와 같았을 것이다.:

```C
static ulong jiffies_till_first_fqs = ULONG_MAX;
static ulong jiffies_till_next_fqs = ULONG_MAX;
```

`rcu_init_geometry` 의 다음 단계는, `rcu_fanout_leaf` 값이 수정되지 않았고 `CONFIG_RCU_FANOUT_LEAF` 의 값과 같은지 확인하여, 그렇다면 그냥 반환하고 마무리한다.:

```C
if (rcu_fanout_leaf == CONFIG_RCU_FANOUT_LEAF &&
    nr_cpu_ids == NR_CPUS)
    return;
```

확인이 끝나고, 우리는 `rcu_node` 트리가 주어진 레벨의 수를 처리할 수 있도록 node 의 수를 계산한다.:

```C
rcu_capacity[0] = 1;
rcu_capacity[1] = rcu_fanout_leaf;
for (i = 2; i <= MAX_RCU_LVLS; i++)
    rcu_capacity[i] = rcu_capacity[i - 1] * CONFIG_RCU_FANOUT;
```

그리고 마지막 단계로는 [루프](https://github.com/torvalds/linux/blob/master/kernel/rcu/tree.c#L4094)를 통해 트리의 각 레벨에 `rcu_nodes`의 수를 계산한다.

`rcu_node` 트리의 기하를 계산하고 나서 `rcu_init` 함수로 돌아가서, 다음 단계로 `rcu_init_one` 함수와 함께 두 개의 `rcu_state` 구조체를 초기화할 필요가 있다.:

```C
rcu_init_one(&rcu_bh_state, &rcu_bh_data);
rcu_init_one(&rcu_sched_state, &rcu_sched_data);
```

`rcu_init_one` 함수는 두 개의 인자를 받는다.:

* 글로벌 `RCU` 상태
* `RCU`를 위한 Per-CPU data.

두 변수들은 그것의 `percpu` 데이터와 함께 [kernel/rcu/tree.h](https://github.com/torvalds/linux/blob/master/kernel/rcu/tree.h)에 선언되어 있다.:

```
extern struct rcu_state rcu_bh_state;
DECLARE_PER_CPU(struct rcu_data, rcu_bh_data);
```

About this states you can read [here](http://lwn.net/Articles/264090/). As I wrote above we need to initialize `rcu_state` structures and `rcu_init_one` function will help us with it. After the `rcu_state` initialization, we can see the call of the ` __rcu_init_preempt` which depends on the `CONFIG_PREEMPT_RCU` kernel configuration option. It does the same as previous functions - initialization of the `rcu_preempt_state` structure with the `rcu_init_one` function which has `rcu_state` type. After this, in the `rcu_init`, we can see the call of the:


```C
open_softirq(RCU_SOFTIRQ, rcu_process_callbacks);
```

function. This function registers a handler of the `pending interrupt`. Pending interrupt or `softirq` supposes that part of actions can be delayed for later execution when the system is less loaded. Pending interrupts is represented by the following structure:

```C
struct softirq_action
{
        void    (*action)(struct softirq_action *);
};
```

which is defined in the [include/linux/interrupt.h](https://github.com/torvalds/linux/blob/master/include/linux/interrupt.h) and contains only one field - handler of an interrupt. You can check about `softirqs` in the your system with the:

```
$ cat /proc/softirqs
                    CPU0       CPU1       CPU2       CPU3       CPU4       CPU5       CPU6       CPU7
          HI:          2          0          0          1          0          2          0          0
       TIMER:     137779     108110     139573     107647     107408     114972      99653      98665
      NET_TX:       1127          0          4          0          1          1          0          0
      NET_RX:        334        221     132939       3076        451        361        292        303
       BLOCK:       5253       5596          8        779       2016      37442         28       2855
BLOCK_IOPOLL:          0          0          0          0          0          0          0          0
     TASKLET:         66          0       2916        113          0         24      26708          0
       SCHED:     102350      75950      91705      75356      75323      82627      69279      69914
     HRTIMER:        510        302        368        260        219        255        248        246
         RCU:      81290      68062      82979      69015      68390      69385      63304      63473
```

The `open_softirq` function takes two parameters:

* index of the interrupt;
* interrupt handler.

and adds interrupt handler to the array of the pending interrupts:

```C
void open_softirq(int nr, void (*action)(struct softirq_action *))
{
        softirq_vec[nr].action = action;
}
```

In our case the interrupt handler is - `rcu_process_callbacks` which is defined in the [kernel/rcu/tree.c](https://github.com/torvalds/linux/blob/master/kernel/rcu/tree.c) and does the `RCU` core processing for the current CPU. After we registered `softirq` interrupt for the `RCU`, we can see the following code:

```C
cpu_notifier(rcu_cpu_notify, 0);
pm_notifier(rcu_pm_notify, 0);
for_each_online_cpu(cpu)
    rcu_cpu_notify(NULL, CPU_UP_PREPARE, (void *)(long)cpu);
```

Here we can see registration of the `cpu` notifier which needs in systems which supports [CPU hotplug](https://www.kernel.org/doc/Documentation/cpu-hotplug.txt) and we will not dive into details about this theme. The last function in the `rcu_init` is the `rcu_early_boot_tests`:

```C
void rcu_early_boot_tests(void)
{
        pr_info("Running RCU self tests\n");

        if (rcu_self_test)
                 early_boot_test_call_rcu();
         if (rcu_self_test_bh)
                 early_boot_test_call_rcu_bh();
         if (rcu_self_test_sched)
                early_boot_test_call_rcu_sched();
}
```

which runs self tests for the `RCU`.

That's all. We saw initialization process of the `RCU` subsystem. As I wrote above, more about the `RCU` will be in the separate chapter about synchronization primitives.

Rest of the initialization process
--------------------------------------------------------------------------------

Ok, we already passed the main theme of this part which is `RCU` initialization, but it is not the end of the linux kernel initialization process. In the last paragraph of this theme we will see a couple of functions which work in the initialization time, but we will not dive into deep details around this function for different reasons. Some reasons not to dive into details are following:

* They are not very important for the generic kernel initialization process and depend on the different kernel configuration;
* They have the character of debugging and not important for now;
* We will see many of this stuff in the separate parts/chapters.

After we initialized `RCU`, the next step which you can see in the [init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c) is the - `trace_init` function. As you can understand from its name, this function initialize [tracing](http://en.wikipedia.org/wiki/Tracing_%28software%29) subsystem. You can read more about linux kernel trace system - [here](http://elinux.org/Kernel_Trace_Systems).

After the `trace_init`, we can see the call of the `radix_tree_init`. If you are familiar with the different data structures, you can understand from the name of this function that it initializes kernel implementation of the [Radix tree](http://en.wikipedia.org/wiki/Radix_tree). This function is defined in the [lib/radix-tree.c](https://github.com/torvalds/linux/blob/master/lib/radix-tree.c) and you can read more about it in the part about [Radix tree](https://0xax.gitbooks.io/linux-insides/content/DataStructures/radix-tree.html).

In the next step we can see the functions which are related to the `interrupts handling` subsystem, they are:

* `early_irq_init`
* `init_IRQ`
* `softirq_init`

We will see explanation about this functions and their implementation in the special part about interrupts and exceptions handling. After this many different functions (like `init_timers`, `hrtimers_init`, `time_init`, etc.) which are related to different timing and timers stuff. We will see more about these function in the chapter about timers.

The next couple of functions are related with the [perf](https://perf.wiki.kernel.org/index.php/Main_Page) events - `perf_event-init` (there will be separate chapter about perf), initialization of the `profiling` with the `profile_init`. After this we enable `irq` with the call of the:

```C
local_irq_enable();
```

which expands to the `sti` instruction and making post initialization of the [SLAB](http://en.wikipedia.org/wiki/Slab_allocation) with the call of the `kmem_cache_init_late` function (As I wrote above we will know about the `SLAB` in the [Linux memory management](http://0xax.gitbooks.io/linux-insides/content/mm/index.html) chapter).

After the post initialization of the `SLAB`, next point is initialization of the console with the `console_init` function from the [drivers/tty/tty_io.c](https://github.com/torvalds/linux/blob/master/drivers/tty/tty_io.c).

After the console initialization, we can see the `lockdep_info` function which prints information about the [Lock dependency validator](https://www.kernel.org/doc/Documentation/locking/lockdep-design.txt). After this, we can see the initialization of the dynamic allocation of the `debug objects` with the `debug_objects_mem_init`, kernel memory leak [detector](https://www.kernel.org/doc/Documentation/kmemleak.txt) initialization with the `kmemleak_init`, `percpu` pageset setup with the `setup_per_cpu_pageset`, setup of the [NUMA](http://en.wikipedia.org/wiki/Non-uniform_memory_access) policy with the `numa_policy_init`, setting time for the scheduler with the `sched_clock_init`, `pidmap` initialization with the call of the `pidmap_init` function for the initial `PID` namespace, cache creation with the `anon_vma_init` for the private virtual memory areas and early initialization of the [ACPI](http://en.wikipedia.org/wiki/Advanced_Configuration_and_Power_Interface) with the `acpi_early_init`.

This is the end of the ninth part of the [linux kernel initialization process](http://0xax.gitbooks.io/linux-insides/content/Initialization/index.html) and here we saw initialization of the [RCU](http://en.wikipedia.org/wiki/Read-copy-update). In the last paragraph of this part (`Rest of the initialization process`) we will go through many functions but did not dive into details about their implementations. Do not worry if you do not know anything about these stuff or you know and do not understand anything about this. As I already wrote many times, we will see details of implementations in other parts or other chapters.

Conclusion
--------------------------------------------------------------------------------

It is the end of the ninth part about the linux kernel [initialization process](http://0xax.gitbooks.io/linux-insides/content/Initialization/index.html). In this part, we looked on the initialization process of the `RCU` subsystem. In the next part we will continue to dive into linux kernel initialization process and I hope that we will finish with the `start_kernel` function and will go to the `rest_init` function from the same [init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c) source code file and will see the start of the first process.

If you have any questions or suggestions write me a comment or ping me at [twitter](https://twitter.com/0xAX).

**Please note that English is not my first language, And I am really sorry for any inconvenience. If you find any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-insides).**

Links
--------------------------------------------------------------------------------

* [lock-free data structures](http://en.wikipedia.org/wiki/Concurrent_data_structure)
* [kmemleak](https://www.kernel.org/doc/Documentation/kmemleak.txt)
* [ACPI](http://en.wikipedia.org/wiki/Advanced_Configuration_and_Power_Interface)
* [IRQs](http://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29)
* [RCU](http://en.wikipedia.org/wiki/Read-copy-update)
* [RCU documentation](https://github.com/torvalds/linux/tree/master/Documentation/RCU)
* [integer ID management](https://lwn.net/Articles/103209/)
* [Documentation/memory-barriers.txt](https://www.kernel.org/doc/Documentation/memory-barriers.txt)
* [Runtime locking correctness validator](https://www.kernel.org/doc/Documentation/locking/lockdep-design.txt)
* [Per-CPU variables](http://0xax.gitbooks.io/linux-insides/content/Concepts/per-cpu.html)
* [Linux kernel memory management](http://0xax.gitbooks.io/linux-insides/content/mm/index.html)
* [slab](http://en.wikipedia.org/wiki/Slab_allocation)
* [i2c](http://en.wikipedia.org/wiki/I%C2%B2C)
* [Previous part](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-8.html)
* [선점 비활성화](http://egloos.zum.com/nimhaplz/v/5683475)
* [정수 ID 관리-IDR](http://egloos.zum.com/studyfoss/v/5187192)
* [정수 ID 관리](http://jake.dothome.co.kr/idr/)
* [rcu_init 설명](http://jake.dothome.co.kr/rcu_init/)
* [rcu 설명](https://kukuruku.co/post/lock-free-data-structures-the-inside-rcu/)
