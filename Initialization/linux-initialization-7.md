커널 초기화. Part 7.
================================================================================

아키텍처 의존적인 초기화의 거의(?) 마지막
================================================================================

리눅스 커널 초기화 과정의 7번째 파트이고 [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/setup.c#L861) 의 `setup_arch`의 내부를 다루어 볼 것이다. 이전 [파트](https://github.com/daeseokyoun/linux-insides/blob/master/Initialization/linux-initialization-6.md) 에서 `setup_arch` 함수의 커널 코드/데이터/bss 영역을 위한 메모리 예약, [Desktop Management Interface](http://en.wikipedia.org/wiki/Desktop_Management_Interface) 의 초기 스캐닝, [PCI](http://en.wikipedia.org/wiki/PCI) 장치의 초기 덤프 등의 아키텍처 의존적인(우리의 경우 `x86_64` 아키텍처) 초기화를 살펴보았다. 만약 이전 파트를 읽었다면, 우리가 `setup_real_mode` 함수에서 마무리되었다는 것을 알 것이다. 다음 단계에서는 [memblock](http://0xax.gitbooks.io/linux-insides/content/mm/linux-mm-1.html)의 제한을 모든 맵핑된 모든 페이지에게 설정하고, [kernel/printk/printk.c](https://github.com/torvalds/linux/blob/master/kernel/printk/printk.c)에 있는 `setup_log_buf` 함수의 호출을 볼 것이다.

`setup_log_buf` 함수는 커널 순환 버퍼와 `CONFIG_LOG_BUF_SHIFT` 구성 옵션에 따라 크기를 설정한다. `CONFIG_LOG_BUF_SHIFT` 관련된 문서를 읽어보면, 그것은 `12`와 `21` 사이의 값이 된다. 내부적으로, buffer 는 char 타입의 배열로 선언된다.:

```C
#define __LOG_BUF_LEN (1 << CONFIG_LOG_BUF_SHIFT)
static char __log_buf[__LOG_BUF_LEN] __aligned(LOG_ALIGN);
static char *log_buf = __log_buf;
```

이제 `setup_log_buf` 함수의 구현을 살펴보자. 그것은 현재 버퍼가 비어 있는지 확인하는 것부터 시작하고(버퍼는 반드시 비어 있는 상태여야 한다, 왜냐면 지금 막 설정했기 때문이다.) 초기 설정인지 아닌지 검사한다. 만얀 커널 로그 버퍼의 설정이 초기(early)가 아니라면, 모든 CPU 를 위한 버퍼의 크기를 증가시키는 `log_buf_add_cpu` 함수를 호출한다.:

```C
if (log_buf != __log_buf)
    return;

if (!early && !new_log_buf_len)
    log_buf_add_cpu();
```

`log_buf_add_cpu` 함수에 대해서는 더 조사 하지는 않는다. 이유는 `setup_arch`내에서는 아래와 같이 `setup_log_buf`를 호출하여 사용하기 때문이다.:

```C
setup_log_buf(1);
```

`1` 은 초기 설정이라는 의미이다. 다음 단계는 `new_log_buf_len` 변수의 값이 커널 로그 버퍼의 길이를 업데이트 했는지 확인하고 버퍼를 위한 새로운 공간을 할당하기 위해 `memblock_virt_alloc` 함수를 사용한다. 업데이트 안되었다면 함수를 그냥 종료한다.

커널 로그 버퍼가 준비되면, 다음 호출 함수는 `reserve_initrd` 이다. 우리는 이미 [커널 초기화 part 4](https://github.com/daeseokyoun/linux-insides/blob/master/Initialization/linux-initialization-4.md)에서 `early_reserve_initrd` 의 호출에 관해 살펴 보았다. 이제, `init_mem_mapping` 함수에서 직접 메모리 맵핑 영역(direct memeory mapping)을 재구성하고, [initrd](http://en.wikipedia.org/wiki/Initrd)를 직접적으로 맵핑된 메모리 영역으로 옮길 필요가 있다. `reserve_initrd` 함수는 `initrd` 의 시작과 끝 주소를 정의하는 것부터 시작하고 부트로더에서 제공된 `initrd` 인지 확인한다. `early_reserve_initrd` 함수에서 봤던 내용과 똑같다. 하지만 `memblock_reserve` 함수의 호출로 `memblock` 영역내에 공간을 예약하는 대신에 직접 메모리 영역(direct memory area) 에 맵핑된 크기를 얻고 `initrd` 의 크기가 이 영역보다 크지 않는지 아래의 코드로 확인한다.:

```C
mapped_size = memblock_mem_size(max_pfn_mapped);
if (ramdisk_size >= (mapped_size>>1))
    panic("initrd too large to handle, "
	      "disabling initrd (%lld needed, %lld available)\n",
	      ramdisk_size, mapped_size>>1);
```

여기서 `memblock_mem_size` 함수의 호출과 그 함수의 인자로 직접 맵핑 영역(direct mapped area)의 가장 높은 페이지 프레임 번호인 `max_pfn_mapped` 를 넘기는 걸 확인 할 수 있다. 만약 `page frame number(페이지 프레임 번호)` 가 무엇이었는지 기억나지 않는다면, 설명은 간단하다.: 첫 가상 주소에서 첫 `12` 비트는 물리적 페이지 혹은 페이지 프레임내의 오프셋을 표현한다. 만약 이 가상 주소의 `12`를 우측으로 쉬프트를 끝까지 한다면, 오프셋 부분은 버릴 것이고 `페이지 프레임 번호`를 얻을 수 있을 것이다. `memblock_mem_size` 함수에서는 모든 memblock `mem`(예약 되지 않은 영역) 영역을 확인하고 맵핑괸 페이지들의 크기를 계산한 뒤, `mapped_size` 변수에 반환한다.(아래에서 살펴 보자.) 직접 맵핑된 메모리(direct mapped memory)의 총 양을 알았으니, `initrd` 의 크기가 맵핑된 페이지들 보다 크지 않은지 확인한다. 만약 그것이 크다면, `panic`을 호출하여 유명한 [Kernel panic](http://en.wikipedia.org/wiki/Kernel_panic) 메세지와 함께 시스템이 멈출 것이다.  다음 단계에서 `initrd` 크기에 관련된 정보를 출력한다. `dmesg`의 출력으로 이 사항을 확인 할 수 있다.:

```C
[0.000000] RAMDISK: [mem 0x36d20000-0x37687fff]
```

그리고 `relocate_initrd` 함수를 통해 `initrd`를 직접 맵핑 영역(direct mapping area)로 재배치 한다. `relocate_initrd` 함수의 시작은 `memblock_find_in_range` 함수를 통해 여유 공간을 찾는 것부터 한다.:

```C
relocated_ramdisk = memblock_find_in_range(0, PFN_PHYS(max_pfn_mapped), area_size, PAGE_SIZE);

if (!relocated_ramdisk)
    panic("Cannot find place for new RAMDISK of size %lld\n",
	       ramdisk_size);
```

`memblock_find_in_range` 함수는 주어진 영역내에서 여유 공간을 찾으려고 시도한다, 우리의 경우에는 `0` 에서 최대 매핑 물리 주소까지 이고, 할당 해야 하는 공간은 `initrd` 의 정렬된(aligned) 크기와 같아야 한다. 만약 적절한 크기의 영역을 찾지 못한다면, `panic`을 호출 하고, 모두 좋다면, 다음 단계에서 램 디스크를 직접 팹핑된 영역의 아래쪽에 재배치를 시작할 것이다.

`reserve_initrd` 함수의 끝 부분에는, 아래의 호출로 램디스크가 차지하고 있는 memblock 메모리를 해제한다.:

```C
memblock_free(ramdisk_image, ramdisk_end - ramdisk_image);
```

`initrd` 램 디스크를 재배치 한 후에, 다음 호출되는 함수는 [arch/x86/kernel/vsmp_64.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/vsmp_64.c) 에 구현된 `vsmp_init` 이다. 이 함수는 `ScaleMP vSMP` 의 지원을 위해 초기화한다. 이미 언급했지만, `x86_64` 초기화 파트와 관련없는 내용은 다루지 않을 것이다.(예를 들면, `ACPI` 등) 그래서 이 함수의 구현부는 지금은 넘어갈 것이고 병렬 컴퓨팅의 기술을 다루는 파트에서 다시 한번 살펴 보도록 하자.

다음 함수는 [arch/x86/kernel/io_delay.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/io_delay.c) 에 있는 `io_delay_init` 이다. 이 함수는 기본 I/O 지연 `0x80` 포트를 오버라이드를 가능하게 한다. [보호 모드로 전환전에 마지막 준비 작업](https://github.com/daeseokyoun/linux-insides/blob/korean-trans/Booting/linux-bootstrap-3.md) 에서 이미 I/O 지연에 관해 보았고, 이제 `io_delay_init` 구현을 살펴 보자.:

```C
void __init io_delay_init(void)
{
    if (!io_delay_override)
        dmi_check_system(io_delay_0xed_port_dmi_table);
}
```

이 함수는 `io_delay_override` 변수를 확인하여, 만약 `io_delay_override` 가 설정되어 있다면, I/O 지연 포트를 오버라이드 한다. 커널 명령 라인에서 `io_delay` 옵션을 전달함으로써 `io_delay_override`를 설정할 수 있다. [Documentation/kernel-parameters.txt](https://github.com/torvalds/linux/blob/master/Documentation/kernel-parameters.txt) 을 읽어 본다면, `io_delay` 옵션은 다음과 같다.:

```
io_delay=	[X86] I/O delay method I/O 지연 방법
    0x80
        Standard port 0x80 based delay 표준 포트 0x80 기반 지연
    0xed
        Alternate port 0xed based delay (needed on some systems) 대체의 포트 0xed 기반 지연
    udelay
        Simple two microseconds delay 단지 2 마이크로초 지연
    none
        No delay 지연 없음.
```

[arch/x86/kernel/io_delay.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/io_delay.c) 에서 `early_param` 매크로를 이용하여 `io_delay` 명령 라인 파라미터를 설정할 수 있다.

```C
early_param("io_delay", io_delay_param);
```

이전 [파트](https://github.com/daeseokyoun/linux-insides/blob/master/Initialization/linux-initialization-6.md) 에서 `early_param` 에 대한 내용을 더 볼 수 있다. 즉, `io_delay_override` 변수를 설정하는 `io_delay_param` 함수는 [do_early_param](https://github.com/torvalds/linux/blob/master/init/main.c#L413) 함수에서 호출 될 것이다. `io_delay_param` 함수는 `io_delay` 커널 명령 라인 파라미터의 인자로 받을 것이고, 받은 값으로 부터 `io_delay_type`의 설정을 한다.:

```C
static int __init io_delay_param(char *s)
{
        if (!s)
                return -EINVAL;

        if (!strcmp(s, "0x80"))
                io_delay_type = CONFIG_IO_DELAY_TYPE_0X80;
        else if (!strcmp(s, "0xed"))
                io_delay_type = CONFIG_IO_DELAY_TYPE_0XED;
        else if (!strcmp(s, "udelay"))
                io_delay_type = CONFIG_IO_DELAY_TYPE_UDELAY;
        else if (!strcmp(s, "none"))
                io_delay_type = CONFIG_IO_DELAY_TYPE_NONE;
        else
                return -EINVAL;

        io_delay_override = 1;
        return 0;
}
```

`io_delay_init` 이후에 호출되는 함수들은 `acpi_boot_table_init`, `early_acpi_boot_init` 와 `initmem_init` 이다. 하지만, 이미 언급했듯이 `리눅스 커널 초기화 과정` 챕터에서는 [ACPI](http://en.wikipedia.org/wiki/Advanced_Configuration_and_Power_Interface)에 관련된 내용은 다루지 않을 것이다.

DMA 를 위한 영역 할당
--------------------------------------------------------------------------------

다음 단계는 [drivers/base/dma-contiguous.c](https://github.com/torvalds/linux/blob/master/drivers/base/dma-contiguous.c)에 구현된 `dma_contiguous_reserve` 함수의 호출로 [직접 메모리 접근-Direct memory access](http://en.wikipedia.org/wiki/Direct_memory_access)를 위한 영역을 할당하는 일이다. `DMA`는 CPU 가 관여하지 않고 장치가 메모리와 통신하기 위한 특별한 모드이다. `dma_contiguous_reserve` 에 `max_pfn_mapped << PAGE_SHIFT` 라는 값의 하나의 인자가 주어지고, 함수의 이름에서 알 수 있듯이 예약된 메모리의 제한을 설정하는 것이다. 이 함수의 구현을 살펴 보자. 아래의 변수들의 선언으로 부터 시작된다.:

```C
phys_addr_t selected_size = 0;
phys_addr_t selected_base = 0;
phys_addr_t selected_limit = limit;
bool fixed = false;
```

첫 번째 변수는 예약된 공간의 크기(바이트), 두 번째는 예약된 공간의 시작 주소, 세 번째는 예약된 공간의 끝 주소 그리고 마지막 `fixed` 는 이 예약된 공간이 어디에 위치 할 것인지 결정한다. 만약, `fixed` 가 `1`이면, `memblock_reserve` 로 공간을 예약하고, `0` 이라면 `kmemleak_alloc` 으로 공간을 할당한다. 다음 단계는 `size_cmdline` 변수를 확인하는 것인데, 만약 이 변수가 `-1`과 같지 않으면, 위에서 보여진 값들을 `cma` 커널 명령 라인 파라미터들로 부터 받아 설정한다.:

```C
if (size_cmdline != -1) {
   ...
   ...
   ...
}
```

같은 소스 파일에 초기 파라미터의 선언을 찾아 볼 수 있다.:

```C
early_param("cma", early_cma);
```

`cma` 커널 명령 라인은:

```
cma=nn[MG]@[start[MG][-end[MG]]]
		[ARM,X86,KNL]
		Sets the size of kernel global memory area for
		contiguous memory allocations and optionally the
		placement constraint by the physical address range of
		memory allocations. A value of 0 disables CMA
		altogether. For more information, see
		include/linux/dma-contiguous.h

    연속적인 메모리 할당을 위한 커널 글로벌 메모리 영역의 크기를 설정하고 추가적으로 메모리 할당의
    물리 주소 범위를 줘서 할당 위치를 설정할 수도 있다. 0 의 값은 CMA 를 비활성화 한다.
    더 많은 정보는 include/linux/dma-contiguous.h를 참조 바란다.
```

만약 커널 명령 라인으로 `cma` 옵션을 넘기지 않았다면, `size_cmdline` 는 `-1`과 같을 것이다. 커널 구성 옵션에 따라 예약된 영역의 크기를 계산 할 필요가 있다.:

* `CONFIG_CMA_SIZE_SEL_MBYTES` - 메가 바이크 크기, 기본 글로벌 `CMA` 영역, `CMA_SIZE_MBYTES * SZ_1M` 또는 `CONFIG_CMA_SIZE_MBYTES * 1M`
* `CONFIG_CMA_SIZE_SEL_PERCENTAGE` - 총 메모리의 백분율
* `CONFIG_CMA_SIZE_SEL_MIN` - 최소값을 사용
* `CONFIG_CMA_SIZE_SEL_MAX` - 최대값을 사용

예약된 영역의 크기를 계산했다면, `dma_contiguous_reserve_area` 함수의 호출로 영역을 예약한다.:

```
ret = cma_declare_contiguous(base, size, limit, 0, 0, fixed, res_cma);
```

`cma_declare_contiguous` 함수는 주어진 베이스 주소와 크기를 갖고 연속적인 공간을 예약한다. `DMA` 를 위한 영역을 예약하고 나면, `memblock_find_dma_reserve` 함수를 호출한다. 이름에서도 알 수 있듯이, 이 함수는 `DMA` 영역에 예약된 페이지들의 개수를 센다. 이 파트에서는 `CMA`와 `DMA`에 관련된 상세내용은 다루지 않는다. 리눅스 커널 메모리 관리를 위한 특별한 파트에서 조금 더 상세히 다루어 보도록 하자.

sparse 메모리 초기화
--------------------------------------------------------------------------------

다음 단계는 `x86_init.paging.pagetable_init` 함수의 호출이다. 만약 리눅스 커널 소스 코드에서 이 함수를 찾아 보려 한다면, 마침내는 아래와 같은 매크로를 볼 수 있을 것이다.:

```C
#define native_pagetable_init        paging_init
```

[arch/x86/mm/init_64.c](https://github.com/torvalds/linux/blob/master/arch/x86/mm/init_64.c) 에 구현된 `paging_init` 함수의 호출을 확장하는 매크로이다. `paging_init` 함수는 sparse 메모리와 zone 크기를 초기화한다. 먼저, zone 과 `Sparsemem` 이 무엇인지 알아보자. `Sparsemem` 은 [NUMA](http://en.wikipedia.org/wiki/Non-uniform_memory_access) 시스템에서 서로 다른 메모리 뱅크내에 메모리를 영역을 나누어 사용하도록 하는 리눅스 커널 메모리 모델이다. `paginig_init` 함수의 구현을 살펴 보자.:

```C
void __init paging_init(void)
{
        sparse_memory_present_with_active_regions(MAX_NUMNODES);
        sparse_init();

        node_clear_state(0, N_MEMORY);
        if (N_MEMORY != N_NORMAL_MEMORY)
                node_clear_state(0, N_NORMAL_MEMORY);

        zone_sizes_init();
}
```

위에서 보듯이 모든 `NUMA` 노드의 메모리 영역을 `struct page` 배열의 구조체에 대한 포인터를 포함하는 `mem_section` 구조체 배열에 기록하는 `sparse_memory_present_with_active_regions` 함수 호출이 있다. 다음은 `sparse_init` 함수인데, 연속적이지 않은 `mem_section` 와 `mem_map` 을 할당한다. 다음 단계로는 이동 가능한(movable) 메모리 노드들의 상태 클리어와 zone 들의 크기를 초기화 한다. 모든 `NUMA` 노드는 `zones` 라는 몇몇의 조각으로 나누어 질수 있다. 그래서 [arch/x86/mm/init.c](https://github.com/torvalds/linux/blob/master/arch/x86/mm/init.c) 에 있는 `zone_sizes_init` 함수는 zone 들의 크기를 초기화 한다.

이 파트와 다음 파트에서도 완전히 자세히 이와 관련된 내용을 살펴 보진 않을 것이다. `NUMA`에 관련된 특별한 파트를 준비 할 것이다.

vsyscall 맵핑
--------------------------------------------------------------------------------

`SparseMem` 초기화가 마무리되면, `cr4` [Control register](http://en.wikipedia.org/wiki/Control_register) 의 내용을 반드시 포함하게 하는 `trampoline_cr4_features` 의 설정을 해야 한다. 무엇보다도 CPU 가 `cr4` 레지스터의 지원이 가능하지 확인하고, 만약 그렇다면, real 모드에서 `cr4` 를 위한 저장소인 `trampoline_cr4_features` 에 그 내용을 저장한다.:

```C
if (boot_cpu_data.cpuid_level >= 0) {
    mmu_cr4_features = __read_cr4();
	if (trampoline_cr4_features)
	    *trampoline_cr4_features = mmu_cr4_features;
}
```

다음 호출되는 함수로는 [arch/x86/kernel/vsyscall_64.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/vsyscall_64.c)에 구현된 `map_vsyscal` 을 볼 수 있다. 이 함수를 포함한 소스코드는 `CONFIG_X86_VSYSCALL_EMULATION` 구성 옵션에 의해 빌드되고 [vsyscalls](https://lwn.net/Articles/446528/) 를 위한 메모리 공간을 맵핑한다. 실제 `vsyscall` 은 `getcpu` 와 같은 특정 시스템 콜에 빠른 접근을 제공하기 위한 특별한 세그먼트이다. 이 함수의 구현을 살펴 보자.:

```C
void __init map_vsyscall(void)
{
        extern char __vsyscall_page;
        unsigned long physaddr_vsyscall = __pa_symbol(&__vsyscall_page);

        if (vsyscall_mode != NONE)
                __set_fixmap(VSYSCALL_PAGE, physaddr_vsyscall,
                             vsyscall_mode == NATIVE
                             ? PAGE_KERNEL_VSYSCALL
                             : PAGE_KERNEL_VVAR);

        BUILD_BUG_ON((unsigned long)__fix_to_virt(VSYSCALL_PAGE) !=
                     (unsigned long)VSYSCALL_ADDR);
}
```

`map_vsyscall` 함수의 초반부에서 두 개의 변수 선언을 볼 수 있다. 첫 번째는 `extern` 변수인 `__vsyscall_page`이다. extern 변수는 다른 소스 코드 파일 어딘가에서 선언된 것을 가져오는 것이다. 실제 [arch/x86/kernel/vsyscall_emu_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/vsyscall_emu_64.S)에서 `__vsyscall_page`의 정의를 볼 수 있다. `__vsyscall_page` 심볼은 `vsyscalls` 의 정렬된 호출들을 `gettimeofday` 등으로 가리킨다.:

```assembly
	.globl __vsyscall_page
	.balign PAGE_SIZE, 0xcc
	.type __vsyscall_page, @object
__vsyscall_page:

	mov $__NR_gettimeofday, %rax
	syscall
	ret

	.balign 1024, 0xcc
	mov $__NR_time, %rax
	syscall
	ret
    ...
    ...
    ...
```

두 번째 변수는 `__vsyscall_page` 심볼의 물리 주소를 저장하는 `physaddr_vsyscall` 이다. 다음 단계에서는 `vsyscall_mode` 변수를 확인하여, `NONE` 가 아니라면, 기본값으로 `EMULATE` 을 가진다.:

```C
static enum { EMULATE, NATIVE, NONE } vsyscall_mode = EMULATE;
```

그리고 이 다음에는 같은 인자들로 `native_set_fixmap` 함수를 호출하는 `__set_fixmap` 호출을 볼 수 있다.:

```C
void native_set_fixmap(enum fixed_addresses idx, unsigned long phys, pgprot_t flags)
{
        __native_set_fixmap(idx, pfn_pte(phys >> PAGE_SHIFT, flags));
}

void __native_set_fixmap(enum fixed_addresses idx, pte_t pte)
{
        unsigned long address = __fix_to_virt(idx);

        if (idx >= __end_of_fixed_addresses) {
                BUG();
                return;
        }
        set_pte_vaddr(address, pte);
        fixmaps_set++;
}
```

여기서 주어진 물리 주소로 부터 `Page Table Entry(페이지 테이블 엔트리)` (우리의 경우에는 `__vsyscall_page` 심볼의 물리 주소가 될 것이다.) 의 값을 만드는 `native_set_fixmap` 를 볼 수 있고, 그 함수는 내부적으로 `__native_set_fixmap` 를 호출 한다. 그 호출은 주어진 `fixed_addresses` 인덱스(우리의 경우 `VSYSCALL_PAGE`)의 가상주소를 얻고 주어진 인덱스가 고정 매핑 주소(fix-mapped address)의 끝보다 큰지 확인 한다. 이것까지 진행하고 나면, `set_pte_vaddr` 함수의 호출로 페이지 테이블 엔트리를 설정하고 고정 맵핑 주소의 카운트를 증가시킨다. 그리고 `map_vsyscall` 의 마지막에는 `VSYSCALL_PAGE`(`fixed_addresses`의 첫 번째 인덱스)의 가상주소를 `-10UL << 20` 또는 `ffffffffff600000` 값을 가지는 `VSYSCALL_ADDR` 보다 큰지 `BUILD_BUG_ON` 매크로와 함께 확인한다.:

```C
BUILD_BUG_ON((unsigned long)__fix_to_virt(VSYSCALL_PAGE) !=
                     (unsigned long)VSYSCALL_ADDR);
```

이제 `vsyscall` 영역은 `fix-mapped(고정 맵핑)` 영역에 있다. 이것이 `map_vsyscall`에 관한 모든 것이다. 만약 고정-맵핑 주소에 관해 이해가 되지 않는다면, [고정-맵핑 주소와 ioremap](https://github.com/daeseokyoun/linux-insides/blob/master/mm/linux-mm-2.md) 를 읽어보길 바란다. `vsyscalls and vdso` 파트에서 `vsyscalls` 관해 더 살펴 볼 것이다.

SMP 구성 설정 가져오기
--------------------------------------------------------------------------------

이전 [파트](https://github.com/daeseokyoun/linux-insides/blob/master/Initialization/linux-initialization-6.md)에서 어떻게 [SMP](http://en.wikipedia.org/wiki/Symmetric_multiprocessing) 설정의 검색을 하는지 기억할 것이다. 이제 우리는 만약 찾았다면, `SMP` 설정을 얻을 필요가 있다. 이것을 위해 `smp_scan_config` 함수에서 설정한 `smp_found_config` 변수를 확인하고 `get_smp_config` 함수를 호출한다.:

```C
if (smp_found_config)
	get_smp_config();
```

`get_smp_config` 함수는 [arch/x86/kernel/mpparse.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/mpparse.c) 에 선언된 `x86_init.mpparse.default_get_smp_config` 함수를 호출한다. 이 함수는 인텔 MP 플로팅 포인터 구조체인 `mpf_intel` (이전 [파트](https://github.com/daeseokyoun/linux-insides/blob/master/Initialization/linux-initialization-6.md)에서 확인했을 것이다.)을 선언하고 몇가지 확인을 한다.:
```C
struct mpf_intel *mpf = mpf_found;

if (!mpf)
    return;

if (acpi_lapic && early)
   return;
```

여기서 멀리프로세서 구성을 `smp_scan_config` 함수내에서 찾을 수 있거나 만약 멀티프로세서가 아니라면 그냥 반환된다. 다음은 `acpi_lapic` 와 `early`을 확인하는 것이다. 그리고 이 확인이 정상적이라면, `SMP` 구성 설정을 읽을 수 있다. 다음 단계는 사용 가능한 CPU 들의 확인을 위해 `cpumask` 를 채우는 `prefill_possible_map` 함수이다. ([cpumasks 소개](https://github.com/daeseokyoun/linux-insides/blob/master/Concepts/cpumask.md)에서 더 많은 내용을 확인 하자.)

setup_arch 의 나머지 부분
--------------------------------------------------------------------------------

이제 `setup_arch` 함수의 막바지에 다달았다. 이 나머지 함수들은 아주 중요하지만, 자세한 사항은 이 파트에서 다루지 않을 것이다. 우리는 간단히 이 함수들을 살펴 볼 것이다. 이유는 이들은 중요하긴 하지만 일반적으로 다루어지는 커널 항목들이 아닌 `NUMA`, `SMP`, `ACPI` 그리고 `APICs`등과 연관된 것들이기 때문이다. 첫 번째로 다음으로 호출되는 함수는 `init_apic_mappings` 함수이다. 이 함수는 지역(local) [APIC](http://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller) 의 주소를 설정한다. 다음 호출은 `x86_io_apic_ops.init` 인데, 이 함수는 I/O APIC 를 초기화 한다. `APIC` 관련해서는 인터럽트와 예외 처리 챕터에서 조금 더 자세히 볼 수 있다. 다음 단계에서는 `DMA`, `TIMER`, `FPU` 등과 같이 표준 I/O 자원을 `x86_init.resources.reserve_resources` 함수를 통해 예약한다. 그 다음에는 `Machine check Exception` 을 초기화하는 `mcheck_init` 함수이고, 마지막으로는 [jiffy](http://en.wikipedia.org/wiki/Jiffy_%28time%29) 를 등록하는 `register_refined_jiffies` 함수이다. (커널의 타이머와 관련된 내용을 다루는 분리된 챕터를 만들것이다.)

마침내, 크디큰 `setup_arch` 함수를 이번 파트에서 마무리 지었다. 물론, 이 함수의 모든 내용을 다룬 것은 아니라는 점을 기억하자. 하지만 걱정말자. 이 함수에 대한 내용을 다른 챕터에서 다룰 예정이다.

이제 `setup_arch` 에서 `start_kernel` 로 돌아가보자.

main.c 로 복귀
================================================================================

이제 `setup_arch` 함수를 마무리 짓고, 이제 [init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c) 에 있는 `start_kernel` 함수로 돌아가자. 기억하겠지만, `start_kernel` 또한 `setup_arch` 만큼 큰 함수이다. 다음 몇 파트에서는 이 함수에서 일어나는 일들을 알아 볼 것이다. 이제 계속 진행해보자. `setup_arch` 함수 이후에, `mm_init_cpumask` 함수의 호출을 볼 수 있다. 이 함수는 [cpumask](https://github.com/daeseokyoun/linux-insides/blob/master/Concepts/cpumask.md) 포인터를 메모리 디스크립터의 `cpumask` 로 설정한다. 그것의 구현을 살펴 보자.:

```C
static inline void mm_init_cpumask(struct mm_struct *mm)
{
#ifdef CONFIG_CPUMASK_OFFSTACK
        mm->cpu_vm_mask_var = &mm->cpumask_allocation;
#endif
        cpumask_clear(mm->cpu_vm_mask_var);
}
```

이 함수는 [init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c) 에서 볼 수 있고, init 프로세스의 메모리 디스크립터 구조체를 `mm_init_cpumask` 함수로 넘겨주고 `CONFIG_CPUMASK_OFFSTACK` 구성 옵션의 선택에 따라 [TLB](http://en.wikipedia.org/wiki/Translation_lookaside_buffer) 를 클리어 할지를 결정한다.

다음 단계에서는 아래의 함수 호출을 볼 수 있다.:

```C
setup_command_line(command_line);
```

이 함수는 커널 명령 라인의 포인터를 받아 명령라인을 저장 하기 위한 몇개의 버퍼들을 할당한다. 그 중 하나는 미래에 참조와 명령라인 접근을 위해 만들어지고 다른 하나는 파라미터 파싱을 위해 사용된다. 다음과 같은 버퍼들을 위해 공간을 할당한다.:

* `saved_command_line` - 부트 명령 라인을 포함한다.
* `initcall_command_line` - 부트 명령 라인을 포함한다. `do_initcall_level`에서 사용될 것이다.
* `static_command_line` - 파라미터 파싱을 위해 명령 라인을 저장한다.

여기서 `memblock_virt_alloc` 함수를 통해 공간을 할당한다. 이 함수는 [slab](http://en.wikipedia.org/wiki/Slab_allocation) 을 사용 가능하지 않다면, `memblock_reserve` 함수를 통해 부트 메모리 블럭을 할당하는 `memblock_virt_alloc_try_nid` 함수를 호출하고, `slab` 이 사용 가능하다면, `kzalloc_node` 를 호출한다.(리눅스 메모리 관리 챕터에서 더 살펴보자.) `memblock_virt_alloc` 함수는 `BOOTMEM_LOW_LIMIT`(`(PAGE_OFFSET + 0x1000000)` 값의 물리주소) 와  `BOOTMEM_ALLOC_ACCESSIBLE`(`memblock.current_limit` 의 현재 값과 동일) 을 사용하여 메모리 영역의 최소 주소값과 최대값으로 사용한다.

`setup_command_line` 함수의 구현을 살펴보자.:

```C
static void __init setup_command_line(char *command_line)
{
        saved_command_line =
                memblock_virt_alloc(strlen(boot_command_line) + 1, 0);
        initcall_command_line =
                memblock_virt_alloc(strlen(boot_command_line) + 1, 0);
        static_command_line = memblock_virt_alloc(strlen(command_line) + 1, 0);
        strcpy(saved_command_line, boot_command_line);
        strcpy(static_command_line, command_line);
 }
 ```

여기서 우리는 서로 다른 목적으로 사용될 커널 명령 라인을 저장하는 3 개의 버퍼를 위한 공간 할당을 볼 수 있다. 공간을 할당하고 나서는, `boot_command_line` 를 `saved_command_line` 에 저장하고 `command_line` 을 `static_command_line` 에 저장한다.

`setup_command_line` 함수 다음으로 호출되는 함수는 `setup_nr_cpu_ids` 이다. 이 함수는 `cpu_possible_mask` 의 마지막 비트에 따라 `nr_cpu_ids`(CPU 의 수) 를 설정한다. (이것에 관해 조금 더 자세히 알고 싶다면, [cpumasks](https://github.com/daeseokyoun/linux-insides/blob/master/Concepts/cpumask.md)를 살펴 보자.) 이제 그 구현을 살펴 보자.:

```C
void __init setup_nr_cpu_ids(void)
{
        nr_cpu_ids = find_last_bit(cpumask_bits(cpu_possible_mask),NR_CPUS) + 1;
}
```

여기서 `nr_cpu_ids` 는 CPU 의 개수를 나타내고, `NR_CPUS` 는 최대로 구성할 수 있는 CPU 의 최대 개수를 나타낸다.:

![CONFIG_NR_CPUS](http://oi59.tinypic.com/28mh45h.jpg)

실제로, 우리는 이 함수의 호출이 필요한데, 그 이유는 `NR_CPUS` 는 당신이 가지고 있는 컴퓨터의 실제 CPU 개수 보다 더 클 수 있기 때문이다. 여기서 우리는 `find_last_bit` 함수의 호출을 볼 수 있고, 이 함수는 두 개의 인자를 가진다.:

* `cpu_possible_mask` 비트들
* 최대 CPU 개수

`setup_arch` 함수에서 우리는 `prefill_possible_map` 함수 호출을 통해 CPU 의 개수를 계산하고 `cpu_possible_mask` 에 써 놓았다. 이제 `find_last_bit` 함수를 주소와 최대 크기를 인자로 넘겨 처음으로 셋된 비트의 인덱스를 반환한다. `cpu_possible_mask` 비트들과 CPU 개수의 최대값을 넘겨주었다. `find_last_bit` 함수에서 처음으로 하는 것은 주어진 `unsigned long` 주소를 [words](http://en.wikipedia.org/wiki/Word_%28computer_architecture%29) 단위로 쪼개는 것이다.:

```C
words = size / BITS_PER_LONG;
```

`BITS_PER_LONG` 은 `x86_64` 에서 `64` 값을 가진다. 찾아야 하는 데이터의 주어진 크기에서 word 의 개수를 얻고, 그 크기를 아래의 코드로 `0` 이 된 것이 아닌지 확인한다.:

```C
if (size & (BITS_PER_LONG-1)) {
         tmp = (addr[words] & (~0UL >> (BITS_PER_LONG
                                 - (size & (BITS_PER_LONG-1)))));
         if (tmp)
                 goto found;
}
```

word 단위에서 비트가 포함되어 있다면, 마지막 word 를 마스킹하고 확인한다. 만약 마지막 word가 0이 아니라면, 현재 word 가 적어도 하나의 비트를 1로 가지고 있다는 의미이다. 찾았다면 `found` 라벨로 이동하자.:

```C
found:
    return words * BITS_PER_LONG + __fls(tmp);
```

여기서 `bsr` 명령어의 도움으로 주어진 word 단위에 최상위 비트를 찾아 반환한다.:

```C
static inline unsigned long __fls(unsigned long word)
{
        asm("bsr %1,%0"
            : "=r" (word)
            : "rm" (word));
        return word;
}
```

`bsr` 명령어는 주어진 피연산자(operand) 의 최상위 비트(Most significant bit) 를 찾아준다. 만약 마지막 word 가 비트를 포함하지 않았다면, 주어진 주소에서 모든 word 단위를 검사하여 첫 비트를 찾기위해 시도 할 것이다.:

```C
while (words) {
    tmp = addr[--words];
    if (tmp) {
found:
        return words * BITS_PER_LONG + __fls(tmp);
    }
}
```

마지막 word 를 `tmp` 변수에 넣고 `tmp` 가 적어도 하나의 비트를 갖고 있는지 확인한다. 만약 설정된 비트를 찾았다면, 그 비트가 몇번째 비트인지 반환한다. 만약 어떤 비트도 설정되어 있지 않았다면, 인자로 주어진 크기를 반환한다.:

```C
return size;
```

`nr_cpu_ids` 는 가용한 CPU 의 정확한 개수를 가질 것이다.

결론
================================================================================

리눅스 초기화 과정의 7번째 파트가 마무리 되었다. 이 파트에서는 마침내, `setup_arch` 함수를 마무리 지었고, `start_kernel` 함수로 복귀했다. 다음 파트에서 `start_kernel` 의 일반적인 커널 코드를 계속 해서 배워 볼 것이고, 첫 프로세스인 `init`까지 가볼 것이다.

어떤 질문이나 제안이 있다면, twitter [0xAX](https://twitter.com/0xAX), [email](anotherworldofworld@gmail.com) 또는 [issue](https://github.com/0xAX/linux-insides/issues/new) 를 만들어 주길 바란다.

**나는 영어권의 사람이 아니고 이런 것에 대해 매우 미안해 하고 있다. 만약 어떤 실수를 발견한다면, 나에게 PR을 [linux-insides](https://github.com/0xAX/linux-internals)을 보내줘**

Links
================================================================================

* [Desktop Management Interface](http://en.wikipedia.org/wiki/Desktop_Management_Interface)
* [x86_64](http://en.wikipedia.org/wiki/X86-64)
* [initrd](http://en.wikipedia.org/wiki/Initrd)
* [Kernel panic](http://en.wikipedia.org/wiki/Kernel_panic)
* [Documentation/kernel-parameters.txt](https://github.com/torvalds/linux/blob/master/Documentation/kernel-parameters.txt)
* [ACPI](http://en.wikipedia.org/wiki/Advanced_Configuration_and_Power_Interface)
* [Direct memory access](http://en.wikipedia.org/wiki/Direct_memory_access)
* [NUMA](http://en.wikipedia.org/wiki/Non-uniform_memory_access)
* [Control register](http://en.wikipedia.org/wiki/Control_register)
* [vsyscalls](https://lwn.net/Articles/446528/)
* [SMP](http://en.wikipedia.org/wiki/Symmetric_multiprocessing)
* [jiffy](http://en.wikipedia.org/wiki/Jiffy_%28time%29)
* [Previous part](http://0xax.gitbooks.io/linux-insides/content/Initialization/%20linux-initialization-6.html)
