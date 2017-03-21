커널 부팅 과정. part 3.
================================================================================

비디오 모드 초기화와 보호 모드 전환
--------------------------------------------------------------------------------

`커널 부팅 과정` 시리즈의 3번 째 파트이다. 이전 [part](linux-bootstrap-2.md#kernel-booting-process-part-2)는 [main.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c#L181) 에서 `set_video` 부르기 전에서 마무리 되었다. 이번 파트에서 우리가 살펴볼 것은:

- 커널 설정 코드 에서 비디오 모드 초기화,
- 보호 모드 진입전 준비사항,
- 보호 모드 전환

**NOTE** 만약 당신이 보호 모드에 대해 아무것도 모르다면, 당신은 이전 [파트](linux-bootstrap-2.md#protected-mode)에서 보호 모드에 관해 정보를 찾아 봐야 할 것이다. 또한 [링크](linux-bootstrap-2.md#links) 에서 당신에게 도움이 될 만한 것들이 있다.

[arch/x86/boot/video.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/video.c#L315) 소스 코드에 구현된 `set_video` 함수에서 시작 할 수 있다. 우리는 `boot_params.hdr` 구조체에서 첫 비디오 모드를 시작할 수 있다는 것을 알 수 있다.:

```C
u16 mode = boot_params.hdr.vid_mode;
```

`copy_boot_params` 함수에서 이 구조체는 채워졌다. (이전 파트에서 확인해 보시길) `vid_mode` 는 부트로더에서 의무적으로 채워줘야 하는 항목이다. 당신은 커널 부트 프로토콜에서 이와 관련된 정보를 찾을 수 있을 것이다.:

```
Offset	Proto	Name		Meaning
/Size
01FA/2	ALL	    vid_mode	Video mode control
```

우리는 리눅스 부트 프로토콜로 부터 아래의 내용을 읽을 수 있다.:

```
vga=<mode>
	<mode> here is either an integer (in C notation, either
	decimal, octal, or hexadecimal) or one of the strings
	"normal" (meaning 0xFFFF), "ext" (meaning 0xFFFE) or "ask"
	(meaning 0xFFFD).  This value should be entered into the
	vid_mode field, as it is used by the kernel before the command
	line is parsed.
vga=<모드>
  <모드> 는 정수형(C 표기법에서, 10진수, 8진수 또는 16진수) 이거나 문자열
	"normal" (0xFFFF 을 의미), "ext" (0xFFFE 를 의미) 또는 "ask"(0xFFFD 를 의미)
	이다. 이 값은 vid_mode 항목에 들어가야 하며, 명령 라인이 해석되기 전에
	커널에 의해 사용된다.
```

그래서 우리는 `vga` 옵션을 grup 이나 다른 부트로터 설정 파일에 넣고 커널 명령 라인에 이 옵션을 넘겨 줄 것이다. 이 옵션은 설명에 나와 있는 대로 다름 값을 가질 수 있다. 예를 들면, 그것은 정수형 `0xFFFD` 나 `ask` 가 될 수 있다. 만약 `ask`를 `vga` 에 넘겨주면, 당신은 아래와 같은 메뉴를 볼 수 있을 것이다.:

![video mode setup menu](http://oi59.tinypic.com/ejcz81.jpg)

비디오 모드 선택을 위해 물어볼 것이다. 우리는 이것의 구현을 살펴보기 전에 다른 몇가지를 먼저 살펴봐야 한다.

Kernel data types (커널 자료 타입)
--------------------------------------------------------------------------------

우리는 커널 설정 코드에서 `u16` 와 같은 자료 타입의 정의를 보았다. 커널에서 제공되는 몇 가지의 자료 타입을 알아보자.:


| Type | char | short | int | long | u8 | u16 | u32 | u64 |
|------|------|-------|-----|------|----|-----|-----|-----|
| Size |  1   |   2   |  4  |   8  |  1 |  2  |  4  |  8  |

만얀 당신이 커널 코드를 읽는다면, 당신은 이 자료 타입을 매우 자주 볼 것이다. 그래서 이것들을 기억해두면 좋을 것이다.

Heap API (힙 API)
--------------------------------------------------------------------------------

`set_video` 함수에서 `boot_params.hdr` 로 부터 `vid_mode` 를 얻은 다음에, 우리는 `RESET_HEAP` 함수를 호출 할 것이다. `RESET_HEAP` 는 [boot.h](https://github.com/torvalds/linux/blob/master/arch/x86/boot/boot.h#L199) 에 정의된 매크로 인데 아래처럼 되어 있다:

```C
#define RESET_HEAP() ((void *)( HEAP = _end )) //* TODO: this line should be removed!!!
```

만약 당신이 두 번째 파트를 읽었다면, [`init_heap`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c#L116) 함수를 통해 힙이 초기화 되었다는 것을 기억할 수 있을 것이다. 우리는 `boot.h` 에 정의된 힙 을 위한 유틸리티 함수들을 갖고 있다.:

```C
#define RESET_HEAP()
```

위에서 확인했듯이, 그것은 `HEAP` 변수를 `extern char _end[];` 로 선언된 `_end` 로 같게 해서 heap 을 리셋시켜준다.

다음은 `GET_HEAP` 매크로이다.:

```C
#define GET_HEAP(type, n) \
	((type *)__get_heap(sizeof(type),__alignof__(type),(n))) //* TODO: this line should be removed!!!
```

힙 할당을 위해, `GET_HEAP` 을 쓴다. 이것은 내부 함수인 `__get_heap` 을 인자 3개와 호출한다.:

* 할당하고자 하는 타입의 크기
* `__alignof__(type)` 로 주어진 타입의 정렬에 필요한 바이트수를 파악
* `n` 할당 받을 타입의 개수
* [__get_heap 상세](http://www.iamroot.org/ldocs/linux.html#sec-22-1)

`__get_heap` 의 구현은:

```C
static inline char *__get_heap(size_t s, size_t a, size_t n)
{
	char *tmp;

	HEAP = (char *)(((size_t)HEAP+(a-1)) & ~(a-1));
	tmp = HEAP;
	HEAP += s*n;
	return tmp;
}
//* TODO: this line should be removed!!!
```

그리고 아래와 같이 사용하면 된다:

```C
saved.data = GET_HEAP(u16, saved.x * saved.y);
```

어떻게 `__get_heap` 이 동작하는지 알아보자. 우리는 `HEAP`(`RESET_HEAP()` 을 통해 `_end`와 같다는 것을 알고 있다.) 은 `a` 인자에 따라 정렬된 메모리의 주소가 될 것이다. 이 다음에 우리는 `HEAP`을 `tmp` 로 임시 저장을 해놓고, `HEAP` 의 주소를 할당된 블락의 주소 만큼 이동 시킨 후, 할당된 메모리의 시작 주소가 되는 `tmp` 를 반환하도록 했다.

그리고 마지막 함수는:

```C
static inline bool heap_free(size_t n)
{
	return (int)(heap_end - HEAP) >= (int)n;
}
```

`heap_end`([이전 파트](linux-bootstrap-2.md) 에서 이 값이 계산되었다) 에서 `HEAP` 의 주소 값을 빼준 값이 `n` 보다 크다면 "참"을 반환한다. 이 함수는 힙 영역에 `n` 크기의 공간이 있는지 확인을 해주기 위한 것이다.

끝이다. 우리는 힙을 위한 간단한 API 를 알아보았고, 이제 비디오 모드 설정을 알아보자.

비디오 모드 설정
--------------------------------------------------------------------------------

이제 비디오 모드 초기화로 바로 넘어가보자. 우리는 `set_video` 함수에서 `RESET_HEAP()` 호출 까지 알아보았다. 이 다음은, [include/uapi/linux/screen_info.h](https://github.com/0xAX/linux/blob/master/include/uapi/linux/screen_info.h) 에 정의된 `boot_params.screen_info` 구조체내에 비디오 모드 인자를 저장하기 위한 `store_mode_params`을 호출 한다.

만약 우리가 `store_mode_params` 함수를 본다면, 우리는 `store_cursor_position` 함수에서 시작한다는 것을 알 수 있을 것이다. 함수 이름에서 부터 이해 가능하겠지만, 그것은 커서의 정보를 얻고 그것을 저장하는 함수 이다.

`store_cursor_position` 에서 처음 하는 것은 `AH = 0x3` 과 `0x10` BIOS 인터럽트를 통해 커서의 위치를 얻어와 biosregs 구조체에 저장한다. 인터럽트가 성공적으로 실행되고 나면, 그것은 열과 행의 정보를 각각 `DL`과 `DH` 레지스터에 저장한다. 행과 열은 각각 `boot_params.screen_info` 구조체의 `orig_x` 와 `orig_y` 항목에 저장될 것이다.

`store_cursor_position` 함수가 실행되고 나서, `store_video_mode` 함수가 호출된다. 그것은 단지 현재 비디오 모드를 얻어와 그 값을 `boot_params.screen_info.orig_video_mode` 에 저장한다.

이 다음에는, 그것은 현재 비디오 모드를 확인하고 `video_segment` 를 설정한다. BIOS 에서 boot sector 로 제어권이 넘어간 뒤로 비디오를 위해 사용되는 주소는 다음과 같다.:

```
0xB000:0x0000 	32 Kb 	Monochrome Text Video Memory 단색 텍스트 비디오 메모리
0xB800:0x0000 	32 Kb 	Color Text Video Memory 컬러 텍스트 비디오 메모리
```

그래서 만약에 현재 비디오 모드가 단색 모드에서 MDA, HGC 또는 VGA 라면 `video_segment` 변수를 `0xB800` 설정하고, 만약 현재 비티오 모드가 컬러 모드이면 `0xB000` 로 설정한다. 비디오 세그먼트 주소 설정이 된 후에, `boot_params.screen_info.orig_video_points` 에 폰트 크기가 아래와 같은 방법으로 저장이 되어야 한다.:

```C
set_fs(0);
font_size = rdfs16(0x485);
boot_params.screen_info.orig_video_points = font_size;
```

첫째로 우리는 `set_fs` 함수 안에서 `FS` 레지스터에 0을 넣는다. 우리는 `set_fs` 와 같은 함수들을 이전 파트에서 이미 다루었다. 그 모든 정의는 [boot.h](https://github.com/0xAX/linux/blob/master/arch/x86/boot/boot.h)에 되어 있다. 다음으로 우리는 `0x485` 주소(이 주소는 폰트 크기를 얻기위한 메모리 위치이다.)에 위치하고 있는 값을 읽고, `boot_params.screen_info.orig_video_points` 로 폰트 크기를 저장한다.

```
 x = rdfs16(0x44a);
 y = (adapter == ADAPTER_CGA) ? 25 : rdfs8(0x484)+1;
```

그 다음으로는 `0x44a` 주소에서 행의 크기와 `0x484` 주소로 부터 열의 크기를 얻어와 각각 `boot_params.screen_info.orig_video_cols` 와 `boot_params.screen_info.orig_video_lines` 에 저장을 한다. 그러면 `store_mode_params` 의 실행은 완료된다.

다음으로 살펴 볼 함수는 `save_screen` 인데, 스크린의 내용(contents)을 힙로 저장하는 함수이다. 이 함수는 행/열의 크기 등과 같은 이전 함수로 얻은 모든 데이터를 모으고 `saved_screen` 구조체에 그것들을 저장한다. 구조체는 아래와 같이 정의되어 있다:

```C
static struct saved_screen {
	int x, y;
	int curx, cury;
	u16 *data;
} saved;
//* TODO: this line should be removed!!!
```

이 스크린 내용을 저장하기 위해 힙이 충분한 공간을 가지고 있는지 확인한다.:

```C
if (!heap_free(saved.x*saved.y*sizeof(u16)+512))
		return;
```

그리고 힙에 충분한 공간이 있다면, 공간을 할당하고 `saved_screen` 을 힙에 저장한다.

다음 호출은 [arch/x86/boot/video-mode.c](https://github.com/0xAX/linux/blob/master/arch/x86/boot/video-mode.c#L33) 에 있는 `probe_cards(0)` 함수이다. 이것은 모든 video_cards 를 순회하고 각 카드마다 지원되는 모드의 수의 정보를 모은다. 여기에 재미난 부분이 있는데, 아래의 루프를 보자.:

```C
for (card = video_cards; card < video_cards_end; card++) {
  /* collecting number of modes here */
}
```
//* TODO: this line should be removed!!!

하지만 `video_cards` 는 어디에도 선언된 변수가 아니다. 답은 간단하다: x86 커널 설정 코드에서 존재하는 모든 비디오 모드는 아래와 같은 형태로 정의되어 있다.:
```C
static __videocard video_vga = {
	.card_name	= "VGA",
	.probe		= vga_probe,
	.set_mode	= vga_set_mode,
};
```

`__videocard` 매크로는 어디에 있는가 하면:

```C
#define __videocard struct card_info __attribute__((used,section(".videocards")))
```

`card_info` 구조체는:

```C
struct card_info {
	const char *card_name;
	int (*set_mode)(struct mode_info *mode);
	int (*probe)(void);
	struct mode_info *modes;
	int nmodes;
	int unsafe;
	u16 xmode_first;
	u16 xmode_n;
};
```
//* TODO: this line should be removed!!!

`.videocards` 세그먼트내에 이 정보가 있다. [arch/x86/boot/setup.ld](https://github.com/0xAX/linux/blob/master/arch/x86/boot/setup.ld) 링커 스크립트를 살펴보자, 우리는 아래와 같이 videocards 를 찾을 수 있다:

```
	.videocards	: {
		video_cards = .;
		*(.videocards)
		video_cards_end = .;
	}
```

위의 내용은 `video_cards` 는 단지 주소이고 모든 `card_info` 구조체는 `.videocards` 세그먼트 내에 위치한다는 것을 의미한다. 또한 `card_info` 구조체들은 `video_cards` 와 `video_cards_end` 사이에 있다는 것도 알 수 있다. 그래서 우리는 그것들을 루프 내에서 접근하여 사용할 수 있다. 다음으로 `probe_cards`는 `static __videocard video_vga` 내에 있는 `nmodes` (비디오 모드 번호) 를 채워 넣는다.

`probe_cards` 실행이 끝나면, 우리는 `set_video` 함수 내에 있는 메인 루프로 진입한다. 무한 루프 내에서 `set_mode` 함수로 현재 비디오 모드를 설정하거나 `vid_mode=ask` 커널 명령 라인이나 비디오 모드가 정해지지 않은 경우에 메뉴를 출력하는 등의 작업을 한다.

[video-mode.c](https://github.com/0xAX/linux/blob/master/arch/x86/boot/video-mode.c#L147) 에 구현되어 있는 `set_mode` 함수는 단 하나의 인자만 받는데, 그것은 비디오 모드 번호를 갖는 `mode` 이다.(우리는 그것을 커널 설정 헤더로 부터 `setup_video` 의 시작부나 메뉴에서 얻었다.)

`set_mode` 함수는 `mode` 를 확인하고 `raw_set_mode` 함수를 호출한다. `raw_set_mode` 는 선택된 card 를 위해 `set_mode` 를 호출한다.(예, `card->set_mode(struct mode_info*)`). 우리는 `card_info` 구조체를 통해 이 함수에 접근할 수 있다. 모든 비디오 모드는 비디오 모드에 맞게 채워진 `card_info` 구조체를 통해 정의된다.(예를 들어, `vga` 는 `video_vga.set_mode` 를 갖고 있다. 위에서 보면, `vga`를 위한 `card_info` 구조체가 있을 것이다.) `video_vga.set_mode` 는 vga mode 를 확인하는 `vga_set_mode` 함수가 될 것이다.:

```C
static int vga_set_mode(struct mode_info *mode)
{
	vga_set_basic_mode();

	force_x = mode->x;
	force_y = mode->y;

	switch (mode->mode) {
	case VIDEO_80x25:
		break;
	case VIDEO_8POINT:
		vga_set_8font();
		break;
	case VIDEO_80x43:
		vga_set_80x43();
		break;
	case VIDEO_80x28:
		vga_set_14font();
		break;
	case VIDEO_80x30:
		vga_set_80x30();
		break;
	case VIDEO_80x34:
		vga_set_80x34();
		break;
	case VIDEO_80x60:
		vga_set_80x60();
		break;
	}
	return 0;
}
```
//* TODO: this line should be removed!!!

Every function which sets up video mode just calls the `0x10` BIOS interrupt with a certain value in the `AH` register.

After we have set video mode, we pass it to `boot_params.hdr.vid_mode`.

Next `vesa_store_edid` is called. This function simply stores the [EDID](https://en.wikipedia.org/wiki/Extended_Display_Identification_Data) (**E**xtended **D**isplay **I**dentification **D**ata) information for kernel use. After this `store_mode_params` is called again. Lastly, if `do_restore` is set, the screen is restored to an earlier state.

After this we have set video mode and now we can switch to the protected mode.
**** //<-- TODO: this line should be removed!!!

Last preparation before transition into protected mode
--------------------------------------------------------------------------------

We can see the last function call - `go_to_protected_mode` - in [main.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c#L184). As the comment says: `Do the last things and invoke protected mode`, so let's see these last things and switch into protected mode.

`go_to_protected_mode` is defined in [arch/x86/boot/pm.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/pm.c#L104). It contains some functions which make the last preparations before we can jump into protected mode, so let's look at it and try to understand what they do and how it works.

First is the call to the `realmode_switch_hook` function in `go_to_protected_mode`. This function invokes the real mode switch hook if it is present and disables [NMI](http://en.wikipedia.org/wiki/Non-maskable_interrupt). Hooks are used if the bootloader runs in a hostile environment. You can read more about hooks in the [boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt) (see **ADVANCED BOOT LOADER HOOKS**).

The `realmode_switch` hook presents a pointer to the 16-bit real mode far subroutine which disables non-maskable interrupts. After `realmode_switch` hook (it isn't present for me) is checked, disabling of Non-Maskable Interrupts(NMI) occurs:

```assembly
asm volatile("cli");
outb(0x80, 0x70);	/* Disable NMI */
io_delay();
```

At first there is an inline assembly instruction with a `cli` instruction which clears the interrupt flag (`IF`). After this, external interrupts are disabled. The next line disables NMI (non-maskable interrupt).

An interrupt is a signal to the CPU which is emitted by hardware or software. After getting the signal, the CPU suspends the current instruction sequence, saves its state and transfers control to the interrupt handler. After the interrupt handler has finished it's work, it transfers control to the interrupted instruction. Non-maskable interrupts (NMI) are interrupts which are always processed, independently of permission. It cannot be ignored and is typically used to signal for non-recoverable hardware errors. We will not dive into details of interrupts now, but will discuss it in the next posts.

Let's get back to the code. We can see that second line is writing `0x80` (disabled bit) byte to `0x70` (CMOS Address register). After that, a call to the `io_delay` function occurs. `io_delay` causes a small delay and looks like:

```C
static inline void io_delay(void)
{
	const u16 DELAY_PORT = 0x80;
	asm volatile("outb %%al,%0" : : "dN" (DELAY_PORT));
}
```

To output any byte to the port `0x80` should delay exactly 1 microsecond. So we can write any value (value from `AL` register in our case) to the `0x80` port. After this delay `realmode_switch_hook` function has finished execution and we can move to the next function.

The next function is `enable_a20`, which enables [A20 line](http://en.wikipedia.org/wiki/A20_line). This function is defined in [arch/x86/boot/a20.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/a20.c) and it tries to enable the A20 gate with different methods. The first is the `a20_test_short` function which checks if A20 is already enabled or not with the `a20_test` function:

```C
static int a20_test(int loops)
{
	int ok = 0;
	int saved, ctr;

	set_fs(0x0000);
	set_gs(0xffff);

	saved = ctr = rdfs32(A20_TEST_ADDR);

    while (loops--) {
		wrfs32(++ctr, A20_TEST_ADDR);
		io_delay();	/* Serialize and make delay constant */
		ok = rdgs32(A20_TEST_ADDR+0x10) ^ ctr;
		if (ok)
			break;
	}

	wrfs32(saved, A20_TEST_ADDR);
	return ok;
}
```

First of all we put `0x0000` in the `FS` register and `0xffff` in the `GS` register. Next we read the value in address `A20_TEST_ADDR` (it is `0x200`) and put this value into the `saved` variable and `ctr`.

Next we write an updated `ctr` value into `fs:gs` with the `wrfs32` function, then delay for 1ms, and then read the value from the `GS` register by address `A20_TEST_ADDR+0x10`, if it's not zero we already have enabled the A20 line. If A20 is disabled, we try to enable it with a different method which you can find in the `a20.c`. For example with call of `0x15` BIOS interrupt with `AH=0x2041` etc.

If the `enabled_a20` function finished with fail, print an error message and call function `die`. You can remember it from the first source code file where we started - [arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S):

```assembly
die:
	hlt
	jmp	die
	.size	die, .-die
```

After the A20 gate is successfully enabled, the `reset_coprocessor` function is called:
 ```C
outb(0, 0xf0);
outb(0, 0xf1);
```
This function clears the Math Coprocessor by writing `0` to `0xf0` and then resets it by writing `0` to `0xf1`.

After this, the `mask_all_interrupts` function is called:
```C
outb(0xff, 0xa1);       /* Mask all interrupts on the secondary PIC */
outb(0xfb, 0x21);       /* Mask all but cascade on the primary PIC */
```
This masks all interrupts on the secondary PIC (Programmable Interrupt Controller) and primary PIC except for IRQ2 on the primary PIC.

And after all of these preparations, we can see the actual transition into protected mode.

Set up Interrupt Descriptor Table
--------------------------------------------------------------------------------

Now we set up the Interrupt Descriptor table (IDT). `setup_idt`:

```C
static void setup_idt(void)
{
	static const struct gdt_ptr null_idt = {0, 0};
	asm volatile("lidtl %0" : : "m" (null_idt));
}
```

which sets up the Interrupt Descriptor Table (describes interrupt handlers and etc.). For now the IDT is not installed (we will see it later), but now we just the load IDT with the `lidtl` instruction. `null_idt` contains address and size of IDT, but now they are just zero. `null_idt` is a `gdt_ptr` structure, it as defined as:
```C
struct gdt_ptr {
	u16 len;
	u32 ptr;
} __attribute__((packed));
```

where we can see the 16-bit length(`len`) of the IDT and the 32-bit pointer to it (More details about the IDT and interruptions will be seen in the next posts). ` __attribute__((packed))` means that the size of `gdt_ptr` is the minimum required size. So the size of the `gdt_ptr` will be 6 bytes here or 48 bits. (Next we will load the pointer to the `gdt_ptr` to the `GDTR` register and you might remember from the previous post that it is 48-bits in size).

Set up Global Descriptor Table
--------------------------------------------------------------------------------

Next is the setup of the Global Descriptor Table (GDT). We can see the `setup_gdt` function which sets up GDT (you can read about it in the [Kernel booting process. Part 2.](linux-bootstrap-2.md#protected-mode)). There is a definition of the `boot_gdt` array in this function, which contains the definition of the three segments:

```C
	static const u64 boot_gdt[] __attribute__((aligned(16))) = {
		[GDT_ENTRY_BOOT_CS] = GDT_ENTRY(0xc09b, 0, 0xfffff),
		[GDT_ENTRY_BOOT_DS] = GDT_ENTRY(0xc093, 0, 0xfffff),
		[GDT_ENTRY_BOOT_TSS] = GDT_ENTRY(0x0089, 4096, 103),
	};
```

For code, data and TSS (Task State Segment). We will not use the task state segment for now, it was added there to make Intel VT happy as we can see in the comment line (if you're interested you can find commit which describes it - [here](https://github.com/torvalds/linux/commit/88089519f302f1296b4739be45699f06f728ec31)). Let's look at `boot_gdt`. First of all note that it has the `__attribute__((aligned(16)))` attribute. It means that this structure will be aligned by 16 bytes. Let's look at a simple example:
```C
#include <stdio.h>

struct aligned {
	int a;
}__attribute__((aligned(16)));

struct nonaligned {
	int b;
};

int main(void)
{
	struct aligned    a;
	struct nonaligned na;

	printf("Not aligned - %zu \n", sizeof(na));
	printf("Aligned - %zu \n", sizeof(a));

	return 0;
}
```

Technically a structure which contains one `int` field must be 4 bytes, but here `aligned` structure will be 16 bytes:

```
$ gcc test.c -o test && test
Not aligned - 4
Aligned - 16
```

`GDT_ENTRY_BOOT_CS` has index - 2 here, `GDT_ENTRY_BOOT_DS` is `GDT_ENTRY_BOOT_CS + 1` and etc. It starts from 2, because first is a mandatory null descriptor (index - 0) and the second is not used (index - 1).

`GDT_ENTRY` is a macro which takes flags, base and limit and builds GDT entry. For example let's look at the code segment entry. `GDT_ENTRY` takes following values:

* base  - 0
* limit - 0xfffff
* flags - 0xc09b

What does this mean? The segment's base address is 0, and the limit (size of segment) is - `0xffff` (1 MB). Let's look at the flags. It is `0xc09b` and it will be:

```
1100 0000 1001 1011
```

in binary. Let's try to understand what every bit means. We will go through all bits from left to right:

* 1    - (G) granularity bit
* 1    - (D) if 0 16-bit segment; 1 = 32-bit segment
* 0    - (L) executed in 64 bit mode if 1
* 0    - (AVL) available for use by system software
* 0000 - 4 bit length 19:16 bits in the descriptor
* 1    - (P) segment presence in memory
* 00   - (DPL) - privilege level, 0 is the highest privilege
* 1    - (S) code or data segment, not a system segment
* 101  - segment type execute/read/
* 1    - accessed bit

You can read more about every bit in the previous [post](linux-bootstrap-2.md) or in the [Intel® 64 and IA-32 Architectures Software Developer's Manuals 3A](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html).

After this we get the length of the GDT with:

```C
gdt.len = sizeof(boot_gdt)-1;
```

We get the size of `boot_gdt` and subtract 1 (the last valid address in the GDT).

Next we get a pointer to the GDT with:

```C
gdt.ptr = (u32)&boot_gdt + (ds() << 4);
```

Here we just get the address of `boot_gdt` and add it to the address of the data segment left-shifted by 4 bits (remember we're in the real mode now).

Lastly we execute the `lgdtl` instruction to load the GDT into the GDTR register:

```C
asm volatile("lgdtl %0" : : "m" (gdt));
```

Actual transition into protected mode
--------------------------------------------------------------------------------

This is the end of the `go_to_protected_mode` function. We loaded IDT, GDT, disable interruptions and now can switch the CPU into protected mode. The last step is calling the `protected_mode_jump` function with two parameters:

```C
protected_mode_jump(boot_params.hdr.code32_start, (u32)&boot_params + (ds() << 4));
```

which is defined in [arch/x86/boot/pmjump.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/pmjump.S#L26). It takes two parameters:

* address of protected mode entry point
* address of `boot_params`

Let's look inside `protected_mode_jump`. As I wrote above, you can find it in `arch/x86/boot/pmjump.S`. The first parameter will be in the `eax` register and second is in `edx`.

First of all we put the address of `boot_params` in the `esi` register and the address of code segment register `cs` (0x1000) in `bx`. After this we shift `bx` by 4 bits and add the address of label `2` to it (we will have the physical address of label `2` in the `bx` after this) and jump to label `1`. Next we put data segment and task state segment in the `cs` and `di` registers with:

```assembly
movw	$__BOOT_DS, %cx
movw	$__BOOT_TSS, %di
```

As you can read above `GDT_ENTRY_BOOT_CS` has index 2 and every GDT entry is 8 byte, so `CS` will be `2 * 8 = 16`, `__BOOT_DS` is 24 etc.

Next we set the `PE` (Protection Enable) bit in the `CR0` control register:

```assembly
movl	%cr0, %edx
orb	$X86_CR0_PE, %dl
movl	%edx, %cr0
```

and make a long jump to protected mode:

```assembly
	.byte	0x66, 0xea
2:	.long	in_pm32
	.word	__BOOT_CS
```

where
* `0x66` is the operand-size prefix which allows us to mix 16-bit and 32-bit code,
* `0xea` - is the jump opcode,
* `in_pm32` is the segment offset
* `__BOOT_CS` is the code segment.

After this we are finally in the protected mode:

```assembly
.code32
.section ".text32","ax"
```

Let's look at the first steps in protected mode. First of all we set up the data segment with:

```assembly
movl	%ecx, %ds
movl	%ecx, %es
movl	%ecx, %fs
movl	%ecx, %gs
movl	%ecx, %ss
```

If you paid attention, you can remember that we saved `$__BOOT_DS` in the `cx` register. Now we fill it with all segment registers besides `cs` (`cs` is already `__BOOT_CS`). Next we zero out all general purpose registers besides `eax` with:

```assembly
xorl	%ecx, %ecx
xorl	%edx, %edx
xorl	%ebx, %ebx
xorl	%ebp, %ebp
xorl	%edi, %edi
```

And jump to the 32-bit entry point in the end:

```
jmpl	*%eax
```

Remember that `eax` contains the address of the 32-bit entry (we passed it as first parameter into `protected_mode_jump`).

That's all. We're in the protected mode and stop at it's entry point. We will see what happens next in the next part.

Conclusion
--------------------------------------------------------------------------------

This is the end of the third part about linux kernel insides. In next part, we will see first steps in the protected mode and transition into the [long mode](http://en.wikipedia.org/wiki/Long_mode).

If you have any questions or suggestions write me a comment or ping me at [twitter](https://twitter.com/0xAX).

**Please note that English is not my first language, And I am really sorry for any inconvenience. If you find any mistakes, please send me a PR with corrections at [linux-insides](https://github.com/0xAX/linux-internals).**

Links
--------------------------------------------------------------------------------

* [VGA](http://en.wikipedia.org/wiki/Video_Graphics_Array)
* [VESA BIOS Extensions](http://en.wikipedia.org/wiki/VESA_BIOS_Extensions)
* [Data structure alignment](http://en.wikipedia.org/wiki/Data_structure_alignment)
* [Non-maskable interrupt](http://en.wikipedia.org/wiki/Non-maskable_interrupt)
* [A20](http://en.wikipedia.org/wiki/A20_line)
* [GCC designated inits](https://gcc.gnu.org/onlinedocs/gcc-4.1.2/gcc/Designated-Inits.html)
* [GCC type attributes](https://gcc.gnu.org/onlinedocs/gcc/Type-Attributes.html)
* [Previous part](linux-bootstrap-2.md)
