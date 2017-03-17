커널 부팅 과정. part 1.
================================================================================

부트로더에서 커널로
--------------------------------------------------------------------------------

만약 당신이 나의 이전 [블로그 포스트](http://0xax.blogspot.com/search/label/asm)를 읽어봤다면, 얼마간 나는 low-level 프로그래밍에 관련된 일을 시작했음을 알 수 있을 것이다. 나는 그와 동시에 리눅스에서 x86_64 어셈블리 프로그래밍에 관련에서 글을 썻고, 리눅스 커널 소스 분석을 시작했다. 나는 low-level 에서 일어나는 일, 컴퓨터에서 프로그램이 어떻게 수행되는지, 프로그램이 메모리에 어떻게 위치하는지, 커널은 프로세스와 메모리를 어떻게 관리하는지, low-level 관점에서 network stack 은 어떻게 동작하는지 그리고 많은 다른 부분에 대해 큰 관심을 갖게 되었다. 그래서, 나는 **x86_64** 를 위한 리눅스 커널에 대해 또 다른 일련의 시리즈의 글을 쓰기로 결정했다.

나는 전문적인 리눅스 커널 해커도 아니고 리눅스 커널과 관련된 일을 하지도 않는다. 이것은 단지 취미일 뿐이다. 나는 low-level 에서 일어나는 것을 좋아한다. 그래서 만약 혼란스러운 어떤 것을 알려주거나 질문들이 있다면, twither [0xAX](https://twitter.com/0xAX) 나 [email](anotherworldofworld@gmail.com)을 보내거나 그냥 [issue](https://github.com/0xAX/linux-insides/issues/new) 를 만들어 달라. 나는 그런 것들에 감사한다. 모든 글을은 [linux-insides](https://github.com/0xAX/linux-insides) 에서 접근가능하고, 영어사용이나 글 내용에 문제가 있다면, 수정 후 pull request 를 보내길 바란다. 한글로 번역된 내용은 [linux-insides](https://github.com/0xAX/linux-insides) 이고, 번역에 문제나 글 내용의 수정 사항이 있다면 [issue](https://github.com/daeseokyoun/linux-insides/issues/new)를 이용 바란다.

*이것은 공식적인 문서가 아니라 단지 배우고 지식을 공유하기 위함이라는 것을 알려드린다.*

**요구 지식**

* C 언어를 이해
* 어셈블리 언어의 이해(AT&T syntax)

여기까지는 단순한 소개였고 이제는 리눅스 커널과 low-level 의 내용을 배워보도록 하자.

모든 코드는 3.18 커널 기준으로 되어 있다. 만약 여기에 변경이 있다면, 나는 그 포스트를 업데이트 할 것이다.

컴퓨터의 전원 인가시(파워 버트을 누르면), 다음에 일어나는 일은 무엇일까?
--------------------------------------------------------------------------------

비록 이 문서는 리눅스 커널에 관련된 시리즈의 글이지만, 우리는 리눅스 커널에서 부터 시작하지는 않을 것이다. 당신의 노트북이나 PC의 파워 버튼을 눌렀을 때, 일어나는 일부터 시작한다. 마더보드는 처음 신호를 [power supply-한글](https://namu.wiki/w/%ED%8C%8C%EC%9B%8C%20%EC%84%9C%ED%94%8C%EB%9D%BC%EC%9D%B4)로 보낸다. 신호를 받고 나면, Power supply 는 컴퓨터에 알맞은 전력으로 전원을 공급한다. 일단 마더보드에서 [power good signal](https://en.wikipedia.org/wiki/Power_good_signal)를 받으면, 마더보드는 CPU를 수행하려고 한다. CPU 는 모든 레지스트의 남은 데이터를 리셋하고 각각을 위해 미리 정의된 값으로 설정한다.

[80386-한글](https://ko.wikipedia.org/wiki/%EC%9D%B8%ED%85%94_80386) 과 이후의 CPU 들은 컴퓨터 리셋 이후에 CPU의 레지스터에 다음과 같은 미리 정의된 값으로 설정한다:

```
IP          0xfff0
CS selector 0xf000
CS base     0xffff0000
```

프로세서는 리얼 모드([real mode-한글](https://ko.wikipedia.org/wiki/%EB%A6%AC%EC%96%BC_%EB%AA%A8%EB%93%9C))에서 시작한다. 이 모드에서 메모리 세그먼트를 이해 해보도록 하자. Real mode 는 모든 x86 호완 프로세서([8086](https://ko.wikipedia.org/wiki/%EC%9D%B8%ED%85%94_8086) 에서 부터 최근 64비트 CPU 까지)에서 지원된다. 8086 프로세서는 20 비트의 주소 버스를 갖고 있고, 이것은 주소 공간 0 ~ 0xFFFFF (1 MB)에서 수행가능 하다는 의미 이다. 하지만 그 CPU는 16비트 레지스터만 갖고 있고, 이것은 최대 주소는 2^16 - 1(0xffff) -64 KB- 이라는 것이다. [메모리 세그먼트](https://ko.wikipedia.org/wiki/%EB%A9%94%EB%AA%A8%EB%A6%AC_%EC%84%B8%EA%B7%B8%EB%A8%BC%ED%8A%B8)는 모든 가용한 주소 공간을 사용할 수 있도록 한다. 모든 메모리는 작은, 65536 바이트(64KB)의 고정 크기 세그먼트로 나누어진다. 16비트 레지스터로 64 KB 이상의 메모리 접근이 불가능 하기 때문에 고안된 방법이다. 하나의 주소는 두 개의 부분으로 구성된다.: 베이스 주소인 세그먼트 셀렉터, 그리고 베이스 주소의 오프셋이다. Real mode 에서, 세그먼트 셀렉터의 연관된 기본 주소는 `세그먼트 셀렉터 * 16` 이다. 그러므로, 메모리의 물리적 주소를 얻기 위해서는, 우리는 세그먼트 셀렉터 부분에 16을 곱하고 오프셋을 더하면 된다:

```
물리 주소 = 세그먼트 셀렉터 * 16 + 오프셋
(PhysicalAddress = Segment Selector * 16 + Offset)
```

예를 들어, 만약 `CD:IP` 가 `0x2000:0x0010`이라면, 그에 대응되는 물리 주소는 아래와 같다:
```python
>>> hex((0x2000 << 4) + 0x0010)
'0x20010'
```

그러나, 만약 우리가 가장 큰 주소를 사용하려면, `0xffff:0xffff`을 사용할 수 있다. 그러면 아래와 같은 주소를 얻을 수 있다:
```python
>>> hex((0xffff << 4) + 0xffff)
'0x10ffef'
```

[A20 라인](https://sites.google.com/site/simplicitydrivendevelopment/un-yeongcheje/x86-un-yeongcheje-guhyeon/interrupt-and-exception)이 켜지기 전에 real mode 에서는 단지 1MB 만 접근이 가능하므로, `0x10ffef`의 주소는 `0x00ffef` 가 될 것이다.

이제 우리는 real mode 와 메모리 주소지정에 대해 알아보았다. 이제 리셋 이후의 레지스터 값에 대한 논의로 돌아가자.:

`CS` 레지스터는 두 개의 부분으로 구성된다: 보이는 부분-Visable part-(세그먼트 셀렉터)와 보이지 않는 부분-Hidden Part-(베이스 주소)로 나뉜다. 베이스 주소는 일반적으로 세그먼트 셀렉터에 16을 곱한 값으로 만들어지는 반면에 하드웨어 리셋 동안 CS 레지스터의 세그먼트 셀렉터에는 0xf000 이 로드되고 베이스 주소는 0xffff0000 이 로드 된다. 프로세서는 이 특별한 베이스 주소를 'CS' 가 바뀔 때까지 사용한다.
[추가 잠조 자료-한글](http://zkakira.tistory.com/entry/%EC%9A%B4%EC%98%81%EC%B2%B4%EC%A0%9C%EC%9D%98-%EB%A9%94%EB%AA%A8%EB%A6%AC-%EC%A3%BC%EC%86%8C-%EC%A7%80%EC%A0%95%EB%B0%A9%EC%8B%9D-Segment)

시작 주소는 EIP 레지스터 값에 베이스 주소를 더한 형태이다.:

```python
>>> 0xffff0000 + 0xfff0
'0xfffffff0'
```

우리는 `0xfffffff0`(4GB - 16 bytes) 를 얻었다. 이 지점은 [리셋 벡터](http://en.wikipedia.org/wiki/Reset_vector) 라 불린다. 리셋 이후에 실행되기 위한 첫 명령어를 찾을 수 있는 메모리 위치 이다. 그것은 BIOS 엔트리 포인트로 진입할 수 있는 [jump](https://ko.wikipedia.org/wiki/%EB%AA%85%EB%A0%B9%EC%96%B4_%EC%A7%91%ED%95%A9) (`jmp`) 명령어를 포함한다. 예를 들어, 만약 우리가 [coreboot](http://www.coreboot.org/) 소스를 보게 된다면, 아래와 같이 나온다:

```assembly
    .section ".reset"
    .code16
.globl  reset_vector
reset_vector:
    .byte  0xe9
    .int   _start - ( . + 2 )
    ...
```

여기에 우리가 `jmp` 명령의 [opcode](http://ref.x86asm.net/coder32.html#xE9)를 보면, 0xe9 라는 것을 알수 있고, `_start - ( . + 2)`에 목적지 주소가 있다. 우리는 또한 `reset` 섹션이 16바이트라는 것을 볼 수 있고 그것의 시작 주소는 `0xfffffff0` 이라는 걸 알 수 있다.:

```
SECTIONS {
    _ROMTOP = 0xfffffff0;
    . = _ROMTOP;
    .reset . : {
        *(.reset)
        . = 15 ;
        BYTE(0x00);
    }
}
```

이제 BIOS 가 시작된다; 하드웨어의 확인 및 초기화 이후에, BIOS는 부팅 가능한 장치를 찾는다. 부트 순서는 BIOS 설정에 저장되어 있고, BIOS 가 어떤 장치로 부터 시작해야 하는지 컨트롤 한다. 하드 드라이브로 부터 부팅을 시도할 때, BIOS는 부트섹터를 찾으려고 한다. 하드 드라이브 MBR 파티션을 포함하여 파티션이 나누어져 있는데, 부트 섹터는 첫번째 섹터의 446 바이트에 저장되어 있다. 첫 섹터의 마지막 두 바이트는 `0x55` 와 `0xaa` 인데 이것은 이 장치는 부팅에 사용 가능한 장치라는 것을 알려준다. 예를 들면:

```assembly
;
; Note: this example is written in Intel Assembly syntax
;
[BITS 16]
[ORG  0x7c00]

boot:
    mov al, '!'
    mov ah, 0x0e
    mov bh, 0x00
    mov bl, 0x07

    int 0x10
    jmp $

times 510-($-$$) db 0

db 0x55
db 0xaa
```

위 어셈블리를 빌드하고 실행하기:

```
nasm -f bin boot.nasm && qemu-system-x86_64 boot
```

이것은 디스크 이미지로 빌드된 `boot` 바이너리로 [QEMU](http://qemu.org)에서 사용되어 질 것이다. 부트섹터의 요구사항을 만족하는 위의 어셈블리 코드로 생성된 바이너리를 이용하면(BIOS는 이 코드를 0x7c00 에 로드하여 수행한다.), QEMU는 디스크 이미지의 마스터 부트 레코드(MBR-Master Boot Record) 로 생각할 것이다.

결과는:

![`!` 찍는 간단한 부트로더](http://oi60.tinypic.com/2qbwup0.jpg)

이 예제에서 우리는 16 비트 Real mode 에서 수행 될 것이고 메모리 `0x7c00` 번지에서 시작할 것이라는 것을 볼 수 있다. 시작 이후에, 부트로더는 `!`를 단지 출력하는 [0x10](http://www.ctyme.com/intr/rb-0106.htm) 인터럽트를 호출한다. 또한 남은 510 바이트를 영으로 채우고 마지막 2 바이트를 매직 값인 `0xaa`와 `0x55`를 넣는 것으로 마무리 한다.

당신은 `objdump` 유틸러티를 사용해서 이미지의 바이너리 덤프를 볼 수 있다.:
```
nasm -f bin boot.nasm
objdump -D -b binary -mi386 -Maddr16,data16,intel boot
```

실제 부트 섹터는 부트 과정을 계속할 수 있는 코드와 느낌표와 0으로 채워지는 대신에 파티션 테이블을 갖고 있다. 이 시점 이후로, BIOS는 부트로더에게 제어권을 넘겨준다.

**NOTE**: 위에서 설명했듯이, CPU는 real mode 내에 실행된다. real mode 내에서는 메모리의 물리주소를 계산 할때 아래와 같이 진행한다.:
```
PhysicalAddress = Segment Selector * 16 + Offset
```

이전 설명을 했듯이, 우리는 16 비트 범용 레지스터(General purpose register) 만 갖고 있다.; 16 비트의 최대 값은 `0xffff` 이며, 만얀 제일 큰 값을 만드려고 한다면, 아래와 같이 할 수 있을 것이다:

```python
>>> hex((0xffff * 16) + 0xffff)
'0x10ffef'
```

`0x10ffef`는 `1MB + 64KB - 16 bytes` 와 같다. 반대로 [8086](https://ko.wikipedia.org/wiki/%EC%9D%B8%ED%85%94_8086) 프로세서는(real mode 와 함께 한 첫 프로세서) 20 비트의 주소 라인을 갖는다. `2^20 = 1048576` 은 1MB 이기 때문에, 그것은 1MB 실제 가용한 메모리를 가질 수 있다는 의미 이다.

일반 적인 real mode 의 메모리 맵은 아래와 같다:
```
0x00000000 - 0x000003FF - Real Mode Interrupt Vector Table (Real mode 인터럽트 벡터 테이블)
0x00000400 - 0x000004FF - BIOS Data Area (BIOS 데이터 영역)
0x00000500 - 0x00007BFF - Unused (사용 안함)
0x00007C00 - 0x00007DFF - Our Bootloader (부트로더)
0x00007E00 - 0x0009FFFF - Unused (사용 안함)
0x000A0000 - 0x000BFFFF - Video RAM (VRAM) Memory (비디오 메모리)
0x000B0000 - 0x000B7777 - Monochrome Video Memory (단색 비디오 메모리)
0x000B8000 - 0x000BFFFF - Color Video Memory (컬러 비디오 메모리)
0x000C0000 - 0x000C7FFF - Video ROM BIOS (비디오 ROM BIOS)
0x000C8000 - 0x000EFFFF - BIOS Shadow Area (BIOS 쉐도우 영역)
0x000F0000 - 0x000FFFFF - System BIOS (시스템 BIOS)
```

이 챕터의 초반에, 나는 첫 명령어를 `0xFFFFFFF0` 주소에 위치에서 실행된다고 썻다. 그 주소는 `0xFFFFF` (1MB) 보다 큰 주소 값이다. 어떻게 Real mode 에서 CPU 는 이 주소에 접근이 가능했던 것일까? 이것에 대한 대답은 [coreboot](http://www.coreboot.org/Developer_Manual/Memory_map) 문서에 나온다:

```
0xFFFE_0000 - 0xFFFF_FFFF: 128 kilobyte ROM mapped into address space
```

실행의 시작은 BIOS 가 RAM에 있을 때가 아니라 ROM 에 있을 때이다.

부트로더
--------------------------------------------------------------------------------

리눅스를 부팅할 수 있는 [GRUB 2](https://www.gnu.org/software/grub/) 나 [syslinux](http://www.syslinux.org/wiki/index.php/The_Syslinux_Project)와 같은 부트로더들이 있다. 리눅스 커널은 리눅스를 지원하기 위한 부트로더가 구현해야 하는 요구사항들을 [Boot protocol](https://github.com/torvalds/linux/blob/master/Documentation/x86/boot.txt)로 명세했다. 이 예제는 GRUB 2 에서 설명 할 것이다.

계속 진행하기 전에, BIOS 는 부트 장치를 선택했고, 부트 섹터 코드로 제어권을 넘겨 [boot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/boot.S;hb=HEAD) 에서 실행을 시작했다. 이 코드는 공간 제약사항으로 인해 매우 간단하게 만들어져야 하고 GRUB 2의 코어 이미지의 위치로 점프하기 위한 포인터를 포함 하고 있다. 코어 이미지는 첫 파티션 전에 사용되지 않는 첫번째 섹터의 바로 다음에 저장되어 있는 [diskboot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/diskboot.S;hb=HEAD) 에서 시작된다. 위의 코드는 코어 이미지의 GRUB 의 커널과 파일 시스템을 핸들링 하기 위한 드라이버들을 포함한 나머지를 메모리로 로드한다. 코어 이미지의 나머지를 로드 한 후에, [grub_main](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/kern/main.c). 을 실행한다.

`grub_main`은 콘솔을 초기화 하고, 모듈을 위한 베이스 주소를 얻고, 루트 장치를 설정하고, grub 설정 파일을 분석(parse) 및 로드, 모듈 로드 등을 진행한다. 실행 마지막 부분에는, `grub_main`은 일반 모드(normal mode)로 이동한다. `grub_normal_execute`(`grub-core/normal/main.c`) 는 마지막 준비 작업을 완료하고 운영체제를 선택할 수 있는 메뉴를 보여준다. 우리가 grub 메뉴 항목중 하나를 선택했을 때, `grub_menu_execute_entry` 가 수행되고, grub 의 `boot` 명령어와 함께 선택된 운영체제가 부팅된다.

우리가 커널 부트 프로토콜에서 읽을 수 있는 것 처럼, 부트로더는 커널 설정 코드로 부터 `0x01f1` 오프셋에 있는 커널 설정 헤더의 일부 필드들을 반드시 읽고 채워야 한다. 커널 해더 [arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S) 아래와 같이 시작한다.:

```assembly
    .globl hdr
hdr:
    setup_sects: .byte 0
    root_flags:  .word ROOT_RDONLY
    syssize:     .long 0
    ram_size:    .word 0
    vid_mode:    .word SVGA_MODE
    root_dev:    .word 0
    boot_flag:   .word 0xAA55
```

부트로더는 이것과 헤더의 나머지 부분(이것은 단지 리눅스 부트 프로토콜에서 `write` 의 속성을 가지는 것들, 예를 들면 [예제](https://github.com/torvalds/linux/blob/master/Documentation/x86/boot.txt#L354))을 계산되거나 커맨드 라인으로 부터 받은 값으로 반드시 채워 넣어야 한다. (우리는 커널 설정 헤더의 모든 항목들을 위한 설명은 하지 않을 것이다. 대신에 커널이 그것들을 사용할 때, 논의 해보도록 하자; 당신은 [boot protocol](https://github.com/torvalds/linux/blob/master/Documentation/x86/boot.txt#L156)에서 모든 항목에 대한 설명을 찾을 수 있을 것이다.)

우리가 커널 부트 프로토콜에서 볼 수 있듯이, 커널 로딩 이후에 메모리 맵은 다음과 같을 것이다.:

```shell
         | Protected-mode kernel  | 보호 모드 커널
100000   +------------------------+
         | I/O memory hole        | I/O 메모리 구멍(메모리 접근을 해도 의미 없는)
0A0000   +------------------------+
         | Reserved for BIOS      | 가능한한 많이 확보
         ~                        ~
         | Command line           | (X+10000 까지 사용가능)
X+10000  +------------------------+
         | Stack/heap             | 커널 real-mode 코드에 의해 사용
X+08000  +------------------------+
         | Kernel setup           | 커널 real-mode 코드.
         | Kernel boot sector     | 커널 레거시 부트 섹터
       X +------------------------+
         | Boot loader            | <- 부트섹터 엔트리 포인트 0x7C00
001000   +------------------------+
         | Reserved for MBR/BIOS  |
000800   +------------------------+
         | Typically used by MBR  |
000600   +------------------------+
         | BIOS use only          |
000000   +------------------------+

```

그래서, 부트로더가 커널로 제어권을 넘길 때, 아래와 같이 시작:

```
X + sizeof(KernelBootSector) + 1
```

`X` 는 커널 부트 섹터가 로드된 주소이다. 나의 경우에 `X`는 메모리 덤프에서 볼 수 있듯이 `0x10000` 이다.:

![커널 시작 주소](http://oi57.tinypic.com/16bkco2.jpg)

부트로더는 리눅스 커널을 메모리에 로드하고, 헤더 항목들을 채우고 나서 알맞은 메모리 주소로 점프했다. 우리는 바로 커널 설정 코드로 넘어갈 수 있다.

커널 설정의 시작
--------------------------------------------------------------------------------

마침내, 우리는 커널로 진입했다. 기술적으로, 커널은 아직 실행을 한 것은 아니다. 가장 처음, 우리는 커널, 메모리 관리자 그리고 프로세스 관리자 등을 설정할 필요가 있다. 커널 설정 실행은 [arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S)의 [_start](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L293)에서 시작된다. 그것의 실행전에 몇가지의 명령어가 수행되기 때문에, 그것은 처음이라고 하기엔 이상합니다.

아주 오래전에, 리눅스 커널은 자신만의 부트로더를 갖고 사용했었다. 하지만 지금은, 실행한다면, 예를 들어,

```
qemu-system-x86_64 vmlinuz-3.18-generic
```

그러면 당신이 볼 수 있는 것은:

![vmlinuz를 qemu 로 실행](http://oi60.tinypic.com/r02xkz.jpg)

실제로, `header.S` 는 [MZ](https://en.wikipedia.org/wiki/DOS_MZ_executable) 로 부터 시작한다(위의 이미지를 보라.), 에러 메세지는 [PE](https://en.wikipedia.org/wiki/Portable_Executable) header 에 따라 메세지가 출력된다:

```assembly
#ifdef CONFIG_EFI_STUB
# "MZ", MS-DOS header
.byte 0x4d
.byte 0x5a
#endif
...
...
...
pe_header:
    .ascii "PE"
    .word 0
```
그것은 [UEFI](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface) 와 함께 운영체제가 로드되기 위해 필요하다. 우리는 지금 당장 내부 동작을 살펴보진 않고 다음 챕터에서 다루어 보도록 하자.

실제 커널 설정 엔트리 포인트는:

```assembly
// header.S line 292
.globl _start
_start:
```

The bootloader (grub2 and others) knows about this point (`0x200` offset from `MZ`) and makes a jump directly to it, despite the fact that `header.S` starts from the `.bstext` section, which prints an error message:
부트로더(grub2 와 다른 것들)은 이 포인트(`MZ`로 부터 오프셋 `0x200`)를 알고 있고 거기로 직접 jump 를 한다. `header.S` 의 `.bstext` 섹션에서 시작하는 사실에도 불구하고 에러 메세지를 출력한다.
```
//
// arch/x86/boot/setup.ld
//
. = 0;                    // 현 위치
.bstext : { *(.bstext) }  // .bstext 섹션을 위치 0으로 넣기
.bsdata : { *(.bsdata) }
```

커널 설정 엔트리 포인트는:

```assembly
    .globl _start
_start:
    .byte  0xeb
    .byte  start_of_setup-1f
1:
    //
    // rest of the header
    //
```

여기서 우리는 `jmp` 명령코드(`0xeb`) 를 통해 `start_of_setup-1f` 로 점프한다는 것을 볼 수 있다. `Nf` 표기는, `2f`로 했을 때, 로컬에서 `2:` 라벨을 찾아가라는 의미, 현재 상황에는 점프 바로 다음에 나오는 라벨 `1`이 되고, 그것은 설정의 나머지를 포함 한다[header](https://github.com/torvalds/linux/blob/master/Documentation/x86/boot.txt#L156). 설정 헤더의 바로 다음에, 우리는 `start_of_setup` 라벨에서 시작하는 `.entrytext` 섹션을 볼 수 있다.

아래는 실제로 수행되는 첫 코드이다.(물론, 이전 점프 명령들을 제외하고). 부트로더로 부터 제어권을 커널이 받고 나서는, 처음 `jmp` 명령은 커널 real mode의 시작지점(첫 512 바이트 다음)으로 부터 `0x200` 떨어진 곳에 위치한다. 이것을 통해 우리는 리눅스 커널 부터 프로토콜과 grub2 소스 코드를 읽을 수 있게 되었다.:

```C
segment = grub_linux_real_target >> 4;
state.gs = state.fs = state.es = state.ds = state.ss = segment;
state.cs = segment + 0x20;
```

이것의 의미는 세그먼트 레지스터들이 커널 설정이 시작된 이후에 아래와 같은 값을 가지게 된다는 것이다:
```
gs = fs = es = ds = ss = 0x1000
cs = 0x1020
```

나의 경우에는, 커널은 `0x10000`에 로드된다.

`start_of_setup` 으로 점프 후에, 커널을 아래의 작업을 해야 한다.:

* 모든 세그먼트 레지스터는 같은 값을 갖도록 확인해야 한다.
* 필요하다면, 알맞은 스택을 설정.
* [bss](https://en.wikipedia.org/wiki/.bss) 설정
* C 코드인 [main.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c) 로 점프

이제 구현을 살펴보자.

Segment registers align
--------------------------------------------------------------------------------

First of all, the kernel ensures that `ds` and `es` segment registers point to the same address. Next, it clears the direction flag using the `cld` instruction:

```assembly
    movw    %ds, %ax
    movw    %ax, %es
    cld
```

As I wrote earlier, grub2 loads kernel setup code at address `0x10000` and `cs` at `0x1020` because execution doesn't start from the start of file, but from

```assembly
_start:
    .byte 0xeb
    .byte start_of_setup-1f
```

`jump`, which is at a 512 byte offset from [4d 5a](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L47). It also needs to align `cs` from `0x10200` to `0x10000`, as well as all other segment registers. After that, we set up the stack:

```assembly
    pushw   %ds
    pushw   $6f
    lretw
```

which pushes the value of `ds` to the stack with the address of the [6](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L494) label and executes the `lretw` instruction. When the `lretw` instruction is called, it loads the address of label `6` into the [instruction pointer](https://en.wikipedia.org/wiki/Program_counter) register and loads `cs` with the value of `ds`. Afterwards, `ds` and `cs` will have the same values.

Stack Setup
--------------------------------------------------------------------------------

Almost all of the setup code is in preparation for the C language environment in real mode. The next [step](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L467) is checking the `ss` register value and making a correct stack if `ss` is wrong:

```assembly
    movw    %ss, %dx
    cmpw    %ax, %dx
    movw    %sp, %dx
    je      2f
```

This can lead to 3 different scenarios:

* `ss` has valid value 0x10000 (as do all other segment registers beside `cs`)
* `ss` is invalid and `CAN_USE_HEAP` flag is set     (see below)
* `ss` is invalid and `CAN_USE_HEAP` flag is not set (see below)

Let's look at all three of these scenarios in turn:

* `ss` has a correct address (0x10000). In this case, we go to label [2](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L481):

```assembly
2:  andw    $~3, %dx
    jnz     3f
    movw    $0xfffc, %dx
3:  movw    %ax, %ss
    movzwl  %dx, %esp
    sti
```

Here we can see the alignment of `dx` (contains `sp` given by bootloader) to 4 bytes and a check for whether or not it is zero. If it is zero, we put `0xfffc` (4 byte aligned address before the maximum segment size of 64 KB) in `dx`. If it is not zero, we continue to use `sp`, given by the bootloader (0xf7f4 in my case). After this, we put the `ax` value into `ss`, which stores the correct segment address of `0x10000` and sets up a correct `sp`. We now have a correct stack:

![stack](http://oi58.tinypic.com/16iwcis.jpg)

* In the second scenario, (`ss` != `ds`). First, we put the value of [_end](https://github.com/torvalds/linux/blob/master/arch/x86/boot/setup.ld#L52) (the address of the end of the setup code) into `dx` and check the `loadflags` header field using the `testb` instruction to see whether we can use the heap. [loadflags](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L321) is a bitmask header which is defined as:

```C
#define LOADED_HIGH     (1<<0)
#define QUIET_FLAG      (1<<5)
#define KEEP_SEGMENTS   (1<<6)
#define CAN_USE_HEAP    (1<<7)
```

and, as we can read in the boot protocol,

```
Field name: loadflags

  This field is a bitmask.

  Bit 7 (write): CAN_USE_HEAP
    Set this bit to 1 to indicate that the value entered in the
    heap_end_ptr is valid.  If this field is clear, some setup code
    functionality will be disabled.
```

If the `CAN_USE_HEAP` bit is set, we put `heap_end_ptr` into `dx` (which points to `_end`) and add `STACK_SIZE` (minimum stack size, 512 bytes) to it. After this, if `dx` is not carried (it will not be carried, dx = _end + 512), jump to label `2` (as in the previous case) and make a correct stack.

![stack](http://oi62.tinypic.com/dr7b5w.jpg)

* When `CAN_USE_HEAP` is not set, we just use a minimal stack from `_end` to `_end + STACK_SIZE`:

![minimal stack](http://oi60.tinypic.com/28w051y.jpg)

BSS Setup
--------------------------------------------------------------------------------

The last two steps that need to happen before we can jump to the main C code are setting up the [BSS](https://en.wikipedia.org/wiki/.bss) area and checking the "magic" signature. First, signature checking:

```assembly
    cmpl    $0x5a5aaa55, setup_sig
    jne     setup_bad
```

This simply compares the [setup_sig](https://github.com/torvalds/linux/blob/master/arch/x86/boot/setup.ld#L39) with the magic number `0x5a5aaa55`. If they are not equal, a fatal error is reported.

If the magic number matches, knowing we have a set of correct segment registers and a stack, we only need to set up the BSS section before jumping into the C code.

The BSS section is used to store statically allocated, uninitialized data. Linux carefully ensures this area of memory is first zeroed using the following code:

```assembly
    movw    $__bss_start, %di
    movw    $_end+3, %cx
    xorl    %eax, %eax
    subw    %di, %cx
    shrw    $2, %cx
    rep; stosl
```

First, the [__bss_start](https://github.com/torvalds/linux/blob/master/arch/x86/boot/setup.ld#L47) address is moved into `di`. Next, the `_end + 3` address (+3 - aligns to 4 bytes) is moved into `cx`. The `eax` register is cleared (using a `xor` instruction), and the bss section size (`cx`-`di`) is calculated and put into `cx`. Then, `cx` is divided by four (the size of a 'word'), and the `stosl` instruction is used repeatedly, storing the value of `eax` (zero) into the address pointed to by `di`, automatically increasing `di` by four, repeating until `cx` reaches zero). The net effect of this code is that zeros are written through all words in memory from `__bss_start` to `_end`:

![bss](http://oi59.tinypic.com/29m2eyr.jpg)

Jump to main
--------------------------------------------------------------------------------

That's all - we have the stack and BSS, so we can jump to the `main()` C function:

```assembly
    calll main
```

The `main()` function is located in [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c). You can read about what this does in the next part.

Conclusion
--------------------------------------------------------------------------------

This is the end of the first part about Linux kernel insides. If you have questions or suggestions, ping me on twitter [0xAX](https://twitter.com/0xAX), drop me an [email](anotherworldofworld@gmail.com), or just create an [issue](https://github.com/0xAX/linux-internals/issues/new). In the next part, we will see the first C code that executes in the Linux kernel setup, the implementation of memory routines such as `memset`, `memcpy`, `earlyprintk`, early console implementation and initialization, and much more.

**Please note that English is not my first language and I am really sorry for any inconvenience. If you find any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-internals).**

Links
--------------------------------------------------------------------------------

  * [Intel 80386 programmer's reference manual 1986](http://css.csail.mit.edu/6.858/2014/readings/i386.pdf)
  * [Minimal Boot Loader for Intel® Architecture](https://www.cs.cmu.edu/~410/doc/minimal_boot.pdf)
  * [8086](http://en.wikipedia.org/wiki/Intel_8086)
  * [80386](http://en.wikipedia.org/wiki/Intel_80386)
  * [Reset vector](http://en.wikipedia.org/wiki/Reset_vector)
  * [Real mode](http://en.wikipedia.org/wiki/Real_mode)
  * [Linux kernel boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt)
  * [CoreBoot developer manual](http://www.coreboot.org/Developer_Manual)
  * [Ralf Brown's Interrupt List](http://www.ctyme.com/intr/int.htm)
  * [Power supply](http://en.wikipedia.org/wiki/Power_supply)
  * [Power good signal](http://en.wikipedia.org/wiki/Power_good_signal)
