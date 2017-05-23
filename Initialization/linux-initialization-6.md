Kernel initialization. Part 6.
================================================================================

Architecture-specific initialization, again...
================================================================================

이전 [파트](https://github.com/daeseokyoun/linux-insides/blob/master/Initialization/linux-initialization-5.md)에서는 the [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/setup.c)에서 아키텍처 특화된 초기화를 보았고 [NX bit](http://en.wikipedia.org/wiki/NX_bit) 의 지원에 의존적인 `_PAGE_NX` 플래그를 설정하는 `x86_configure_nx` 함수에서 마무리 했다. 누누이 얘기 했듯이 `setup_arch` 함수와 `start_kernel` 은 매우 많은 일을 하는 함수여서, 이 파트와 다음 파트에서 아키텍처 특화된 초기화 과정을 살펴 볼 것이다. `x86_configure_nx` 함수의 다음 호출되는 함수는 `parse_early_param` 이다. 이 함수는 [init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c) 파일에 구현되어 있고 무슨 일을 하는지 이름으로 알 수 있듯이 커널 명령라인을 파싱하고 주어진 인자들과 연관된 다른 서비스를 설정한다. (모든 커널 명령 라인 인자들은 [Documentation/kernel-parameters.txt](https://github.com/torvalds/linux/blob/master/Documentation/kernel-parameters.txt) 파일에서 확인 가능하다.) 거의 초기 [파트](https://github.com/daeseokyoun/linux-insides/blob/korean-trans/Booting/linux-bootstrap-2.md) 에서 어떻게 `earlyprintk` 를 설정했는지 기억할 수 있을 것이다. 초기 단계에서 [arch/x86/boot/cmdline.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/cmdline.c) 에 있는 관련 헬퍼 함수인 `cmdline_find_option`, `__cmdline_find_option`, `__cmdline_find_option_bool` 와 함께 그것들의 값과 커널 인자들을 살펴 보았다. 이제는 아키텍처 의존적이지 않은 일반적인 커널 부분으로 들어왔고 다른 방식으로 커널 인자들을 파싱할 것이다. 만약 커널 소스 코드를 보고 싶다면, 아래와 같이 호출해서 쓴다는 것을 볼 수 있다.:

```C
early_param("gbpages", parse_direct_gbpages_on);
```

`early_param` 매크로는 두개의 인자를 받는다.:

* 명령 라인 인자 이름
* 주어진 인자가 들어오면 불러야 하는 함수

그리고 이 매크로는 [include/linux/init.h](https://github.com/torvalds/linux/blob/master/include/linux/init.h) 에 아래와 같이 구현되어 있다.:

```C
#define early_param(str, fn) \
        __setup_param(str, fn, fn, 1)
```

보시다시피 `early_param` 매크로는 단지 `__setup_param` 매크로를 호출한다.:

```C
#define __setup_param(str, unique_id, fn, early)                \
        static const char __setup_str_##unique_id[] __initconst \
                __aligned(1) = str; \
        static struct obs_kernel_param __setup_##unique_id      \
                __used __section(.init.setup)                   \
                __attribute__((aligned((sizeof(long)))))        \
                = { __setup_str_##unique_id__, fn, early } // TODO id 뒤에 언더바 두개
```

이 매크로는 `__setup_str_*_id` 변수(`*` 은 주어진 함수 이름에 의존적이다.)를 선언하고 그것을 주어진 명령 인자 이름을 할당한다. 다음 라인은 타입은 `obs_kernel_param` 이고, 그것의 초기화하는 `__setup_*` 변수를 선언하는 것을 볼 수 있다. `obs_kernel_param` 구조체는 아래와 같이 선언되어 있다.:

```C
struct obs_kernel_param {
        const char *str;
        int (*setup_func)(char *)*; // TODO 마지막 별
        int early;
};
```

그리고 3개의 필드들을 갖고 있다.:

* 커널 인자의 이름
* 인자에 의존적으로 어떤 일을 하는 함수
* 초기에서 사용(1) 이거나 나중에 사용되거나(0) 를 결정하는 필드

`__set_param` 매크로는 `__section(.init.setup)` 속성으로 선언된다. 그것의 의미는 모든 `__setup_str_*` 는 `.init.setup` 섹션에 위치하게 될 것이고, 그 섹션은 [include/asm-generic/vmlinux.lds.h](https://github.com/torvalds/linux/blob/master/include/asm-generic/vmlinux.lds.h) 파일에서 확인 가능하다. 그래서 그것들은 `__setup_start` 와 `__setup_end` 사이에 놓여진다는 것이다.:

```
#define INIT_SETUP(initsetup_align)                \
                . = ALIGN(initsetup_align);        \
                VMLINUX_SYMBOL(__setup_start) = .; \
                *(.init.setup)                     \
                VMLINUX_SYMBOL(__setup_end) = .;
```

이제 어떻게 인자들이 선언되는지 알았다, `parse_early_param` 구현으로 돌아가보자.:

```C
void __init parse_early_param(void)
{
        static int done __initdata;
        static char tmp_cmdline[COMMAND_LINE_SIZE] __initdata;

        if (done)
                return;

        /* All fall through to do_early_param. */
        strlcpy(tmp_cmdline, boot_command_line, COMMAND_LINE_SIZE);
        parse_early_options(tmp_cmdline);
        done = 1;__ // TODO 마지막 언더바 두개
}
```

`parse_early_param` 함수는 두 개의 정적(static) 변수를 선언한다. 하나는 `done` 으로, `parse_early_param` 함수가 이미 호출되었는지를 확인하게 해주고, 두 번째 변수인 `tmp_cmdline`은 커널 명령 라인의 임시 저장소로 쓰인다. 선언 이후에 이 함수는 `boot_command_line` 을 방금 선언한 임시 명령 라인으로 복사하고 `main.c` 파일에 있는 `parse_early_options` 함수를 호출한다. `parse_early_options` 함수는 [kernel/params.c](https://github.com/torvalds/linux/blob/master/) 에 구현된 주어진 명령 라인을 파싱하고 `do_early_param` 함수를 호출하는 `parse_args`를 호출한다. 이 [함수](https://github.com/torvalds/linux/blob/master/init/main.c#L419)는 ` __setup_start` 부터 ` __setup_end` 까지 루프를 수행하면서 `obs_kernel_param` 으로 이 파라미터가 초기에 쓰이는 것이라면, 관련 설정 함수를 호출한다. 초기 명령 라인 파라미터에 의존적인 모든 서비스들이 설정되면 `parse_early_param` 함수의 다음으로 `x86_report_nx` 를 호출한다. 이미 언급 했듯이, 이 파트의 초반에서 `x86_configure_nx` 와 `NX-bit` 설정을 이미 설정했었다. [arch/x86/mm/setup_nx.c](https://github.com/torvalds/linux/blob/master/arch/x86/mm/setup_nx.c) 파일에 있는 `x86_report_nx` 호출하여 단지 `NX` 관련 정보를 출력한다. `x86_report_nx` 함수는 `x86_configure_nx` 바로 뒤에 호출되지 않고, `parse_early_param` 호출 다음에 불린다. 왜 그런가 하면, 커널은 `noexec` 파라미터를 지원하기 때문에 `parse_early_param` 함수 다음에 `x86_report_nx` 호출해야 한다.:

```
noexec		[X86]
			On X86-32 available only on PAE configured kernels.
			noexec=on: enable non-executable mappings (default)
			noexec=off: disable non-executable mappings

      X86-32 에서는 PAE 가 활성화 되어 있는 커널에서만 가용합니다.
      noexec=on: 실행불가능한 메모리 맵핑을 허용
      noexec=off: 실행불가능한 메모리 맵핑을 허용하지 않음.
```

부팅 때 아래와 같은 내용을 볼 수 있다.:

![NX](http://oi62.tinypic.com/swwxhy.jpg)

이 다음에 호출하는 함수는:

```C
	memblock_x86_reserve_range_setup_data();
```

이 함수는 [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/setup.c) 소스 코드에 구현되어 있으며, `setup_data` 를 위해 메모리 재 맵핑을 하고 `setup_data`를 위해 메모리 블럭을 예약한다. (이전 [파트](https://github.com/daeseokyoun/linux-insides/blob/master/Initialization/linux-initialization-5.md) 에서 `setup_data`, `ioremap` 그리고 `memblock`에 대한 내용을 더 볼 수 있다.)

다음 단계에서 다음과 같은 if 문을 볼 수 있다.:

```C
	if (acpi_mps_check()) {
#ifdef CONFIG_X86_LOCAL_APIC
		disable_apic = 1;
#endif
		setup_clear_cpu_cap(X86_FEATURE_APIC);
	}
```

`acpi_mps_check` 함수는 [arch/x86/kernel/acpi/boot.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/acpi/boot.c) 에 구현되어 있고, `CONFIG_X86_LOCAL_APIC` 와 `CONFIG_x86_MPPARSE` 구성 옵션에 의존적이다.:

```C
int __init acpi_mps_check(void)
{
#if defined(CONFIG_X86_LOCAL_APIC) && !defined(CONFIG_X86_MPPARSE)
        /* mptable code is not built-in*/
        if (acpi_disabled || acpi_noirq) {
                printk(KERN_WARNING "MPS support code is not built-in.\n"
                       "Using acpi=off or acpi=noirq or pci=noacpi "
                       "may have problem\n");
                 return 1;
        }
#endif
        return 0;
}
```

이 함수는 빌트인 `MPS` 혹은 [MultiProcessor Specification](http://en.wikipedia.org/wiki/MultiProcessor_Specification) 테이블을 확인한다. 만약 `CONFIG_X86_LOCAL_APIC` 이 셋되어 있고 `CONFIG_x86_MPPAARSE`은 셋이 되어 있지 않고, 명령라인에서 `acpi=off`, `acpi=noirq` 또는 `pci=noacpi` 가 커널로 넘어왔다면, `acpi_mps_check` 함수는 경고 메세지를 출력할 것이다. 만약 `acpi_mps_check` 가 `1`을 반환한다면, 그것은 로컬 [APIC](http://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller)를 비활성화 한다는 의미이고, 현재 CPU 가 갖고 있는 `X86_FEATURE_APIC` 비트를 `setup_clear_cpu_cap` 매크로를 통해 클리어 할 것이다. (CPU mask 에 대해 더 읽고 싶다면, [CPU mask](https://github.com/daeseokyoun/linux-insides/blob/master/Concepts/cpumask.md) 를 읽어보자.)

초기 PCI 덤프
--------------------------------------------------------------------------------

다음 단계에서는 [PCI](http://en.wikipedia.org/wiki/Conventional_PCI) 장치들의 덤프(dump)를 만드는 것이다.:

```C
#ifdef CONFIG_PCI
	if (pci_early_dump_regs)
		early_dump_pci_devices();
#endif
```

`pci_early_dump_regs` 변수는 [arch/x86/pci/common.c](https://github.com/torvalds/linux/blob/master/arch/x86/pci/common.c) 선언되어 있고, 그것의 값은 커널 명령 라인에서 `pci=earlydump` 의 값에 의존적이다. [drivers/pci/pci.c](https://github.com/torvalds/linux/blob/master/drivers/pci/pci.c) 에서 이 파라미터의 정의를 찾을 수 있다.:
```C
early_param("pci", pci_setup);
```

`pci_setup` 함수는 `pci=` 뒤에 따르는 문자열을 분석한다. 이 함수는 [drivers/pci/pci.c](https://github.com/torvalds/linux/blob/master/drivers/pci/pci.c) 에 `__weak` 로
선언되어 있는 `pcibios_setup` 함수를 호출하고 모든 아키텍처들은 `__weak` 로 선언되어 있는 함수를 오버라이드 하여 같은 함수를 갖고 있다. 예를 들어, `x86_64` 아키텍쳐 의존적인 함수는 [arch/x86/pci/common.c](https://github.com/torvalds/linux/blob/master/arch/x86/pci/common.c) 에 구현되어 있다.:  

```C
char *__init pcibios_setup(char *str) {
        ...
		...
		...
		} else if (!strcmp(str, "earlydump")) {
                pci_early_dump_regs = 1;
                return NULL;
        }
		...
		...
		...
}
```

그러니, 만약 `CONFIG_PCI` 옵션이 설정되어 있고, 커널 명령 라인으로 `pci=earlydump`를 넘긴다면, 다음 함수는 [arch/x86/pci/early.c](https://github.com/torvalds/linux/blob/master/arch/x86/pci/early.c)에 있는 `early_dump_pci_devices` 호출이 될 것이다. 이 함수는 `noearly` pci 파라미터를 아래의 코드로 확인한다.:

```C
if (!early_pci_allowed())
        return;
```

그리고 만약 `noearly` 가 설정되어 있다면, 그냥 리턴한다. 각 PCI 도엔은 `256` 개 버스들을 호스팅 할 수 있고, 각 호스트는 32 개의 장치를 가질 수 있다. 아래의 루프를 보자.:

```C
for (bus = 0; bus < 256; bus++) {
                for (slot = 0; slot < 32; slot++) {
                        for (func = 0; func < 8; func++) {
						...
						...
						...
                        }
                }
}
```

그리고 `read_pci_config` 함수로 `pci` 설정을 읽는다.

`pci` 에 대해 상세히 들어가진 않을 것이다. `Drivers/PCI` 파트를 통해 조금 더 다루어 볼 예정이다.

메모리 파싱 마무리
--------------------------------------------------------------------------------

`early_dump_pci_devices` 호출 이후에, 가용 메모리와 [커널 설정의 첫 단계](https://github.com/daeseokyoun/linux-insides/blob/master/Booting/linux-bootstrap-2.md)에서 확인한 [e820](http://en.wikipedia.org/wiki/E820)에 관련된 함수 몇 개를 더 살펴 볼 것이다.:

```C
	/* update the e820_saved too */
	e820_reserve_setup_data();
	finish_e820_parsing();
	...
	...
	...
	e820_add_kernel_range();
	trim_bios_range(void);
	max_pfn = e820_end_of_ram_pfn();
	early_reserve_e820_mpc_new();
```

이를 위한 첫 번째 함수는 `e820_reserve_setup_data`이다. 이 함수는 이미 살펴본 `memblock_x86_reserve_range_setup_data` 함수와 거의 비슷하지만, 우리의 경우에 `E820_RESERVED_KERN` 의 타입과 함께 `e820map` 에 새로운 영역을 추가하는 `e820_update_range` 를 호출한다. 그 다음으로 `sanitize_e820_map`를 이용하여 `e820map` 영역이 제대로 되어 있는지 확인하는 `finish_e820_parsing` 함수를 호출한다. 이 두 개의 함수들은 [e820](http://en.wikipedia.org/wiki/E820) 과 관련있는 함수들이다. 그리고 위에 추가적으로 리스팅이 되어 있다. `e820_add_kernel_range` 함수는 커널 시작과 끝의 물리주소를 사용한다.:

```C
u64 start = __pa_symbol(_text);
u64 size = __pa_symbol(_end_) - start; // TODO end 뒤에 언더바 하나.
```

`e820map`에서 `.text`, `.data` 그리고 `.bss` 를 `E820RAM`으로 마킹되어 있는지 확인하고, 그렇지 않다면 경고 메세지를 출력한다. 이 다음 함수는 `e820Map` 의 첫 4096 바이트를 `E820_RESERVED` 로 업데이트 하고, 그것을 `sanitize_e820_map` 함수의 호출로 정상적으로 되었는지 확인한다. 이 다음에는 `e820_end_of_ram_pfn` 함수의 호출로 마지막 페이지 프레임의 번호를 가져온다. 모든 메모리 페이지는 `Page frame number(페이지 프레임 번호)`라는 유일한 번호를 갖고, `e820_end_of_ram_pfn` 함수에서 `e820_end_pfn` 의 호출로 최대 번호를 반환한다.:

```C
unsigned long __init e820_end_of_ram_pfn(void)
{
	return e820_end_pfn(MAX_ARCH_PFN);
}
```

`e820_end_pfn` 는 아키텍처마다 정해진 최대 페이지 프레임 넘버를 받는다(`x86_64` 의 경우, `MAX_ARCH_PFN` 은 `0x400000000`이다). `e820_end_pfn`에서 모든 `e820` 슬롯 들을 확인하는데, `E820_RAM` 나 `E820_PRAM` 의 타입을 확인할 것이고 페이지 프레임 번호들을 계산한다. 현재 `e820` 엔트리를 위한 페이지 프레임의 시작 주소와 끝 주소를 갖고 몇몇 확인 작업을 한다.:

```C
for (i = 0; i < e820.nr_map; i++) {
		struct e820entry *ei* = &e820.map[i]; // TODO ei 뒤에 별
		unsigned long start_pfn;
		unsigned long end_pfn;

		if (ei->type != E820_RAM && ei->type != E820_PRAM)
			continue;

		start_pfn = ei->addr >> PAGE_SHIFT;
		end_pfn = (ei->addr + ei->size) >> PAGE_SHIFT;

    if (start_pfn >= limit_pfn)
			continue;
		if (end_pfn > limit_pfn) {
			last_pfn = limit_pfn;
			break;
		}
		if (end_pfn > last_pfn)
			last_pfn = end_pfn;
}
```

```C
	if (last_pfn > max_arch_pfn)
		last_pfn = max_arch_pfn;

	printk(KERN_INFO "e820: last_pfn = %#lx max_arch_pfn = %#lx\n",
			 last_pfn, max_arch_pfn);
	return last_pfn;
```

확인 작업이 끝나면, `last_pfn` 이 아키텍처에서 지정한(여기서는 `x86_64`) 최대 페이지 프레임 번호보다 크지 않는지 검사를 하고, 마지막 페이지 프레임 번호를 출력하고 그것을 반환하여 마무리 한다. 우리는 `dmegs` 출력에서 `last_pfn`의 내용을 볼 수 있다.:

```
...
[    0.000000] e820: last_pfn = 0x41f000 max_arch_pfn = 0x400000000
...
```

우리는 가장 큰 페이지 프레임 번호를 계산했고, `low memory` 나 첫 `4` 기가 바이트 이하의 메모리에서 최대 페이지 프레임 번호를 가지는 `max_low_pfn` 를 계산한다. 만약 RAM이 4기가 이상 설치되어 있다면, `max_low_pfn`는 `e820_end_of_low_ram_pfn` 함수의 결과값을 가지게 될텐데, 4 기가 제한에서는 `e820_end_of_ram_pfn`의 결과와 같을 것이고, 이것은 `max_low_pfn` 와 `max_pfn`가 같아진다는 의미이다.:

```C
if (max_pfn > (1UL<<(32 - PAGE_SHIFT)))
	max_low_pfn = e820_end_of_low_ram_pfn();
else
	max_low_pfn = max_pfn;

high_memory = (void *)__va(max_pfn * PAGE_SIZE* - 1) + 1; // TODO PAGE_SIZE 뒤에 별 하나
```

다음은 주어진 물리 메모리에서 가상 주소로 반환해주는 `__va` 매크로를 이용해서 `high_memory`(직접 맵핑된 메모리 상위에 있는 메모리)를 계산한다.

DMI 스캐닝
-------------------------------------------------------------------------------

서로 다른 메모리 영역과 `e820` 슬롯의 설정 이후에 컴퓨터의 정보를 모은다. 우리는 [데스크탑 관리 인터페이스-Desktop Management Interface](http://en.wikipedia.org/wiki/Desktop_Management_Interface) 와 연관된 함수를 통해 모든 정보를 모을 것이다.:


```C
dmi_scan_machine();
dmi_memdev_walk();
```

첫 번째는 [drivers/firmware/dmi_scan.c](https://github.com/torvalds/linux/blob/master/drivers/firmware/dmi_scan.c)에 있는 `dmi_scan_machine` 함수이다. 이 함수는 [System Management BIOS](http://en.wikipedia.org/wiki/System_Management_BIOS) 데이터 구조와 정보를 추출한다. `SMBIOS` 테이블에 접근하기 위해 정의된 두 가지 방법이 있다.: [EFI](http://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface) 의 구성 테이블로 부터 `SMBIOS` 테이블의 포인터를 얻고 물리 메모리를 `0xF0000` 와 `0x10000` 주소들 사이를 스캐닝한다. 두 번째 접근을 보자. `dmi_scan_machine` 함수는 단지 `early_ioremap`를 호출하는 `dmi_early_remap`을 이용해서 `0xf0000` 와 `0x10000` 사이를 재 매핑한다.:

```C
void __init dmi_scan_machine(void)
{
	char __iomem__ *p, *q*; // TODO iomem 뒤에 언더바 2개 q 뒤에 별하나
	char buf[32];
	...
	...
	...
	p = dmi_early_remap(0xF0000, 0x10000);
	if (p == NULL)
			goto error;
```

그리고 모든 `DMI` 해더 주소를 확인하기 위해 반복하고 `_SM_` 문자열을 검색한다.:

```C
memset(buf, 0, 16);
for (q = p; q < p + 0x10000; q += 16) {
		memcpy_fromio(buf + 16, q, 16);
		if (!dmi_smbios3_present(buf) || !dmi_present(buf)) {
			dmi_available = 1;
			dmi_early_unmap(p, 0x10000);
			goto out;
		}
		memcpy(buf, buf + 16, 16);
}
```

`_SM_` 문자열은 반드시 `000F0000h` 와 `0x000FFFFF` 사이에 있어야 한다. 여기서 우리는 `buf` 에 `memcpy`와 비슷한 `memcpy_fromio` 를 이용해서 16바이트를 복사하고 `buf`를 갖고  `dmi_smbios3_present` 와 `dmi_present`를 수행한다. 이 함수는 첫 4바이트가 `_SM_` 문자열인지 확인하고, `SMBIOS` 버전을 얻고, 버퍼에서 `_DMI_` 문자열을 찾으면, `DMI` 구조체 관련 속성(테이블 길이, 테이블 주소 등등)을 가져온다. 이 함수들이 마무리된 후에, 다음와 같은 커널 메시지를 볼 수 있을 것이다.(`dmesg`) :

```
[    0.000000] SMBIOS 2.7 present.
[    0.000000] DMI: Gigabyte Technology Co., Ltd. Z97X-UD5H-BK/Z97X-UD5H-BK, BIOS F6 06/17/2014
```

`dmi_scan_machine` 의 마지막에는 재 매핑한(remap) 메모리를 해제한다.:

```C
dmi_early_unmap(p, 0x10000);
```

두 번째 함수는 `dmi_memdev_walk` 이다. 이것은 메모리 장치와 연관되어 있는 함수이다. 함수를 살펴 보자.:

```C
void __init dmi_memdev_walk(void)
{
	if (!dmi_available)
		return;

	if (dmi_walk_early(count_mem_devices) == 0 && dmi_memdev_nr) {
		dmi_memdev = dmi_alloc(sizeof(*dmi_memdev) * dmi_memdev_nr);
		if (dmi_memdev)
			dmi_walk_early(save_mem_devices);
	}
}
```

`DMI` 가 가능한지 확인하고 (우리는 이전 함수에서 그것을 알수 있었다. - `dmi_scan_machine`) `dmi_walk_early` 와 `dmi_alloc` 함수로 메모리 장치에 관한 정보를 모은다.:
```
#ifdef CONFIG_DMI
RESERVE_BRK(dmi_alloc, 65536);
#endif
```

`RESERVE_BRK`는 [arch/x86/include/asm/setup.h](http://en.wikipedia.org/wiki/Desktop_Management_Interface) 에 정의되어 있고, 주어진 크기로 `brk` 섹션에 공간을 예약한다.

-------------------------
	init_hypervisor_platform();
	x86_init.resources.probe_roms();
	insert_resource(&iomem_resource, &code_resource);
	insert_resource(&iomem_resource, &data_resource);
	insert_resource(&iomem_resource, &bss_resource);
	early_gart_iommu_check();


SMP 설정
--------------------------------------------------------------------------------

다음 단계는 [SMP](http://en.wikipedia.org/wiki/Symmetric_multiprocessing) 설정을 파싱하는 것이다. `find_smp_config` 함수 호출로 이것을 할 것이다.:

```C
static inline void find_smp_config(void)
{
        x86_init.mpparse.find_smp_config();
}
```

`x86_init.mpparse.find_smp_config` 는 [arch/x86/kernel/mpparse.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/mpparse.c) 에 있는 `default_find_smp_config` 함수이다. `default_find_smp_config` 함수 내에 `SMP` 설정을 위한 몇몇 메모리 영역을 스캐닝하고 찾는다면 그것을 반환한다.:

```C
if (smp_scan_config(0x0, 0x400) ||
            smp_scan_config(639 * 0x400, 0x400) ||
            smp_scan_config(0xF0000, 0x10000))
            return;
```

`smp_scan_config`  함수는 몇몇의 변수를 선언한다.:

```C
unsigned int *bp = phys_to_virt(base);
struct mpf_intel *mpf;
```

우리가 `SMP` 설정을 스캔하기 위한 메모리 영역의 가상 주소를 갖는 변수 선언이고, 두 번째는 `mpf_intel` 구조체 포인터이다. `mpf_intel` 구조체를 조금 더 살펴 보자. 모든 정보는 멀티 프로세서 구성 데이터 구조체에 저장된다. `mpf_intel` 는 아래와 같이 구성된다.:

```C
struct mpf_intel {
        char signature[4];
        unsigned int physptr;
        unsigned char length;
        unsigned char specification;
        unsigned char checksum;
        unsigned char feature1;
        unsigned char feature2;
        unsigned char feature3;
        unsigned char feature4;
        unsigned char feature5;
};
```

As we can read in the documentation - one of the main functions of the system BIOS is to construct the MP floating pointer structure and the MP configuration table. And operating system must have access to this information about the multiprocessor configuration and `mpf_intel` stores the physical address (look at second parameter) of the multiprocessor configuration table. So, `smp_scan_config` going in a loop through the given memory range and tries to find `MP floating pointer structure` there. It checks that current byte points to the `SMP` signature, checks checksum, checks if `mpf->specification` is 1 or 4(it must be `1` or `4` by specification) in the loop:

```C
while (length > 0) {
if ((*bp == SMP_MAGIC_IDENT) &&
    (mpf->length == 1) &&
    !mpf_checksum((unsigned char *)bp, 16) &&
    ((mpf->specification == 1)
    || (mpf->specification == 4))) {

        mem = virt_to_phys(mpf);
        memblock_reserve(mem, sizeof(*mpf));
        if (mpf->physptr)
            smp_reserve_memory(mpf);
	}
}
```

reserves given memory block if search is successful with `memblock_reserve` and reserves physical address of the multiprocessor configuration table. You can find documentation about this in the - [MultiProcessor Specification](http://www.intel.com/design/pentium/datashts/24201606.pdf). You can read More details in the special part about `SMP`.

Additional early memory initialization routines
--------------------------------------------------------------------------------

In the next step of the `setup_arch` we can see the call of the `early_alloc_pgt_buf` function which allocates the page table buffer for early stage. The page table buffer will be placed in the `brk` area. Let's look on its implementation:

```C
void  __init early_alloc_pgt_buf(void)
{
        unsigned long tables = INIT_PGT_BUF_SIZE;
        phys_addr_t base;

        base = __pa(extend_brk(tables, PAGE_SIZE));

        pgt_buf_start = base >> PAGE_SHIFT;
        pgt_buf_end = pgt_buf_start;
        pgt_buf_top = pgt_buf_start + (tables >> PAGE_SHIFT);
}
```

First of all it get the size of the page table buffer, it will be `INIT_PGT_BUF_SIZE` which is `(6 * PAGE_SIZE)` in the current linux kernel 4.0. As we got the size of the page table buffer, we call `extend_brk` function with two parameters: size and align. As you can understand from its name, this function extends the `brk` area. As we can see in the linux kernel linker script `brk` is in memory right after the [BSS](http://en.wikipedia.org/wiki/.bss):

```C
	. = ALIGN(PAGE_SIZE);
	.brk : AT(ADDR(.brk) - LOAD_OFFSET) {
		__brk_base = .;
		. += 64 * 1024;		/* 64k alignment slop space */
		*(.brk_reservation)	/* areas brk users have reserved */
		__brk_limit = .;
	}
```

Or we can find it with `readelf` util:

![brk area](http://oi61.tinypic.com/71lkeu.jpg)

After that we got physical address of the new `brk` with the `__pa` macro, we calculate the base address and the end of the page table buffer. In the next step as we got page table buffer, we reserve memory block for the brk area with the `reserve_brk` function:

```C
static void __init reserve_brk(void)
{
	if (_brk_end > _brk_start)
		memblock_reserve(__pa_symbol(_brk_start),
				 _brk_end - _brk_start);

	_brk_start = 0;
}
```

Note that in the end of the `reserve_brk`, we set `brk_start` to zero, because after this we will not allocate it anymore. The next step after reserving memory block for the `brk`, we need to unmap out-of-range memory areas in the kernel mapping with the `cleanup_highmap` function. Remember that kernel mapping is `__START_KERNEL_map` and `_end - _text` or `level2_kernel_pgt` maps the kernel `_text`, `data` and `bss`. In the start of the `clean_high_map` we define these parameters:

```C
unsigned long vaddr = __START_KERNEL_map;
unsigned long end = roundup((unsigned long)_end, PMD_SIZE) - 1;
pmd_t *pmd = level2_kernel_pgt;
pmd_t *last_pmd = pmd + PTRS_PER_PMD;
```

Now, as we defined start and end of the kernel mapping, we go in the loop through the all kernel page middle directory entries and clean entries which are not between `_text` and `end`:

```C
for (; pmd < last_pmd; pmd++, vaddr += PMD_SIZE) {
        if (pmd_none(*pmd))
            continue;
        if (vaddr < (unsigned long) _text || vaddr > end)
            set_pmd(pmd, __pmd(0));
}
```

After this we set the limit for the `memblock` allocation with the `memblock_set_current_limit` function (read more about `memblock` you can in the [Linux kernel memory management Part 2](https://github.com/0xAX/linux-insides/blob/master/mm/linux-mm-2.md)), it will be `ISA_END_ADDRESS` or `0x100000` and fill the `memblock` information according to `e820` with the call of the `memblock_x86_fill` function. You can see the result of this function in the kernel initialization time:

```
MEMBLOCK configuration:
 memory size = 0x1fff7ec00 reserved size = 0x1e30000
 memory.cnt  = 0x3
 memory[0x0]	[0x00000000001000-0x0000000009efff], 0x9e000 bytes flags: 0x0
 memory[0x1]	[0x00000000100000-0x000000bffdffff], 0xbfee0000 bytes flags: 0x0
 memory[0x2]	[0x00000100000000-0x0000023fffffff], 0x140000000 bytes flags: 0x0
 reserved.cnt  = 0x3
 reserved[0x0]	[0x0000000009f000-0x000000000fffff], 0x61000 bytes flags: 0x0
 reserved[0x1]	[0x00000001000000-0x00000001a57fff], 0xa58000 bytes flags: 0x0
 reserved[0x2]	[0x0000007ec89000-0x0000007fffffff], 0x1377000 bytes flags: 0x0
```

The rest functions after the `memblock_x86_fill` are: `early_reserve_e820_mpc_new` allocates additional slots in the `e820map` for MultiProcessor Specification table, `reserve_real_mode` - reserves low memory from `0x0` to 1 megabyte for the trampoline to the real mode (for rebooting, etc.), `trim_platform_memory_ranges` - trims certain memory regions started from `0x20050000`, `0x20110000`, etc. these regions must be excluded because [Sandy Bridge](http://en.wikipedia.org/wiki/Sandy_Bridge) has problems with these regions, `trim_low_memory_range` reserves the first 4 kilobyte page in `memblock`, `init_mem_mapping` function reconstructs direct memory mapping and setups the direct mapping of the physical memory at `PAGE_OFFSET`, `early_trap_pf_init` setups `#PF` handler (we will look on it in the chapter about interrupts) and `setup_real_mode` function setups trampoline to the [real mode](http://en.wikipedia.org/wiki/Real_mode) code.

That's all. You can note that this part will not cover all functions which are in the `setup_arch` (like `early_gart_iommu_check`, [mtrr](http://en.wikipedia.org/wiki/Memory_type_range_register) initialization, etc.). As I already wrote many times, `setup_arch` is big, and linux kernel is big. That's why I can't cover every line in the linux kernel. I don't think that we missed something important, but you can say something like: each line of code is important. Yes, it's true, but I missed them anyway, because I think that it is not realistic to cover full linux kernel. Anyway we will often return to the idea that we have already seen, and if something is unfamiliar, we will cover this theme.

Conclusion
--------------------------------------------------------------------------------

It is the end of the sixth part about linux kernel initialization process. In this part we continued to dive in the `setup_arch` function again and it was long part, but we are not finished with it. Yes, `setup_arch` is big, hope that next part will be the last part about this function.

If you have any questions or suggestions write me a comment or ping me at [twitter](https://twitter.com/0xAX).

**Please note that English is not my first language, And I am really sorry for any inconvenience. If you find any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-insides).**

Links
--------------------------------------------------------------------------------

* [MultiProcessor Specification](http://en.wikipedia.org/wiki/MultiProcessor_Specification)
* [NX bit](http://en.wikipedia.org/wiki/NX_bit)
* [Documentation/kernel-parameters.txt](https://github.com/torvalds/linux/blob/master/Documentation/kernel-parameters.txt)
* [APIC](http://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller)
* [CPU masks](http://0xax.gitbooks.io/linux-insides/content/Concepts/cpumask.html)
* [Linux kernel memory management](http://0xax.gitbooks.io/linux-insides/content/mm/index.html)
* [PCI](http://en.wikipedia.org/wiki/Conventional_PCI)
* [e820](http://en.wikipedia.org/wiki/E820)
* [System Management BIOS](http://en.wikipedia.org/wiki/System_Management_BIOS)
* [System Management BIOS](http://en.wikipedia.org/wiki/System_Management_BIOS)
* [EFI](http://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface)
* [SMP](http://en.wikipedia.org/wiki/Symmetric_multiprocessing)
* [MultiProcessor Specification](http://www.intel.com/design/pentium/datashts/24201606.pdf)
* [BSS](http://en.wikipedia.org/wiki/.bss)
* [SMBIOS specification](http://www.dmtf.org/sites/default/files/standards/documents/DSP0134v2.5Final.pdf)
* [Previous part](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-5.html)
