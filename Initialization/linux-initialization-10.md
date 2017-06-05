커널 초기화. Part 10.
================================================================================

리눅스 커널 초기화 과정의 마지막
================================================================================

이 파트는 [리눅스 커널 초기화 과정](https://github.com/daeseokyoun/linux-insides/blob/master/Initialization/README.md)의 10 번째 파트이고 이전 [파트](https://github.com/daeseokyoun/linux-insides/blob/master/Initialization/linux-initialization-9.md) 에서 [RCU](http://en.wikipedia.org/wiki/Read-copy-update) 초기화를 살펴 보았고 `acpi_early_init` 함수의 호출에서 마무리 지었었다. 이 파트에서는 커널 초기화 과정의 마지막이 될 것이다. 이제 마무리를 지어보자.

[init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c) 소스 파일에서 `acpi_early_init` 함수의 호출이 있는 뒤로 아래의 코드를 볼 수 있다.:

```C
#ifdef CONFIG_X86_ESPFIX64
	init_espfix_bsp();
#endif
```

여기서 `CONFIG_X86_ESPFIX64` 커널 구성 옵션에 의존적인 `init_espfix_bsp` 함수의 호출을 볼 수 있다. 이름에서 볼 수 있듯이, 스택과 관련된 무엇인가를 하는 것이다. 이 함수는 [arch/x86/kernel/espfix_64.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/espfix_64.c)에 구현되어 있고 16 비트 스택에서 돌아보는 과정에서 `esp` 레지스터의 `31:16` 비트의 누수(leacking)를 방지한다. 먼저, `init_espfix_bs` 에서 `espfix` page upper directory 를 커널 페이지 디렉토리로 설정한다.:

```C
pgd_p = &init_level4_pgt[pgd_index(ESPFIX_BASE_ADDR)];
pgd_populate(&init_mm, pgd_p, (pud_t *)espfix_pud_page);
```

`ESPFIX_BASE_ADDR` 의 선언은:

```C
#define PGDIR_SHIFT     39
#define ESPFIX_PGD_ENTRY _AC(-2, UL)
#define ESPFIX_BASE_ADDR (ESPFIX_PGD_ENTRY << PGDIR_SHIFT)
```

또한 [Documentation/x86/x86_64/mm](https://github.com/torvalds/linux/blob/master/Documentation/x86/x86_64/mm.txt) 에서도 내용을 찾아 볼 수 있다.:

```
... unused hole ...
ffffff0000000000 - ffffff7fffffffff (=39 bits) %esp fixup stacks
... unused hole ...
```

`espfix` pud 와 함께 페이지 글로벌 디렉토리(pgd) 를 채우고 나서, 다음 단계는 `init_espfix_random` 와 `init_espfix_ap` 함수의 호출이다. 첫 번째 함수는 `espfix` 페이지를 위한 랜덤 위치를 반환하고 두 번째 함수는 현재 CPU 를 위한 `espfix`를 활성화 한다. `init_espfix_bsp` 함수가 마무리되고 나서, [kernel/fork.c](https://github.com/torvalds/linux/blob/master/kernel/fork.c) 에 구현된 `thread_info_cache_init` 함수가 호출되고 `THREAD_SIZE` 크기가 `PAGE_SIZE` 보다 작다면 `thread_info`를 위한 캐쉬를 할당한다.:

```C
# if THREAD_SIZE >= PAGE_SIZE
...
...
...
void thread_info_cache_init(void)
{
        thread_info_cache = kmem_cache_create("thread_info", THREAD_SIZE,
                                              THREAD_SIZE, 0, NULL);
        BUG_ON(thread_info_cache == NULL);
}
...
...
...
#endif
```

As we already know the `PAGE_SIZE` is `(_AC(1,UL) << PAGE_SHIFT)` or `4096` bytes and `THREAD_SIZE` is `(PAGE_SIZE << THREAD_SIZE_ORDER)` or `16384` bytes for the `x86_64`. The next function after the `thread_info_cache_init` is the `cred_init` from the [kernel/cred.c](https://github.com/torvalds/linux/blob/master/kernel/cred.c). This function just allocates cache for the credentials (like `uid`, `gid`, etc.):
`PAGE_SIZE` 는 `(_AC(1,UL) << PAGE_SHIFT)` 이거나 `4096` 바이트라고 알고 있고 `x86_64`를 위힌 `THREAD_SIZE`는 `(PAGE_SIZE << THREAD_SIZE_ORDER)` 혹은 `16384` 바이트가 된다. `thread_info_cache_init` 함수 다음으로 호출되는 것은 [kernel/cred.c](https://github.com/torvalds/linux/blob/master/kernel/cred.c) 에 구현된, `cred_init` 함수이다. 이 함수는 단지 credentials 을 위한(`uid`, `gid` 등과 같은) 캐쉬를 할당한다.

```C
void __init cred_init(void)
{
         cred_jar = kmem_cache_create("cred_jar", sizeof(struct cred),
                                     0, SLAB_HWCACHE_ALIGN|SLAB_PANIC, NULL);
}
```

credentials(자격) 에 관련된 내용은 [Documentation/security/credentials.txt](https://github.com/torvalds/linux/blob/master/Documentation/security/credentials.txt) 에서 더 확인을 해보자. `fork_init` 함수는 `task_struct` 를 위한 공간을 할당한다. `fork_init` 함수의 구현을 살펴보자. 처음에 `ARCH_MIN_TASKALIGN` 매크로가 선언을 볼수 있고, `task_struct` 가 할당될 slab 의 생성을 볼 수 있다.:

```C
#ifndef CONFIG_ARCH_TASK_STRUCT_ALLOCATOR
#ifndef ARCH_MIN_TASKALIGN
#define ARCH_MIN_TASKALIGN      L1_CACHE_BYTES
#endif
        task_struct_cachep =
                kmem_cache_create("task_struct", sizeof(struct task_struct),
                        ARCH_MIN_TASKALIGN, SLAB_PANIC | SLAB_NOTRACK, NULL);
#endif
```

이 코드는 `CONFIG_ARCH_TASK_STRUCT_ACLLOCATOR` 에 의존적이다. 이 구성 옵션은 주어진 아키텍처를 위한 `alloc_task_struct` 존재 여부를 보여준다. `x86_64`에서는 `alloc_task_struct` 함수는 없으며, 이 코드는 수행되지 않을 뿐만 아니라 `x86_64`에서는 컴파일도 되지 않는다.

init 태스크를 위한 캐쉬 할당
--------------------------------------------------------------------------------

`fork_init` 함수에서 다음 단계는 `arch_task_cache_init` 함수 호출이다.:

```C
void arch_task_cache_init(void)
{
        task_xstate_cachep =
                kmem_cache_create("task_xstate", xstate_size,
                                  __alignof__(union thread_xstate),
                                  SLAB_PANIC | SLAB_NOTRACK, NULL);
        setup_xstate_comp();
}
```

`arch_task_cache_init` 는 아키텍처 의존적인 캐쉬의 초기화를 진행한다. `x86_64` 의 경우, `arch_task_cache_init` 함수는 [FPU](http://en.wikipedia.org/wiki/Floating-point_unit) 상태를 표현하는 `task_xstate`를 위한 메모리를 할당하고 `setup_xstate_comp` 함수의 호출을 통해 [xsave](http://www.felixcloutier.com/x86/XSAVES.html)  영역 내에 있는 확장된 상태들의 크기와 오프셋을 설정한다. `arch_task_cache_init` 함수 이후에는 기본 최대 쓰레드 개수를 계산하여 설정한다.:

```C
set_max_threads(MAX_THREADS);
```

기본 최대 쓰레드 개수는 다음과 같다.:

```C
#define FUTEX_TID_MASK  0x3fffffff
#define MAX_THREADS     FUTEX_TID_MASK
```

`fork_init` 함수의 마지막 부분에 [signal](http://www.win.tue.nl/~aeb/linux/lk/lk-5.html) 핸들러를 초기화 했다.:

```C
init_task.signal->rlim[RLIMIT_NPROC].rlim_cur = max_threads/2;
init_task.signal->rlim[RLIMIT_NPROC].rlim_max = max_threads/2;
init_task.signal->rlim[RLIMIT_SIGPENDING] =
		init_task.signal->rlim[RLIMIT_NPROC];
```

`init_task` 는 `task_struct` 구조체의 인스턴스이란 것을 알고 있고, 시그널 핸들러를 관리하는 `signal` 항목을 포함한다. 그것은 `struct signal_struct` 타입이다. 위의 코드에서 첫 두라인은 `resource limits`의 현재와 최대치를 설정한다. 모든 프로세스는 resource limits(자원 제약)의 설정과 연관되어 있다. 이런 제약들은 현재 프로세스에서 사용될 수 있는 자원의 양을 특정하다. 여기서 `rlim` 은 아래와 같이 표현된다.:

```C
struct rlimit {
        __kernel_ulong_t        rlim_cur;
        __kernel_ulong_t        rlim_max;
};
```

이 구조체는 [include/uapi/linux/resource.h](https://github.com/torvalds/linux/blob/master/include/uapi/linux/resource.h)에 선언되어 있다. 사용자가 가질 수 있는 최대 프로세서의 개수는 `RLIMIT_NPROC` 이고 `RLIMIT_SIGPENDING` 는 최대 지연된 signal 을 개수를 표현한다. 우리는 아래와 같이 확인 가능하다.:

```C
cat /proc/self/limits
Limit                     Soft Limit           Hard Limit           Units     
...
...
...
Max processes             63815                63815                processes
Max pending signals       63815                63815                signals   
...
...
...
```

캐쉬 초기화
--------------------------------------------------------------------------------

`fork_init` 함수 다음에는 [kernel/fork.c](https://github.com/torvalds/linux/blob/master/kernel/fork.c) 에 있는 `proc_caches_init` 호출이다. 이 함수는 메모리 디스크립터(즉, `mm_struct`)를 위한 메모리를 할당한다. `proc_caches_init` 함수의 초반에 `kmem_cache_create` 함수의 호출로 [SLAB](http://en.wikipedia.org/wiki/Slab_allocation) 캐쉬를 아래의 항목들을 할당한다.:

* `sighand_cachep` - 설정된 시그널 핸들러를 관련하여 정보 관리
* `signal_cachep` - 프로세스 시그널 디스크립터에 관련된 정보 관리
* `files_cachep` - 열러진 파일들을 위한 정보 관리
* `fs_cachep` - 파일시스템 정보 관리

이 다음에는 `mm_struct` 구조체를 위한 `SLAB` 캐쉬를 할당한다.:

```C
mm_cachep = kmem_cache_create("mm_struct",
                         sizeof(struct mm_struct), ARCH_MIN_MMSTRUCT_ALIGN,
                         SLAB_HWCACHE_ALIGN|SLAB_PANIC|SLAB_NOTRACK, NULL);
```

이 다음에는 커널에 의해 사용되어 가상 메모리를 관리하기 위해 중요한 `vm_area_struct` 를 위해 `SLAB` 캐쉬를 할당한다.:

```C
vm_area_cachep = KMEM_CACHE(vm_area_struct, SLAB_PANIC);
```

여기서는 `kmem_cache_create` 함수 대신에 `KMEM_CACHE` 매크로를 사용한다. 이 매크로는 [include/linux/slab.h](https://github.com/torvalds/linux/blob/master/include/linux/slab.h) 에 선언되어 있고 `kmem_cache_create` 호출을 확장한다.:

```C
#define KMEM_CACHE(__struct, __flags) kmem_cache_create(#__struct,\
                sizeof(struct __struct), __alignof__(struct __struct),\
                (__flags), NULL)
```

`KMEM_CACHE` 는 `kmem_cache_create` 호출과 한가지 다른 점이 있다. `__alignof__` 에 주목해보자. `KMEM_CACHE` 매크로는 주어진 구조체의 크기에 `SLAB`을 정렬(align)한다. 하지만, `kmem_cache_create` 함수는 정렬(align) 공간에 주어진 값을 사용한다. 이것 다음에는 `mmap_init` 와 `nsproxy_cache_init` 함수를 호출한다. 첫 번째는 가상 메모리 영역을 초기화 하고 두 번째 함수는 네임스페이스를 위한 `SLAB`을 초기화한다.

`proc_caches_init` 함수 이후에 불리는 함수는 `buffer_init` 함수이다. 이 함수는 [fs/buffer.c](https://github.com/torvalds/linux/blob/master/fs/buffer.c) 에 구현되어 있고 `buffer_head` 를 위해 캐쉬를 할당한다. `buffer_head` 는 [include/linux/buffer_head.h](https://github.com/torvalds/linux/blob/master/include/linux/buffer_head.h) 에 선언된 특별한 구조체이고 버퍼를 관리하기 위해 사용된다. `buffer_init` 함수의 시작은 이전 함수에서 했던 방식대로 `kmem_cache_create` 함수의 호출로 `struct buffer_head` 구조체를 위한 캐쉬를 할당한다.  그리고 메모리에 버퍼의 최대 크기를 계산한다.:

```C
nrpages = (nr_free_buffer_pages() * 10) / 100;
max_buffer_heads = nrpages * (PAGE_SIZE / sizeof(struct buffer_head));
```

최대 크기는 `ZONE_NORMAL` (`x86_64` 에서는 4GB 이후에 구성되는 모든 RAM) 의 `10%` 와 같다. `buffer_init` 함수 후에 불리는 `vfs_caches_init` 이다. 이 함수는 `SLAB` 캐쉬와 [VFS](http://en.wikipedia.org/wiki/Virtual_file_system) 캐쉬를 위해 hashtable 을 할당한다. `vfs_caches_init_early` 함수는   `dcache` (or directory-cache) 와 [inode](http://en.wikipedia.org/wiki/Inode) cache를 초기화 했던 리눅스 커널 초기화 8번째 파트에서 확인 해보았다. `vfs_caches_init` 함수는 `dcache` 와 `inode` 캐쉬, private 데이터 캐쉬, 마운트 포인트를 위한 해쉬 테일이블 등의 초기화를 한다. [VFS](http://en.wikipedia.org/wiki/Virtual_file_system) 에 대한 자세한 내용은 다른 파트에서 다루어 보도록 하자. 이 다름 함수는 `signals_init` 함수이다. 이 함수는 [kernel/signal.c](https://github.com/torvalds/linux/blob/master/kernel/signal.c) 에 구현되어 있고 실시간 시그널의 큐를 관리하기 위한 `sigqueue` 구조체 캐쉬를 할당한다. 다음 함수는 `page_writeback_init` 이다. 이 함수는 dirty 페이지를 위한 비율(ratio)을 초기화한다. 모든 낮은 수준 페이지 엔트리는 메모리내에 로드된 후에 수정 사항이 발생되었는지 확인 하도록 하는 `dirty` 비트를 포함하고 있다.

procfs 를 위한 루트의 생성
--------------------------------------------------------------------------------

모든 이 준비들이 끝나고 [proc](http://en.wikipedia.org/wiki/Procfs) 파일 시스템을 위한 루트를 생성할 필요가 있다. [fs/proc/root.c](https://github.com/torvalds/linux/blob/master/fs/proc/root.c) 에 있는 `proc_root_init` 의 호출을 볼 수 있다. `proc_root_init` 함수의 시작에는 inode를 위해 캐쉬를 할당하고 시스템에 새로운 파일 시스템을 등록한다.:

```C
err = register_filesystem(&proc_fs_type);
      if (err)
                return;
```

여기서는 [VFS](http://en.wikipedia.org/wiki/Virtual_file_system)에 관련해서 더 자세히 살펴보진 않을 것이지만 `VFS` 관한 다른 챕터에서 볼 것이다. 우리 시스템 내에 새로운 파일시스템을 등록한 이 후에, [fs/proc/self.c](https://github.com/torvalds/linux/blob/master/fs/proc/self.c) 파일에 있는 `proc_self_init` 함수를 호출하여 `self`(`/proc/self` 디렉토리는 `/proc` 파일시스템을 접근하여 프로세스 정보를 볼 수 있다.)을 위한 `inode` 번호를 할당한다. `proc_self_init` 함수 이후에 다음 단계는 현재 쓰레드에 관련된 정보를 포함하는 `/proc/thread-self` 설정하는 `proc_setup_thread_self` 함수이다. 이 다음에는 아래의 함수 호출로 마운트 포인트를 갖고 있는 `/proc/self/mounts` 심볼릭 링크를 생성한다.:

```C
proc_symlink("mounts", NULL, "self/mounts");
```

그리고 다른 구성 옵션들에 의존적인 디렉토리들을 만든다.:

```C
#ifdef CONFIG_SYSVIPC
        proc_mkdir("sysvipc", NULL);
#endif
        proc_mkdir("fs", NULL);
        proc_mkdir("driver", NULL);
        proc_mkdir("fs/nfsd", NULL);
#if defined(CONFIG_SUN_OPENPROMFS) || defined(CONFIG_SUN_OPENPROMFS_MODULE)
        proc_mkdir("openprom", NULL);
#endif
        proc_mkdir("bus", NULL);
        ...
        ...
        ...
        if (!proc_mkdir("tty", NULL))
                 return;
        proc_mkdir("tty/ldisc", NULL);
        ...
        ...
        ...
```

`proc_root_init` 함수의 끝에는 `/proc/sys` 디렉토리를 생성하고 [Sysctl](http://en.wikipedia.org/wiki/Sysctl) 를 초기화하는 `proc_sys_init` 함수 호출한다.

이제 `start_kernel` 함수의 끝이다. `start_kernel` 함수 내에 있는 모든 함수들을 기술하진 않았다. 일반적인 커널 초기화를 위한 중요한 부분이 아니거나 단지 커널 구성 옵션에 의존적인 내용을 넘어 갔다. 그것들은 태스크당 통계를 유저 영역에 전달하는 `taskstats_init_early` 함수, 태스크당 지연 acounting 관련 초기화하는 `delayacct_init`, `key_init` 와 `security_init`는 다른 보안관련 것들을 초기화, 몇 몇의 아키텍처 의존적인 버그를 수정하는 `check_bugs`, [ftrace](https://www.kernel.org/doc/Documentation/trace/ftrace.txt) 의 초기화를 수행하는 `ftrace_init`, [cgroup](http://en.wikipedia.org/wiki/Cgroups) 서브 시스템의 초기화를 하는 `cgroup_init` 등이 있다. 많은 부분들은 다른 챕터에서 기술될 것이다.

마침내 길고 길었던 `start_kernel` 함수가 끝이 났다. 그러나 이것이 리눅스 커널 초기화 과정의 모두가 끝난것이 아니다. 아직 첫 프로세스가 시작되지 않았다. `start_kernel` 의 끝에는 마지막으로 `rest_init` 함수의 호출을 볼 수 있다. 계속 보자.

start_kernel 이후에 첫 단계
--------------------------------------------------------------------------------

`rest_init` 함수는 `start_kernel` 함수의 구현부가 있는 같은 소스 파일인 [init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c) 에 있다. `rest_init` 함수으 시작은 아래 두 함수의 호출로 된다.:

```C
	rcu_scheduler_starting();
	smpboot_thread_init();
```

첫 `rcu_scheduler_starting` 함수는 [RCU](http://en.wikipedia.org/wiki/Read-copy-update) 스케줄러를 활성화하고 두 번째 `smpboot_thread_init` 함수는 `smpboot_thread_notifier` CPU notifier 를 등록한다.([CPU hotplug documentation](https://www.kernel.org/doc/Documentation/cpu-hotplug.txt) 에서 조금 더 자세히 보자.) 이것 이후에는 아래의 호출을 볼 수 있다.:

```C
kernel_thread(kernel_init, NULL, CLONE_FS);
pid = kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES);
```

여기서 `kernel_thread` 함수([kernel/fork.c](https://github.com/torvalds/linux/blob/master/kernel/fork.c)에 구현됨)는 새로운 커널 쓰레드를 만든다. `kernel_thread` 함수는 3개의 인자를 받는다.:

* 새로운 쓰레드 내에서 실행될 함수
* `kernel_init` 함수를 위한 인자
* 플래그

`kernel_thread` 함수 구현에 대해서는 자세히 살펴보지 않을 것이다.(스케줄러 파트에서 다룰 것이고, 지금은 단지 `kernel_thread` 는 [clone](http://www.tutorialspoint.com/unix_system_calls/clone.htm)을 수행한다는 정도만 알고 넘어가자.) 이제 `kernel_thread` 함수로 새로운 커널 쓰레드를 생성했다. 그러면 쓰레드의 부모와 자식은 파일 시스템에 관련해서는 정보를 공유할 것이고 `kernel_init` 함수를 시작할 것이다. 커널 쓰레드는 커널 모드에서 수행되기 때문에 사용자 쓰레드와는 다르다. 여기서 두 개의 `kernel_thread` 호출로 새로운 커널 쓰레드를 만드며 `init` 프로세스를 위해 `PID = 1` 로 할당하고 `kthreadd`를 위해 `PID = 2`를 할당한다. 우리는 이미 `init` 프로세스가 무엇인지 않다. `kthreadd`를 살펴 보자. 이 특별한 커널 쓰레드는 커널 쓰레드를 생성하고 위해 커널의 다른 부분들을 지원하고 관리한다. `ps` 유틸리티의 출력으로 그것을 확인해 볼 수 있다.:

```C
$ ps -ef | grep kthread
root         2     0  0 Jan11 ?        00:00:00 [kthreadd]
```

`kernel_init` 와 `kthreadd` 를 잠시 제쳐 두고 `rest_init`의 내용을 계속 보자. 새로운 커널 쓰레드 두 개를 생성한 뒤의 다음 단계는 아래와 같다.:

```C
	rcu_read_lock();
	kthreadd_task = find_task_by_pid_ns(pid, &init_pid_ns);
	rcu_read_unlock();
```

`rcu_read_lock` 함수는 [RCU](http://en.wikipedia.org/wiki/Read-copy-update) 의 읽기 임계 영역의 진입을 뜻하고 `rcu_read_unlock` 은 마지막을 알려준다. 이 함수들이 호출되는 이유는 `find_task_by_pid_ns` 호출의 보호가 필요하기 때문이다. `find_task_by_pid_ns` 함수는 주어진 Pid 에 맞는 `task_struct` 구조체의 포인터를 반환한다. 그래서, 여기 보면 `PID = 2`를 위한 `task_struct` 의 포인터를 얻어온다.(이것은 `kernel_thread`로 생성된 `kthreadd` 이다.) 다음 단계로는 `complete` 함수의 호출이다.:

```C
complete(&kthreadd_done);
```

그리고 `kthreadd_done` 의 주소를 넘긴다. `kthreadd_done` 은 아래와 같이 선언되어 있다.:

```C
static __initdata DECLARE_COMPLETION(kthreadd_done);
```

`DECLARE_COMPLETION` 매크로는 아래로 선언되어 있고:

```C
#define DECLARE_COMPLETION(work) \
         struct completion work = COMPLETION_INITIALIZER(work)
```

`completion` 구조체의 선언으로 확장된다. 이 구조체는 [include/linux/completion.h](https://github.com/torvalds/linux/blob/master/include/linux/completion.h) 에 선언되어 있고 `completions` 의 개념을 나타낸다. Completion(완료)는 쓰레들들이 특정한 상태나 특정한 위치에 도달할 때까지 기다리도록 하는 경쟁 상태에서 해방될 수 있도록 하는 코드 동기화 매커니즘이다. completion을 사용하기 위한 3 가지 부분이 있다.: 첫 번째는 `complete` 구조체의 선언이고 `DECLARE_COMPLETION` 매크로를 이용해서 할 것이다. 두 번째는 `wait_for_completion` 함수의 호출이다. 이 함수가 호출되고 나면, 그것을 호출한 쓰레드는 실행을 계속 하지 않고 다른 쓰레드에서 `complete` 함수를 호출 할때 까지 기다릴 것이다. `kernel_init_freeable` 함수 초반에 `kthreadd_done`의 주소를 인자로 하는 `wait_for_completion`를 호출 할 것이다.:

```C
wait_for_completion(&kthreadd_done);
```

그리고 마지막 단계로는 `complete` 함수를 호출 하는 것이다. 이후에 `kernel_init_freeable` 함수는 `kthreadd` 쓰레드가 설정되지 않는다면 실행되지 않을 것이다. `kthreadd` 가 설정되고 나면, `rest_init` 함수내의 다음 함수들의 호출을 볼 수 있다.:

```C
	init_idle_bootup_task(current);
	schedule_preempt_disabled();
    cpu_startup_entry(CPUHP_ONLINE);
```

[kernel/sched/core.c](https://github.com/torvalds/linux/blob/master/kernel/sched/core.c) 에 구현된 `init_idle_bootup_task` 함수는 현재 프로세스를 위해 스케줄링 클래스를 설정한다.(우리의 경우 `idle` 클래스 이다.):

```C
void init_idle_bootup_task(struct task_struct *idle)
{
         idle->sched_class = &idle_sched_class;
}
```

`idle` 클래스는 낮은 태스크 우선순위이며 이 태스크는 프로세서에서 아무것도 실행되고 있지 않을 때만 실행된다. 두 번째 함수는 `schedule_preempt_disabled` 로 `idle` 태스크내에 선점을 비활성화 한다. 3 번째 함수는 [kernel/sched/idle.c](https://github.com/torvalds/linux/blob/master/sched/idle.c) 내에 구현된 `cpu_startup_entry` 함수와 같은 소스 파일에 구현된 `cpu_idle_loop` 함수 호출이다. `cpu_idle_loop` 함수는 `PID = 0` 의 값을 가진 프로세스이고 백그라운드에서 실행된다. `cpu_idle_loop` 함수의 주요 목적은 idle CPU 사이클을 소모한다. 수행되는 프로세스가 없을 때, 이 프로세스는 실행된다. 우리는 `idle` 스케줄링 클래스를 가진 하나의 프로세스만 수행되고 있고(`init_idle_bootup_task` 함수의 호출로 `current` 태스크를 `idle` 로 설정한다.) `idle` 쓰레드는 유용한 작업을 하는 것은 아니지만 만약 수행할 수 있는 태스크가 확인이되면 그 태스크로 전환한다.:

```C
static void cpu_idle_loop(void)
{
        ...
        ...
        ...
        while (1) {
                while (!need_resched()) {
                ...
                ...
                ...
                }
        ...
        }
```

스케줄러에 관한 챕터를 보면 더 자세히 알 수 있다. `start_kernel` 함수가 `rest_init` 함수를 호출하는 시점에 `init`(`kernel_init` 함수) 프로세를 생성하고 그 프로세스는 `idle` 프로세스가 된다. 이제 `kernel_init` 을 볼 시간이다. `kernel_init` 함수의 실행은 `kernel_init_freeable` 함수의 호출로 부터 시작한다. `kernel_init_freeable` 함수는 `kthreadd` 설정이 완료될 때까지(completion) 기다린다.:

```C
wait_for_completion(&kthreadd_done);
```

이 다음에는 `gfp_allowed_mask` 값을 시스템이 이미 수행중이다는 의미의 `__GFP_BITS_MASK` 값으로 설정하여, 모든 CPU 들을 [cpus/mems](https://www.kernel.org/doc/Documentation/cgroups/cpusets.txt) 에 허가된 것으로 설정하고 `set_mems_allowed` 함수를 통해 [NUMA](http://en.wikipedia.org/wiki/Non-uniform_memory_access) 노드를 설정, `set_cpus_allowed_ptr` 함수와 함께 어떤 CPU 든 `init` 프로세스의 수행을 허가하고, `Ctrl-Alt-Delete` 혹은 `cad` 를 위한 pid 설정, `smp_prepare_cpus` 함수의 호출로 다른 CPU 들의 부팅을 위해 준비를 하고, `do_pre_smp_initcalls` 함수로 초기 [initcalls](http://kernelnewbies.org/Documents/InitcallMechanism) 호출하고, `smp_init` 함수로 `SMP` 를 초기화 하고 `sched_init_smp` 함수로 스케줄러를 초기화한다.

이 모든 것들이 끝나고 `do_basic_setup` 함수의 호출을 볼 수 있다. `do_basic_setup` 함수가 호출되기 전에 우리의 커널은 이미 초기화 되어 있는 것이다. 함수의 주석을 보면:

```
Now we can finally start doing some real work..
```

The `do_basic_setup` will reinitialize [cpuset](https://www.kernel.org/doc/Documentation/cgroups/cpusets.txt) to the active CPUs, initialize the `khelper` - which is a kernel thread which used for making calls out to userspace from within the kernel, initialize [tmpfs](http://en.wikipedia.org/wiki/Tmpfs), initialize `drivers` subsystem, enable the user-mode helper `workqueue`  and make post-early call of the `initcalls`. We can see opening of the `dev/console` and dup twice file descriptors from `0` to `2` after the `do_basic_setup`:
`do_basic_setup` 함수는 [cpuset](https://www.kernel.org/doc/Documentation/cgroups/cpusets.txt) 을 활동중인(active) CPU들을 위해 재 초기화할 것이고, 커널 내에서 사용자 영역으로 호출을 하기 위해 사용되는 커널 쓰레드인 `khelper`를 초기화, [tmpfs](http://en.wikipedia.org/wiki/Tmpfs) 초기화, `drivers` 서브시스템 초기화, 사용자 모드 helper 인 `workqueue` 활성화 그리고 `initcalls` 의 조기 호출을 한다. `do_basic_setup` 함수 이후에 `dev/console` 를 여는 것을 볼 수 있고, 그 파일 디스크립터를 `0` 에서 `2` 까지 복사한다.(dup)


```C
if (sys_open((const char __user *) "/dev/console", O_RDWR, 0) < 0)
	pr_err("Warning: unable to open an initial console.\n");

(void) sys_dup(0);
(void) sys_dup(0);
```

여기서 두 개의 시스템 콜인 `sys_open` 와 `sys_dup` 호출을 볼 수 있다. 초기 콘솔이 열린 이후에, 커널 명령라인으로 부터 받은 `rdinit=` 를 확인하거나 램디스크의 기본 경로를 설정한다.:

```C
if (!ramdisk_execute_command)
	ramdisk_execute_command = "/init";
```

`ramdisk` 를 위한 사용자 권한을 확인하고 [init/do_mounts.c](https://github.com/torvalds/linux/blob/master/init/do_mounts.c) 에 구현된 [initrd](http://en.wikipedia.org/wiki/Initrd)를 확인하고 마운트하는 `prepare_namespace` 함수를 호출한다.:

```C
if (sys_access((const char __user *) ramdisk_execute_command, 0) != 0) {
	ramdisk_execute_command = NULL;
	prepare_namespace();
}
```

이것은 `kernel_init_freeable` 함수의 끝이고 `kernel_init` 에서 복귀할 필요가 있다. `kernel_init_freeable` 함수가 마무리되고 다음 실행은 `async_synchronize_full` 함수이다. 이 함수는 모든 동기화 함수의 호출이 완료될 때까지 기다리고 그게 끝나면 초기화 하는 과정(`__init_begin` 와 `__init_end` 사이에 위치함)에서 점유한 모든 메모리를 해제하는 `free_initmem` 호출한다. 이 다음에는 `mark_rodata_ro` 함수로 `.rodata`를 보호하고 시스템 상태를 `SYSTEM_BOOTING` 에서 `SYSTEM_RUNNING` 로 업데이트 한다.

```C
system_state = SYSTEM_RUNNING;
```

그리고 `init` 프로세스를 실행하려고 시도한다.:

```C
if (ramdisk_execute_command) {
	ret = run_init_process(ramdisk_execute_command);
	if (!ret)
		return 0;
	pr_err("Failed to execute %s (error %d)\n",
	       ramdisk_execute_command, ret);
}
```

`kernel_init_freeable` 함수내에서 설정한 `ramdisk_execute_command` 를 확인하고 `rdinit=` 커널 명령 라인 인자들과 값이 같게 하고나 `/init`을 기본으로 설정한다. `run_init_process` 한수는 `argv_init` 배열의 첫 번째 항목을 채운다.:

```C
static const char *argv_init[MAX_INIT_ARGS+2] = { "init", NULL, };
```

`init` 프로그램의 인자들을 나타내고, `do_execve` 함수를 호출한다.:

```C
argv_init[0] = init_filename;
return do_execve(getname_kernel(init_filename),
	(const char __user *const __user *)argv_init,
	(const char __user *const __user *)envp_init);
```

[include/linux/sched.h](https://github.com/torvalds/linux/blob/master/include/linux/sched.h) 에 구현되어 있는 `do_execve` 함수이고 주어진 파일 이름과 인자들고 프로그램을 수행한다. 만약 커널 명령라인으로 `rdinit=` 옵션을 넘겨주지 않았다면, 커널은 `init=` 커널 명령라인 인자의 값과 같은 `execute_command` 를 확인한다.:

```C
	if (execute_command) {
		ret = run_init_process(execute_command);
		if (!ret)
			return 0;
		panic("Requested init %s failed (error %d).",
		      execute_command, ret);
	}
```

만약 커널 명령라인으로 `init=`를 넘겨주지 않았다면, 커널은 아래와 같이 실행가능한 파일들 중 하나를 실행하려고 한다.:

```C
if (!try_to_run_init_process("/sbin/init") ||
    !try_to_run_init_process("/etc/init") ||
    !try_to_run_init_process("/bin/init") ||
    !try_to_run_init_process("/bin/sh"))
	return 0;
```

그렇지 않으면, [panic](http://en.wikipedia.org/wiki/Kernel_panic) 과 함께 시스템을 멈춘다.:

```C
panic("No working init found.  Try passing init= option to kernel. "
      "See Linux Documentation/init.txt for guidance.");
```

이제 끝이다! 리눅스 커널 초기화 과정이 마무리 되었다!!

결론
--------------------------------------------------------------------------------

리눅스 커널 초기화 과정의 10 번째 파트가 마무리 되었다. 10 번째 파트이자 리눅스 커널 초기화를 기술하는 마지막 파트이다. 이 챕터의 첫 번째 파트에서 부터 커널 초기화 모든 과정을 설명하려고 했고, 완료하였다. 우리는 아키텍처 독립적인 함수인 `start_kernel`에서 시작하였고 첫 `init` 프로세스가 실행함으로 마무리하였다. 나는 커널의 몇몇 서브 시스템에 관한 상세를 그냥 넘겼는데, 예를 들면 스케줄러, 인터럽트, 예외 처리 등에서 다루지 못했던 것들이다. 다음 파트 부터는 커널의 서브 시스템중 하나를 더 자세히 살펴 보도록 하려한다

어떤 질문이나 제안이 있다면, twitter [0xAX](https://twitter.com/0xAX), [email](anotherworldofworld@gmail.com) 또는 [issue](https://github.com/0xAX/linux-insides/issues/new) 를 만들어 주길 바란다.

**나는 영어권의 사람이 아니고 이런 것에 대해 매우 미안해 하고 있다. 만약 어떤 실수를 발견한다면, 나에게 PR을 [linux-insides](https://github.com/0xAX/linux-internals)을 보내줘**

Links
--------------------------------------------------------------------------------

* [SLAB](http://en.wikipedia.org/wiki/Slab_allocation)
* [xsave](http://www.felixcloutier.com/x86/XSAVES.html)
* [FPU](http://en.wikipedia.org/wiki/Floating-point_unit)
* [Documentation/security/credentials.txt](https://github.com/torvalds/linux/blob/master/Documentation/security/credentials.txt)
* [Documentation/x86/x86_64/mm](https://github.com/torvalds/linux/blob/master/Documentation/x86/x86_64/mm.txt)
* [RCU](http://en.wikipedia.org/wiki/Read-copy-update)
* [VFS](http://en.wikipedia.org/wiki/Virtual_file_system)
* [inode](http://en.wikipedia.org/wiki/Inode)
* [proc](http://en.wikipedia.org/wiki/Procfs)
* [man proc](http://linux.die.net/man/5/proc)
* [Sysctl](http://en.wikipedia.org/wiki/Sysctl)
* [ftrace](https://www.kernel.org/doc/Documentation/trace/ftrace.txt)
* [cgroup](http://en.wikipedia.org/wiki/Cgroups)
* [CPU hotplug documentation](https://www.kernel.org/doc/Documentation/cpu-hotplug.txt)
* [completions - wait for completion handling](https://www.kernel.org/doc/Documentation/scheduler/completion.txt)
* [NUMA](http://en.wikipedia.org/wiki/Non-uniform_memory_access)
* [cpus/mems](https://www.kernel.org/doc/Documentation/cgroups/cpusets.txt)
* [initcalls](http://kernelnewbies.org/Documents/InitcallMechanism)
* [Tmpfs](http://en.wikipedia.org/wiki/Tmpfs)
* [initrd](http://en.wikipedia.org/wiki/Initrd)
* [panic](http://en.wikipedia.org/wiki/Kernel_panic)
* [kmem_cache_init 설명](http://jake.dothome.co.kr/kmem_cache_init/)
* [zone type](http://jake.dothome.co.kr/zone-types/)
