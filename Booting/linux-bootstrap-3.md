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
#define RESET_HEAP() ((void *)( HEAP = _end ))
```

만약 당신이 두 번째 파트를 읽었다면, [`init_heap`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c#L116) 함수를 통해 힙이 초기화 되었다는 것을 기억할 수 있을 것이다. 우리는 `boot.h` 에 정의된 힙 을 위한 유틸리티 함수들을 갖고 있다.:

```C
#define RESET_HEAP()
```

위에서 확인했듯이, 그것은 `HEAP` 변수를 `extern char _end[];` 로 선언된 `_end` 로 같게 해서 heap 을 리셋시켜준다.

다음은 `GET_HEAP` 매크로이다.:

```C
#define GET_HEAP(type, n) \
	((type *)__get_heap(sizeof(type),__alignof__(type),(n)))
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

비디오 모드 설정을 위한 모든 함수는 `AH` 레지스터에 설정값을 넣고 `0x10` BIOS 인터럽	트 호출을 한다.

비디오 모드를 설정하고 나면, 그 값을 `boot_params.hdr.vid_mode` 로 넘긴다.

다음은 `vesa_store_edid` 호출이다. 이 함수는 단순히 커널에서 사용하기 위해 [EDID](https://en.wikipedia.org/wiki/Extended_Display_Identification_Data)(**E**xtended **D**isplay **I**dentification **D**ata) 정보를 저장한다. 이후에 `store_mode_params`를 다시 한번 호출한다. 마지막으로, 만약 `do_restore` 이 셋되어 있다면, 스크린은 최초의 상태로 복귀될 것이다.

우리는 비디오 모드 설정을 했고 이제 보호 모드로 전환을 할 수 있다.

보호 모드로 전환전에 마지막 준비 작업
--------------------------------------------------------------------------------

우리는 [main.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c#L181) 에 있는 `go_to_protected_mode` 함수가 마지막이라는 것을 볼 수 있다. 이 함수 주석의 내용을 보면, 마지막 작업을 하고 보호 모드로 전환한다라는 내용이 있다. 여기서 마지막 작업은 무엇인지 살펴보고 보호 모드로 전환하자.

[arch/x86/boot/pm.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/pm.c#L104) 에 `go_to_protected_mode` 함수가 정의되어 있다. 그것은 보호 모드로 진입하기 전에 마지막으로 완료해야 할 몇몇 함수들을 가지다. 그것을 살펴봄으로써 무엇을 하고 어떻게 동작하는지 살펴보자.

`go_to_protected_mode` 함수에서 처음 수행하는 함수는 `realmode_switch_hook` 함수이다. 이 함수에서 기본 hook 은 인터럽트를 비활성화하고 [NMI](http://en.wikipedia.org/wiki/Non-maskable_interrupt) 를 금지 한다.
만약 real 모드 hook 을 `boot_params.hdr.realmode_swtch` 에 등록했다면 해당 등록된 것을 수행한다. Hook 은 만약 부트로더가 hostile 환경에서 수행된다면 사용된다. 당신은  [boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt)에서 **ADVANCED BOOT LOADER HOOKS** 챕터에서 볼 수 있다.

`realmode_switch` hook 은 16비트 real 모드의 NMI 를 비활성화하는 서브 루틴을 가리키는 포인터이다. `realmode_switch` hook 을 확인해보고(현재는 등록된 것이 없으니), 차단 불가능 인터럽트 (NMI) 를 불가능하게 만든다.

```assembly
asm volatile("cli");
outb(0x80, 0x70);	/* Disable NMI */
io_delay();
```

위의 어셈블리 코드에서 첫 명령어는 인터럽트 플래그(`IF`)를 모두 클리어 하는 `cli` 이다. 이 수행 다음에 모든 외부의 인터럽트가 비활성화 된다. 다음 라인에서 NMI (non-maskable interrupt)를 비활성화 한다.

인터럽트는 하드웨어나 소프트웨에가 CPU 로 전달하는 신호(signal) 이다. 이 신호를 받으면, CPU 는 현재 명령을 중지하고, 현재 상태를 저장 한뒤에 인터럽트 핸들러로 그 제어권을 넘긴다. 인터럽트 핸들러는 자신의 작업을 마친뒤에, 인터럽트로 중지된 명령어에서 시작 할 수 있도록 컨트롤을 돌려준다. 차단 불가능 인터럽트(NMI) 는 가장 우선순위가 높은 인터럽트이며, 항상 수행을 해야 한다. NMI 는 무시 될 수 없으며, 일반적으로 복구 불가능한 기계적 오류에 대한 신호를 처리하기 위해 사용된다. 우리는 지금은 인터럽트에 대해 자세히 다루지 않을 것이지만, 향후 다른 챕터에서 살펴 보도록 한다.

코드로 다시 돌아오자. 우리는 두 번째 라인에서 `0x70` (CMOS RAM/RTC 포트) 에 `0x80` 값을 쓰는데, 이것은 NMI 기능을 무력화한다. 이후에 `io_delay` 함수를 호출한다. `io_delay` 는 아래에서 보듯이 약간의 지연을 위한 코드다.:

```C
static inline void io_delay(void)
{
	const u16 DELAY_PORT = 0x80;
	asm volatile("outb %%al,%0" : : "dN" (DELAY_PORT));
}
```

`0x80` 포트에 어떤 바이트 값을 넣으면 정확이 1 밀리세컨드만큼 지연시킨다. 그래서 우리는 어떤 값(`AL` 레지스터에 있는 값)을 `0x80` 에 쓴다. 이 `realmode_switch_hook` 함수는 이 지연 이후에 실행을 마무리하고 다음 함수로 넘어간다:

다음 함수는 [A20 라인](http://en.wikipedia.org/wiki/A20_line) 을 활성화 하는 `enable_a20` 이다. 이 함수는 [arch/x86/boot/a20.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/a20.c) 에 정의되어 있으며 그것은 다른 방법으로 A20 게이트를 활성화 시키려고 한다. 첫 번째 수행은 A20 라인이 이미 활성화 되어 있는지 확인 하는 `a20_test_short` 함수를 호출한다.:

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

처음으로 `FS` 레지스터에 `0x0000`을 `GS` 레스터에는 `0xffff`을 넣는다. 다음에 코드는 `A20_TEST_ADDR` (`0x200` 이다.) 에 값을 읽고, `saved` 와 `ctr`의 변수에 저장을 해둔다.

다음에는 `ctr` 값을 `fs:gs` 에 `wrfs32` 함수로 업데이트하고, `io_delay`로 1 ms 동안 지연한다. 그런 다음 `A20_TEST_ADDR+0x10` 주소로 `Gs` 레지스터의 값을 읽고 이것이 만약 0 가 아니라면, 이미 A20라인은 활성화 되어 있는 것이다. 만약, A20 이 비활성화 상태라면, 우리는 그것을 `a20.c` 에서 찾을 수 있는 다른 방법으로 활성화 할 것이다. 예를 들어 `AH=0x2041`값 등과 `0x15` BIOS 인터럽트의 호출이 될 것이다.

만약 `enabled_a20` 함수가 실패한다면, 에러 메세지를 출력하고 `die` 함수를 호출한다. die 함수는 [arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S) 에서 확인 할 수 있다.:

```assembly
die:
	hlt
	jmp	die
	.size	die, .-die
```

A20 게이트가 성공적으로 활성화된 후에, `reset_coprocessor` 함수가 불린다:
 ```C
outb(0, 0xf0);
outb(0, 0xf1);
```

이 함수는 `0xf0` 에 `0` 을 써서 수학 코 프로세서(Coprocessor) 를 클리어 하고 `0xf1`에 `0`을 써서 리셋한다.

이 다음에, `mask_all_interrupts` 함수가 호출된다.:
```C
outb(0xff, 0xa1);       /* Mask all interrupts on the secondary PIC */
outb(0xfb, 0x21);       /* Mask all but cascade on the primary PIC */
```
슬레이브 PIC (Programmable Interrupt Controller) 의 모든 인터럽트를 마스크하고 마스터 PIC 에서는 IRQ2(슬레이브 PIC) 를 제외하고 마스크하여 모든 인터럽트를 금지한다.
[PIC 개념-한글](http://itguava.tistory.com/17)

이 모든 준비작업이 끝나면, 우리는 실제 보호 모드로 전환 할 수 있다.

인터럽트 디스크립터 테이블 설정
--------------------------------------------------------------------------------

이제 우리는 인터럽트 디크크립터 테이블(IDT - Interrupt Descriptor Table) 을 `setup_idt` 함수로 설정한다.:

```C
static void setup_idt(void)
{
	static const struct gdt_ptr null_idt = {0, 0};
	asm volatile("lidtl %0" : : "m" (null_idt));
}
```

위의 코드는 인터럽트 디크크립터 테이블(인터럽트 핸들러를 기술)를 설정한다. 아직은 IDT 가 설치되지 않은 상태이지만, 우리는 `lidtl` 명령어로 IDT 를 로드해둬야 한다. `null_idt` 는 주소와 IDT 의 크기르 갖고 있지만, 그 내용은 0 으로 채워져 있다. `null_idt` 는 `gdt_ptr` 구조체이며, 아래 처럼 정의되어 있다:

```C
struct gdt_ptr {
	u16 len;
	u32 ptr;
} __attribute__((packed));
```

우리는 IDT 의 16 비트 길이(`len`)와 그것을 가리킬 수 있는 32 비트 포인터를 볼 수 있다. (자세한 IDT 와 인터럽트에 관한 내용은 다음에 살펴보자) ` __attribute__((packed))` 은 `gdt_ptr` 의 크기를 최소 요구한 크기로 하겠다는 의미이다. 그래서 `gdt_ptr` 의 크기는 현재 6 바이트 또는 48 비트가 될 것이다. (다음에 우리는 `gdt_ptr`가 `GDTR` 레지스터를 가리키고 이전 글에서 살펴봤듯이 그것의 크기는 48 비트이다.)

글로벌 디스크립터 테이블(GDT) 설정
--------------------------------------------------------------------------------

여기서는 GDT(Global Descriptor Table) 설정을 본다. 우리는 GDT ([커널 부팅 과정. part2](linux-bootstrap-2.md#protected-mode) 에서 확인 할 수 있다.) 를 설정하는 `setup_gdt` 함수를 볼 수 있을 것이다. 이 함수에서  3개의 세그먼트를 정의하는 것을 포함한 `boot_gdt` 배열의 선언이 있다.:

```C
	static const u64 boot_gdt[] __attribute__((aligned(16))) = {
		[GDT_ENTRY_BOOT_CS] = GDT_ENTRY(0xc09b, 0, 0xfffff),
		[GDT_ENTRY_BOOT_DS] = GDT_ENTRY(0xc093, 0, 0xfffff),
		[GDT_ENTRY_BOOT_TSS] = GDT_ENTRY(0x0089, 4096, 103),
	};
```

코드, 데이터 와 TSS (Task State 세그먼트)를 위함 이다. 우리는 TSS 는 인텔 VT 를 행복(?) 하게 하기 위해 이 시점에 설정한 것이며, 지금은 TSS 를 사용하지 않을 것이다.(만약 당신이 여기에 관심이 있다면, 이 [커밋](https://github.com/torvalds/linux/commit/88089519f302f1296b4739be45699f06f728ec31)을 확인해보시길 바란다 ) `boot_gdt`를 보자. 처음 `__attribute__((aligned(16)))` 속성을 갖고 있다. 이 구조체는 16 바이트 정렬이 되어야 한다는 것이다. 아래의 간단 예제를 보자:

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

기술적으로 하나의 `int` 항목을 가지는 구조체는 4바이트가 되어야 하지만, `aligned` 구조체는 16바이트가 될 것이다.:

```
$ gcc test.c -o test && test
Not aligned - 4
Aligned - 16
```

`GDT_ENTRY_BOOT_CS` 는 인덱스를 가진다 - 여기서는 2, `GDT_ENTRY_BOOT_DS` 는 `GDT_ENTRY_BOOT_CS + 1` 이다. 그것은 2부터 시작한다. 인덱스 0 은 필수적으로 null(널) 디스크립터이고 인덱스 1은 사용하지 않는다.

`GDT_ENTRY` 는 플래그, 베이스 그리고 제한(limit) 를 가져와서 GDT 엔트리를 만드는 매크로 이다. 예를 들어 코드 세그먼트 엔트리를 보자. `GDT_ENTRY` 는 다음 값을 가진다.:

* base(베이스)  - 0
* limit(제한) - 0xfffff
* flags(플래그) - 0xc09b

이것이 어떤 의미일까? 그 세그먼트의 베이스 주소는 0, 그리고 제한은(세그먼트 크기) 는 `0xffff` (1MB). 플래그를 살펴보자. 그것은 `0xc09b` 고 2진수로는:

```
1100 0000 1001 1011
```

모든 비트의 의미를 이해해보자. 우리는 모든 비트를 왼쪽에서 오른쪽으로 살펴볼 것이다.:

* 1    - (G) granularity 비트; 바이트 단위 세그먼트 사이즈, 0 이면 4 바이트 단위 세그먼트 사이즈
* 1    - (D) 0 이면 16-bit 세그먼트; 1 = 32 비트 세그먼트
* 0    - (L) 만약 1이면 64 비트 모드로 수행
* 0    - (AVL) available for use by system software 시스템 소프트웨어에 사용 가능한 비트
* 0000 - 디스크립터 내에 4 비트 길이(19:16)
* 1    - (P) 메모리에 세그먼트의 존재 여부
* 00   - (DPL) - 특권 레벨, 0 이 가장 큰 특권 레벨
* 1    - (S) 코드 또는 데이터 세그먼트, 그렇지 않으면 시스템 세그먼트
* 101  - (EWA) 세그먼트 타입 실행/읽기/쓰기
* 1    - 접근 허용 비트

이전 [파트](linux-bootstrap-2.md)에서 보면 모든 비트에 대한 설명이 있다. 또는 [Intel® 64 and IA-32 Architectures Software Developer's Manuals 3A](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html) 보시길 바란다.

아래와 같이 GDT 의 길이를 얻을 수 있다:

```C
gdt.len = sizeof(boot_gdt)-1;
```

우리는 `boot_gdt` 의 크기를 얻고 1 을 빼준다.

다음은 우리는 GDT 를 위한 포인터를 얻는다:

```C
gdt.ptr = (u32)&boot_gdt + (ds() << 4);
```

여기서 우리는 단지 `boot_gdt` 주소를 얻고, 데이터 세그먼트 주소에 좌측 쉬프트 4비트 한 값을 더한다. (아직 우리는 real 모드인것을 기억하자.)

마지막으로 우리는 `lgdtl` 명령어를 실행하여 GDT 를 GDTR 레지스터에 로드한다.

```C
asm volatile("lgdtl %0" : : "m" (gdt));
```

드디어 보호 모드로의 진입
--------------------------------------------------------------------------------

여기는 `go_to_protected_mode` 함수의 마지막이다. 우리는 IDT, GDT, 인터럽트 비활성화 했으니 이제는 CPU 를 보호 모드로 전환할 수 있다. 마지막 단계는 2개의 인자와 `protected_mode_jump` 함수 호출로 이루어 진다.:

```C
protected_mode_jump(boot_params.hdr.code32_start, (u32)&boot_params + (ds() << 4));
```

[arch/x86/boot/pmjump.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/pmjump.S#L26) 에 정의되어 있다. 두 인자를 보자:

* 보호 모드 엔트리 포인트 주소
* `boot_params` 의 주소

`protected_mode_jump` 를 들여다 보자. 내가 위에서 썻듯이, 당신은 `arch/x86/boot/pmjump.S`에서 이것을 찾아 볼 수 있다. 첫 번째 인자는 `eax` 레지스터에 두 번째 인자는 `edx`에 있을 것이다.

무엇보다더 먼저 우리는 `boot_params` 의 주소를 `esi` 레지스터에 넣는다. 그리고 코드 세그먼트 레지스터인 `cs` (0x1000) 의 주소를 `bx` 에 넣는다. `bx` 를 4비트 만큼 왼쪽 쉬프트하고 `2` 라벨 주소를 더한다. (우리는 이것 다음에 `bx` 에서 라벨 `2`의 물리주소를 가질 수 있을 것이다.) 그리고 라벨 `1`로 점프한다. 다음에 우리는 데이터 세그먼트와 타스크 상태 세그먼트(TSS)를 각각 `cs`와 `di` 레지스터에 넣어준다.:

```assembly
movw	$__BOOT_DS, %cx
movw	$__BOOT_TSS, %di
```

`GDT_ENTRY_BOOT_CS` 는 인덱스 2이고 모든 GDT 엔트리는 8 바이트 이다. 그래서 `CS` 는 `2 * 8 = 16`이 될 것이고, `__BOOT_DS` 는 24가 될 것이다.

그다음 우리는 `CR0` 레지스터에 있는 `PE` (Protection Enable) 비트를 설정한다:

```assembly
movl	%cr0, %edx
orb	$X86_CR0_PE, %dl
movl	%edx, %cr0
```

그리고 보호 모드를 위해 긴 점프(long jump)를 만든다.:

```assembly
	.byte	0x66, 0xea
2:	.long	in_pm32
	.word	__BOOT_CS
```

where
* `0x66` 는 ljmpl 코드를 위한 피연산자(operand) 크기 설정,
* `0xea` - jump 코드,
* `in_pm32` 세그먼트 오프셋
* `__BOOT_CS` 코드 세그먼트.

여기서 `0x66 0xea` 의 의미를 조금 더 짚어보면, `0xea` 로 ljmpl 의 코드를 사용하는데 16 비트의 코드이기 때문에 jump 코드(`0xea`)는 주소 16:16 레지스터를 사용한다. 여기서 `0x66`(operand size override prefix) 를 사용해야 피연산자 크기가 32비트로 확장되어 주소 16:32가 된다.

마침내 보호 모드로 진입이 완료:

```assembly
.code32
.section ".text32","ax"
```

보호 모드에서 첫번째 수행을 살펴 보자. 제일 먼저 데이터 세그먼트를 설정한다:

```assembly
movl	%ecx, %ds
movl	%ecx, %es
movl	%ecx, %fs
movl	%ecx, %gs
movl	%ecx, %ss
```

집중을 했다면, 당신은 우리가 `cx` 레지스터에 `$__BOOT_DS` 값을 저장했다는 것을 기억할 것이다. 이제 우리는 모든 세그먼트 레지스터를 `cx` 값으로 채울 것이다.(`cs`는 이미 `__BOOT_CS` 이다). 다음으로 우리는 `eax`를 제왼한 모든 범용 레지스터를 0으로 채울 것이다.:

```assembly
xorl	%ecx, %ecx
xorl	%edx, %edx
xorl	%ebx, %ebx
xorl	%ebp, %ebp
xorl	%edi, %edi
```

그리고 32비트 엔트리 포인트로 점프:

```
jmpl	*%eax
```

`eax` 는 32 비트 엔트리의 주소를 갖고 있었다는 것을 기억하자.(우리는 그것은 `protected_mode_jump` 함수의 첫 번째 인자로 넘겼다)

이번 파트는 여기까지다. 우리는 보호 모드를 알아봤고 그 엔트리 포인터로 진입까지 진행했다. 다음 파트에서 이어가도록 하자.

결론
--------------------------------------------------------------------------------

리눅스 커널 인사이드의 3번째 파트의 끝이다. 다음 파트는 보호 모드에서 [롱 모드](http://en.wikipedia.org/wiki/Long_mode) 에 진입하는 과정을 살펴본다.

당신이 어떤 질문이나 제안이 있다면 [twitter](https://twitter.com/0xAX) - 원저자 에게 알려주길 바란다.

**나는 영어권의 사람이 아니고 이런 것에 대해 매우 미안해 하고 있다. 만약 어떤 실수를 발견한다면, 나에게 PR을 [linux-insides](https://github.com/0xAX/linux-internals)을 보내줘**

링크
--------------------------------------------------------------------------------

* [VGA](http://en.wikipedia.org/wiki/Video_Graphics_Array)
* [VESA BIOS Extensions](http://en.wikipedia.org/wiki/VESA_BIOS_Extensions)
* [Data structure alignment](http://en.wikipedia.org/wiki/Data_structure_alignment)
* [Non-maskable interrupt](http://en.wikipedia.org/wiki/Non-maskable_interrupt)
* [A20](http://en.wikipedia.org/wiki/A20_line)
* [GCC designated inits](https://gcc.gnu.org/onlinedocs/gcc-4.1.2/gcc/Designated-Inits.html)
* [GCC type attributes](https://gcc.gnu.org/onlinedocs/gcc/Type-Attributes.html)
* [Previous part](linux-bootstrap-2.md)
* [PIC 개념-한글](http://itguava.tistory.com/17)
* [GDT 의 필드 살펴보기-한글](http://kcats.tistory.com/156)
