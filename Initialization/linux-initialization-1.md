커널 초기화. Part 1.
================================================================================

커널 압축 해제 후 첫 단계
--------------------------------------------------------------------------------

이전 [챕터](https://github.com/daeseokyoun/linux-insides/blob/master/Booting/linux-bootstrap-5.md) 는 리눅스 커널 부팅 과정의 마지막 부분이었고 이제는 리눅스 커널의 초기화 과정에 대해 알아볼 것이다. 리눅스 커널 이미지가 압축 해제되고 메모리의 적절한 곳에 위치 했다면, 커널이 일하기(?) 시작한다. 모든 이전 파트들은 리눅스 커널 코드의 첫 바이트가 실행되기 전의 준비과정의 커널 설정 코드의 작업을 기술 한것이다. 이제 우리는 커널로 진입했고, 이 챕터의 모든 파트는 [pid](https://en.wikipedia.org/wiki/Process_identifier) `1` 프로세스가 실행되기 전에 커널의 초기화 과정에 대해 집중할 것이다. `init` 프로세스가 수행되기 전에 많은 일들이 일어난다. [arch/x86/kernel/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head_64.S)에 있는 커널 엔트리 포인트에서 부터 시작할 것이고, 계속 앞으로 진행할 것이다. 우리는 [init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c#L489) 에 있는 `start_kernel`  함수가 불리기 전의 초기 페이지 테이블 초기화와 같은 첫 준비과정, 커널 공간에서 새로운 디스크립터로 전환등을 살펴 볼 것이다.

[arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_64.S) 어셈블리 소스 코드에서 [jmp](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_64.S#L429) 명령어를 수행하는 곳까지 [이전 파트](https://github.com/daeseokyoun/linux-insides/blob/master/Booting/linux-bootstrap-5.md) 에서 살펴 보았다.:

```assembly
jmp	*%rax
```

이 순간 `rax` 레스터에는 [arch/x86/boot/compressed/misc.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/misc.c) 소스 파일에서 `decompress_kernel` 함수 호출에 의해 얻어진 리눅스 커널 엔트리 포인터의 주소를 갖고 있다. 그래서, 리눅스 커널 설정 코드의 마지막 명령어는 커널 엔트리 포인트로 점프하는 것이다. 우리는 이미 리눅스 커널 엔트리 포인트가 어디에 정의되어 있는지 알고 있고, 우리는 시작 이후에 리눅스 커널이 무엇을 하는지 배울 수 있도록 준비가 되어 있다.

커널에서의 첫 단계
--------------------------------------------------------------------------------

`decompress_kernel` 함수에서 압축 해제된 커널 이미지의 주소를 `rax` 레지스터에 저장을 해놓았고 거기로 점프 할 것이다. [arch/x86/kernel/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head_64.S) 어셈블리 소스 코드 파일에서 압축 해제된 커널 이미지의 시작 엔트리 포인트를 알고 있고 그것으 시작 부분은 아래와 같다:

```assembly
    .text
	__HEAD
	.code64
	.globl startup_64
startup_64:
	...
	...
	...
```

`.head.text` 섹션의 정의를 매크로인 `__HEAD` 로 하고 그 섹션 내에 `startup_64` 루틴의 정의를 볼 수 있다.:

```C
#define __HEAD		.section	".head.text","ax"
```

이 섹션은 [arch/x86/kernel/vmlinux.lds.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/vmlinux.lds.S#L93)  링커 스크립트에서 확인 가능하다:

```
.text : AT(ADDR(.text) - LOAD_OFFSET) {
	_text = .;
	...
	...
	...
} :text = 0x9090
```

`.text` 섹션 정의 외에도, 링커 스크립터로 부터 기본 가상/물리 주소들이 어떻게 되어 있는지 이해해야 한다. [x86_64](https://en.wikipedia.org/wiki/X86-64) 을 위해 `_text` 는 위치 카운터(location counter) 에서 아래와 같이 주소 지정을 한다.:

```
. = __START_KERNEL;
```

`__START_KERNEL` 매크로의 정의는 [arch/x86/include/asm/page_types.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/page_types.h) 헤더 파일에 위치 하고 있고 커널 맵핑 가상 베이스 주소와 물리 주소의 시작을 더하는 것이다:

```C
#define __START_KERNEL	(__START_KERNEL_map + __PHYSICAL_START) //__ <-- TODO 지우기

#define __PHYSICAL_START  ALIGN(CONFIG_PHYSICAL_START, CONFIG_PHYSICAL_ALIGN)
```

주소를 계산해보면:

* 리눅스 커널의 물리 베이스 주소는 - `0x1000000`;
* 리눅스 커널의 가상 베이스 주소는 - `0xffffffff81000000`.

이제 우리는 `startup_64` 루틴의 기본 물리/가상 주소들을 알았다. 하지만 실제 주소를 알기 위해서는 아래의 코드와 함께 계산이되어야 한다.:

```assembly
	leaq	_text(%rip), %rbp
	subq	$_text - __START_KERNEL_map, %rbp
```

맞다, 그것은 `0x1000000` 로 정의되어 있지만, 그것은 만약 [kASLR](https://en.wikipedia.org/wiki/Address_space_layout_randomization#Linux) 이 활성화 되어 있다면(예를 들면) 실제랑 다를 것이다. 그래서 우리의 목표는 `0x1000000` 와 실제 로드된 곳의 차이 값을 계산하는 것이다. 여기에 `rip-relative` 주소를 `rbp` 레지스터에 넣고 그것에서 부터 `$_text - __START_KERNEL_map` 를 빼준다. `_text` 의 컴파일된 가상 주소는 `0xffffffff81000000` 이고, 그것의 물리 주소는 `0x1000000` 이다. `__START_KERNEL_map` 매크로는 `0xffffffff80000000` 주소로 확장하는 것이고, 그래서 어셈블리 코드의 두 번째 라인을 아래와 같이 값을 대입함으로써 한눈에 볼 수 있다.:

```
rbp = 0x1000000 - (0xffffffff81000000 - 0xffffffff80000000)
```

계산이 끝나면, `rbp`는 컴파일된 코드와 실제 로드된 주소의 차이를 표현하게 되면 `0`을 담게 된다. `0` 은 [kASLR](https://en.wikipedia.org/wiki/Address_space_layout_randomization#Linux) 비활성화 되어 있는 상태에서 리눅스 커널이 로드된 주소가 되는 것이다.:

`startup_64` 의 주소를 얻은 후에, 이 주소가 알맞게 정렬(aligned) 되어 있는지 확인이 필요하다. 아래의 코드를 보자:

```assembly
	testl	$~PMD_PAGE_MASK, %ebp
	jnz	bad_address
```

여기서 `ebp` 레지스터의 low 부분과 `PMD_PAGE_MASK` 값에 NOT(~) 연산을 하여 비교한다. `PMD_PAGE_MASK` 는 `Page middle directory` 를 위한 masking 값이다.([페이징](https://github.com/daeseokyoun/linux-insides/blob/master/Theory/Paging.md) 그리고 아래와 같이 정의되어 있다:

```C
#define PMD_PAGE_MASK           (~(PMD_PAGE_SIZE-1))
```

`PMD_PAGE_SIZE` 매크로는 아래와 같이:

```
#define PMD_PAGE_SIZE           (_AC(1, UL) << PMD_SHIFT)
#define PMD_SHIFT       21
```

쉽게 계산을 할 수 있는데, `PMD_PAGE_SIZE` 는 `2` MB 이다. 정렬을 확인하여 만약 `text` 주소가 `2` MB 정렬되지 않았다면, `bad_address` 라벨로 점프한다.

이 확인 이후에, 상위 `18` 비트를 확인하여 너무 크지 않는지 또 확인한다:

```assembly
	leaq	_text(%rip), %rax
	shrq	$MAX_PHYSMEM_BITS, %rax
	jnz	bad_address
```

주소는 `46` 비트보다 크면 안된다.:

```C
#define MAX_PHYSMEM_BITS       46
```

이제 초기 확인은 했고, 다음으로 넘어가자.

페이지 테이블의 고정 베이스 주소들
--------------------------------------------------------------------------------

페이징을 설정하는 것을 시작하기 전에 다음의 주소들을 수정해야 한다:

```assembly
	addq	%rbp, early_level4_pgt + (L4_START_KERNEL*8)(%rip)
	addq	%rbp, level3_kernel_pgt + (510*8)(%rip)
	addq	%rbp, level3_kernel_pgt + (511*8)(%rip)
	addq	%rbp, level2_fixmap_pgt + (506*8)(%rip)
```

만약 `startup_64` 함수가 기본 `0x1000000` 주소가 아니라면, `early_level4_pgt`, `level3_kernel_pgt` 그리고 다른 주소들 모두는 잘못되어 있을 것이다. `rbp` 레지스터는 주소의 차이 값을 갖고 있는데, `early_level4_pgt`, `level3_kernel_pgt` 와 `level2_fixmap_pgt` 의 정확한 엔트리에서 그 값을 더하는데 사용한다. 이 라벨들의 의미를 한번 생각해보자. 일단 이것들의 정의를 먼저 살펴 본다.:

```assembly
NEXT_PAGE(early_level4_pgt)
  /* fill <repeat>, <size>, <value> */
	.fill	511,8,0
	.quad	level3_kernel_pgt - __START_KERNEL_map + _PAGE_TABLE

NEXT_PAGE(level3_kernel_pgt)
	.fill	L3_START_KERNEL,8,0
	.quad	level2_kernel_pgt - __START_KERNEL_map + _KERNPG_TABLE
	.quad	level2_fixmap_pgt - __START_KERNEL_map + _PAGE_TABLE

NEXT_PAGE(level2_kernel_pgt)
	PMDS(0, __PAGE_KERNEL_LARGE_EXEC,
		KERNEL_IMAGE_SIZE/PMD_SIZE)

NEXT_PAGE(level2_fixmap_pgt)
	.fill	506,8,0
	.quad	level1_fixmap_pgt - __START_KERNEL_map + _PAGE_TABLE
	.fill	5,8,0

NEXT_PAGE(level1_fixmap_pgt)
	.fill	512,8,0
```

어려워 보이지만, 정작 그렇지는 않다. 맨 먼저 `early_level4_pgt`를 보자. 이것은 0 으로 채워진 (4096 - 8) 바이트에서 시작한다. 이것은 `511` 엔트리는 사용하지 않겠다는 의미이다. 그리고 이것다음에는 `level3_kernel_pgt` 엔트리가 있다. 여기서 `level3_kernel_pgt` 에 `__START_KERNEL_map + _PAGE_TABLE` 를 빼주는 것을 보자. `__START_KERNEL_map` 는 알고 있듯이, 커널 텍스트 영역의 가상 베이스 주소이다. 그래서 만약 `__START_KERNEL_map` 를 빼준다는 것은 `level3_kernel_pgt` 의 물리 주소를 얻는다는 것이다. 이제 `_PAGE_TABLE`을 살펴보자. 이것은 단지 페이지 엔트리의 접근 권한을 표현한다.:

```C
#define _PAGE_TABLE     (_PAGE_PRESENT | _PAGE_RW | _PAGE_USER | \
                         _PAGE_ACCESSED | _PAGE_DIRTY)_ // TODO 여기 마지막 언더바 지우기
```

[페이징](https://github.com/daeseokyoun/linux-insides/blob/master/Theory/Paging.md) 파트를 한번 읽어 보길 바란다.

`level3_kernel_pgt` - 커널 영역을 맵핑하는 두 개의 엔트리들을 저장한다. 이것의 정의의 시작은, `L3_START_KERNEL` 또는 `510` 반복하여 8 바이트씩 0으로 채운다. `L3_START_KERNEL` 은 `__START_KERNEL_map` 주소를 갖고 있는 `page upper directory` 내의 인덱스이고 그것은 `510`과 같다. 이 다음에 우리는 두 `level3_kernel_pgt` 엔트리들의 정의를 볼 수 있다.: `level2_kernel_pgt` 와 `level2_fixmap_pgt` 이다. 첫번째는 간단하다. 그것은 커널 영역을 맵핑하는 `page middle directory` 의 포인터를 가지는 페이지 테이블 엔트리들이다.:

```C
#define _KERNPG_TABLE   (_PAGE_PRESENT | _PAGE_RW | _PAGE_ACCESSED | \
                         _PAGE_DIRTY)_ // TODO 여기도 마지막 언더바 지우기
```

위의 `_KERNPG_TABLE` 는 페이지의 권한을 나타낸다. 두 번째는 - `level2_fixmap_pgt` 는 커널 영역 안에서 어떤 물리 주소도 참조할 수 있는 가상 주소이다. 이 주소들은 [vsyscalls](https://lwn.net/Articles/446528/) 맵핑을 위해 `10` 메가 바이트의 공간 함께 하나의 `level2_fixmap_pgt` 엔트리로 표현된다. 다음 `level2_kernel_pgt` 은 커널 `.text` 를 위한 `__START_KERNEL_map` 에서 부터 `512` MB 를 만드는 `PDMS` 매크로를 호출한다.(`512` MB 뒤에는 모듈을 위한 메모리 공간이 될 것이다.)

Now, after we saw definitions of these symbols, let's get back to the code which is described at the beginning of the section. Remember that the `rbp` register contains delta between the address of the `startup_64` symbol which was got during kernel [linking](https://en.wikipedia.org/wiki/Linker_%28computing%29) and the actual address. So, for this moment, we just need to add this delta to the base address of some page table entries, that they'll have correct addresses. In our case these entries are:

```assembly
	addq	%rbp, early_level4_pgt + (L4_START_KERNEL*8)(%rip)
	addq	%rbp, level3_kernel_pgt + (510*8)(%rip)
	addq	%rbp, level3_kernel_pgt + (511*8)(%rip)
	addq	%rbp, level2_fixmap_pgt + (506*8)(%rip)
```

or the last entry of the `early_level4_pgt` which is the `level3_kernel_pgt`, last two entries of the `level3_kernel_pgt` which are the `level2_kernel_pgt` and the `level2_fixmap_pgt` and five hundreds seventh entry of the `level2_fixmap_pgt` which is `level1_fixmap_pgt` page directory.

After all of this we will have:

```
early_level4_pgt[511] -> level3_kernel_pgt[0]
level3_kernel_pgt[510] -> level2_kernel_pgt[0]
level3_kernel_pgt[511] -> level2_fixmap_pgt[0]
level2_kernel_pgt[0]   -> 512 MB kernel mapping
level2_fixmap_pgt[507] -> level1_fixmap_pgt
```

Note that we didn't fixup base address of the `early_level4_pgt` and some of other page table directories, because we will see this during of building/filling of structures for these page tables. As we corrected base addresses of the page tables, we can start to build it.

Identity mapping setup
--------------------------------------------------------------------------------

Now we can see the set up of identity mapping of early page tables. In Identity Mapped Paging, virtual addresses are mapped to physical addresses that have the same value, `1 : 1`. Let's look at it in detail. First of all we get the `rip-relative` address of the `_text` and `_early_level4_pgt` and put they into `rdi` and `rbx` registers:

```assembly
	leaq	_text(%rip), %rdi
	leaq	early_level4_pgt(%rip), %rbx
```

After this we store address of the `_text` in the `rax` and get the index of the page global directory entry which stores `_text` address, by shifting `_text` address on the `PGDIR_SHIFT`:

```assembly
	movq	%rdi, %rax
	shrq	$PGDIR_SHIFT, %rax
```

where `PGDIR_SHIFT` is `39`. `PGDIR_SHFT` indicates the mask for page global directory bits in a virtual address. There are macro for all types of page directories:

```C
#define PGDIR_SHIFT     39
#define PUD_SHIFT       30
#define PMD_SHIFT       21
```

After this we put the address of the first entry of the `early_dynamic_pgts` page table to the `rdx` register with the `_KERNPG_TABLE` access rights (see above) and fill the `early_level4_pgt` with the 2 `early_dynamic_pgts` entries:

```assembly
	leaq	(4096 + _KERNPG_TABLE)(%rbx), %rdx
	movq	%rdx, 0(%rbx,%rax,8)
	movq	%rdx, 8(%rbx,%rax,8)
```

The `rbx` register contains address of the `early_level4_pgt` and `%rax * 8` here is index of a page global directory occupied by the `_text` address. So here we fill two entries of the `early_level4_pgt` with address of two entries of the `early_dynamic_pgts` which is related to `_text`. The `early_dynamic_pgts` is array of arrays:

```C
extern pmd_t early_dynamic_pgts[EARLY_DYNAMIC_PAGE_TABLES][PTRS_PER_PMD];
```

which will store temporary page tables for early kernel which we will not move to the `init_level4_pgt`.

After this we add `4096` (size of the `early_level4_pgt`) to the `rdx` (it now contains the address of the first entry of the `early_dynamic_pgts`) and put `rdi` (it now contains physical address of the `_text`)  to the `rax`. Now we shift address of the `_text` ot `PUD_SHIFT` to get index of an entry from page upper directory which contains this address and clears high bits to get only `pud` related part:

```assembly
	addq	$4096, %rdx
	movq	%rdi, %rax
	shrq	$PUD_SHIFT, %rax
	andl	$(PTRS_PER_PUD-1), %eax
```

As we have index of a page upper directory we write two addresses of the second entry of the `early_dynamic_pgts` array to the first entry of this temporary page directory:

```assembly
	movq	%rdx, 4096(%rbx,%rax,8)
	incl	%eax
	andl	$(PTRS_PER_PUD-1), %eax
	movq	%rdx, 4096(%rbx,%rax,8)
```

In the next step we do the same operation for last page table directory, but filling not two entries, but all entries to cover full size of the kernel.

After our early page table directories filled, we put physical address of the `early_level4_pgt` to the `rax` register and jump to label `1`:

```assembly
	movq	$(early_level4_pgt - __START_KERNEL_map), %rax
	jmp 1f
```

That's all for now. Our early paging is prepared and we just need to finish last preparation before we will jump into C code and kernel entry point later.

Last preparation before jump at the kernel entry point
--------------------------------------------------------------------------------

After that we jump to the label `1` we enable `PAE`, `PGE` (Paging Global Extension) and put the physical address of the `phys_base` (see above) to the `rax` register and fill `cr3` register with it:

```assembly
1:
	movl	$(X86_CR4_PAE | X86_CR4_PGE), %ecx
	movq	%rcx, %cr4

	addq	phys_base(%rip), %rax
	movq	%rax, %cr3
```

In the next step we check that CPU supports [NX](http://en.wikipedia.org/wiki/NX_bit) bit with:

```assembly
	movl	$0x80000001, %eax
	cpuid
	movl	%edx,%edi
```

We put `0x80000001` value to the `eax` and execute `cpuid` instruction for getting the extended processor info and feature bits. The result will be in the `edx` register which we put to the `edi`.

Now we put `0xc0000080` or `MSR_EFER` to the `ecx` and call `rdmsr` instruction for the reading model specific register.

```assembly
	movl	$MSR_EFER, %ecx
	rdmsr
```

The result will be in the `edx:eax`. General view of the `EFER` is following:

```
63                                                                              32
 --------------------------------------------------------------------------------
|                                                                               |
|                                Reserved MBZ                                   |
|                                                                               |
 --------------------------------------------------------------------------------
31                            16  15      14      13   12  11   10  9  8 7  1   0
 --------------------------------------------------------------------------------
|                              | T |       |       |    |   |   |   |   |   |   |
| Reserved MBZ                 | C | FFXSR | LMSLE |SVME|NXE|LMA|MBZ|LME|RAZ|SCE|
|                              | E |       |       |    |   |   |   |   |   |   |
 --------------------------------------------------------------------------------
```

We will not see all fields in details here, but we will learn about this and other `MSRs` in a special part about it. As we read `EFER` to the `edx:eax`, we check `_EFER_SCE` or zero bit which is `System Call Extensions` with `btsl` instruction and set it to one. By the setting `SCE` bit we enable `SYSCALL` and `SYSRET` instructions. In the next step we check 20th bit in the `edi`, remember that this register stores result of the `cpuid` (see above). If `20` bit is set (`NX` bit) we just write `EFER_SCE` to the model specific register.

```assembly
	btsl	$_EFER_SCE, %eax
	btl	    $20,%edi
	jnc     1f
	btsl	$_EFER_NX, %eax
	btsq	$_PAGE_BIT_NX,early_pmd_flags(%rip)
1:	wrmsr
```

If the [NX](https://en.wikipedia.org/wiki/NX_bit) bit is supported we enable `_EFER_NX`  and write it too, with the `wrmsr` instruction. After the [NX](https://en.wikipedia.org/wiki/NX_bit) bit is set, we set some bits in the `cr0` [control register](https://en.wikipedia.org/wiki/Control_register), namely:

* `X86_CR0_PE` - system is in protected mode;
* `X86_CR0_MP` - controls interaction of WAIT/FWAIT instructions with TS flag in CR0;
* `X86_CR0_ET` - on the 386, it allowed to specify whether the external math coprocessor was an 80287 or 80387;
* `X86_CR0_NE` - enable internal x87 floating point error reporting when set, else enables PC style x87 error detection;
* `X86_CR0_WP` - when set, the CPU can't write to read-only pages when privilege level is 0;
* `X86_CR0_AM` - alignment check enabled if AM set, AC flag (in EFLAGS register) set, and privilege level is 3;
* `X86_CR0_PG` - enable paging.

by the execution following assembly code:

```assembly
#define CR0_STATE	(X86_CR0_PE | X86_CR0_MP | X86_CR0_ET | \
			 X86_CR0_NE | X86_CR0_WP | X86_CR0_AM | \
			 X86_CR0_PG)
movl	$CR0_STATE, %eax
movq	%rax, %cr0
```

We already know that to run any code, and even more [C](https://en.wikipedia.org/wiki/C_%28programming_language%29) code from assembly, we need to setup a stack. As always, we are doing it by the setting of [stack pointer](https://en.wikipedia.org/wiki/Stack_register) to a correct place in memory and resetting [flags](https://en.wikipedia.org/wiki/FLAGS_register) register after this:

```assembly
movq initial_stack(%rip), %rsp
pushq $0
popfq
```

The most interesting thing here is the `initial_stack`. This symbol is defined in the [source](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head_64.S) code file and looks like:

```assembly
GLOBAL(initial_stack)
    .quad  init_thread_union+THREAD_SIZE-8
```

The `GLOBAL` is already familiar to us from. It defined in the [arch/x86/include/asm/linkage.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/linkage.h) header file expands to the `global` symbol definition:

```C
#define GLOBAL(name)    \
         .globl name;           \
         name:
```

The `THREAD_SIZE` macro is defined in the [arch/x86/include/asm/page_64_types.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/page_64_types.h) header file and depends on value of the `KASAN_STACK_ORDER` macro:

```C
#define THREAD_SIZE_ORDER       (2 + KASAN_STACK_ORDER)
#define THREAD_SIZE  (PAGE_SIZE << THREAD_SIZE_ORDER)
```

We consider when the [kasan](http://lxr.free-electrons.com/source/Documentation/kasan.txt) is disabled and the `PAGE_SIZE` is `4096` bytes. So the `THREAD_SIZE` will expands to `16` kilobytes and represents size of the stack of a thread. Why is `thread`? You may already know that each [process](https://en.wikipedia.org/wiki/Process_%28computing%29) may have parent [processes](https://en.wikipedia.org/wiki/Parent_process) and [child](https://en.wikipedia.org/wiki/Child_process) processes. Actually, a parent process and child process differ in stack. A new kernel stack is allocated for a new process. In the Linux kernel this stack is represented by the [union](https://en.wikipedia.org/wiki/Union_type#C.2FC.2B.2B) with the `thread_info` structure.

And as we can see the `init_thread_union` is represented by the `thread_union` [union](https://en.wikipedia.org/wiki/Union_type#C.2FC.2B.2B). Earlier this union looked like:

```C
union thread_union {
         struct thread_info thread_info;
         unsigned long stack[THREAD_SIZE/sizeof(long)];
};
```

but from the Linux kernel `4.9-rc1` release, `thread_info` was moved to the `task_struct` structure which represents a thread. So, for now `thread_union` looks like:

```C
union thread_union {
#ifndef CONFIG_THREAD_INFO_IN_TASK
	struct thread_info thread_info;
#endif
	unsigned long stack[THREAD_SIZE/sizeof(long)];
};
```

where the `CONFIG_THREAD_INFO_IN_TASK` kernel configuration option is enabled for `x86_64` architecture. So, as we consider only `x86_64` architecture in this book, an instance of `thread_union` will contain only stack and `thread_info` structure will be placed in the `task_struct`.

The `init_thread_union` looks like:

```
union thread_union init_thread_union __init_task_data = {
#ifndef CONFIG_THREAD_INFO_IN_TASK
	INIT_THREAD_INFO(init_task)
#endif
};
```

which represents just thread stack. Now we may understand this expression:

```assembly
GLOBAL(initial_stack)
    .quad  init_thread_union+THREAD_SIZE-8
```


that `initial_stack` symbol points to the start of the `thread_union.stack` array + `THREAD_SIZE` which is 16 killobytes and - 8 bytes. Here we need to subtract `8` bytes at the to of stack. This is necessary to guarantee illegal access of the next page memory.

After the early boot stack is set, to update the [Global Descriptor Table](https://en.wikipedia.org/wiki/Global_Descriptor_Table) with the `lgdt` instruction:

```assembly
lgdt	early_gdt_descr(%rip)
```

where the `early_gdt_descr` is defined as:

```assembly
early_gdt_descr:
	.word	GDT_ENTRIES*8-1
early_gdt_descr_base:
	.quad	INIT_PER_CPU_VAR(gdt_page)
```

We need to reload `Global Descriptor Table` because now kernel works in the low userspace addresses, but soon kernel will work in its own space. Now let's look at the definition of `early_gdt_descr`. Global Descriptor Table contains `32` entries:

```C
#define GDT_ENTRIES 32
```

for kernel code, data, thread local storage segments and etc... it's simple. Now let's look at the definition of the `early_gdt_descr_base`.

First of `gdt_page` defined as:

```C
struct gdt_page {
	struct desc_struct gdt[GDT_ENTRIES];
} __attribute__((aligned(PAGE_SIZE)));
```

in the [arch/x86/include/asm/desc.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/desc.h). It contains one field `gdt` which is array of the `desc_struct` structure which is defined as:

```C
struct desc_struct {
         union {
                 struct {
                         unsigned int a;
                         unsigned int b;
                 };
                 struct {
                         u16 limit0;
                         u16 base0;
                         unsigned base1: 8, type: 4, s: 1, dpl: 2, p: 1;
                         unsigned limit: 4, avl: 1, l: 1, d: 1, g: 1, base2: 8;
                 };
         };
 } __attribute__((packed));
```

and presents familiar to us `GDT` descriptor. Also we can note that `gdt_page` structure aligned to `PAGE_SIZE` which is `4096` bytes. It means that `gdt` will occupy one page. Now let's try to understand what is `INIT_PER_CPU_VAR`. `INIT_PER_CPU_VAR` is a macro which defined in the [arch/x86/include/asm/percpu.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/percpu.h) and just concats `init_per_cpu__` with the given parameter:

```C
#define INIT_PER_CPU_VAR(var) init_per_cpu__##var
```

After the `INIT_PER_CPU_VAR` macro will be expanded, we will have `init_per_cpu__gdt_page`. We can see in the [linker script](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/vmlinux.lds.S):

```
#define INIT_PER_CPU(x) init_per_cpu__##x = x + __per_cpu_load
INIT_PER_CPU(gdt_page);
```

As we got `init_per_cpu__gdt_page` in `INIT_PER_CPU_VAR` and `INIT_PER_CPU` macro from linker script will be expanded we will get offset from the `__per_cpu_load`. After this calculations, we will have correct base address of the new GDT.

Generally per-CPU variables is a 2.6 kernel feature. You can understand what it is from its name. When we create `per-CPU` variable, each CPU will have will have its own copy of this variable. Here we creating `gdt_page` per-CPU variable. There are many advantages for variables of this type, like there are no locks, because each CPU works with its own copy of variable and etc... So every core on multiprocessor will have its own `GDT` table and every entry in the table will represent a memory segment which can be accessed from the thread which ran on the core. You can read in details about `per-CPU` variables in the [Theory/per-cpu](http://0xax.gitbooks.io/linux-insides/content/Concepts/per-cpu.html) post.

As we loaded new Global Descriptor Table, we reload segments as we did it every time:

```assembly
	xorl %eax,%eax
	movl %eax,%ds
	movl %eax,%ss
	movl %eax,%es
	movl %eax,%fs
	movl %eax,%gs
```

After all of these steps we set up `gs` register that it post to the `irqstack` which represents special stack where [interrupts](https://en.wikipedia.org/wiki/Interrupt) will be handled on:

```assembly
	movl	$MSR_GS_BASE,%ecx
	movl	initial_gs(%rip),%eax
	movl	initial_gs+4(%rip),%edx
	wrmsr
```

where `MSR_GS_BASE` is:

```C
#define MSR_GS_BASE             0xc0000101
```

We need to put `MSR_GS_BASE` to the `ecx` register and load data from the `eax` and `edx` (which are point to the `initial_gs`) with `wrmsr` instruction. We don't use `cs`, `fs`, `ds` and `ss` segment registers for addressing in the 64-bit mode, but `fs` and `gs` registers can be used. `fs` and `gs` have a hidden part (as we saw it in the real mode for `cs`) and this part contains descriptor which mapped to [Model Specific Registers](https://en.wikipedia.org/wiki/Model-specific_register). So we can see above `0xc0000101` is a `gs.base` MSR address. When a [system call](https://en.wikipedia.org/wiki/System_call) or [interrupt](https://en.wikipedia.org/wiki/Interrupt) occurred, there is no kernel stack at the entry point, so the value of the `MSR_GS_BASE` will store address of the interrupt stack.

In the next step we put the address of the real mode bootparam structure to the `rdi` (remember `rsi` holds pointer to this structure from the start) and jump to the C code with:

```assembly
	movq	initial_code(%rip), %rax
	pushq	$__KERNEL_CS	# set correct cs
	pushq	%rax		# target address in negative space
	lretq
```

Here we put the address of the `initial_code` to the `rax` and push fake address, `__KERNEL_CS` and the address of the `initial_code` to the stack. After this we can see `lretq` instruction which means that after it return address will be extracted from stack (now there is address of the `initial_code`) and jump there. `initial_code` is defined in the same source code file and looks:

```assembly
	.balign	8
	GLOBAL(initial_code)
	.quad	x86_64_start_kernel
	...
	...
	...
```

As we can see `initial_code` contains address of the `x86_64_start_kernel`, which is defined in the [arch/x86/kerne/head64.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head64.c) and looks like this:

```C
asmlinkage __visible void __init x86_64_start_kernel(char * real_mode_data) {
	...
	...
	...
}
```

It has one argument is a `real_mode_data` (remember that we passed address of the real mode data to the `rdi` register previously).

This is first C code in the kernel!

Next to start_kernel
--------------------------------------------------------------------------------

We need to see last preparations before we can see "kernel entry point" - start_kernel function from the [init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c#L489).

First of all we can see some checks in the `x86_64_start_kernel` function:

```C
BUILD_BUG_ON(MODULES_VADDR < __START_KERNEL_map);
BUILD_BUG_ON(MODULES_VADDR - __START_KERNEL_map < KERNEL_IMAGE_SIZE);
BUILD_BUG_ON(MODULES_LEN + KERNEL_IMAGE_SIZE > 2*PUD_SIZE);
BUILD_BUG_ON((__START_KERNEL_map & ~PMD_MASK) != 0);
BUILD_BUG_ON((MODULES_VADDR & ~PMD_MASK) != 0);
BUILD_BUG_ON(!(MODULES_VADDR > __START_KERNEL));
BUILD_BUG_ON(!(((MODULES_END - 1) & PGDIR_MASK) == (__START_KERNEL & PGDIR_MASK)));
BUILD_BUG_ON(__fix_to_virt(__end_of_fixed_addresses) <= MODULES_END);
```

There are checks for different things like virtual addresses of modules space is not fewer than base address of the kernel text - `__STAT_KERNEL_map`, that kernel text with modules is not less than image of the kernel and etc... `BUILD_BUG_ON` is a macro which looks as:

```C
#define BUILD_BUG_ON(condition) ((void)sizeof(char[1 - 2*!!(condition)]))
```

Let's try to understand how this trick works. Let's take for example first condition: `MODULES_VADDR < __START_KERNEL_map`. `!!conditions` is the same that `condition != 0`. So it means if `MODULES_VADDR < __START_KERNEL_map` is true, we will get `1` in the `!!(condition)` or zero if not. After `2*!!(condition)` we will get or `2` or `0`. In the end of calculations we can get two different behaviors:

* We will have compilation error, because try to get size of the char array with negative index (as can be in our case, because `MODULES_VADDR` can't be less than `__START_KERNEL_map` will be in our case);
* No compilation errors.

That's all. So interesting C trick for getting compile error which depends on some constants.

In the next step we can see call of the `cr4_init_shadow` function which stores shadow copy of the `cr4` per cpu. Context switches can change bits in the `cr4` so we need to store `cr4` for each CPU. And after this we can see call of the `reset_early_page_tables` function where we resets all page global directory entries and write new pointer to the PGT in `cr3`:

```C
for (i = 0; i < PTRS_PER_PGD-1; i++)
	early_level4_pgt[i].pgd = 0;

next_early_pgt = 0;

write_cr3(__pa_nodebug(early_level4_pgt));
```

Soon we will build new page tables. Here we can see that we go through all Page Global Directory Entries (`PTRS_PER_PGD` is `512`) in the loop and make it zero. After this we set `next_early_pgt` to zero (we will see details about it in the next post) and write physical address of the `early_level4_pgt` to the `cr3`. `__pa_nodebug` is a macro which will be expanded to:

```C
((unsigned long)(x) - __START_KERNEL_map + phys_base)
```

After this we clear `_bss` from the `__bss_stop` to `__bss_start` and the next step will be setup of the early `IDT` handlers, but it's big concept so we will see it in the next part.

Conclusion
--------------------------------------------------------------------------------

This is the end of the first part about linux kernel initialization.

If you have questions or suggestions, feel free to ping me in twitter [0xAX](https://twitter.com/0xAX), drop me [email](anotherworldofworld@gmail.com) or just create [issue](https://github.com/0xAX/linux-insides/issues/new).

In the next part we will see initialization of the early interruption handlers, kernel space memory mapping and a lot more.

**Please note that English is not my first language and I am really sorry for any inconvenience. If you found any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-insides).**

Links
--------------------------------------------------------------------------------

* [Model Specific Register](http://en.wikipedia.org/wiki/Model-specific_register)
* [Paging](http://0xax.gitbooks.io/linux-insides/content/Theory/Paging.html)
* [Previous part - Kernel decompression](http://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-5.html)
* [NX](http://en.wikipedia.org/wiki/NX_bit)
* [ASLR](http://en.wikipedia.org/wiki/Address_space_layout_randomization)
