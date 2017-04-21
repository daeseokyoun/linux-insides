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
#define __va(x)                 ((void *)*((unsigned long)(x)+PAGE_OFFSET)) // TODO void *) 다음에 * 지우기
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

제일 먼저, 이 함수는 `__init`__ 접두사가 붙는다는 것을 주목하자. 이 함수는 초기화 과정에서 사용될 것이며 사용된 메모리를 해제될 것이다는 의미를 포함한다. // TODO init 뒤에 언더바 두개 지우기

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

그리고 디버깅을 위해 사용된다. `CFI_STARTPROC` 매크로 바로 다음에 `eax` 레지스터를 0으로 만들고 64 의 값을 `ecx` 레지스터에 넣어준다.(이것은 카운터 값으로 사용될 것이다.) 다음에는 `.Lloop` 라벨로 시작하는 루프를 볼 수 있고 그것은 `ecx`를 하나 감소하는 코드로 시작한다. `rax` 레지스터의 0 값을 현재 `init_level4_pgt` 의 베이스 주소값을 갖고 있는 `rdi` 에 넣고 이와 같은 작업을 7번 더 해주는데 이 때, 매번마다 `rdi` 의 오프셋을 8씩 이동시킨다. 이렇게 하면, 우리는 `init_level4_pgt` 의 첫 64 바이트를 0으로 채우게 된다. 다음 단계는 `rdi` 의 64 바이트 오프셋을 더해 `init_level4_pgt` 의 주소를 넣어주는 일을 `ecx` 의 값이 0이 될 때까지 한다. 결국 `init_level4_pgt` 의 모든 내용을 0으로 채우는 것이다.())( // TODO 이괄호들 지우기

As we have `init_level4_pgt` filled with zeros, we set the last `init_level4_pgt` entry to kernel high mapping with the:

```C
init_level4_pgt[511] = early_level4_pgt[511];
```

Remember that we dropped all `early_level4_pgt` entries in the `reset_early_page_table` function and kept only kernel high mapping there.

The last step in the `x86_64_start_kernel` function is the call of the:

```C
x86_64_start_reservations(real_mode_data);
```

function with the `real_mode_data` as argument. The `x86_64_start_reservations` function defined in the same source code file as the `x86_64_start_kernel` function and looks:

```C
void __init x86_64_start_reservations(char *real_mode_data)
{
	if (!boot_params.hdr.version)
		copy_bootdata(__va(real_mode_data));

	reserve_ebda_region();

	start_kernel();
}
```

You can see that it is the last function before we are in the kernel entry point - `start_kernel` function. Let's look what it does and how it works.

Last step before kernel entry point
--------------------------------------------------------------------------------

First of all we can see in the `x86_64_start_reservations` function the check for `boot_params.hdr.version`:

```C
if (!boot_params.hdr.version)
	copy_bootdata(__va(real_mode_data));
```

and if it is zero we call `copy_bootdata` function again with the virtual address of the `real_mode_data` (read about its implementation).

In the next step we can see the call of the `reserve_ebda_region` function which defined in the [arch/x86/kernel/head.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head.c). This function reserves memory block for the `EBDA` or Extended BIOS Data Area. The Extended BIOS Data Area located in the top of conventional memory and contains data about ports, disk parameters and etc...

Let's look on the `reserve_ebda_region` function. It starts from the checking is paravirtualization enabled or not:

```C
if (paravirt_enabled())
	return;
```

we exit from the `reserve_ebda_region` function if paravirtualization is enabled because if it enabled the extended bios data area is absent. In the next step we need to get the end of the low memory:

```C
lowmem = *(unsigned short *)__va(BIOS_LOWMEM_KILOBYTES);
lowmem <<= 10;
```

We're getting the virtual address of the BIOS low memory in kilobytes and convert it to bytes with shifting it on 10 (multiply on 1024 in other words). After this we need to get the address of the extended BIOS data are with the:

```C
ebda_addr = get_bios_ebda();
```

where `get_bios_ebda` function defined in the [arch/x86/include/asm/bios_ebda.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/bios_ebda.h) and looks like:

```C
static inline unsigned int get_bios_ebda(void)
{
	unsigned int address = *(unsigned short *)phys_to_virt(0x40E);
	address <<= 4;
	return address;
}
```

Let's try to understand how it works. Here we can see that we converting physical address `0x40E` to the virtual, where `0x0040:0x000e` is the segment which contains base address of the extended BIOS data area. Don't worry that we are using `phys_to_virt` function for converting a physical address to virtual address. You can note that previously we have used `__va` macro for the same point, but `phys_to_virt` is the same:

```C
static inline void *phys_to_virt(phys_addr_t address)
{
         return __va(address);
}
```

only with one difference: we pass argument with the `phys_addr_t` which depends on `CONFIG_PHYS_ADDR_T_64BIT`:

```C
#ifdef CONFIG_PHYS_ADDR_T_64BIT
	typedef u64 phys_addr_t;
#else
	typedef u32 phys_addr_t;
#endif
```

This configuration option is enabled by `CONFIG_PHYS_ADDR_T_64BIT`. After that we got virtual address of the segment which stores the base address of the extended BIOS data area, we shift it on 4 and return. After this `ebda_addr` variables contains the base address of the extended BIOS data area.

In the next step we check that address of the extended BIOS data area and low memory is not less than `INSANE_CUTOFF` macro

```C
if (ebda_addr < INSANE_CUTOFF)
	ebda_addr = LOWMEM_CAP;

if (lowmem < INSANE_CUTOFF)
	lowmem = LOWMEM_CAP;
```

which is:

```C
#define INSANE_CUTOFF		0x20000U
```

or 128 kilobytes. In the last step we get lower part in the low memory and extended bios data area and call `memblock_reserve` function which will reserve memory region for extended bios data between low memory and one megabyte mark:

```C
lowmem = min(lowmem, ebda_addr);
lowmem = min(lowmem, LOWMEM_CAP);
memblock_reserve(lowmem, 0x100000 - lowmem);
```

`memblock_reserve` function is defined at [mm/block.c](https://github.com/torvalds/linux/blob/master/mm/block.c) and takes two parameters:

* base physical address;
* region size.

and reserves memory region for the given base address and size. `memblock_reserve` is the first function in this book from linux kernel memory manager framework. We will take a closer look on memory manager soon, but now let's look at its implementation.

First touch of the linux kernel memory manager framework
--------------------------------------------------------------------------------

In the previous paragraph we stopped at the call of the `memblock_reserve` function and as i sad before it is the first function from the memory manager framework. Let's try to understand how it works. `memblock_reserve` function just calls:

```C
memblock_reserve_region(base, size, MAX_NUMNODES, 0);
```

function and passes 4 parameters there:

* physical base address of the memory region;
* size of the memory region;
* maximum number of numa nodes;
* flags.

At the start of the `memblock_reserve_region` body we can see definition of the `memblock_type` structure:

```C
struct memblock_type *_rgn = &memblock.reserved;
```

which presents the type of the memory block and looks:

```C
struct memblock_type {
         unsigned long cnt;
         unsigned long max;
         phys_addr_t total_size;
         struct memblock_region *regions;
};
```

As we need to reserve memory block for extended bios data area, the type of the current memory region is reserved where `memblock` structure is:

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

and describes generic memory block. You can see that we initialize `_rgn` by assigning it to the address of the `memblock.reserved`. `memblock` is the global variable which looks:

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

We will not dive into detail of this variable, but we will see all details about it in the parts about memory manager. Just note that `memblock` variable defined with the `__initdata_memblock` which is:

```C
#define __initdata_memblock __meminitdata
```

and `__meminit_data` is:

```C
#define __meminitdata    __section(.meminit.data)
```

From this we can conclude that all memory blocks will be in the `.meminit.data` section. After we defined `_rgn` we print information about it with `memblock_dbg` macros. You can enable it by passing `memblock=debug` to the kernel command line.

After debugging lines were printed next is the call of the following function:

```C
memblock_add_range(_rgn, base, size, nid, flags);
```

which adds new memory block region into the `.meminit.data` section. As we do not initialize `_rgn` but it just contains `&memblock.reserved`, we just fill passed `_rgn` with the base address of the extended BIOS data area region, size of this region and flags:

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

After we filled our region we can see the call of the `memblock_set_region_node` function with two parameters:

* address of the filled memory region;
* NUMA node id.

where our regions represented by the `memblock_region` structure:

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

NUMA node id depends on `MAX_NUMNODES` macro which is defined in the [include/linux/numa.h](https://github.com/torvalds/linux/blob/master/include/linux/numa.h):

```C
#define MAX_NUMNODES    (1 << NODES_SHIFT)
```

where `NODES_SHIFT` depends on `CONFIG_NODES_SHIFT` configuration parameter and defined as:

```C
#ifdef CONFIG_NODES_SHIFT
  #define NODES_SHIFT     CONFIG_NODES_SHIFT
#else
  #define NODES_SHIFT     0
#endif
```

`memblick_set_region_node` function just fills `nid` field from `memblock_region` with the given value:

```C
static inline void memblock_set_region_node(struct memblock_region *r, int nid)
{
         r->nid = nid;
}
```

After this we will have first reserved `memblock` for the extended bios data area in the `.meminit.data` section. `reserve_ebda_region` function finished its work on this step and we can go back to the [arch/x86/kernel/head64.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head64.c).

We finished all preparations before the kernel entry point! The last step in the `x86_64_start_reservations` function is the call of the:

```C
start_kernel()
```

function from [init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c) file.

That's all for this part.

Conclusion
--------------------------------------------------------------------------------

It is the end of the third part about linux kernel insides. In next part we will see the first initialization steps in the kernel entry point - `start_kernel` function. It will be the first step before we will see launch of the first `init` process.

If you have any questions or suggestions write me a comment or ping me at [twitter](https://twitter.com/0xAX).

**Please note that English is not my first language, And I am really sorry for any inconvenience. If you find any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-insides).**

Links
--------------------------------------------------------------------------------

* [BIOS data area](http://stanislavs.org/helppc/bios_data_area.html)
* [What is in the extended BIOS data area on a PC?](http://www.kryslix.com/nsfaq/Q.6.html)
* [Previous part](https://github.com/0xAX/linux-insides/blob/master/Initialization/linux-initialization-2.md)
