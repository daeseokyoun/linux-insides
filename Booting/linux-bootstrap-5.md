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

Last preparation before kernel decompression
--------------------------------------------------------------------------------

In the previous paragraph we saw that the `.text` section starts with the `relocated` label. The first thing it does is clearing the `bss` section with:

```assembly
	xorl	%eax, %eax
	leaq    _bss(%rip), %rdi
	leaq    _ebss(%rip), %rcx
	subq	%rdi, %rcx
	shrq	$3, %rcx
	rep	stosq
```

We need to initialize the `.bss` section, because we'll soon jump to [C](https://en.wikipedia.org/wiki/C_%28programming_language%29) code. Here we just clear `eax`, put the address of `_bss` in `rdi` and `_ebss` in `rcx`, and fill it with zeros with the `rep stosq` instruction.

At the end, we can see the call to the `decompress_kernel` function:

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

Again we set `rdi` to a pointer to the `boot_params` structure and call `decompress_kernel` from [arch/x86/boot/compressed/misc.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/misc.c) with seven arguments:

* `rmode` - pointer to the [boot_params](https://github.com/torvalds/linux/blob/master//arch/x86/include/uapi/asm/bootparam.h#L114) structure which is filled by bootloader or during early kernel initialization;
* `heap` - pointer to the `boot_heap` which represents start address of the early boot heap;
* `input_data` - pointer to the start of the compressed kernel or in other words pointer to the `arch/x86/boot/compressed/vmlinux.bin.bz2`;
* `input_len` - size of the compressed kernel;
* `output` - start address of the future decompressed kernel;
* `output_len` - size of decompressed kernel;
* `run_size` - amount of space needed to run the kernel including `.bss` and `.brk` sections.

All arguments will be passed through the registers according to [System V Application Binary Interface](http://www.x86-64.org/documentation/abi.pdf). We've finished all preparation and can now look at the kernel decompression.

Kernel decompression
--------------------------------------------------------------------------------

As we saw in previous paragraph, the `decompress_kernel` function is defined in the [arch/x86/boot/compressed/misc.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/misc.c) source code file and takes seven arguments. This function starts with the video/console initialization that we already saw in the previous parts. We need to do this again because we don't know if we started in [real mode](https://en.wikipedia.org/wiki/Real_mode) or a bootloader was used, or whether the bootloader used the 32 or 64-bit boot protocol.

After the first initialization steps, we store pointers to the start of the free memory and to the end of it:

```C
free_mem_ptr     = heap;
free_mem_end_ptr = heap + BOOT_HEAP_SIZE;
```

where the `heap` is the second parameter of the `decompress_kernel` function which we got in the [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_64.S):

```assembly
leaq	boot_heap(%rip), %rsi
```

As you saw above, the `boot_heap` is defined as:

```assembly
boot_heap:
	.fill BOOT_HEAP_SIZE, 1, 0
```

where the `BOOT_HEAP_SIZE` is macro which expands to `0x8000` (`0x400000` in a case of `bzip2` kernel) and represents the size of the heap.

After heap pointers initialization, the next step is the call of the `choose_random_location` function from [arch/x86/boot/compressed/kaslr.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/kaslr.c#L425) source code file. As we can guess from the function name, it chooses the memory location where the kernel image will be decompressed. It may look weird that we need to find or even `choose` location where to decompress the compressed kernel image, but the Linux kernel supports [kASLR](https://en.wikipedia.org/wiki/Address_space_layout_randomization) which allows decompression of the kernel into a random address, for security reasons. Let's open the [arch/x86/boot/compressed/kaslr.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/kaslr.c#L425) source code file and look at `choose_random_location`.

First, `choose_random_location` tries to find the `kaslr` option in the Linux kernel command line if `CONFIG_HIBERNATION` is set, and `nokaslr` otherwise:

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

If the `CONFIG_HIBERNATION` kernel configuration option is enabled during kernel configuration and there is no `kaslr` option in the Linux kernel command line, it prints `KASLR disabled by default...` and jumps to the `out` label:

```C
out:
	return (unsigned char *)choice;
```

which just returns the `output` parameter which we passed to the `choose_random_location`, unchanged. If the `CONFIG_HIBERNATION` kernel configuration option is disabled and the `nokaslr` option is in the kernel command line, we jump to `out` again.

For now, let's assume the kernel was configured with randomization enabled and try to understand what `kASLR` is. We can find information about it in the [documentation](https://github.com/torvalds/linux/blob/master/Documentation/kernel-parameters.txt):

```
kaslr/nokaslr [X86]

Enable/disable kernel and module base offset ASLR
(Address Space Layout Randomization) if built into
the kernel. When CONFIG_HIBERNATION is selected,
kASLR is disabled by default. When kASLR is enabled,
hibernation will be disabled.
```

It means that we can pass the `kaslr` option to the kernel's command line and get a random address for the decompressed kernel (you can read more about ASLR [here](https://en.wikipedia.org/wiki/Address_space_layout_randomization)). So, our current goal is to find random address where we can `safely` to decompress the Linux kernel. I repeat: `safely`. What does it mean in this context? You may remember that besides the code of decompressor and directly the kernel image, there are some unsafe places in memory. For example, the [initrd](https://en.wikipedia.org/wiki/Initrd) image is in memory too, and we must not overlap it with the decompressed kernel.

The next function will help us to find a safe place where we can decompress kernel. This function is `mem_avoid_init`. It defined in the same source code [file](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/kaslr.c), and takes four arguments that we already saw in the `decompress_kernel` function:

* `input_data` - pointer to the start of the compressed kernel, or in other words, the pointer to `arch/x86/boot/compressed/vmlinux.bin.bz2`;
* `input_len` - the size of the compressed kernel;
* `output` - the start address of the future decompressed kernel;
* `output_len` - the size of decompressed kernel.

The main point of this function is to fill array of the `mem_vector` structures:

```C
#define MEM_AVOID_MAX 5

static struct mem_vector mem_avoid[MEM_AVOID_MAX];
```

where the `mem_vector` structure contains information about unsafe memory regions:

```C
struct mem_vector {
	unsigned long start;
	unsigned long size;
};
```

The implementation of the `mem_avoid_init` is pretty simple. Let's look on the part of this function:

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

Here we can see calculation of the [initrd](http://en.wikipedia.org/wiki/Initrd) start address and size. The `ext_ramdisk_image` is the high `32-bits` of the `ramdisk_image` field from the setup header, and `ext_ramdisk_size` is the high 32-bits of the `ramdisk_size` field from the [boot protocol](https://github.com/torvalds/linux/blob/master/Documentation/x86/boot.txt):

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

And `ext_ramdisk_image` and `ext_ramdisk_size` can be found in the [Documentation/x86/zero-page.txt](https://github.com/torvalds/linux/blob/master/Documentation/x86/zero-page.txt):

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

So we're taking `ext_ramdisk_image` and `ext_ramdisk_size`, shifting them left on `32` (now they will contain low 32-bits in the high 32-bit bits) and getting start address of the `initrd` and size of it. After this we store these values in the `mem_avoid` array.

The next step after we've collected all unsafe memory regions in the `mem_avoid` array will be searching for a random address that does not overlap with the unsafe regions, using the `find_random_addr` function. First of all we can see the alignment of the output address in the `find_random_addr` function:

```C
minimum = ALIGN(minimum, CONFIG_PHYSICAL_ALIGN);
```

You can remember `CONFIG_PHYSICAL_ALIGN` configuration option from the previous part. This option provides the value to which kernel should be aligned and it is `0x200000` by default. Once we have the aligned output address, we go through the memory regions which we got with the help of the BIOS [e820](https://en.wikipedia.org/wiki/E820) service and collect regions suitable for the decompressed kernel image:

```C
for (i = 0; i < real_mode->e820_entries; i++) {
	process_e820_entry(&real_mode->e820_map[i], minimum, size);
}
```

Recall that we collected `e820_entries` in the second part of the [Kernel booting process part 2](https://github.com/0xAX/linux-insides/blob/master/Booting/linux-bootstrap-2.md#memory-detection). The `process_e820_entry` function does some checks that an `e820` memory region is not `non-RAM`, that the start address of the memory region is not bigger than maximum allowed `aslr` offset, and that the memory region is above the minimum load location:

```C
struct mem_vector region, img;

if (entry->type != E820_RAM)
	return;

if (entry->addr >= CONFIG_RANDOMIZE_BASE_MAX_OFFSET)
	return;

if (entry->addr + entry->size < minimum)
	return;
```

After this, we store an `e820` memory region start address and the size in the `mem_vector` structure (we saw definition of this structure above):

```C
region.start = entry->addr;
region.size = entry->size;
```

As we store these values, we align the `region.start` as we did it in the `find_random_addr` function and check that we didn't get an address that is outside the original memory region:

```C
region.start = ALIGN(region.start, CONFIG_PHYSICAL_ALIGN);

if (region.start > entry->addr + entry->size)
	return;
```

In the next step, we reduce the size of the memory region to not include rejected regions at the start, and ensure that the last address in the memory region is smaller than `CONFIG_RANDOMIZE_BASE_MAX_OFFSET`, so that the end of the kernel image will be less than the maximum `aslr` offset:

```C
region.size -= region.start - entry->addr;

if (region.start + region.size > CONFIG_RANDOMIZE_BASE_MAX_OFFSET)
		region.size = CONFIG_RANDOMIZE_BASE_MAX_OFFSET - region.start;
```

Finally, we go through all unsafe memory regions and check that the region does not overlap unsafe areas, such as kernel command line, initrd, etc...:

```C
for (img.start = region.start, img.size = image_size ;
	     mem_contains(&region, &img) ;
	     img.start += CONFIG_PHYSICAL_ALIGN) {
		if (mem_avoid_overlap(&img))
			continue;
		slots_append(img.start);
	}
```

If the memory region does not overlap unsafe regions we call the `slots_append` function with the start address of the region. `slots_append` function just collects start addresses of memory regions to the `slots` array:

```C
slots[slot_max++] = addr;
```

which is defined as:

```C
static unsigned long slots[CONFIG_RANDOMIZE_BASE_MAX_OFFSET /
			   CONFIG_PHYSICAL_ALIGN];
static unsigned long slot_max;
```

After `process_e820_entry` is done, we will have an array of addresses that are safe for the decompressed kernel. Then we call `slots_fetch_random` function to get a random item from this array:

```C
if (slot_max == 0)
	return 0;

return slots[get_random_long() % slot_max];
```

where `get_random_long` function checks different CPU flags as `X86_FEATURE_RDRAND` or `X86_FEATURE_TSC` and chooses a method for getting random number (it can be the RDRAND instruction, the time stamp counter, the programmable interval timer, etc...). After retrieving the random address, execution of the `choose_random_location` is finished.

Now let's back to [misc.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/misc.c#L404). After getting the address for the kernel image, there need to be some checks to be sure that the retrieved random address is correctly aligned and address is not wrong.

After all these checks we will see the familiar message:

```
Decompressing Linux...
```

and call the `__decompress` function which will decompress the kernel. The `__decompress` function depends on what decompression algorithm was chosen during kernel compilation:

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

After kernel is decompressed, the last two functions are `parse_elf` and `handle_relocations`. The main point of these functions is to move the uncompressed kernel image to the correct memory place. The fact is that the decompression will decompress [in-place](https://en.wikipedia.org/wiki/In-place_algorithm), and we still need to move kernel to the correct address. As we already know, the kernel image is an [ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) executable, so the main goal of the `parse_elf` function is to move loadable segments to the correct address. We can see loadable segments in the output of the `readelf` program:

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

The goal of the `parse_elf` function is to load these segments to the `output` address we got from the `choose_random_location` function. This function starts with checking the [ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) signature:

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

and if it's not valid, it prints an error message and halts. If we got a valid `ELF` file, we go through all program headers from the given `ELF` file and copy all loadable segments with correct address to the output buffer:

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

That's all. From now on, all loadable segments are in the correct place. The last `handle_relocations` function adjusts addresses in the kernel image, and is called only if the `kASLR` was enabled during kernel configuration.

After the kernel is relocated, we return back from the `decompress_kernel` to [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_64.S). The address of the kernel will be in the `rax` register and we jump to it:

```assembly
jmp	*%rax
```

That's all. Now we are in the kernel!

Conclusion
--------------------------------------------------------------------------------

This is the end of the fifth and the last part about linux kernel booting process. We will not see posts about kernel booting anymore (maybe updates to this and previous posts), but there will be many posts about other kernel internals.

Next chapter will be about kernel initialization and we will see the first steps in the Linux kernel initialization code.

If you have any questions or suggestions write me a comment or ping me in [twitter](https://twitter.com/0xAX).

**Please note that English is not my first language, And I am really sorry for any inconvenience. If you find any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-internals).**

Links
--------------------------------------------------------------------------------

* [address space layout randomization](https://en.wikipedia.org/wiki/Address_space_layout_randomization)
* [initrd](http://en.wikipedia.org/wiki/Initrd)
* [long mode](http://en.wikipedia.org/wiki/Long_mode)
* [bzip2](http://www.bzip.org/)
* [RDdRand instruction](http://en.wikipedia.org/wiki/RdRand)
* [Time Stamp Counter](http://en.wikipedia.org/wiki/Time_Stamp_Counter)
* [Programmable Interval Timers](http://en.wikipedia.org/wiki/Intel_8253)
* [Previous part](https://github.com/0xAX/linux-insides/blob/master/Booting/linux-bootstrap-4.md)
