커널 부팅 과정. Part 5.
================================================================================

커널 압축 해제
--------------------------------------------------------------------------------

`커널 부팅 과정` 시리즈의 5번째 파트이다. 우리는 이전 [파트](https://github.com/daeseokyoun/linux-insides/blob/master/Booting/linux-bootstrap-4.md#transition-to-the-long-mode)에서 64 비트 모드에서 전환을 살펴 보았고 이 파트에서 이후에 일들을 계속 진행할 것이다. 우리는 커널 압축 해제, 커널 재배치 그리고 직접 커널 압축해제를 위한 준비하기 위한 커널코드에 점프하기 전에 마지막 단계부터 살펴 볼 것이다.

커널 압축해제 전에 준비
--------------------------------------------------------------------------------

우리는 64 비트 엔트리 포인트인 `startup_64`([arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_64.S) 에 있음)로 점프하기 바로 직전에 마무리 지었었다. 우리는 `startup_32` 에서  `startup_64` 로 점프하는 것은 벌써 살펴보았다.:

```assembly
	pushl	$__KERNEL_CS
	leal	startup_64(%ebp), %eax
	...
	...
	...
	pushl	%eax
	...
	...
	...
	lret
```

이전 파트에서, `startup_64` 가 시작했다. 우리는 새로운 Global Descriptor Table 을 로드했고, CPU 는 64 비트 모드로 전환이 되었다. 우리는 데이터 세그먼트들도 설정되는 것을 볼 수 있다.:

```assembly
	.code64
	.org 0x200
ENTRY(startup_64)
	xorl	%eax, %eax
	movl	%eax, %ds
	movl	%eax, %es
	movl	%eax, %ss
	movl	%eax, %fs
	movl	%eax, %gs
```

위의 코드는 `startup_64` 의 시작 부분이다. `cs` 를 포함한 모든 세그먼트 레지스터는 `0x18`의 값을 가지는 `ds` 로 가리키게 된다.(만약 왜 `0x18` 인지 이해할 수 없다면, 이전 파트를 다시 보라.)

다음 단계는 컴파일 단계에서는 컴파일 시점과 실제 로드된 주소의 차이를 계산한다:

```assembly
#ifdef CONFIG_RELOCATABLE
	leaq	startup_32(%rip), %rbp
	movl	BP_kernel_alignment(%rsi), %eax
	decl	%eax
	addq	%rax, %rbp
	notq	%rax
	andq	%rax, %rbp
	cmpq	$LOAD_PHYSICAL_ADDR, %rbp
	jge	1f
#endif
	movq	$LOAD_PHYSICAL_ADDR, %rbp
1:
	leaq	z_extract_offset(%rbp), %rbx
```

`rbp` 레지스터는 압축 해제된 커널의 시작 주소를 담고 있고 `rbx` 레지스터는 압축 해제를 위한 커널 코드를 재배치 하기 위한 주소를 갖고 있다. 우리는 이미 `startup_32` 에서 이와 비슷한 일을 한 것을 이전 파트의 [재배치 주소 계산](https://github.com/daeseokyoun/linux-insides/blob/master/Booting/linux-bootstrap-4.md#calculate-relocation-address) 에서 확인 할 수 있다. 이 계산이 왜 다시 필요하냐면, 부트로더는 64 비트 부트 프로토콜을 사용할 수 있고 `startup_32` 는 단지 이 경우에서는 실행되지 않았기 때문이다.

다음 단계에서는 스택 포인터를 설정하고 플래그 레지스터 들을 리셋하는 것을 확인한다:

```assembly
	leaq	boot_stack_end(%rbx), %rsp

	pushq	$0
	popfq
```

이전 코드에서 봤듯이, `rbx` 레지스터는 커널 압축해제 코드의 시작 주소를 담고 있고, 우리는 이 주소를 `boot_stack_end` 오프셋을 적용하여 스택의 맨 상위를 가리키는 `rsp` 레지스터에 넣어준다. 이 단계 이후에 스택은 제대로 동작할 것이다. 또한 `boot_stack_end` 의 정의는 [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_64.S) 어셈블리 파일의 마지막 부분에 찾아 볼 수 있을 것이다.:

```assembly
	.bss
	.balign 4
boot_heap:
	.fill BOOT_HEAP_SIZE, 1, 0
boot_stack:
	.fill BOOT_STACK_SIZE, 1, 0
boot_stack_end:
```

이것은 `.bss` 섹션의 마지막 부분부터 `.pgtable` 의 바로 전까지 자리를 잡고 있다. 만약 당신이 [arch/x86/boot/compressed/vmlinux.lds.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/vmlinux.lds.S) 링커 스크립트를 본다면, `.bss` 와 `.pgtable` 이 정의되어 있는 것을 찾을 수 잇을 것이다.

스택을 설정하고, 이제는 압축된 커널을 앞서 압축해제된 커널의 재배치 주소를 계산하여 얻은 주소로 복사할 수 있다. 자세한 사항을 살펴보기전에 아래 어셈블리 코드를 보자:

```assembly
	pushq	%rsi
	leaq	(_bss-8)(%rip), %rsi
	leaq	(_bss-8)(%rbx), %rdi
	movq	$_bss, %rcx
	shrq	$3, %rcx
	std
	rep	movsq
	cld
	popq	%rsi
```

맨 먼저 `rsi` 를 스택에 넣는다. 우리는 `rsi` 의 값을 보존할 필요가 있는데, 이유는 이 레지스터가 부팅 관련 데이터를 갖고 있는 real 모드 구조체인 `boot_params` 를 가리키는 포인터를 저장하고 있다.(`boot_params` 는 커널 설정 시작 부분에서 채워진다.) 마미막 부분에 이 코드는 스택에 있었던 `rsi`에 `boot_params` 의 포인터를 되돌려 놓을 것이다.

`rsi` 의 값을 스택에 넣고 바로 다음 두 `leaq` 명령어는 `rip` 와 `rbx` 에 `_bss - 8` 의 값을 오프셋으로 유효 주소(effective address)를 계산하여 각각 `rsi` 와 `rdi` 레지스터에 넣는다. 왜 이 주소들을 계산해야 할까? 실제 압축된 커널이미지는 이 복사 코드(`startup_32` 에서 현재 코드로)와 압축해제 코드 사이에 위치한다. 당신은 링커 스크립트 [arch/x86/boot/compressed/vmlinux.lds.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/vmlinux.lds.S)를 확인해 봄으로써 이 사실을 알 수 있다.:

```
	. = 0;
	.head.text : {
		_head = . ;
		HEAD_TEXT
		_ehead = . ;
	}
	.rodata..compressed : {
		*(.rodata..compressed)
	}
	.text :	{
		_text = .; 	/* Text */
		*(.text)
		*(.text.*)
		_etext = . ;
	}
```

`.head.text` 섹션은 `startup_32` 를 포함한다. 이전 파트에서 아래에 대해 설명했을 것이다.:

```assembly
	__HEAD
	.code32
ENTRY(startup_32)
...
...
...
```

`.text` 섹션은 압축 해제 코드를 포함한다:

```assembly
	.text
relocated:
...
...
...
/*
 * Do the decompression, and jump to the new kernel..
 */
...
```

그리고 `.rodata..compressed` 는 압축된 커널 이미지를 갖고 있다. 그래서 `rsi` 는 `_bss - 8` 의 절대 주소를 갖게 될 것이다, 그리고 `rdi` 는 `_bss - 8` 의 재배치 상태 주소를 갖게 될 것이다. 레지스터에 이 주소들이 저장됨에 따라 우리는 `rcx` 레지스터에 `_bss` 의 주소를 넣는다. `vmlinux.lds.S` 링커 스크립트를 보면, 설정/커널 코드와 연관된 모든 섹션의 끝에 그것이 놓여진 것을 알 수 있다. 우리는 이제 데이터를 `8` 바이트 단위로 `movsq` 명령어를 사용하여 `rsi` 에서 `rdi` 로 복사를 시작할 것이다.

데이터 복사 전에 `std` 명령어가 있을 것이다: 이것은 `DF` 플래그(방향 지정 플래그)를 설정한다, 이것의 의미는 `rsi` 와 `rdi` 가 감소하며 접근될 것이라는 의미이다. 간단히 말하면, 거꾸로 복사진행 한다는 것이다. 마지막으로 `cld` 명령어로 `DF` 플래그를 클리어 하고, `boot_params` 구조체의 포인터를 `rsi` 로 복구 한다.

이제 우리는 재배치 이후에 `.text` 섹션 주소를 갖게 되었고, 그곳으로 점프할 것이다.:

```assembly
	leaq	relocated(%rbx), %rax
	jmp	*%rax
```

커널 압축 해제 전 마지막 준비
--------------------------------------------------------------------------------

이전 절에서 `.text` 섹션은 `relocated` 라벨에서 시작한다는 것을 보았다. 처음으로 하는 일은 `bss` 섹션을 클리어 하는 것이다.:

```assembly
	xorl	%eax, %eax
	leaq    _bss(%rip), %rdi
	leaq    _ebss(%rip), %rcx
	subq	%rdi, %rcx
	shrq	$3, %rcx
	rep	stosq
```

이제 [C](https://en.wikipedia.org/wiki/C_%28programming_language%29) 코드로 점프해야 하기 때문에, `.bss` 섹션을 초기화 할 필요가 있다. `eax` 를 클리어 하고, `_bss` 의 주소를 `rdi` 에, `_ebss` 를 `rcx` 에 넣고, `req stosq` 명령어로 그 영역에 0 을 채워 준다.

마지막으로, `decompress_kernel` 함수를 호출한다.:

```assembly
	pushq	%rsi
	movq	$z_run_size, %r9
	pushq	%r9
	movq	%rsi, %rdi
	leaq	boot_heap(%rip), %rsi
	leaq	input_data(%rip), %rdx
	movl	$z_input_len, %ecx
	movq	%rbp, %r8
	movq	$z_output_len, %r9
	call	decompress_kernel
	popq	%r9
	popq	%rsi
```

다시 `rdi` 를 `boot_params` 구조체를 가리키도록 설정하고 7 개의 인자와 함께 [arch/x86/boot/compressed/misc.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/misc.c) 에 있는 `extract_kernel` 를 호출 한다.:

* `rmode` - 초기 커널 초기화 또는 부트로더에서 채워진 [boot_params](https://github.com/torvalds/linux/blob/master//arch/x86/include/uapi/asm/bootparam.h#L114) 구조체의 포인터
* `heap` - 초기 부트 힙의 시작 주소를 갖고 있는 `boot_heap` 에 대한 포인터
* `input_data` - 압축된 커널의 시작 주소 포인터 또는 다른 말로 `arch/x86/boot/compressed/vmlinux.bin.bz2` 에 대한 포인터
* `input_len` - 압축된 커널의 크기
* `output` - 미래의 압축 해제된 커널의 시작 주소
* `output_len` - 압축 해제된 커널의 크기
* `run_size` - `.bss` 와 `.brk` 를 포함한 커널이 수행될 수 있도록 하는 필요한 공간

모든 인자는 [System V Application Binary Interface](http://www.x86-64.org/documentation/abi.pdf) 에 따라 레지스터를 통해 전달된다. 우리는 커널 압축 해제를 위한 모든 준비를 마치고 커널 압축 해제를 보자.

커널 압축 해제
--------------------------------------------------------------------------------

[arch/x86/boot/compressed/misc.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/misc.c) 에 정의되어 있는 `extract_kernel` 함수가 있고, 7 ~ 8 개의 인자를 받는다. 이 함수는 이전 파트에선 본 비디오/콘솔 초기화화 함께 진행된다. 만약, [real mode](https://en.wikipedia.org/wiki/Real_mode) 이나 부트로더에서 사용된지 안되어 있는지, 부트로더가 32 비트나 64 비트 부트 프로토콜인지 알 수 없기 때문에 이 작업을 다시 해야한다.

이 첫 초기화 단계 다음에, 우리는 가용 메모리의 시작 주소와 마지막 주소의 포인터를 저장한다.:
```C
free_mem_ptr     = heap;
free_mem_end_ptr = heap + BOOT_HEAP_SIZE;
```

`extract_kernel` 함수의 두 번째 인자는 에서 넘어온 [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_64.S) `heap` 이다.:

```assembly
leaq	boot_heap(%rip), %rsi
```

위에서 보면, `boot_heap` 은 아래와 같이 되어 있다:

```assembly
boot_heap:
	.fill BOOT_HEAP_SIZE, 1, 0
```

`BOOT_HEAP_SIZE` 매크로는 `0x10000` 로 확장하고(`bzip2` 커널의 경우 `0x400000` 로) `boot_heap` 은 힙의 크기를 갖는다.

힙 포인터 초기화 이후에, 다음 단계는 [arch/x86/boot/compressed/kaslr.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/kaslr.c#L559) 소스파일에 구현된 `choose_random_location` 함수 호출이다. 함수 이름에서 유추 가능한 사항으로, 커널 이미지가 압축해제된 메모리 위치를 선택한다. 우리가 압축된 이미지를 해제하는 메모리 공간을 찾고 선택을 하는 것이 이상하게 보일 수도 있으나, 리눅스 커널은 커널을 보안의 이유로, 랜덤 주소에 압축 해제 하는 [kASLR](https://en.wikipedia.org/wiki/Address_space_layout_randomization) 을 허용한다. [arch/x86/boot/compressed/kaslr.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/kaslr.c#L559) 소스 코드 파일을 열어 `choose_random_location` 함수를 보자.

첫째로, `choose_random_location` 함수는 `CONFIG_HIBERNATION` 옵션이 설정되어 있다면, 리눅스 커널 명령 라인을 확인하여 `kaslr` 옵션을 찾는다. `CONFIG_HIBERNATION`이 설정되어 있지 않다면 `nokaslr` 을 찾는다.:

```C
#ifdef CONFIG_HIBERNATION
	if (!cmdline_find_option_bool("kaslr")) {
		debug_putstr("KASLR disabled by default...\n");
		goto out;
	}
#else
	if (cmdline_find_option_bool("nokaslr")) {
		debug_putstr("KASLR disabled by cmdline...\n");
		goto out;
	}
#endif
```

커널 구성 옵션에서 `CONFIG_HIBERNATION` 이 활성화 되어 있고 커널 명령 라인에서 `kaslr` 옵션이 없다면, 커널은 `KASLR disabled by default...` 출력하고 `out` 라벨로 점프한다:

```C
out:
	return (unsigned char *)choice;
```

`choose_random_location` 함수로 넘어온 `output` 인자는 변경되지 않고 그냥 리턴된다. 만약 `CONFIG_HIBERNATION` 커널 구성 옵션이 비활성화 상태고 커널 명령 라인의 `nokaslr` 옵션이 있다면 `out` 라벨로 점프한다.
-현재 `CONFIG_HIBERNATION` 정의에 대한 구분은 없는 상태이며, `nokaslr` 옵션 확인만 남아 있는 상태이다.-

현재는, `kASLR` 을 이해하기 위해, 커널은 주소 랜덤 접근이 활성화 되어 옵션이 구성되어 있다고 가정한다. 이것과 관련된 내용을 이 [문서](https://github.com/torvalds/linux/blob/master/Documentation/admin-guide/kernel-parameters.txt#L1723)에서 찾아 볼 수 있다.:

```
kaslr/nokaslr [X86]

Enable/disable kernel and module base offset ASLR
(Address Space Layout Randomization) if built into
the kernel. When CONFIG_HIBERNATION is selected,
kASLR is disabled by default. When kASLR is enabled,
hibernation will be disabled.

최신 버전에서는 kaslr 이 없어짐.
nokaslr		[KNL]
		When CONFIG_RANDOMIZE_BASE is set, this disables
		kernel and module base offset ASLR (Address Space
		Layout Randomization).

		CONFIG_RANDOMIZE_BASE 이 설정되어 있다면, 이것은 커널과 모듈의 기본 오프셋을
		ASLR 하지 않는다.
```

`kaslr` 옵션은 커널 명령 라인에서 넘어오고 압축 해제 커널을 위한 랜던 주소를 얻게 한다.(당신은 ASLR 에 대한 내용을 [여기](https://en.wikipedia.org/wiki/Address_space_layout_randomization)에서 확인 해보자). 그리고, 현재 우리의 목표는 리눅스 커널을 압축 해제 하기 위해 `safely` 한 랜덤 주소를 어떻게 찾는 냐는 것이다. `safely` 란 말을 다시 한번 확인하자. 이것은 문맥상 어떤 의미를 가지는가? 당신은 커널 이미지를 직접적으로 안전하지 않는 메모리의 위치에 압축해제하는 코드를 기억할 것이다. 예를 들면, [initrd](https://en.wikipedia.org/wiki/Initrd) 이미지도 메모리에 있기 때문에 압축 해제된 커널이 그것을 덮어쓰면 안되는 것이다.

다음 함수는 압축 해제시에 안전한 위치를 찾을 수 있도록 도와주는 함수를 알아보자. 이 함수는 `mem_avoid_init` 이다. 이 함수는 같은 소스 [파일]((https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/kaslr.c) 에 구현되어 있고, `extract_kernel` 함수에서 본 4 개의 인자를 받는다.:

* `input_data` - 압축된 커널의 시작 포인터, 이 문서의 경우 `arch/x86/boot/compressed/vmlinux.bin.bz2` 의 포인터
* `input_len` - 압축된 커널의 크기
* `output` - 미래에 압축 해제될 커널의 시작 주소
* `output_len` - 압축 해제된 커널의 크기

이 함수의 주요 사항은 `mem_vector` 구조체의 배열을 채우는 것이다.:

```C
#define MEM_AVOID_MAX 5

static struct mem_vector mem_avoid[MEM_AVOID_MAX];
```

`mem_vector`는 안전하지 않은(?) 메모리 영역에 대한 정보를 담고 있다:

```C
struct mem_vector {
	unsigned long start;
	unsigned long size;
};
```

`mem_avoid_init` 의 구현은 꽤나 간단하다. 이 함수의 일부를 살펴 보자.:

```C
	...
	...
	...
	initrd_start  = (u64)real_mode->ext_ramdisk_image << 32;
	initrd_start |= real_mode->hdr.ramdisk_image;
	initrd_size  = (u64)real_mode->ext_ramdisk_size << 32;
	initrd_size |= real_mode->hdr.ramdisk_size;
	mem_avoid[1].start = initrd_start;
	mem_avoid[1].size = initrd_size;
	...
	...
	...
```

여기서 [initrd](http://en.wikipedia.org/wiki/Initrd) 의 시작 주소와 크기의 계산을 볼 수 있다. `ext_ramdisk_image` 의 32 비트와 설정 헤더로 부터 `ramdisk_image` 의 값으로 `initrd_start`를 계산한다. `initrd_size` 는 `ext_ramdisk_size` 의 `32 비트` 상위 비트와 [boot protocol](https://github.com/torvalds/linux/blob/master/Documentation/x86/boot.txt)에서 `ramdisk_size` 값의 조합으로 계산된다.:

```
Offset	Proto	Name		Meaning
/Size
...
...
...
0218/4	2.00+	ramdisk_image	initrd load address (set by boot loader)
021C/4	2.00+	ramdisk_size	initrd size (set by boot loader)
...
```

`ext_ramdisk_image` 와 `ext_ramdisk_size` 는 [Documentation/x86/zero-page.txt](https://github.com/torvalds/linux/blob/master/Documentation/x86/zero-page.txt) 에서 찾아 볼 수 있다.:

```
Offset	Proto	Name		Meaning
/Size
...
...
...
0C0/004	ALL	ext_ramdisk_image ramdisk_image high 32bits
0C4/004	ALL	ext_ramdisk_size  ramdisk_size high 32bits
...
```

우리는 `ext_ramdisk_image` 와 `ext_ramdisk_size` 를 `32` 비트 왼쪽 쉬프트 해서 사용했고(이제 상위 32 비트를 하위 32 비트로 사용한다), `initrd`의 시작 주소와 크기를 얻을 수 있다. 이것 이후에 이 값들을 `mem_avoid` 배열에 저장한다.

`mem_avoid` 배열에 모든 안전하지 않는 메모리 영역을 모은 다음 단계는 안전하지 않는 영역과 겹치지 않도록 랜덤한 주소를 찾는 것이다. 그 작업을 위해 `find_random_phys_addr` 를 호출 하고 제일 처음 하는 것은 결과 주소를 정렬하는 것이다.:

```C
minimum = ALIGN(minimum, CONFIG_PHYSICAL_ALIGN);
```

`CONFIG_PHYSICAL_ALIGN` 구성 옵션을 이전 파트에서 봤을 것이다. 이 옵션은 커널이 정렬되어야 하는 값을 제공하고, 그 값은 기본으로 `0x200000` 이다. 일단 정렬된 minimum 주소를 가졌다면, BIOS [e820](https://en.wikipedia.org/wiki/E820) 서비스를 이용해서 메모리 영역을 얻고 압축 해제된 커널 이미지에 적합안 영역을 확인 한다.:

```C
for (i = 0; i < real_mode->e820_entries; i++) {
	process_e820_entry(&real_mode->e820_map[i], minimum, size);
}
```

[커널 부팅 과정 파트 2 의 메모리 검출](https://github.com/daeseokyoun/linux-insides/blob/master/Booting/linux-bootstrap-2.md#메모리-검출) 에서 `e820_entries` 의 값을 찾았을 것이다. `process_e820_entry` 함수는 `e820` 메모리 영역이 `non-RAM` 아닌지 확인하고, 메모리 영역의 시작 주소가 허용된 `kaslr` 오프셋의 최대값보다 크지 않는지 확인한다. 그리고 메모리 영역은 `minimum` 위치보다는 커야 한다.:
```C
struct mem_vector region, img;

if (entry->type != E820_RAM)
	return;

if (entry->addr >= CONFIG_RANDOMIZE_BASE_MAX_OFFSET)
	return;

if (entry->addr + entry->size < minimum)
	return;
```

이 다음에, 우리는 `e820` 메모리 영역의 시작 주소와 크기를 `mem_vector` 구조체 안에 저장한다.(위에서 이 구조체에 대한 정의를 살펴보았다.):

```C
region.start = entry->addr;
region.size = entry->size;
```

이 값들이 저장되면, `find_random_phys_addr` 에서 했던 것과 마찬가지로 `region.start` 를 정렬하고 원래 메모리 영역을 벗어나진 않았는지 검사한다.:

```C
region.start = ALIGN(region.start, CONFIG_PHYSICAL_ALIGN);

if (region.start > entry->addr + entry->size)
	return;
```

In the next step, we reduce the size of the memory region to not include rejected regions at the start, and ensure that the last address in the memory region is smaller than `CONFIG_RANDOMIZE_BASE_MAX_OFFSET`, so that the end of the kernel image will be less than the maximum `aslr` offset:
다음 단계, 우리는 시작으로써 배척된 영역을 제외하기 위해 메모리 영역의 크기를 줄이고 `CONFIG_RANDOMIZE_BASE_MAX_OFFSET` 보다 작은 주소의 메모리 영역안에 마지막 주소가 있도록 해야 한다.

```C
region.size -= region.start - entry->addr;

if (region.start + region.size > CONFIG_RANDOMIZE_BASE_MAX_OFFSET)
		region.size = CONFIG_RANDOMIZE_BASE_MAX_OFFSET - region.start;
```

마침내, 우리는 모든 안전하지 않는 메모리 영역을 뒤졌고, 우리가 원하는 영역은 안전하지 않는 영역(커널 명령 라인, initrd 등)에 들어가지 않음을 확인했다.:
```C
for (img.start = region.start, img.size = image_size ;
	     mem_contains(&region, &img) ;
	     img.start += CONFIG_PHYSICAL_ALIGN) {
		if (mem_avoid_overlap(&img))
			continue;
		slots_append(img.start);
	}
```

만약 메모리 영역이 안전하지 않은 영역과 겹치지 않는다면, 영역의 시작 주소와 함께 `slots_append` 함수를 호출한다. `slots_append` 함수는 `slots` 배열로 메모리 영역의 시작 주소를 모으는 일을 한다:

```C
slots[slot_max++] = addr;
```

which is defined as:

```C
static unsigned long slots[CONFIG_RANDOMIZE_BASE_MAX_OFFSET /
			   CONFIG_PHYSICAL_ALIGN];
static unsigned long slot_max;
```

`process_e820_entry` 가 끝나면, 우리는 압축 해제를 위한 안전한 주소의 배열을 갖게 된다. 이 배열 내에서 랜덤한 값을 얻도록 하는 `slots_fetch_random` 함수를 다음으로 호출한다:
```C
if (slot_max == 0)
	return 0;

return slots[get_random_long() % slot_max];
```

where `get_random_long` function checks different CPU flags as `X86_FEATURE_RDRAND` or `X86_FEATURE_TSC` and chooses a method for getting random number (it can be the RDRAND instruction, the time stamp counter, the programmable interval timer, etc...). After retrieving the random address, execution of the `choose_random_location` is finished.
`get_random_long` 함수를 호출 하여 CPU 플래그인 `X86_FEATURE_RDRAND` 또는 `X86_FEATURE_TSC` 확인하여, 랜덤 숫자를 얻기 위한 방법을 선택한다.(RDRAND 명령어, 타임 스탬프 카운터, 프로그램어블 인터벌 타이머(PIC) 중 하나가 될 수 있다.) 랜덤 주소를 얻고 나서는 `choose_random_location` 호출을 마무리 한다.

다시 [misc.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/misc.c#L380)로 돌아오자.커널 이미지를 위한 주소를 얻고 나서, 얻은 랜덤 주소가 알맞게 정렬되어 있는지 주소가 틀린것은 아닌지 확인을 해야한다.

모든 것을 마치면, 우리는 아래와 같은 메세지를 볼 것이다:

```
Decompressing Linux...
```

그리고 커널 압축 해제를 위한 `__decompress` 함수를 호출한다. `__decompress` 함수는 어떤 압축 해제 알고리즘을 사용하는냐에 따라 다른 소스 파일들이 선택되어 빌드될 것이다.:

```C
#ifdef CONFIG_KERNEL_GZIP
#include "../../../../lib/decompress_inflate.c"
#endif

#ifdef CONFIG_KERNEL_BZIP2
#include "../../../../lib/decompress_bunzip2.c"
#endif

#ifdef CONFIG_KERNEL_LZMA
#include "../../../../lib/decompress_unlzma.c"
#endif

#ifdef CONFIG_KERNEL_XZ
#include "../../../../lib/decompress_unxz.c"
#endif

#ifdef CONFIG_KERNEL_LZO
#include "../../../../lib/decompress_unlzo.c"
#endif

#ifdef CONFIG_KERNEL_LZ4
#include "../../../../lib/decompress_unlz4.c"
#endif
```

커널 압축 해제 후에, 이제 마지막으로 두 개의 함수가 있다. 그것은 `parse_elf` 와 `handle_relocations` 함수 이다. 이 함수들의 주요 목적은 압축 해제된 커널 이미지를 제대로 된 메모리 영역에 옮겨 주는 것이다. 압축 해제는 [in-place](https://en.wikipedia.org/wiki/In-place_algorithm) 알고리즘으로 압축 해제 될 것이며, 우리는 커널을 알맞은 주소로 옮겨줘야 할 필요가 있다. 우리가 이미 살펴 봤듯이, 커널 이미지는 [ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) 실행 파일 포멧이다. 그래서 `parse_elf` 함수는 로드 가능한 세그먼트들을 알맞은 주소로 옮겨준다. 우리는 `readelf` 프로그램의 출력으로 로드 가능한 세그먼트 들을 볼 수 있다.:

```
readelf -l vmlinux

Elf file type is EXEC (Executable file)
Entry point 0x1000000
There are 5 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  LOAD           0x0000000000200000 0xffffffff81000000 0x0000000001000000
                 0x0000000000893000 0x0000000000893000  R E    200000
  LOAD           0x0000000000a93000 0xffffffff81893000 0x0000000001893000
                 0x000000000016d000 0x000000000016d000  RW     200000
  LOAD           0x0000000000c00000 0x0000000000000000 0x0000000001a00000
                 0x00000000000152d8 0x00000000000152d8  RW     200000
  LOAD           0x0000000000c16000 0xffffffff81a16000 0x0000000001a16000
                 0x0000000000138000 0x000000000029b000  RWE    200000
```

`parse_elf` 함수의 목표는 이 세그먼트들을 `choose_random_location`에서 얻은 `output` 주소에 로드하는 것이다. 이 함수는 [ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) 사인을 먼저 확인한다.:

```C
Elf64_Ehdr ehdr;
Elf64_Phdr *phdrs, *phdr;

memcpy(&ehdr, output, sizeof(ehdr));

if (ehdr.e_ident[EI_MAG0] != ELFMAG0 ||
   ehdr.e_ident[EI_MAG1] != ELFMAG1 ||
   ehdr.e_ident[EI_MAG2] != ELFMAG2 ||
   ehdr.e_ident[EI_MAG3] != ELFMAG3) {
   error("Kernel is not a valid ELF file");
   return;
}
```

만약 이것이 유효하지 않다면, 에러 메시지를 출력하고 시스템을 중지한다. 만약, 유효한 `ELF` 파일이면, 우리는 `ELF` 파일로 부터 모든 프로그램 헤더를 살펴 보고 모든 로드 가능한 세그먼트 들을 알맞은 주소와 함께 `output` 버퍼에 복사한다.:

```C
	for (i = 0; i < ehdr.e_phnum; i++) {
		phdr = &phdrs[i];

		switch (phdr->p_type) {
		case PT_LOAD:
#ifdef CONFIG_RELOCATABLE
			dest = output;
			dest += (phdr->p_paddr - LOAD_PHYSICAL_ADDR);
#else
			dest = (void *)(phdr->p_paddr);
#endif
			memcpy(dest,
			       output + phdr->p_offset,
			       phdr->p_filesz);
			break;
		default: /* Ignore other PT_* */ break;
		}
	}
```

이제 모두 마무리 되었다. 모든 로드 가능한 세그먼트들은 정확한 위치에 있다. 마지막으로 `handle_relocations` 함수는 커널 이미지 내에 주소들을 조정하고, 만약 `kASLR` 이 활성화 되어 있었다면, 아무것도 하지 않는 함수가 될 것이다.

커널이 재배치 되고 나면, 우리는 [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_64.S) `decompress_kernel` 로 돌아간다. 커널의 주소는 이제 `rax` 레지스터에 있을 것이고, 거기로 점프 할 것이다.:

```assembly
jmp	*%rax
```

이제 커널로 진입했다!

결론
--------------------------------------------------------------------------------

커널 부팅 과정의 마지막 파트가 끝이 났다. 우리는 커널 부팅 과정에 대해서는 더 이상 다루지 않을 것이지만, 커널 내부를 살펴볼 다른 많은 글들이 있을 것이다.

다음 챕터는 커널 초기화에 관련된 것이고, 리눅스 커널 초기화 코드를 살펴 보도록 하자.

당신이 어떤 질문이나 제안이 있다면 [twitter](https://twitter.com/0xAX) - 원저자 에게 알려주길 바란다.

**나는 영어권의 사람이 아니고 이런 것에 대해 매우 미안해 하고 있다. 만약 어떤 실수를 발견한다면, 나에게 PR을 [linux-insides](https://github.com/0xAX/linux-internals)을 보내줘**

링크
--------------------------------------------------------------------------------

* [address space layout randomization](https://en.wikipedia.org/wiki/Address_space_layout_randomization)
* [initrd](http://en.wikipedia.org/wiki/Initrd)
* [long mode](http://en.wikipedia.org/wiki/Long_mode)
* [bzip2](http://www.bzip.org/)
* [RDdRand instruction](http://en.wikipedia.org/wiki/RdRand)
* [Time Stamp Counter](http://en.wikipedia.org/wiki/Time_Stamp_Counter)
* [Programmable Interval Timers](http://en.wikipedia.org/wiki/Intel_8253)
* [Previous part](https://github.com/0xAX/linux-insides/blob/master/Booting/linux-bootstrap-4.md)
* [리눅스 메모리 보호](https://bpsecblog.wordpress.com/2016/05/16/memory_protect_linux_1/)
