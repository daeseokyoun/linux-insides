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

이 심볼들의 정의를 본다음, 섹션의 시작 부분으로 돌아가보자. `rbp` 레지스터는 커널 [링킹](https://en.wikipedia.org/wiki/Linker_%28computing%29) 과정에서 얻을 수 있는 `startup_64` 심볼의 주소와 실제 주소의 차이를 갖고 있다. 그래서, 어떤 페이지 테이블 엔트리들의 베이스 주소에 이 차이값만 더하면 실제 주소들을 얻을 수 있다. 우리의 경우에 이 엔트리들은 다음과 같다.:

```assembly
	addq	%rbp, early_level4_pgt + (L4_START_KERNEL*8)(%rip)
	addq	%rbp, level3_kernel_pgt + (510*8)(%rip)
	addq	%rbp, level3_kernel_pgt + (511*8)(%rip)
	addq	%rbp, level2_fixmap_pgt + (506*8)(%rip)
```

`early_level4_pgt` 마지막 엔트리는 `level3_kernel_pgt` 이고, `level3_kernel_pgt` 의 마지막 두 개의 엔트리는 `level2_kernel_pgt` 와 `level2_fixmap_pgt` 이다. 그리고 `level2_fixmap_pgt`의 507번째 엔트리는 `level1_fixmap_pgt` 가 된다.

이 모든 것들이 아래와 같이 표현될 수 있다:

```
early_level4_pgt[511] -> level3_kernel_pgt[0]
level3_kernel_pgt[510] -> level2_kernel_pgt[0]
level3_kernel_pgt[511] -> level2_fixmap_pgt[0]
level2_kernel_pgt[0]   -> 512 MB kernel mapping
level2_fixmap_pgt[507] -> level1_fixmap_pgt
```

`early_level4_pgt`와 몇몇의 다른 페이지 테이블 디렉토리들의 베이스 주소를 수정하지 않았다. 이는 이 페이지 테이블의 구조체를 만들거나 채우는 과정에서 볼 수 있기 때문이다. 페이지 테이블의 베이스 주소들을 고친 다음, 페이지 테이블을 빌드할 수 있게 되었다.

Identity 맵핑 설정
--------------------------------------------------------------------------------

초기 페이지 테이블의 identity 맵핑을 설정하는 것을 보자. Identity 맵핑 페이지는, 가상 주소들을 물리 주소로 `1:1`, 같은 값으로 맵핑하기 위한 작업을 한다. 자세하게 살펴보도록 하자. 먼저, 우리는 `_text` 와 `_early_level4_pgt` 의  `rip-상대적인` 주소를 얻고, 그 주소들을 `rdi` 와 `rbx` 레지스터에 넣는다:

```assembly
	leaq	_text(%rip), %rdi
	leaq	early_level4_pgt(%rip), %rbx
```

`_text` 의 주소를 `rax` 레지스터에 저장한 뒤에, `_text` 주소에 `PGDIR_SHIFT` 만큼 우측 쉬프트 하여 페이지 글로벌 디렉토리 엔트리의 인덱스를 얻어온다.:

```assembly
	movq	%rdi, %rax
	shrq	$PGDIR_SHIFT, %rax
```

`PGDIR_SHIFT` 는 `39` 의 값을 가진다. `PGDIR_SHFT` 는 가상 주소에서 페이지 글로벌 디렉토리 비트를 갖고 올 수 있도록 한다. 페이지 디렉토리들의 모든 타입을 위한 매크로가 있다:

```C
#define PGDIR_SHIFT     39
#define PUD_SHIFT       30
#define PMD_SHIFT       21
```

`early_dynamic_pgts` 페이지 테이블의 첫 엔트리의 주소를 `_KERNPG_TABLE` 접근권한과 함께, `rdx` 레지스터에 넣는다. 그리고 2 개의 `early_dynamic_pgts` 엔트리에 그 값을 채워준다.:

```assembly
	leaq	(4096 + _KERNPG_TABLE)(%rbx), %rdx
	movq	%rdx, 0(%rbx,%rax,8)
	movq	%rdx, 8(%rbx,%rax,8)
```

The `rbx` register contains address of the `early_level4_pgt` and `%rax * 8` here is index of a page global directory occupied by the `_text` address. So here we fill two entries of the `early_level4_pgt` with address of two entries of the `early_dynamic_pgts` which is related to `_text`. The `early_dynamic_pgts` is array of arrays:
`rbx` 레지스터는 `early_level4_pgt` 의 주소를 담고 있고, `%rax * 8` 에 `_text` 주소에 의해 점유된 페이지 글로벌 디렉토리의 인텍스가 있다. 그래서 우리는 `_text` 와 연관된 `early_dynamic_pgts` 의 두 개의 엔트리 주소값으로 `early_level4_pgt`의 두개의 엔트리를 채울 것이다. `early_dynamic_pgts`은 이차원 배열이다:

```C
extern pmd_t early_dynamic_pgts[EARLY_DYNAMIC_PAGE_TABLES][PTRS_PER_PMD];
```

초기 커널을 위한 임시의 페이지 테이블을 저장하겠지만, `init_level4_pgt` 에 옮겨지지는 않는다.

`4096` (`early_level4_pgt` 의 크기) 를 `rdx` (이제 `early_dynamic_pgts` 의 첫 엔트리의 주소를 가질 것이다.) 에 더해주고, `rdi` (`_text` 의 물리주소를 갖는다) 를 `rax` 에 넣는다. 이제 우리는 `_text` 의 주소를 `PUD_SHIFT` 만큼 우측 쉬프트를 하여 page upper directory 로 부터 엔트리의 인덱스를 얻을 수 있다. 이는 주소에  `pud` 와 관련된 부분만 얻기 위해 상위 비트를 클리어하여 얻을 수 있다.:

```assembly
	addq	$4096, %rdx
	movq	%rdi, %rax
	shrq	$PUD_SHIFT, %rax
	andl	$(PTRS_PER_PUD-1), %eax
```

`pud` 의 인텍스를 갖고 있기 때문에, 우리는 `early_dynamic_pgts` 배열의 두 번째 엔트리의 주소값을 임시 페이지 디렉토리의 첫번째 엔트리에 써준다.:

```assembly
	movq	%rdx, 4096(%rbx,%rax,8)
	incl	%eax
	andl	$(PTRS_PER_PUD-1), %eax
	movq	%rdx, 4096(%rbx,%rax,8)
```

다음 단계에서는 마지막 페이지 테이블 디렉토리를 위해 같은 작업을 수행 한다, 하지만 두 개의 엔트리를 채우지는 않고, 모든 엔트리는 커널의 전체 크기를 관리한다.

초기 페이지 테이블 디렉토리들을 채운다음, 우리는 `early_level4_pgt` 의 물리주소를 `rax`에 넣고 라벨 `1` 로 점프한다.:
```assembly
	movq	$(early_level4_pgt - __START_KERNEL_map), %rax
	jmp 1f
```

초기 페이징은 준비되었고, 우리는 C 코드와 커널 엔트리 포인트로 들어가기 전에 마지막 준비를 마무리해야 한다.

커널 엔트리 포인트로 점프하기전에 마지막 준비 작업
--------------------------------------------------------------------------------

라벨 `1` 로 점프하고, 우리는 `PAE` 와 `PGE` (Paging Global Extension) 를 활성화 한다. 그리고 `phys_base` 의 물리주소를 `rax` 에 넣고 `cr3`에 그 값을 넣는다:

```assembly
1:
	movl	$(X86_CR4_PAE | X86_CR4_PGE), %ecx
	movq	%rcx, %cr4

	addq	phys_base(%rip), %rax
	movq	%rax, %cr3
```

다음 단계는, 확장된 cpuid 인 [NX](http://en.wikipedia.org/wiki/NX_bit) 비트를 확인한다:

```assembly
	movl	$0x80000001, %eax
	cpuid
	movl	%edx,%edi
```

`0x80000001` 값을 `eax` 에 넣고, 확장된 프로세서의 정보와 관련 정보를 얻기 위해 `cpuid` 명령어를 실행한다. 그러면 그 결과가 `edx` 레지스터로 들어 갈 것이고, 우리는 그 값을 `edi` 에 넣을 것이다.

이제 `0xc0000080` 나 `MSR_EFER` 를 `ecx` 에 넣고 모델 특정 레지스터를 읽기 위해 `rdmsr` 명령어를 호출한다.

```assembly
	movl	$MSR_EFER, %ecx
	rdmsr
```

그 결과는 `edx:eax` 에 저장될 것이다. 일반 적으로 `EFER`는 아래와 같은 항목을 가질 것이다.:

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

여기서는 모든 항목들을 자세히 다루지 않을 것이지만 다른 `MSRs` 관련된 특별한 파트에서 배워볼 수 있을 것이다. `EFER` 를 읽어서 `edx:eax` 로 저장 했고, 우리는 `_EFER_SCE` 필드나 0 비트에 있는  `System Call Extensions` 의 값을 `btsl` 명령어 호출로 확인 하고, 1 로 설정할 것이다. `SCE` 비트를 설정하면, `SYSCALL` 와 `SYSRET` 명령어를 허용할 것이다. 다음 단계는 `edi` 의 20 비트를 확인하는 것이다. 이 레지스터는 `cpuid` 의 결과를 저장(NX 비트 확인을 위함)하고 있다는 것을 기억하자. 만약 `20` 비트가 설정되어 있다면(`NX` 비트), `EFER_SCE` 를 모델 특정 레지스터에 써준다.


```assembly
	btsl	$_EFER_SCE, %eax
	btl	    $20,%edi
	jnc     1f
	btsl	$_EFER_NX, %eax
	btsq	$_PAGE_BIT_NX,early_pmd_flags(%rip)
1:	wrmsr
```

만약 [NX](https://en.wikipedia.org/wiki/NX_bit) 가 지원된다면, 우리는 `_EFER_NX` 활성화 하고 그 값을 `wrmsr` 명령어로 써준다. [NX](https://en.wikipedia.org/wiki/NX_bit)  가 설정된 후에, 우리는 `cr0` [control register](https://en.wikipedia.org/wiki/Control_register) 내에 몇 몇 비트들을 설
정한다:

* `X86_CR0_PE` - 시스템은 보호 모드에 있다.
* `X86_CR0_MP` - CR0 에 있는 TS 비트와 함께 WAIT/FWAIT 명령어를 컨트롤 한다. EM, TS 비트가 활성화 되어 있다면, #NM Exception 발생 시킨다.
* `X86_CR0_ET` - P4, P6 그리고 Xeon 을 위해 예약된 비트. 386/486 에서는, 이 비트가 외부 수학 coprocessor 를 지원하는 지에 대한 것을 나타낸다.
* `X86_CR0_NE` - x87 FPU 연산 에러에 대한 reporting 하는 방법 결정, 만약 설정되어 있다면 internal, 아니라면 PC-style 이다.
* `X86_CR0_WP` - 설정되어 있다면, 특권 레벨 0 에서(supervisor) read-only 페이지에 쓰는 것을 금한다.
* `X86_CR0_AM` - 특권 레벨 3이고 AC 플래그(EFLAGS 레지스터), AM 이 설정되어 있다면, alignment 확인을 활성화 한다.
* `X86_CR0_PG` - 페이징을 사용한다.

아래 어셈블리 코드에 의해 실행된다.:

```assembly
#define CR0_STATE	(X86_CR0_PE | X86_CR0_MP | X86_CR0_ET | \
			 X86_CR0_NE | X86_CR0_WP | X86_CR0_AM | \
			 X86_CR0_PG)
movl	$CR0_STATE, %eax
movq	%rax, %cr0
```

어떤 코드든 그리고, 특히나 어셈블리로 부터 C 코드를 실행하기 위해서는 스택을 설정해야 한다. 메모리에 알맞은 위치의 값으로 [stack pointer](https://en.wikipedia.org/wiki/Stack_register) 의 설정함으로써 설정을 하고, 설정한 다음에는 [EFLAGS](https://en.wikipedia.org/wiki/FLAGS_register) 를 리셋해준다.(0으로 초기화):

```assembly
movq initial_stack(%rip), %rsp
pushq $0
popfq
```

정말 흥미로은 것은 `initial_stack` 이다. 이 심볼은 [소스 코드](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head_64.S)에 선언되어 있고 아래 처럼 생겼다.:

```assembly
GLOBAL(initial_stack)
    .quad  init_thread_union+THREAD_SIZE-8
```

`GLOBAL` 은 이미 우리와 아주 친근하다. 이것은 [arch/x86/include/asm/linkage.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/linkage.h) 헤더에 정의되어 있는데, 외부에서 참조 가능하도록 레이블 생성하고 global 로 선언한다.:

```C
#define GLOBAL(name)    \
         .globl name;           \
         name:
```

`THREAD_SIZE` 매크로는 [arch/x86/include/asm/page_64_types.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/page_64_types.h) 헤더 파일에 정의되어 있고, `KASAN_STACK_ORDER` 매크로의 값에 의존적이다.:

```C
#define THREAD_SIZE_ORDER       (2 + KASAN_STACK_ORDER)
#define THREAD_SIZE  (PAGE_SIZE << THREAD_SIZE_ORDER)
```

우리는 [kasan](http://lxr.free-electrons.com/source/Documentation/kasan.txt) 이 비활성화 되어 있고 `PAGE_SIZE` 가 `4096` 일 때 고려해야 한다. 그래서 `THREAD_SIZE` 가 `16` KB 까지 확장될 수 있고 쓰레드의 스택 크기를 알려준다. 왜 `thread(쓰레드)` 일까? 당신은 각 [프로세스](https://en.wikipedia.org/wiki/Process_%28computing%29)는  [부모 프로세스](https://en.wikipedia.org/wiki/Parent_process) 들과 [자식 프로세스](https://en.wikipedia.org/wiki/Child_process) 를 가질 수 있다는 것을 알고 있을 것이다. 실제로, 하나의 부모 프로세스와 자식 프로세스는 다른 스택을 사용한다. 새로운 프로세스를 위해 새로운 커널 스택이 할당된다. 리눅스 커널은 `thread_info` 구조체와 스택이 함께 사용하는 [union](https://en.wikipedia.org/wiki/Union_type#C.2FC.2B.2B) 타입을 사용한다.

`init_thread_union` 은 `thread_union` 이라는 [union](https://en.wikipedia.org/wiki/Union_type#C.2FC.2B.2B) 타입이라는 것을 볼 수 있다. 초기 이 구조체는 아래와 같이 구성된다.:

```C
union thread_union {
         struct thread_info thread_info;
         unsigned long stack[THREAD_SIZE/sizeof(long)];
};
```

그러나 리눅스 커널 `4.9-rc1` 릴리즈 부터, `thread_info` 변수는 쓰레드를 대표하는 `task_struct` 로 이동되었다. 그래서 이제는 `thread_union` 이 아래 처럼 변경되었다:

```C
union thread_union {
#ifndef CONFIG_THREAD_INFO_IN_TASK
	struct thread_info thread_info;
#endif
	unsigned long stack[THREAD_SIZE/sizeof(long)];
};
```

`CONFIG_THREAD_INFO_IN_TASK` 커널 구성 옵션은 `x86_64` 아키텍처를 위해 활성화된다. 우리는 이 책에서 `x86_64` 아키텍처에 대해서만 살펴보기 때문에, `thread_union` 구조체는 스택만 가지고 있을 것이고, `thread_info` 구조체는 `task_struct` 내에 위치 할 것이다.

`init_thread_union` 는 아래와 같이 되어 있다:

```
union thread_union init_thread_union __init_task_data = {
#ifndef CONFIG_THREAD_INFO_IN_TASK
	INIT_THREAD_INFO(init_task)
#endif
};
```

이것은 단지 thread 스택만 가질 것이다. 이제 우리는 아래의 표현을 이해 해야 한다.:

```assembly
GLOBAL(initial_stack)
    .quad  init_thread_union+THREAD_SIZE-8
```

`initial_stack` 심볼은 `thread_union.stack` 배열 + `THREAD_SIZE`(16 KB - 8 바이트) 의 시작을 가리킨다. 스택의 맨 위에서 8 바이트를 빼줄 필요가 있다. 이것은 근접한 페이지 메모리의 잘못된 접근을 막기 위한 gap 이다.

초기 부트 스택이 설정되면, `lgdt` 명령어로 [Global Descriptor Table](https://en.wikipedia.org/wiki/Global_Descriptor_Table) 를 업데이트 한다:
```assembly
lgdt	early_gdt_descr(%rip)
```

`early_gdt_descr` 이 어떻게 정의되어 있냐면:

```assembly
early_gdt_descr:
	.word	GDT_ENTRIES*8-1
early_gdt_descr_base:
	.quad	INIT_PER_CPU_VAR(gdt_page)
```

커널은 낮은(low) userspace 주소 공간에서 수행되기 때문에 `Global Descriptor Table` 를 재로드 할 필요가 있다. 하지만 곧 커널은 자신만의 주소 공간을 가지고 수행될 것이다. `early_gdt_descr` 의 정의를 보자. `Global Descriptor Table` 은 `32` 개의 엔트리들을 가진다:

```C
#define GDT_ENTRIES 32
```

커널 코드, 데이터, 쓰레드 로컬 스토리지 세그먼트 등을 위해 쓰여질 엔트리들이다. `early_gdt_descr_base` 의 정의를 보자.

[arch/x86/include/asm/desc.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/desc.h)내에 정의된 `gdt_page` 를 보면:

```C
struct gdt_page {
	struct desc_struct gdt[GDT_ENTRIES];
} __attribute__((aligned(PAGE_SIZE)));
```

그것은 `desc_struct` 구조체 배열인 `gdt` 라는 것은 하나의 필드로 가져간다. `desc_struct` 구조체를 살펴 보면:

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

그리고 우리에게 익숙한 `GDT` 디스크립터의 내용을 표현한다. 또한 `gdt_page` 구조체는 `PAGE_SIZE` 인 `4096` 바이트에 정렬되어 있다. 그것은 `gdt` 는 하나의 페이지를 차지한다는 의미이다. 이제 `INIT_PER_CPU_VAR` 가 무엇인지 이해해보도록 하자. `INIT_PER_CPU_VAR` 는 [arch/x86/include/asm/percpu.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/percpu.h) 에 정의된 매크로 이고, 주어진 인자와 단지 `init_per_cpu__` 문자열 연결을 한다.:


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
