Kernel initialization. Part 5.
================================================================================

Continue of architecture-specific initialization
================================================================================

이전 [파트](https://github.com/daeseokyoun/linux-insides/blob/master/Initialization/linux-initialization-4.md) 에서, 우리는 [setup_arch](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/setup.c#L856) 함수에서 아키텍처 의존적인 것들을 초기화하는 과정에서 멈추었고 그것을 이 파트에서 계속진행하려 한다. [initrd](http://en.wikipedia.org/wiki/Initrd) 를 위해 메모리 블럭을 예약했고, 다음 단계는 [One Laptop Per Child support](http://wiki.laptop.org/go/OFW_FAQ) 를 검출하는 `olpc_ofw_detect` 이다. 이 책에서는 플랫폼과 관련된 것은 다루지 않을 것이고 그것과 관련된 함수들은 넘어갈 것이다. 계속 진행해보자. 다음 단계는 `early_trap_init` 함수이다. 이 함수는 디버그와(`#DB` rflags 의 `TF` 플래그를 셋할 때, 올라온다) `int3`(`#BP`) 인터럽트 게이트를 초기화한다. 만약 인터럽트에 대해 아무것도 모른다면, 너는 [초기 인터럽트와 Exception](https://github.com/daeseokyoun/linux-insides/blob/master/Initialization/linux-initialization-2.md) 을 읽어보도록 하자. `x86` 아키텍처에서 `INT`, `INTO` 그리고 `INT3` 는 명시적으로 인터럽트 핸들러를 호출하도록 허가하는 특별한 명령어이다. `INT3` 명령어는 브레이크포인트(`#BP`) 핸들러를 호출한다. 이것 또한 [이전 파트](https://github.com/daeseokyoun/linux-insides/blob/master/Initialization/linux-initialization-2.md) 에서 인터럽트와 예외 처리를 알아보았다.:

```
----------------------------------------------------------------------------------------------
|Vector|Mnemonic|Description         |Type |Error Code|Source                   |
----------------------------------------------------------------------------------------------
|3     | #BP    |Breakpoint          |Trap |NO        |INT 3                    |
----------------------------------------------------------------------------------------------
```

디버그 인터럽트인 `#DB`는 디버거를 수행하는 주요 함수인 [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/traps.c) 에 구현되어 있는 `early_trap_init` 를 보자. 이 함수는 `#DB` 는 `#BP` 핸들러를 설정하고 [IDT](http://en.wikipedia.org/wiki/Interrupt_descriptor_table) 를 재 로드 한다.:

```C
void __init early_trap_init(void)
{
        set_intr_gate_ist(X86_TRAP_DB, &debug, DEBUG_STACK);
        set_system_intr_gate_ist(X86_TRAP_BP, &int3, DEBUG_STACK);
        load_idt(&idt_descr);
}
```

인터럽트와 관련된 `set_intr_gate` 의 구현을 이전 파트에서 이미 보았다. 여기에 또 다른 두개의 비슷한 함수인 `set_intr_gate_ist` 와 `set_system_intr_gate_ist`가 있다. 이 두개의 함수는 3개의 인자를 받는다.:

* 인터럽트 번호
* 인터럽트/예외 핸들러의 베이스 주소
* 세번째 인자는 `Interrupt Stack Table` 이다.  `IST`  는 `x86_64` 에서 새로운 매커니즘이고 [TSS](http://en.wikipedia.org/wiki/Task_state_segment) 의 일부이다. 모든 활성화된 커널 모드 쓰레드들은 `16` 킬로바이트의 자신만의 커널 스택을 가진다. 반면, 유저 영역의 쓰레드는 이 커널 스택은 비어 있다.

per-thread 스택 이외에, 각 CPU 와 연관된 특별한 몇개의 스택들이 있다. 이 모든 스택은 리눅스 커널 문서인 [Kernel stacks](https://www.kernel.org/doc/Documentation/x86/x86_64/kernel-stacks) 를 보자. `x86_64` 는 넌 마스커블 인터럽트등과 같은 어떤 이벤트를 처리하는 동안 새로운 `특별한` 스택으로 전환할 수 있도록 하는 기능을 제공한다. 그리고 기능의 이름을 `Interrupt Stack Table` 라 한다. 각 CPU 마다 7 개의 `IST` 이상의 엔트리들이 있을 수 있고 모든 엔트리 포인트들은 전용의 스택을 갖고 있다. 우리의 경우에는 `DEBUG_STACK` 이다.

`set_intr_gate_ist` 와 `set_system_intr_gate_ist` 는 단지 하나의 차이점만 있지 기능적으로는 `set_intr_gate` 와 같다. 이 두 함수는 인터럽트 번호확인과 내부적으로 `_set_gate`를 호출한다.:

```C
BUG_ON((unsigned)n > 0xFF);
_set_gate(n, GATE_INTERRUPT, addr, 0, ist, __KERNEL_CS__); // TODO 마지막 언더바 2개
```

`set_intr_gate`도 이와 같은 일을 한다. 하지만, `set_intr_gate` 는 [dpl](http://en.wikipedia.org/wiki/Privilege_level) 0과 함께 `_set_gate` 호출하고, ist - 0, 하지만 `set_intr_gate_ist` 와 `set_system_intr_gate_ist`는 `ist`로 `DEBUG_STACK` 설정하고 `set_system_intr_gate_ist`는 `dpl` 을 가장 낮은 특권인 `0x3` 으로 설정한다. 인터럽트가 발생하고 하드웨어가 디스크립터를 로드할 때, 하드웨어는 자동적으로 IST 값을 기반으로 새로운 스택 포인터로 설정하고, 그 다음 인터럽트를 실행한다. 모든 특별한 커널 스택은 `cpu_init` 함수내에서 설정될 것이다.(우리는 나중에 볼 것이다.)

`idt_descr` 쓰여진 `#DB` 와 `#BP` 게이트는, `ldtr` 명령어를 단지 호출하는 `load_idt` 으로 `IDT` 테이블을 재로드 한다. 이제 인터럽트 핸들러를 봤고, 그것들이 어떻게 동작하는지 이해해보도록 하자. 물론, 여기서 모든 인터럽트 핸들러를 알아볼 수 없다. 그것은 리눅스 커널 소스에서 매우 흥미로운 부분이며, 이 파트에서는 `debug` 핸들어가 어떻게 구현이 되어 있는지 볼 것이고, 다른 인터럽트 핸들러의 구현 방식은 당신이 차차 알아보면 좋을 것 이다.

#DB handler
--------------------------------------------------------------------------------

위에서 봤듯이, 우리는 `#DB` 핸들러의 주소인 `&debug` 를 `set_intr_gate_ist` 함수로 넘긴다. [lxr.free-electrons.com](http://lxr.free-electrons.com/ident) 는 리눅스 커널 소스에서 심볼들을 찾는데 아주 훌륭한 사이트이다. 하지만 안타깝게도 `debug` 핸들러는 찾아볼 수 없을 것이다. `debug` 의 정의는 [arch/x86/include/asm/traps.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/traps.h) 에 있다.:

```C
asmlinkage void debug(void);
```

`debug` 함수는 [assembly](http://en.wikipedia.org/wiki/Assembly_language) 와 함께 작성된 것이라고 알려주는 `asmlinkage` 속성을 볼 수 있다. 다른 핸들러와 마찬가지로 `#DB` 핸들러의 구현은 [arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/entry/entry_64.S) 에 있고, `idtentry` 어셈블리 매크로와 함께 정의되어 있다.:

```assembly
idtentry debug do_debug has_error_code=0 paranoid=1 shift_ist=DEBUG_STACK
```

`idtentry` 는 인터럽트/예외 엔트리 포인트를 정의하는 매크로이다. 이것은 5 개의 인자를 받는다.:

* 인터럽트 엔트리 포인트의 이름
* 인터럽트 핸들러의 이름
* 인터럽트 에러 코드가 있는지에 대한 여부
* paranoid  - 만약 1이면, 특별한 스택으로 전환 (위에서 다시 보자)
* shift_ist - 인터럽트 동안에 스택 전환

이제 `idtentry` 매크로 구현을 살펴보자. 이 매크로는 entry_64.S 파일에 정의되어 있고, `debug` 함수를 `ENTRY` 매크로를 이용해서 정의한다. `idtentry` 는 스페셜 스택으로 전환할 필요한 경우에 주어진 인자들이 알맞은 것인지 확인한다. 다음 단계로는 인터럽트가 에러 코드를 반환하는지 확인한다. 만약 인터럽트가 에러코드를 반환하지 않는다면(우리의 경우에 `#DB` 핸들러는 에러코드를 반환하지 않는다), `INTR_FRAME` 호출하고, 만약 인터럽트가 에러 코드를 가진다면 `XCPT_FRAME` 를 호출한다. `XCPT_FRAME` 와 `INTR_FRAME` 매크로는 아무것도 하지 않고 인터럽트를 위해 초기 프레임 상태를 만들기 위해 필요한 것이다. 그것들은 `CFI`(Call Frame Infomation) 지시문을 디버깅을 위해 사용한다.  [CFI directives](https://sourceware.org/binutils/docs/as/CFI-directives.html) 에서 자세한 사항을 볼 수 있을 것이다. [arch/x86/kernel/entry_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/entry/entry_64.S) 에 관련 커멘트를 보면, `CFI 매크로는 더 나은 백트레이스를 위해 dwarf2 unwind 정보를 생성할 때 사용된다. 어떤 코드도 변경하지 않는다.` 그래서 우리는 이것들을 무시할 것이다.

```assembly
.macro idtentry sym do_sym has_error_code:req paranoid=0 shift_ist=-1
ENTRY(\sym)
	/* Sanity check */
	.if \shift_ist != -1 && \paranoid == 0
	.error "using shift_ist requires paranoid=1"
	.endif

	.if \has_error_code
	XCPT_FRAME
	.else
	INTR_FRAME
	.endif
	...
	...
	...
```

당신은 이전 파트에서 인터럽트 발생 후에 초기 인터럽트/예외 핸들링에 관한 내용을 기억할 수 있고, 현재 스택은 아래의 포멧으로 구성될 것이다.:

```
    +-----------------------+
    |                       |
+40 |         SS            |
+32 |         RSP           |
+24 |        RFLAGS         |
+16 |         CS            |
+8  |         RIP           |
 0  |       Error Code      | <---- rsp
    |                       |
    +-----------------------+
```

`idtentry` 의 구현에서 다음 두개의 매크로를 볼 수 있다:

```assembly
	ASM_CLAC
	PARAVIRT_ADJUST_EXCEPTION_FRAME
```

첫 `ASM_CLAC` 매크로는 `CONFIG_X86_SMAP` 구성 옵션에 의존적이고 보안을 위해 필요하다. 이것에 관해서는 [여기](https://lwn.net/Articles/517475/)를 참고 바란다. 다음 `PARAVIRT_ADJUST_EXCEPTION_FRAME` 매크로는 Xen-type-exceptions 을 핸들링하기 위한 것이다. (커널 초기화 관련인 이 파트에서는 가상화 관련된 내용은 다루지 않을 것이다.)

다음 코드 조각은 만약 인터럽트가 에러 코드를 가지는지 아닌지 확인하고, 스택에 `$-1` 를 넣는다.(-1 은 `x86_64` 에서 `0xffffffffffffffff` 이다.):

```assembly
	.ifeq \has_error_code
	pushq_cfi $-1
	.endif
```

모든 인터럽트의 스택 일관성을 위한 `dummy` 에러코드가 필요한 것이다. 다음 단계는 스택 포인터로 부터 `$ORIG_RAX-R15` 를 빼준다:

```assembly
	subq $ORIG_RAX-R15, %rsp
```

`ORIRG_RAX`, `R15` 등의 매크로들은 [arch/x86/include/asm/calling.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/calling.h) 에 정의되어 있고 `ORIG_RAX-R15` 는 120 바이트이다. 인터럽트 핸들링을 하는 동안 스택에 모든 레지스터의 값을 저장 할 필요가 있고 범용 레스터의 경우 120 바이트를 차지 하기 때문에, 이렇게 하는 것이다. 범용 레지스터를 위한 스택 설정 이후에, 유저 영역에서 인터럽트가 온것인지 아래와 같이 확인한다:

```assembly
testl $3, CS(%rsp)
jnz 1f
```

여기서 `CS` 의 첫 번째와 두 번째의 비트를 확인한다. `CS` 레지스터는 세그먼트 레지스터를 포함하고 첫 두 비트는 `RPL` 를 위한 것이다. 모든 특권 레벨은 정수로 0-3 까지 값이고, 가장 낮은 번호가 가장 높은 특권을 나타낸다. 그래서 만얀 인터럽트가 커널 모드로 부터 왔다면, `save_paranoid` 를 호출하고, 아니라면 라벨 `1`로 점프한다. `save_paranoid` 에서는 스택에 모든 범용 레지스터를 저장하고, 필요하다면 유저 `gs` 에서 커널 `gs` 로 전환한다.:

```assembly
	movl $1,%ebx
	movl $MSR_GS_BASE,%ecx
	rdmsr
	testl %edx,%edx
	js 1f
	SWAPGS
	xorl %ebx,%ebx
1:	ret
```

다음 단계는 `pt_regs` 포인터를 `rdi` 에 넣고, `rsi` 에 있는 에러코드를 저장하고 만약 인터럽트가 발생했거나 호출이 있었다면, - 우리의 경우 [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/traps.c) 에 있는 `do_debug` 이다.  `do_debug` 는 다른 핸들러와 같이 두개의 인자를 받는다:

* pt_regs - 프로세스의 메모리 영역에 저장되어 있는 CPU 레지스터 셋을 갖고 있는 구조체
* error code - 인터럽트 에러코드.

모든 인터럽트 핸들러가 자신의 일을 마무리 했다면, 스택을 복구하고, 유저영역에서의 호출이었다면 다시 복귀하고 `iret` 을 호출하는 `paranoid_exit` 를 호출한다. 이것이 여기서 설명하는 전부이다. 하지만 우리는 다른 챕터에서 인터럽트에 관해 깊게 살펴보도록 하자.

이것은 `#DB` 인터럽트를 위한 `idtentry` 매크로의 일반적인 설명이다. 모든 인터럽트는 idtentry 와 함께 구현된 사항과 비슷할 것이다. `early_trap_init` 이 마무리되면, 다음 함수는 `early_cpu_init` 이다. 이 함수는 [arch/x86/kernel/cpu/common.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/cpu/common.c) 에 정의되어 있고 CPU 와 벤더 정보를 모은다.

Early ioremap initialization
--------------------------------------------------------------------------------

The next step is initialization of early `ioremap`. In general there are two ways to communicate with devices:

* I/O Ports;
* Device memory.

We already saw first method (`outb/inb` instructions) in the part about linux kernel booting [process](http://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-3.html). The second method is to map I/O physical addresses to virtual addresses. When a physical address is accessed by the CPU, it may refer to a portion of physical RAM which can be mapped on memory of the I/O device. So `ioremap` used to map device memory into kernel address space.

As i wrote above next function is the `early_ioremap_init` which re-maps I/O memory to kernel address space so it can access it. We need to initialize early ioremap for early initialization code which needs to temporarily map I/O or memory regions before the normal mapping functions like `ioremap` are available. Implementation of this function is in the [arch/x86/mm/ioremap.c](https://github.com/torvalds/linux/blob/master/arch/x86/mm/ioremap.c). At the start of the `early_ioremap_init` we can see definition of the `pmd` point with `pmd_t` type (which presents page middle directory entry `typedef struct { pmdval_t pmd; } pmd_t;` where `pmdval_t` is `unsigned long`) and make a check that `fixmap` aligned in a correct way:

```C
pmd_t *pmd;
BUILD_BUG_ON((fix_to_virt(0) + PAGE_SIZE) & ((1 << PMD_SHIFT) - 1));
```

`fixmap` - is fixed virtual address mappings which extends from `FIXADDR_START` to `FIXADDR_TOP`. Fixed virtual addresses are needed for subsystems that need to know the virtual address at compile time. After the check `early_ioremap_init` makes a call of the `early_ioremap_setup` function from the [mm/early_ioremap.c](https://github.com/torvalds/linux/blob/master/mm/early_ioremap.c). `early_ioremap_setup` fills `slot_virt` array of the `unsigned long` with virtual addresses with 512 temporary boot-time fix-mappings:

```C
for (i = 0; i < FIX_BTMAPS_SLOTS; i++)
    slot_virt[i] = __fix_to_virt(FIX_BTMAP_BEGIN - NR_FIX_BTMAPS*i);
```

After this we get page middle directory entry for the `FIX_BTMAP_BEGIN` and put to the `pmd` variable, fills `bm_pte` with zeros which is boot time page tables and call `pmd_populate_kernel` function for setting given page table entry in the given page middle directory:

```C
pmd = early_ioremap_pmd(fix_to_virt(FIX_BTMAP_BEGIN));
memset(bm_pte, 0, sizeof(bm_pte));
pmd_populate_kernel(&init_mm, pmd, bm_pte);
```

That's all for this. If you feeling puzzled, don't worry. There is special part about `ioremap` and `fixmaps` in the [Linux Kernel Memory Management. Part 2](https://github.com/0xAX/linux-insides/blob/master/mm/linux-mm-2.md) chapter.

Obtaining major and minor numbers for the root device
--------------------------------------------------------------------------------

After early `ioremap` was initialized, you can see the following code:

```C
ROOT_DEV = old_decode_dev(boot_params.hdr.root_dev);
```

This code obtains major and minor numbers for the root device where `initrd` will be mounted later in the `do_mount_root` function. Major number of the device identifies a driver associated with the device. Minor number referred on the device controlled by driver. Note that `old_decode_dev` takes one parameter from the `boot_params_structure`. As we can read from the x86 linux kernel boot protocol:

```
Field name:	root_dev
Type:		modify (optional)
Offset/size:	0x1fc/2
Protocol:	ALL

  The default root device device number.  The use of this field is
  deprecated, use the "root=" option on the command line instead.
```

Now let's try to understand what `old_decode_dev` does. Actually it just calls `MKDEV` inside which generates `dev_t` from the give major and minor numbers. It's implementation is pretty simple:

```C
static inline dev_t old_decode_dev(u16 val)
{
         return MKDEV((val >> 8) & 255, val & 255);
}
```

where `dev_t` is a kernel data type to present major/minor number pair.  But what's the strange `old_` prefix? For historical reasons, there are two ways of managing the major and minor numbers of a device. In the first way major and minor numbers occupied 2 bytes. You can see it in the previous code: 8 bit for major number and 8 bit for minor number. But there is a problem: only 256 major numbers and 256 minor numbers are possible. So 16-bit integer was replaced by 32-bit integer where 12 bits reserved for major number and 20 bits for minor. You can see this in the `new_decode_dev` implementation:

```C
static inline dev_t new_decode_dev(u32 dev)
{
         unsigned major = (dev & 0xfff00) >> 8;
         unsigned minor = (dev & 0xff) | ((dev >> 12) & 0xfff00);
         return MKDEV(major, minor);
}
```

After calculation we will get `0xfff` or 12 bits for `major` if it is `0xffffffff` and `0xfffff` or 20 bits for `minor`. So in the end of execution of the `old_decode_dev` we will get major and minor numbers for the root device in `ROOT_DEV`.

Memory map setup
--------------------------------------------------------------------------------

The next point is the setup of the memory map with the call of the `setup_memory_map` function. But before this we setup different parameters as information about a screen (current row and column, video page and etc... (you can read about it in the [Video mode initialization and transition to protected mode](http://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-3.html))), Extended display identification data, video mode, bootloader_type and etc...:

```C
	screen_info = boot_params.screen_info;
	edid_info = boot_params.edid_info;
	saved_video_mode = boot_params.hdr.vid_mode;
	bootloader_type = boot_params.hdr.type_of_loader;
	if ((bootloader_type >> 4) == 0xe) {
		bootloader_type &= 0xf;
		bootloader_type |= (boot_params.hdr.ext_loader_type+0x10) << 4;
	}
	bootloader_version  = bootloader_type & 0xf;
	bootloader_version |= boot_params.hdr.ext_loader_ver << 4;
```

All of these parameters we got during boot time and stored in the `boot_params` structure. After this we need to setup the end of the I/O memory. As you know one of the main purposes of the kernel is resource management. And one of the resource is memory. As we already know there are two ways to communicate with devices are I/O ports and device memory. All information about registered resources are available through:

* /proc/ioports - provides a list of currently registered port regions used for input or output communication with a device;
* /proc/iomem   - provides current map of the system's memory for each physical device.

At the moment we are interested in `/proc/iomem`:

```
cat /proc/iomem
00000000-00000fff : reserved
00001000-0009d7ff : System RAM
0009d800-0009ffff : reserved
000a0000-000bffff : PCI Bus 0000:00
000c0000-000cffff : Video ROM
000d0000-000d3fff : PCI Bus 0000:00
000d4000-000d7fff : PCI Bus 0000:00
000d8000-000dbfff : PCI Bus 0000:00
000dc000-000dffff : PCI Bus 0000:00
000e0000-000fffff : reserved
  000e0000-000e3fff : PCI Bus 0000:00
  000e4000-000e7fff : PCI Bus 0000:00
  000f0000-000fffff : System ROM
```

As you can see range of addresses are shown in hexadecimal notation with its owner. Linux kernel provides API for managing any resources in a general way. Global resources (for example PICs or I/O ports) can be divided into subsets - relating to any hardware bus slot. The main structure `resource`:

```C
struct resource {
        resource_size_t start;
        resource_size_t end;
        const char *name;
        unsigned long flags;
        struct resource *parent, *sibling, *child;
};
```

presents abstraction for a tree-like subset of system resources. This structure provides range of addresses from `start` to `end` (`resource_size_t` is `phys_addr_t` or `u64` for `x86_64`) which a resource covers, `name` of a resource (you see these names in the `/proc/iomem` output) and `flags` of a resource (All resources flags defined in the [include/linux/ioport.h](https://github.com/torvalds/linux/blob/master/include/linux/ioport.h)). The last are three pointers to the `resource` structure. These pointers enable a tree-like structure:

```
+-------------+      +-------------+
|             |      |             |
|    parent   |------|    sibling  |
|             |      |             |
+-------------+      +-------------+
       |
       |
+-------------+
|             |
|    child    |
|             |
+-------------+
```

Every subset of resources has root range resources. For `iomem` it is `iomem_resource` which defined as:

```C
struct resource iomem_resource = {
        .name   = "PCI mem",
        .start  = 0,
        .end    = -1,
        .flags  = IORESOURCE_MEM,
};
EXPORT_SYMBOL(iomem_resource);
```

TODO EXPORT_SYMBOL

`iomem_resource` defines root addresses range for io memory with `PCI mem` name and `IORESOURCE_MEM` (`0x00000200`) as flags. As i wrote above our current point is setup the end address of the `iomem`. We will do it with:

```C
iomem_resource.end = (1ULL << boot_cpu_data.x86_phys_bits) - 1;
```

Here we shift `1` on `boot_cpu_data.x86_phys_bits`. `boot_cpu_data` is `cpuinfo_x86` structure which we filled during execution of the `early_cpu_init`. As you can understand from the name of the `x86_phys_bits` field, it presents maximum bits amount of the maximum physical address in the system. Note also that `iomem_resource` is passed to the `EXPORT_SYMBOL` macro. This macro exports the given symbol (`iomem_resource` in our case) for dynamic linking or in other words it makes a symbol accessible to dynamically loaded modules.

After we set the end address of the root `iomem` resource address range, as I wrote above the next step will be setup of the memory map. It will be produced with the call of the `setup_ memory_map` function:

```C
void __init setup_memory_map(void)
{
        char *who;

        who = x86_init.resources.memory_setup();
        memcpy(&e820_saved, &e820, sizeof(struct e820map));
        printk(KERN_INFO "e820: BIOS-provided physical RAM map:\n");
        e820_print_map(who);
}
```

First of all we call look here the call of the `x86_init.resources.memory_setup`. `x86_init` is a `x86_init_ops` structure which presents platform specific setup functions as resources initialization, pci initialization and etc... initialization of the `x86_init` is in the [arch/x86/kernel/x86_init.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/x86_init.c). I will not give here the full description because it is very long, but only one part which interests us for now:

```C
struct x86_init_ops x86_init __initdata = {
	.resources = {
            .probe_roms             = probe_roms,
            .reserve_resources      = reserve_standard_io_resources,
            .memory_setup           = default_machine_specific_memory_setup,
    },
    ...
    ...
    ...
}
```

As we can see here `memry_setup` field is `default_machine_specific_memory_setup` where we get the number of the [e820](http://en.wikipedia.org/wiki/E820) entries which we collected in the [boot time](http://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-2.html), sanitize the BIOS e820 map and fill `e820map` structure with the memory regions. As all regions are collected, print of all regions with printk. You can find this print if you execute `dmesg` command and you can see something like this:

```
[    0.000000] e820: BIOS-provided physical RAM map:
[    0.000000] BIOS-e820: [mem 0x0000000000000000-0x000000000009d7ff] usable
[    0.000000] BIOS-e820: [mem 0x000000000009d800-0x000000000009ffff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000000e0000-0x00000000000fffff] reserved
[    0.000000] BIOS-e820: [mem 0x0000000000100000-0x00000000be825fff] usable
[    0.000000] BIOS-e820: [mem 0x00000000be826000-0x00000000be82cfff] ACPI NVS
[    0.000000] BIOS-e820: [mem 0x00000000be82d000-0x00000000bf744fff] usable
[    0.000000] BIOS-e820: [mem 0x00000000bf745000-0x00000000bfff4fff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000bfff5000-0x00000000dc041fff] usable
[    0.000000] BIOS-e820: [mem 0x00000000dc042000-0x00000000dc0d2fff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000dc0d3000-0x00000000dc138fff] usable
[    0.000000] BIOS-e820: [mem 0x00000000dc139000-0x00000000dc27dfff] ACPI NVS
[    0.000000] BIOS-e820: [mem 0x00000000dc27e000-0x00000000deffefff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000defff000-0x00000000deffffff] usable
...
...
...
```

Copying of the BIOS Enhanced Disk Device information
--------------------------------------------------------------------------------

The next two steps is parsing of the `setup_data` with `parse_setup_data` function and copying BIOS EDD to the safe place. `setup_data` is a field from the kernel boot header and as we can read from the `x86` boot protocol:

```
Field name:	setup_data
Type:		write (special)
Offset/size:	0x250/8
Protocol:	2.09+

  The 64-bit physical pointer to NULL terminated single linked list of
  struct setup_data. This is used to define a more extensible boot
  parameters passing mechanism.
```

It used for storing setup information for different types as device tree blob, EFI setup data and etc... In the second step we copy BIOS EDD information from the `boot_params` structure that we collected in the [arch/x86/boot/edd.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/edd.c) to the `edd` structure:

```C
static inline void __init copy_edd(void)
{
     memcpy(edd.mbr_signature, boot_params.edd_mbr_sig_buffer,
            sizeof(edd.mbr_signature));
     memcpy(edd.edd_info, boot_params.eddbuf, sizeof(edd.edd_info));
     edd.mbr_signature_nr = boot_params.edd_mbr_sig_buf_entries;
     edd.edd_info_nr = boot_params.eddbuf_entries;
}
```

Memory descriptor initialization
--------------------------------------------------------------------------------

The next step is initialization of the memory descriptor of the init process. As you already can know every process has its own address space. This address space presented with special data structure which called `memory descriptor`. Directly in the linux kernel source code memory descriptor presented with `mm_struct` structure. `mm_struct` contains many different fields related with the process address space as start/end address of the kernel code/data, start/end of the brk, number of memory areas, list of memory areas and etc... This structure defined in the [include/linux/mm_types.h](https://github.com/torvalds/linux/blob/master/include/linux/mm_types.h). As every process has its own memory descriptor, `task_struct` structure contains it in the `mm` and `active_mm` field. And our first `init` process has it too. You can remember that we saw the part of initialization of the init `task_struct` with `INIT_TASK` macro in the previous [part](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-4.html):

```C
#define INIT_TASK(tsk)  \
{
    ...
	...
	...
	.mm = NULL,         \
    .active_mm  = &init_mm, \
	...
}
```

`mm` points to the process address space and `active_mm` points to the active address space if process has no address space such as kernel threads (more about it you can read in the [documentation](https://www.kernel.org/doc/Documentation/vm/active_mm.txt)). Now we fill memory descriptor of the initial process:

```C
	init_mm.start_code = (unsigned long) _text;
	init_mm.end_code = (unsigned long) _etext;
	init_mm.end_data = (unsigned long) _edata;
	init_mm.brk = _brk_end;
```

with the kernel's text, data and brk. `init_mm` is the memory descriptor of the initial process and defined as:

```C
struct mm_struct init_mm = {
    .mm_rb          = RB_ROOT,
    .pgd            = swapper_pg_dir,
    .mm_users       = ATOMIC_INIT(2),
    .mm_count       = ATOMIC_INIT(1),
    .mmap_sem       = __RWSEM_INITIALIZER(init_mm.mmap_sem),
    .page_table_lock =  __SPIN_LOCK_UNLOCKED(init_mm.page_table_lock),
    .mmlist         = LIST_HEAD_INIT(init_mm.mmlist),
    INIT_MM_CONTEXT(init_mm)
};
```

where `mm_rb` is a red-black tree of the virtual memory areas, `pgd` is a pointer to the page global directory, `mm_users` is address space users, `mm_count` is primary usage counter and `mmap_sem` is memory area semaphore. After we setup memory descriptor of the initial process, next step is initialization of the Intel Memory Protection Extensions with `mpx_mm_init`. The next step is initialization of the code/data/bss resources with:

```C
	code_resource.start = __pa_symbol(_text);
	code_resource.end = __pa_symbol(_etext)-1;
	data_resource.start = __pa_symbol(_etext);
	data_resource.end = __pa_symbol(_edata)-1;
	bss_resource.start = __pa_symbol(__bss_start);
	bss_resource.end = __pa_symbol(__bss_stop)-1;
```

We already know a little about `resource` structure (read above). Here we fills code/data/bss resources with their physical addresses. You can see it in the `/proc/iomem`:

```C
00100000-be825fff : System RAM
  01000000-015bb392 : Kernel code
  015bb393-01930c3f : Kernel data
  01a11000-01ac3fff : Kernel bss
```

All of these structures are defined in the [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/setup.c) and look like typical resource initialization:

```C
static struct resource code_resource = {
	.name	= "Kernel code",
	.start	= 0,
	.end	= 0,
	.flags	= IORESOURCE_BUSY | IORESOURCE_MEM
};
```

The last step which we will cover in this part will be `NX` configuration. `NX-bit` or no execute bit is 63-bit in the page directory entry which controls the ability to execute code from all physical pages mapped by the table entry. This bit can only be used/set when the `no-execute` page-protection mechanism is enabled by the setting `EFER.NXE` to 1. In the `x86_configure_nx` function we check that CPU has support of `NX-bit` and it does not disabled. After the check we fill `__supported_pte_mask` depend on it:

```C
void x86_configure_nx(void)
{
        if (cpu_has_nx && !disable_nx)
                __supported_pte_mask |= _PAGE_NX;
        else
                __supported_pte_mask &= ~_PAGE_NX;
}
```

Conclusion
--------------------------------------------------------------------------------

It is the end of the fifth part about linux kernel initialization process. In this part we continued to dive in the `setup_arch` function which makes initialization of architecture-specific stuff. It was long part, but we have not finished with it. As i already wrote, the `setup_arch` is big function, and I am really not sure that we will cover all of it even in the next part. There were some new interesting concepts in this part like `Fix-mapped` addresses, ioremap and etc... Don't worry if they are unclear for you. There is a special part about these concepts - [Linux kernel memory management Part 2.](https://github.com/0xAX/linux-insides/blob/master/mm/linux-mm-2.md). In the next part we will continue with the initialization of the architecture-specific stuff and will see parsing of the early kernel parameters, early dump of the pci devices, direct Media Interface scanning and many many more.

If you have any questions or suggestions write me a comment or ping me at [twitter](https://twitter.com/0xAX).

**Please note that English is not my first language, And I am really sorry for any inconvenience. If you find any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-insides).**

Links
--------------------------------------------------------------------------------

* [mm vs active_mm](https://www.kernel.org/doc/Documentation/vm/active_mm.txt)
* [e820](http://en.wikipedia.org/wiki/E820)
* [Supervisor mode access prevention](https://lwn.net/Articles/517475/)
* [Kernel stacks](https://www.kernel.org/doc/Documentation/x86/x86_64/kernel-stacks)
* [TSS](http://en.wikipedia.org/wiki/Task_state_segment)
* [IDT](http://en.wikipedia.org/wiki/Interrupt_descriptor_table)
* [Memory mapped I/O](http://en.wikipedia.org/wiki/Memory-mapped_I/O)
* [CFI directives](https://sourceware.org/binutils/docs/as/CFI-directives.html)
* [PDF. dwarf4 specification](http://dwarfstd.org/doc/DWARF4.pdf)
* [Call stack](http://en.wikipedia.org/wiki/Call_stack)
* [Previous part](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-4.html)
