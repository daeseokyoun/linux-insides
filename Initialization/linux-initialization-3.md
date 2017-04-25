커널 초기화. Part 3.
================================================================================

커널 엔트리 포인트로 진입 전 마지막 준비
--------------------------------------------------------------------------------

리눅스 커널 초기화 과정 시리즈중 3 번째 파트이다. 이전 [파트](https://github.com/daeseokyoun/linux-insides/blob/master/Initialization/linux-initialization-2.md)에서 초기 인터럽트와 exception 핸들링을 알아보았고 이 파트에서도 리눅스 커널 초기화 과정을 계속 알아볼 것이다. 우리의 다음 확인 사항은 [init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c) 소스 파일에 있는 '커널 엔트리 포인트'인 `start_kernel` 함수이다. 이함수가 실제로 커널 엔트리 포인트는 아니지만 아키텍처에 의존적이지 않은 일반적인 커널 코드이 시작이라고 볼 수 있다. `start_kernel` 을 보기전에 몇가지 준비사항이 있다. 먼저 그것을 살펴 보기로 하자.

boot_params 재조명
--------------------------------------------------------------------------------

이전 파트에서 우리는 인터럽트 디스크립션 테이블과 `IDTR` 레지스터에 그것을 로드하는 곳에서 마무리하였다. 그 다름 단계로 `copy_bootdata` 함수의 호출을 봐야 할 것이다.:

```C
copy_bootdata(__va(real_mode_data));
```

이 함수는 하나의 인자를 받는다. 그것의 `real_mode_data` 의 가상 주소이다. [arch/x86/include/uapi/asm/bootparam.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/uapi/asm/bootparam.h#L114)에 정의되어 있는 `boot_params` 구조체의 주소는 [arch/x86/kernel/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head_64.S) 에서 `x86_64_start_kernel` 함수의 첫번째 인자로 전달 했다는 것을 기억하자.:

```
	/* rsi is pointer to real mode structure with interesting info.
	   pass it to C */
	movq	%rsi, %rdi
```

이제 `__va` 매크로를 보자. 이 매크로는  [init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c)에 정의되어 있다:

```C
#define __va(x)                 ((void *)((unsigned long)(x)+PAGE_OFFSET))
```

`PAGE_OFFSET` 은 `__PAGE_OFFSET` 로 되어 있고, 모든 물리 메모리를 직접 맵핑할 수 있는 가상 주소의 베이스이다. 그 값은 `0xffff880000000000` 이다. 그래서 우리는 `boot_params` 구조체의 가상 주소를 얻고 그것을 `copy_bootdata`함수로 넘겨준다. 그 뒤에 우리는 [arch/x86/kernel/setup.h](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/setup.h)에 정의되어 있는 `boot_params`에 `real_mod_data`를 복사한다.

```C
extern struct boot_params boot_params;
```

`copy_boot_data` 의 구현부를 살펴보자:

```C
static void __init copy_bootdata(char *real_mode_data)
{
	char * command_line;
	unsigned long cmd_line_ptr;

	memcpy(&boot_params, real_mode_data, sizeof boot_params);
	sanitize_boot_params(&boot_params);
	cmd_line_ptr = get_cmd_line_ptr();
	if (cmd_line_ptr) {
		command_line = __va(cmd_line_ptr);
		memcpy(boot_command_line, command_line, COMMAND_LINE_SIZE);
	}
}
```

제일 먼저, 이 함수는 `__init` 접두사가 붙는다는 것을 주목하자. 이 함수는 초기화 과정에서 사용될 것이며 사용된 메모리를 해제될 것이다는 의미를 포함한다.

커널 명령 라인을 위한 두 개의 변수 선언과 `real_mode_data` 를 `boot_params` 로 `memcpy` 함수를 통해 복사한다. 그 다음 `sanitize_boot_params` 함수 호출로 `boot_params` 구조체의 `ext_ramdisk_image` 와 같은 몇 몇 항목 그리고 부트로더가 알수 없는 항목으로 `boot_params` 내에 초기화 하지 못한 것들을 0으로 채워 준다. 이 후에 우리는 `get_cmd_line_ptr` 함수 호출로 명령 라인의 주소를 얻을 것이다.:

```C
unsigned long cmd_line_ptr = boot_params.hdr.cmd_line_ptr;
cmd_line_ptr |= (u64)boot_params.ext_cmd_line_ptr << 32;
return cmd_line_ptr;
```

커널 부트 헤더로 부터 명령라인의 64 비트 주소를 얻고 그것을 반환한다. 마지막 단계에서는 `cmd_line_ptr` 을 확인하고, 그것의 가상 주소를 얻은 다음 바이트 배열인 `boot_command_line` 으로 복사한다.:

```C
extern char __initdata boot_command_line[];
```

이 다음에 우리는 복사된 커널 명령라인과 `boot_params` 구조체를 가질 것이다. 다음 단계에서는 프로세서의 마이크로 코드를 로드하는 `load_ucode_bsp` 함수의 호출을 볼수 있다. 하지만 여기서는 다루지 않을 것이다.

마이크로 코드가 로드되고 나면, `console_loglevel` 을 확인해서 loglevel 에 따라 `early_printk` 에서 `Kernel Alive` 문자열을 출력한다. 하지만, `early_printk` 는 이시점에서 초기화 되지 않기 때문에, 이문자열은 결코 볼 수 없을 것이다. 커널내에 마이너 버그이고, [패치](http://git.kernel.org/cgit/linux/kernel/git/tip/tip.git/commit/?id=91d8f0416f3989e248d3a3d3efb821eda10a85d2)를 보내 현재는 이 출력문의 코드는 없어졌다.

초기 페이지로 넘어가자
--------------------------------------------------------------------------------

복사된 `boot_params` 구조체를 얻은 다음에는 초기 페이지 테이블로 부터 초기 프로세스를 위한 페이지 테이블로 전환을 해야 한다. 우리는 지난 [파트](https://github.com/daeseokyoun/linux-insides/blob/korean-trans/Initialization/linux-initialization-1.md) 에서 이미 초기 페이지 테이블을 설정하는 것을 살펴보았다. `reset_early_page_tables` 함수의 호출로 모든 페이지 테이블을 정리하였고, 단지 커널 high 맵핑만 남겨두었다. 이 다음의 호출은:

```C
	clear_page(init_level4_pgt);
```

`clear_page` 함수에 [arch/x86/kernel/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head_64.S) 에 정의된 `init_level4_pgt` 를 넘겨준다. 그리고 `init_level4_pgt` 는 아래와 같이:

```assembly
NEXT_PAGE(init_level4_pgt)
	.quad   level3_ident_pgt - __START_KERNEL_map + _KERNPG_TABLE
	.org    init_level4_pgt + L4_PAGE_OFFSET*8, 0
	.quad   level3_ident_pgt - __START_KERNEL_map + _KERNPG_TABLE
	.org    init_level4_pgt + L4_START_KERNEL*8, 0
	.quad   level3_kernel_pgt - __START_KERNEL_map + _PAGE_TABLE
```

이것은 첫 2 GB 와 512 MB 를 커널 코드, 데이터 그리고 BSS 위해 맵핑한다. [arch/x86/lib/clear_page_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/lib/clear_page_64.S) 의 `clear_page` 함수를 살펴보자:


```assembly
ENTRY(clear_page)
	CFI_STARTPROC
	xorl %eax,%eax
	movl $4096/64,%ecx
	.p2align 4
	.Lloop:
    decl	%ecx
#define PUT(x) movq %rax,x*8(%rdi)
	movq %rax,(%rdi)
	PUT(1)
	PUT(2)
	PUT(3)
	PUT(4)
	PUT(5)
	PUT(6)
	PUT(7)
	leaq 64(%rdi),%rdi
	jnz	.Lloop
	nop
	ret
	CFI_ENDPROC
	.Lclear_page_end:
	ENDPROC(clear_page)
```

이 함수의 이름을 봐도 이 함수는 페이지 테이블을 모두 0으로 채우는 일을 하는 것임을 알 수 있다. 먼저 이함수는 `CFI_STARTPROC` 에서 시작하고 `CFI_ENDPROC`에서 끝난다. 이 매크로는 GNU 어셈블리 [디렉티브](http://reverseengine.tistory.com/entry/%EC%A0%9C-5%EC%9E%A5-%EB%94%94%EB%A0%89%ED%8B%B0%EB%B8%8C)를 확장한 것이다.:

```C
#define CFI_STARTPROC           .cfi_startproc
#define CFI_ENDPROC             .cfi_endproc
```

그리고 디버깅을 위해 사용된다. `CFI_STARTPROC` 매크로 바로 다음에 `eax` 레지스터를 0으로 만들고 64 의 값을 `ecx` 레지스터에 넣어준다.(이것은 카운터 값으로 사용될 것이다.) 다음에는 `.Lloop` 라벨로 시작하는 루프를 볼 수 있고 그것은 `ecx`를 하나 감소하는 코드로 시작한다. `rax` 레지스터의 0 값을 현재 `init_level4_pgt` 의 베이스 주소값을 갖고 있는 `rdi` 에 넣고 이와 같은 작업을 7번 더 해주는데 이 때, 매번마다 `rdi` 의 오프셋을 8씩 이동시킨다. 이렇게 하면, 우리는 `init_level4_pgt` 의 첫 64 바이트를 0으로 채우게 된다. 다음 단계는 `rdi` 의 64 바이트 오프셋을 더해 `init_level4_pgt` 의 주소를 넣어주는 일을 `ecx` 의 값이 0이 될 때까지 한다. 결국 `init_level4_pgt` 의 모든 내용을 0으로 채우는 것이다.

`init_level4_pgt` 를 0 으로 모두 초기화 하고, 마지막 511 엔트리에는 `early_level4_pgt` 의 511 엔트리의 주소를 대입한다. 이것은 나중에 사용할 level 4 페이지 테이블에 커널이 사용할 커널 텍스트 맵핑 영역을 설정해주는 것이다.:

```C
init_level4_pgt[511] = early_level4_pgt[511];
```

`reset_early_page_table` 함수에서 커널 텍스트 맵핑 영역만 남기고 모두 초기화를 했다는 것을 기억하자.

`x86_64_start_kernel` 함수의 마지막 단계는 아래의 함수 호출이다.:

```C
x86_64_start_reservations(real_mode_data);
```

이 함수는 `real_mode_data` 를 인자로 받는다. `x86_64_start_reservations` 함수는 `x86_64_start_kernel` 함수가 있던 파일과 동일한 소스코드에 있고 아래와 같이 구현되어 있다:

```C
void __init x86_64_start_reservations(char *real_mode_data)
{
	if (!boot_params.hdr.version)
		copy_bootdata(__va(real_mode_data));

	reserve_ebda_region();

	start_kernel();
}
```

이제 커널 엔트리 포인트인 `start_kernel` 에 진입하기 전의 마지막 함수의 호출을 볼 수 있다. 마지막 함수는 어떤 일을 하고 어떻게 동작하는지 살펴보자.

커널 엔트리 포인트 전의 마지막 단계
--------------------------------------------------------------------------------

`x86_64_start_reservations` 함수에서 제일 처음으로 볼 수 있는 것이 `boot_params.hdr.version` 를 확인하는 것이다:

```C
if (!boot_params.hdr.version)
	copy_bootdata(__va__(real_mode_data)); // va 뒤의 언더바 두개 지우기
```

만약 그 값이 0이면, `copy_bootdata` 함수를 `real_mode_data` 의 가상 주소와 함께 다시 호출해준다.

다음 단계에서는 [arch/x86/kernel/head.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head.c) 에 구현되어 있는 `reserve_ebda_region` 함수의 호출이다. 이 함수는 `EBDA` 혹은 `확장된 BIOS 데이터 영역` 을 위해 메모리 블럭을 예약해둔다. 확장된 BIOS 데이터 영역(Extended BIOS Data Area)은 기본 메모리의 꼭대기(TOP) 에 위치하고 포트, 디스크 파라미터등의 데이터를 담고 있다.

`reserve_ebda_region` 함수를 살펴보자. 그것은 반가상화(paravirtualization) 이 활성화 되어 있는지 확인하는 것으로 부터 시작한다.:

```C
if (paravirt_enabled())
	return;
```

만약 반가상화가 활성화 되어 있다면, `reserve_ebda_region` 함수를 빠져나간다. 활성화 되어 있다는 것은 EBDA 데이터가 없다는 것을 의미하기 때문이다. 다음 단계에서는 low 메모리의 마지막을 얻어야 한다.:

```C
lowmem = *(unsigned short *)__va(BIOS_LOWMEM_KILOBYTES);
lowmem <<= 10;
```

우리는 BIOS low 메모리의 가상 주소를 KB 단위의 값으로 갖고 오기 때문에 우측 10 만큼 쉬프트를 해줘야 한다.(이는 1024를 곱하는 것과 같은 의미이다.) 그 다음 확장된 BIOS 데이터의 주소를 아래와 같이 얻는다.:

```C
ebda_addr = get_bios_ebda();
```

`get_bios_ebda` 함수는 [arch/x86/include/asm/bios_ebda.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/bios_ebda.h)에 아래와 같이 구현되어 있다:

```C
static inline unsigned int get_bios_ebda(void)
{
	unsigned int address = *(unsigned short *)phys_to_virt(0x40E);
	address <<= 4;
	return address;
}
```

이 함수가 어떻게 동작하는지 이해해보도록 하자. 우리는 물리주소 `0x40E`를 가상 주소로 변환할 것이다. 이는 `0x0040:0x000e` 로 변환 시도 될 것이며, 이 세그먼트는 확장된 BIOS 데이터 영역의 베이스 주소를 갖고 있다. 물리 주소에서 가상 주소로 변환하기 위해 `phys_to_virt` 함수를 사용할 것이다. 이미 이전에 `__va` 매크로를 통해 이와 동일한 작업을 했었고, `phys_to_virt` 함수도 이와 같은 일을 한다:

```C
static inline void *phys_to_virt(phys_addr_t address)
{
         return __va(address);
}
```

단지 하나의 차이점이 있다면: 우리는 `CONFIG_PHYS_ADDR_T_64BIT` 옵션에 의존적인 `phys_addr_t` 타입의 인자를 넘겨 줄것이다.:

```C
#ifdef CONFIG_PHYS_ADDR_T_64BIT
	typedef u64 phys_addr_t;
#else
	typedef u32 phys_addr_t;
#endif
```

`CONFIG_PHYS_ADDR_T_64BIT`는 단지 64 비트/32 비트 물리주소를 결정한다. 확장된 BIOS 데이터 영역의 베이스 주소가 저장되어 있는 세그먼트의 가상 주소를 얻고 나면, 그 값을 우측 4만큼 쉬프트(16 을 곱함 - 이 값은 real mode 에서 다루는 세그먼크 값이기 때문에)하여 반환한다. `ebda_addr` 변수는 확장된 BIOS 데이터 영역의 베이스 주소를 갖게 된다.

다음 단계에서는 `ebda_addr` 의 주소와 low 메모리가 `INSANE_CUTOFF` 매크로 보다 작지 않는지 확인한다:

```C
if (ebda_addr < INSANE_CUTOFF)
	ebda_addr = LOWMEM_CAP;

if (lowmem < INSANE_CUTOFF)
	lowmem = LOWMEM_CAP;
```

`INSANE_CUTOFF` 은:

```C
#define INSANE_CUTOFF		0x20000U
```

128 킬로 바이트이다. 마지막 단계로 `lowmem` 와 `ebda_addr` 중 작은 값을 선택하여, 사용가능한 메모리의 상한선에서 1 MB 까지 확장된 BIOS 데이터 영역을 위해 `memblock_reserve` 함수 호출로 메모리를 사용하지 못하도록 예약한다.:

```C
lowmem = min(lowmem, ebda_addr);
lowmem = min(lowmem, LOWMEM_CAP);
memblock_reserve(lowmem, 0x100000 - lowmem);
```

`memblock_reserve` 함수는 [mm/block.c](https://github.com/torvalds/linux/blob/master/mm/block.c) 에 구현되어 있고, 두 개의 인자를 받는다:
* 베이스 물리주소
* 영역 크기

그리고 주어진 베이스 주소에서 그 크기만큼 메모리 영역을 예약한다. `memblock_reserve` 는 리눅스 커널 메모리 관리 프레임웍에서 첫번째로 등장하는 함수이다. 메모리 관리는 곧 자세히 살펴 볼 것이고, 지금은 이 함수의 구현을 살펴보자.

리눅스 커널 메모리 관리 프레임웍의 첫 만남.
--------------------------------------------------------------------------------

이전 장에서 `memblock_reserve` 함수의 호출에서 마무리 지었고 메모리 관리 프레임웍에서 처음 접하는 함수라 언급했다. 이 함수가 어떻게 동작하는지 살펴 보도록 하자. `memblock_reserve` 함수는 아래와 같이 불린다:

```C
memblock_reserve_region(base, size, MAX_NUMNODES, 0);
```

함수는 4개의 인자를 가진다:

* 메모리 영역의 물리적 시작 주소
* 메모리 영역의 크기
* NUMA 노드들의 최대 개수
* 플래그

`memblock_reserve_region` 함수 내를 보면, 처음 시작에서 `memblock_type` 구조체의 선언을 볼 수 있을 것이다.:

```C
struct memblock_type *_rgn = &memblock.reserved;
```

메모리 블럭의 타입을 표현하기 위한 구조체이다:

```C
struct memblock_type {
         unsigned long cnt;
         unsigned long max;
         phys_addr_t total_size;
         struct memblock_region *regions;
};
```

확장된 BIOS 데이터 영역을 위해 메모리 블럭을 예약할 필요가 있다. 현재 메모리 영역의 타입은 `memblock` 구조체에서 `reserved` 이다.:

```C
struct memblock {
         bool bottom_up;
         phys_addr_t current_limit;
         struct memblock_type memory;
         struct memblock_type reserved;
#ifdef CONFIG_HAVE_MEMBLOCK_PHYS_MAP
         struct memblock_type physmem;
#endif
};
```

그리고 일반적 메모리 블럭을 기술한다. `memblock.reserved` 의 주소를 `_rgn` 에 할당함으로써 초기화 한다. `memblock` 은 아래와 같이 선언된 글러벌 변수이다.:

```C
struct memblock memblock __initdata_memblock = {
	.memory.regions		= memblock_memory_init_regions,
	.memory.cnt		= 1,
	.memory.max		= INIT_MEMBLOCK_REGIONS,
	.reserved.regions	= memblock_reserved_init_regions,
	.reserved.cnt		= 1,
	.reserved.max		= INIT_MEMBLOCK_REGIONS,
#ifdef CONFIG_HAVE_MEMBLOCK_PHYS_MAP
	.physmem.regions	= memblock_physmem_init_regions,
	.physmem.cnt		= 1,
	.physmem.max		= INIT_PHYSMEM_REGIONS,
#endif
	.bottom_up		= false,
	.current_limit		= MEMBLOCK_ALLOC_ANYWHERE,
};
```

지금은 이 변수에 대해 자세히 살펴 보진 않겠지만, 메모리 관리 파트를 보게 된다면 더 자세히 알수 있을 것이다. 단지 `memblock` 변수는 `__initdata_memblock` 와 함께 선언되었다는 것을 볼 수 있다:

```C
#define __initdata_memblock __meminitdata
```

__initdata_memblock 은 __meminitdata 로 선언되어 있고, __meminitdata 는:

```C
#define __meminitdata    __section(.meminit.data)
```

이것으로 부터, 모든 메모리 블럭은 `.meminit.data` 섹션내에 있다는 것을 알 수 있다. 선언한 `_rgn`을 `memblock_dbg` 매크로를 이용해서 관련 정보를 출력할 수 있다. 커널 명령 라인에 `memblock=debug` 라인을 넘겨주어 그것을 활성화 할 수 있다.

다음 함수의 호출에서 디버깅 라인들을 출력하는 것을 볼 수 있다.:

```C
memblock_add_range(_rgn, base, size, nid, flags);
```

이것은 `.meminit.data` 섹션에 새로운 메모리 블럭 영역을 추가하는 것이다. `_rgn` 단지 `&memblock.reserved`를 갖고 있고, 우리는 확장된 BIOS 데이터 영역의 시작 주소, 영역의 크기 그리고 플래그를 `_rgn` 과 함께 넘겨주었다.:

```C
if (type->regions[0].size == 0) {
    WARN_ON(type->cnt != 1 || type->total_size);
    type->regions[0].base = base;
    type->regions[0].size = size;
    type->regions[0].flags = flags;
    memblock_set_region_node(&type->regions[0], nid);
    type->total_size = size;
    return 0;
}
```

우리는 이 영역의 내용을 채운 다음, 두개의 인자와 함께 `memblock_set_region_node` 함수의 호출을 볼 수 있다.:

* 채워진 메모리 영역의 주소
* NUMA 노드 ID.

`memblock_region` 구조체에 의해 표현된 이 영역은:
```C
struct memblock_region {
    phys_addr_t base;
	  phys_addr_t size;
	  unsigned long flags;
#ifdef CONFIG_HAVE_MEMBLOCK_NODE_MAP
    int nid;
#endif
};
```

NUMA 노드 id 는 [include/linux/numa.h](https://github.com/torvalds/linux/blob/master/include/linux/numa.h) 에 선언된 `MAX_NUMNODES`에 의존적이다.:

```C
#define MAX_NUMNODES    (1 << NODES_SHIFT)
```

여기서 `NODES_SHIFT` 는 `CONFIG_NODES_SHIFT` 구성 옵션에 의존적이며 선언은:

```C
#ifdef CONFIG_NODES_SHIFT
  #define NODES_SHIFT     CONFIG_NODES_SHIFT
#else
  #define NODES_SHIFT     0
#endif
```

`memblick_set_region_node` 함수는 단지 인자로 주어진 값을 `memblock_region` 의 `nid` 항목에 채워준다.:

```C
static inline void memblock_set_region_node(struct memblock_region *r, int nid)
{
         r->nid = nid;
}
```

이것을 통해 `.meminit.data`섹션에 확장된 bios 데이터 영역을 위해 첫 예약된 `memblock` 을 가지게 되었다. `reserve_ebda_region` 함수는 모든 일을 마무리 하였고, [arch/x86/kernel/head64.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head64.c) 로 다시 돌아갈 수 있다.

커널 엔트리 포인트로 진입전 모든 준비를 마쳤다. `x86_64_start_reservations` 내의 마지막 단계는 아래의 `start_kernel` 함수를 호출 하는 것이다.:
```C
start_kernel()
```

이 함수의 구현은 [init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c) 파일에 있다.

이 파트의 모든 것을 마무리 했다.

결론
--------------------------------------------------------------------------------

리눅스 커널 초기화의 3번째 파트가 마무리되었다. 다음 파트는 커널 엔트리 포인트인 `start_kernel` 함수에서 첫 초기화 단계를 볼 수 있다. 그것은 첫 `init` 프로세스를 수행하기 전에 첫 단계가 될 것이다.

어떤 질문이나 제안이 있다면, twitter [0xAX](https://twitter.com/0xAX), [email](anotherworldofworld@gmail.com) 또는 [issue](https://github.com/0xAX/linux-insides/issues/new) 를 만들어 주길 바란다.

**나는 영어권의 사람이 아니고 이런 것에 대해 매우 미안해 하고 있다. 만약 어떤 실수를 발견한다면, 나에게 PR을 [linux-insides](https://github.com/0xAX/linux-internals)을 보내줘**

링크
--------------------------------------------------------------------------------

* [BIOS data area](http://stanislavs.org/helppc/bios_data_area.html)
* [What is in the extended BIOS data area on a PC?](http://www.kryslix.com/nsfaq/Q.6.html)
* [Previous part](https://github.com/0xAX/linux-insides/blob/master/Initialization/linux-initialization-2.md)
