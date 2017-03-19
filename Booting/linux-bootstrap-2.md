커널 부팅 과정. part 2.
================================================================================

커널 설정의 처음 단계
--------------------------------------------------------------------------------

우리는 이전 [part](linux-bootstrap-1.md) 에서 리눅스를 깊게 파보기를 시작했고 커널 설정 코드의 초기화 부분을 보았다. 우리는 `main` 함수[arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c). 의 호출에서 이어 나가기로 한다.(C 로 작성된 첫 함수이다.)

이 파트에서 우리는 커널 설정 코드를 계속해서 살펴볼 것이고
* `보호 모드` 가 무엇인지 보고
* 어떤 준비는 그것에 진입하기 위한 전환을 위한 것이다.
* 힙과 콘솔 초기화,
* memory detection, cpu validation, keyboard initialization
* 메모리 검출, CPU 확인, 키보드 초기화
* 그리고 많은 부분이 남아 있다.

이제 시작해보자.

보호 모드(Protected mode)
--------------------------------------------------------------------------------

우리가 Intel64 [Long mode](http://en.wikipedia.org/wiki/Long_mode) 로 넘어가기 전에, 커널은 CPU 를 보호 모드로 전환해야 한다. ([Long mode-한글](https://ko.wikipedia.org/wiki/%EB%A1%B1_%EB%AA%A8%EB%93%9C))

[보호 모드-한글](https://ko.wikipedia.org/wiki/보호_모드)/[Protected mode](https://en.wikipedia.org/wiki/Protected_mode) 란 무엇인가? 보호 모드는 1982년 x86 아키텍처에 처음 추가된 모드이며, Intel 64 와 long mode 가 나타나지 전에 [80286](http://en.wikipedia.org/wiki/Intel_80286) 프로세서 계열인 인텔 프로세서의 주요(main) 모드였다.

[Real mode](http://wiki.osdev.org/Real_Mode)로 부터 변환되는 주요 이유는 메모리 접근이 매우 제한 적이였기 때문이다. 이전 파트의 내용을 기억한다면, Real mode 에서는 2<sup>20</sup> 바이트 또는 1 메가 바이트만이 있고, 때로는 640 킬로바이트 메모리만 가용적이었다.

보호 모드는 많은 변화를 가져왔고, 메모리 관리에서 주요한 하나의 차이를 갖고 있다. 20 비트 주소 버스는 32 비트 주소 버스로 교체된다. 그것은 real mode 에서 1 MB 접근하던 메모리를 4 GB 까지 접근 허용하겠다는 것이다. 또한 다음 섹션에서 설명할 [paging](http://en.wikipedia.org/wiki/Paging)/[페이징-한글](https://ko.wikipedia.org/wiki/%ED%8E%98%EC%9D%B4%EC%A7%95) 의 지원이 추가된다.

보호 모드에서 메모리 관리는 두 가지로 분리가능 하고, 두 부분이 굉장히 독립적이다.:

* 세그먼트
* 페이징

여기서 우리는 단지 세그먼트의 내용만 볼 것이다. 페이징은 다음 섹션에서 논의해보도록 하자.

이전 파트에서 읽을 수 있었듯이, 주소는 real mode 에서 두 개의 부분으로 구성된다:

* 세그먼트의 베이스 주소
* 베이스 주소에서의 오프셋

만약 우리가 이 두가지의 부분을 다 알고 있다면 물리 주소를 얻을 수 있을 것이다.:

```
PhysicalAddress = Segment Selector * 16 + Offset
```

메모리 세그먼트는 보호 모드에서 완전히 다르게 사용된다. 이제는 64 KB 고정 크기 세그먼트는 없다. 대신에, 각 세그먼트의 크기와 위치는 _세그먼트 디스크립터_ 라 불리는 자료 구조에 연관된 내용이 저장될 것이다. 그 세그먼트 디스크립터는 `Global Descriptor Table`(GDT) 라 불리는 자료구조에 저장된다.

GDT는 메모리에 존재한는 구조체 이다. 이것은 메모리네애 고정된 위치는 아니고, 특수한 `GDTR` 레지스터에 그 주소가 저장된다. 리눅스 커널 코드에서 GDT가 로딩되는 것은 나중에 살펴볼 것이다. 그것을 메모리로 로딩하는 아래와 같은 명령들이 있다.:

```assembly
lgdt gdt
```

`lgdt` 명령어는 베이스 주소와 global descriptor table 의 크기(제약)을 `GDTR` 레지스터에 로드 한다. `GDTR`은 48 비트 레지스터고 두 개의 파트로 구성된다.:

 * global descriptor table 의 크기(16 비트)
 * global descriptor table 의 32비트 주소

위에서 언급했듯이 GDT는 메모리 세그먼트를 기술하는 `세그먼트 디스크립터`를 갖고 있다. 각 디스크립터는 64 비트의 크기를 갖는다. 일반적인 디스크립터의 구성은 다음과 같다:

```
31          24        19      16              7            0
------------------------------------------------------------
|             | |B| |A|       | |   | |0|E|W|A|            |
| BASE 31:24  |G|/|L|V| LIMIT |P|DPL|S|  TYPE | BASE 23:16 | 4
|             | |D| |L| 19:16 | |   | |1|C|R|A|            |
------------------------------------------------------------
|                             |                            |
|        BASE 15:0            |       LIMIT 15:0           | 0
|                             |                            |
------------------------------------------------------------
```

걱정 마라, 나는 real mode 이후에 매우 무섭게 변한 것을 알고 있지만 그것은 알고 보면 쉽다. 예를 들어 LIMIT 15:0 은 디스크립터의 0-15 비트의 값은 제한(limit)의 값을 표현한다. 그리고 LIMIT 19:16 에는 나머지가 있다. 그래서 Limit 의 크기는 0-19 비트로 총 20 비트가 된다. 조금 더 살펴보자:

1. Limit[20 비트] 는 0-15, 16-19 비트에 있다. 그것은 `length_of_segment - 1`을 정의한다. 그것은 `G`(Granularity) 비트에 의존적이다.

  * 만약 `G` (비트 55) 가 0 이고 세그먼트 제한(limit) 이 0 이면 세그먼트의 크기는 1 바이트가 된다.
  * 만약 `G` 가 0 이고 세그먼트 제한이 0xffff 이면, 세그먼트의 크기는 1 MB 이다.
  * 만약 `G` 가 1 이고 세그먼트 제한이 0 이면 세그먼트 크기는 4096 바이트이다.
  * 만약 `G` 가 1 이고 세그먼트 제한이 0xfffff 이면, 세그먼트의 크기는 4GB 이다.

  그래서 그것들이 의미하는 것은,
  * 만약 `G` 가 0 이면, 제한(Limit) 의 크기는 1 바이트 에서 최대 1 MB 가 된다고 해석될 수 있다.
  * 만약 `G` 가 1 이면, 제한(Limit) 의 크기는 4096 바이트 = 4 KB = 1 페이지에서 최대 4 GB 의 크기의 세그먼트를 가질 수 있다고 해석된다. 실제로 `G` 가 1일 때, Limit 의 값은 12 비트 왼쪽 쉬프트가 된다. 그래서 20 비트 + 12 비트 = 32 비트 이고, 2<sup>32</sup> = 4 GB이다.

2. 베이스[32 비트] 는 0-15, 32-39 그리고 56-53 비트에 있다. 그것은 세그먼트의 시작 위치의 물리 주소를 정의한다.

3. 타입/속성 (40-47 비트) 는 세그먼트의 타입과 그것의 접근 권한에 대한 정의를 한다.
  * 44 비트에 있는 `S` 플래그는 디스크립터의 타입을 결정한다. 만얀 `S` 가 0 이면, 그 세그먼트는 시스템 세그먼트 이고, 반대로 `S`가 1 이면, 그 세그먼트는 코드 또는 데이터 세그먼트이다. (스택 세그먼트는 읽기와 쓰기 권한이 주어진 데이터 세그먼트라고 할 수 있다.)

만약 세그먼트가 코드나 데이터 세그먼트인지 결정하기 위해서는 우리는 Ex(43 비트) 의 속성이 0 으로 되어 있는지 확인 해야 한다. 만약 그것이 0이면 데이터 세그먼트 이고 1이면 코드 세그먼트가 된다.

세그먼트는 아래 나오는 타입중에 하나일 것이다.:

```
|           Type Field        | Descriptor Type | Description
|-----------------------------|-----------------|------------------
| Decimal                     |                 |
|             0    E    W   A |                 |
| 0           0    0    0   0 | Data            | Read-Only
| 1           0    0    0   1 | Data            | Read-Only, accessed
| 2           0    0    1   0 | Data            | Read/Write
| 3           0    0    1   1 | Data            | Read/Write, accessed
| 4           0    1    0   0 | Data            | Read-Only, expand-down
| 5           0    1    0   1 | Data            | Read-Only, expand-down, accessed
| 6           0    1    1   0 | Data            | Read/Write, expand-down
| 7           0    1    1   1 | Data            | Read/Write, expand-down, accessed
|                  C    R   A |                 |
| 8           1    0    0   0 | Code            | Execute-Only
| 9           1    0    0   1 | Code            | Execute-Only, accessed
| 10          1    0    1   0 | Code            | Execute/Read
| 11          1    0    1   1 | Code            | Execute/Read, accessed
| 12          1    1    0   0 | Code            | Execute-Only, conforming
| 14          1    1    0   1 | Code            | Execute-Only, conforming, accessed
| 13          1    1    1   0 | Code            | Execute/Read, conforming
| 15          1    1    1   1 | Code            | Execute/Read, conforming, accessed
```

위의 테이블에서 처음 비트가(Ex 43 비트) 0이면 `데이터 세그먼트`고 `1` 이면 `코드 세그먼트` 이다. 다음 3 비트는 (40, 41, 42) 는 각각 `EWA`(Expansion Writeable Accessible) 또는 `CRA` (conforming Readable Accessible) 로 나타난다.
  * 만약 E(비트 42) 가 0 이면, Expand up 세그먼트이고 1이면 expand down 세그먼트다. 더 알고 싶다면, [여기](http://www.sudleyplace.com/dpmione/expanddown.html).
  * 만약 W(비트 41)(데이터 세그먼트를 위함) 이 1이면, 쓰기 접근이 가능하고 0이면 안된다. 읽기 접근은 데이터 세그먼트의 경우 항상 허용된다.
  * A(비트 40) - 프로세서에 의해 이 세그먼트가 접근 가능 한지 아닌지에 대한 설정.
  * C(비트 42) 는 코드 세그먼트를 위한 규칙을 수행하도록 한다. 만약 C 가 1이면, 세그먼트의 코드는 낮은 단계의 특권 레벨로 실행가능 할 것이고 0이라면 같은 특권 레벨에서 수행되어야 할 것이다.
  * R(비트 41)(코드 세그먼트). 만약 1이면 세그먼트의 읽기 허용이 되고 0이면 안된다. 코드 세그먼트로 쓰기 접근은 절대로 허용되지 않는다.

4. DPL[2 비트] (디스크립터 특권 레벨-Privilege level)이고 45-46 비트에 위치한다. 그것은 세그먼트의 특권 레벨을 정의한다. 특권 레벨은 0-3 까지 지정 가능하며 0이 가장 높은 특권을 가리킨다.

5. P 플래그(비트 47) - 만약 세그먼트가 메모리에 현재 로드되어 있는지 안되어 있는지를 알려준다. 만약 0이면 세그먼트는 _invalid_ 상태인 것이며, 프로세서가 그 세그먼트를 읽으려고 할 때, 허용하지 않을 것이다.

6. AVL 플래그(비트 52) - 예약된 비트 공간이다. 리눅스에서는 무시된다.

7. L 플래그(비트 53) - 코드 세그먼트가 64 비트 코드를 포함하는지 여부를 알려준다. 만약 1이면 코드 세그먼트는 64 비트 모드에서 수행 되고 있다는 것이다.

8. D/B 플래그(비트 54) - Default/Big 플래그이며 피연산자(operand)의 크기가 16/32 비트인지를 알려준다. 만약 1이면 32비트이고 아니면 16 비트이다.

세그먼트 레지스터들은 real mode 에서 사용된 세그먼트 셀렉터를 포함한다. 하지만, 보호 모드내에서는, 세그먼트 셀렉터는 다르게 응용된다. 각 세그먼트 디스크립터는 연관된 16비트로 구성된 세그먼트 셀렉터를 가지고 있다.:
```
15              3  2   1  0
-----------------------------
|      Index     | TI | RPL |
-----------------------------
```

위의 테이블에서,
* **Index** GDT 에서 디스크립터의 인덱스 번호를 보여준다.
* **TI**(테이블 지시자-Table Indicator) 은 디스크립의 위치를 알려준다. 만약 0 이면 GDT 에 있다는 것이고 1이면 LDT(Local Descriptor Table) 에 있다는 것이다.
* 그리고 **RPL** 는 요청한 시점에 특권 레벨을 알수 있다.

Every segment register has a visible and hidden part.
모든 세그먼트 레지스터는 보여지는 부분(visible)과 숨겨진(hidden) 부분이 있다.
* 보여지는 부분(Visible) - 세그먼트 셀렉터가 여기에 저장
* 숨겨진 부분(Hidden) - 세그먼트 디스크립터(베이스, 제한(limit), 속성, 플래그들)

보호 모드에서 물리주소를 얻기 위해서는 다음과 같은 단계를 거처야 한다:

* 세그먼트 셀렉터는 세그먼트 레지스터 중 하나에 반드시 로드되어 있어야 한다.
* CPU 는 셀렉터로 부터 GDT 주소 + 인덱스에 있는 세그먼트 디스크립터를 찾고, 세그먼트 레지스터의 *hidden(숨겨진)* 부분에 이 디스크립터를 로드 해야 한다.
* 베이스 주소 (세그먼트 디스크립터로 부터) + 오프셋은 세그먼트의 선형 주소가 될 것이고 이것이 물리주소이다.(만약 페이징이 활성화되지 않은 상태라면)

개략적으로 그것은 아래와 같이 보일 것이다.:

![linear address](http://oi62.tinypic.com/2yo369v.jpg)

이 알고리즘은 real mode 에서 보호 모드로 전환을 하기 위한 것이다:

* 인터럽트 중지
* `lgdt` 명령어로 GDT 를 기술하고 로드
* CR0 (Control Register 0) 에 있는 PE (Protection Enable) 비트를 셋
* 보호 모드로 진입

우리는 다음 파트에서 리눅스 커널의 보호 모드로 전환하는 것을 완벽하게 살펴볼 것이다. 그전에 보호모드에 진입하려면, 몇 가지 준비사항에 대해 알아 볼 필요가 있다.

[arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c)를 살펴보자. 우리는 키보드 초기화, 힙 초기화등의 과정들을 볼 수 있다.

부트 파라미터(인자)를 "zeropage"내로 복사.
--------------------------------------------------------------------------------

우리는 "main.c" 파일에서 `main` 루틴에서 시작할 것이다. `main` 에서 처음 불리는 함수는 [`copy_boot_params(void)`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c#L30). 이다. 이것은 커널 설정 해더를 [arch/x86/include/uapi/asm/bootparam.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/uapi/asm/bootparam.h#L113)에 정의된 `boot_params` 구조체로 복사한다.

`boot_params` 구조체는 `struct setup_header hdr` 자료구조를 포함한다. 이 자료 구조는 [linux boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt)에 정의된 같은 항목들을 포함하고, 이 자료구조는 부트로더에서 또는 커널 컴파일/빌드 시점에 내용이 채워진다. `copy_boot_params` 는 두 가지 일을 한다:

1. `hdr` 을 [header.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L281) 에서 `boot_params` 구조체의 `setup_header` 항목으로 복사한다.

2. 만약 커널이 이미 지난 스타일의 커맨드 라인 프로토콜에 의해 로드되었다면 커널 커맨드 라인을 업데이트 한다.

`hdr` 은 [copy.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/copy.S) 에 구현된 `memcpy`를 이용하여 복사된다. 그 내부를 살펴보자:

```assembly
GLOBAL(memcpy)
    pushw   %si
    pushw   %di
    movw    %ax, %di
    movw    %dx, %si
    pushw   %cx
    shrw    $2, %cx
    rep; movsl
    popw    %cx
    andw    $3, %cx
    rep; movsb
    popw    %di
    popw    %si
    retl
ENDPROC(memcpy)
```

우리는 이제 막 C 코드로 진입을 했고 다시 에셈블리 코드를 보게 되었다. 이 파일의 처음으로 보여지는 `memcpy` 와 다른 함수들의 구현들을 볼 수 있을 것이고, 이 함수들의 시작과 끝을 `GLOBAL` 과 `ENDPROC`이라는 두 개의 매크로를 이용했다. `GLOBAL` 은 [arch/x86/include/asm/linkage.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/linkage.h) 에 정의되어 있고, 전달되는 이름을 `global` 로 심볼을 외부 참조가 가능하도록 만든다. `ENDPROC` 은 [include/linux/linkage.h](https://github.com/torvalds/linux/blob/master/include/linux/linkage.h) 에 있고, 그 함수 이름으로의 `name`의 심볼 크기와 그 끝을 알려주는 역할을 한다.

`memcpy`의 구현은 아주 간단하다. 첫째로, `memcpy`를 수행하는 동안 값이 변경될 `si` 와 `di` 레지스터의 이전 값을 보존하기 위해 스택에 넣어둔다. `memcpy` (그리고 copy.S 의 다른 함수들도)는 `fastcall` 함수 호출 규약(calling conventions)을 사용한다.([calling conventions-한글](http://wendys.tistory.com/22)) 그래서 들어오는 인자들을 각각 순서대로 `ax`, `dx` 그리고 `cx` 레지스터에 저장한다. `memcpy` 호출을 아래와 같이 이루어 진다.:

```c
memcpy(&boot_params.hdr, &hdr, sizeof hdr);
```

그래서,
* `ax` 는 `boot_params.hdr`의 주소를 가진다.
* `dx` 는 `hdr` 의 주소를 가진다.
* `cx` 는 `hdr`의 크기를 바이트 단위로 가진다.

`memcpy` 는 `boot_params.hdr`의 주소를 `di` 에 넣고 스택에 그 크기를 저장한다. 다음에 크기를 2 만큰 우측 쉬프트 연산(4로 나누는 것)을 하고 `si` 에서 `di` 에 4 바이트 만큼 복사한다. 이 다음엔 다시 `hdr` 의 크기를 스택에서 꺼내고, 4 바이트 정렬을 한다. 그리고 나머지 바이트를 `si`에서 `di`로 바이트 단위로 복사한다.(복사할 것이 남았다면) 복사가 완료되면 스택으로 부터 원래 `si`와 `di` 값을 꺼내어 복구한다.

콘솔 초기화
--------------------------------------------------------------------------------

`hdr`이 `boot_params.hdr`로 복사 되면, 다음 단계로는 [arch/x86/boot/early_serial_console.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/early_serial_console.c)에 정의된 `console_init` 함수를 호출하여 콘솔을 초기화한다.

그것은 커맨드 라인에 `earlyprintk` 옵션이 있는지 찾고, 만약 검색이 성공적이었다면, 그것은 포트 주소와 시리얼 포트의 baud rate 를 파싱하고 시리얼 포트를 초기화한다. `earlyprintk` 와 연관된 커맨드 라인 옵션의 값을 아래와 같은 형식으로 찾을 수 있다.:

* serial,0x3f8,115200
* serial,ttyS0,115200
* ttyS0,115200

After serial port initialization we can see the first output:
시리얼 포트 초기화 이후에 우리는 첫 출력을 볼 수 있다.

```C
if (cmdline_find_option_bool("debug"))
    puts("early console in setup code\n");
```

`puts` 는 [tty.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/tty.c) 에 구현되어 있다. 이것은 `putchar` 함수를 루프 내에서 호출하여 character 단위로 출력한다는 것을 볼 수 있다. `putchar`의 구현을 살펴보자.:

```C
void __attribute__((section(".inittext"))) putchar(int ch)
{
    if (ch == '\n')
        putchar('\r');

    bios_putchar(ch);

    if (early_serial_base != 0)
        serial_putchar(ch);
}
```

`__attribute__((section(".inittext")))` 은 `.inittext` 내에 이 코드를 놓을 것이라는 의미이다. 우리는 [setup.ld](https://github.com/torvalds/linux/blob/master/arch/x86/boot/setup.ld#L19) 의 링커 파일에서 찾을 수 있다.

첫번재로, `putchar`은 `\n` 심볼을 확인하고 만약 `\n`이라면, `\r`을 출력한다. 그 다음에 BIOS 에서 `0x10` 인터럽트 호출과 함께 VGA 화면에 문자 하나가 출력된다.:

```C
static void __attribute__((section(".inittext"))) bios_putchar(int ch)
{
    struct biosregs ireg;

    initregs(&ireg);
    ireg.bx = 0x0007;
    ireg.cx = 0x0001;
    ireg.ah = 0x0e;
    ireg.al = ch;
    intcall(0x10, &ireg, NULL);
}
```

여기에 `initregs`는 `biosregs` 구조체를 받아 `biosregs`를 `memset`을 이용해서 0 으로 채우고 거기에 레지스터가 갖고 있는 값들로 채워준다.

```C
    memset(reg, 0, sizeof \*reg);
    reg->eflags |= X86_EFLAGS_CF;
    reg->ds = ds();
    reg->es = ds();
    reg->fs = fs();
    reg->gs = gs();
```

[memset](https://github.com/torvalds/linux/blob/master/arch/x86/boot/copy.S#L36) 구현을 살펴보자:

```assembly
GLOBAL(memset)
    pushw   %di
    movw    %ax, %di
    movzbl  %dl, %eax
    imull   $0x01010101,%eax
    pushw   %cx
    shrw    $2, %cx
    rep; stosl
    popw    %cx
    andw    $3, %cx
    rep; stosb
    popw    %di
    retl
ENDPROC(memset)
```

위의 코드를 보면, `memcpy`와 같이 `fastcall` 함수 호출 규약을 사용한다. 그래서 각 인자들을 `ax`, `dx` 그리고 `cx` 레지스터에 저장한다.

일반적으로 `memset` 은 memcpy 구현과 같다. 그것을 `di`레지스터 값을 스택에 저장하고 `ax`의 값(`biosregs` 구조체의 주소)을 `di`로 넣는다. 다음에 `movzbl` 명령어를 사용하여 `dl` 의 값을 하위 2 바이트를 `eax` 레지스터로 복사한다. `eax`의 남은 상위 2바이트는 0으로 채워질 것이다.

다음 명령어는 `eax`에 `0x01010101`을 곱하는 것이다. 이것은 `memset` 이 이와 동시에 4 바이트 복사를 하기 때문에 필요하다. 예를 들어, 우리가 memset 으로 어떤 구조체에 `0x7` 을 채운다면, `eax`는 `0x00000007` 의 값을 갖고 있을 것이다. 만약 `eax`에 `0x01010101`을 곱하면, 우리는 `0x07070707` 을 얻을 것이고, 이제 우리는 이 4 바이트를 구조체로 복사 할 수 있을 것이다. `memset`은 `eax`의 값을 `es:di`에 복사하기 위해 `rep; stosl` 명령어를 사용한다.

`memset` 함수의 나머지 부분은 `memcpy`와 거의 같다.

`biosregs` 구조체가 `memset`으로 채워지고 나면, `bios_putchar` 가 [0x10](http://www.ctyme.com/intr/rb-0106.htm) 인터럽트와 함께 호출되어 문자 하나를 출력한다. 그런 다음에 이 함수는 시리억 포트가 초기화 되어 있는지 확인 하여 설정되어 있다면 문자 하나를 [serial_putchar](https://github.com/torvalds/linux/blob/master/arch/x86/boot/tty.c#L30)와 `inb/outb` 명령어를 통해 쓴다.

Heap initialization
--------------------------------------------------------------------------------

After the stack and bss section were prepared in [header.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S) (see previous [part](linux-bootstrap-1.md)), the kernel needs to initialize the [heap](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c#L116) with the [`init_heap`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c#L116) function.

First of all `init_heap` checks the [`CAN_USE_HEAP`](https://github.com/torvalds/linux/blob/master/arch/x86/include/uapi/asm/bootparam.h#L21) flag from the [`loadflags`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L321) in the kernel setup header and calculates the end of the stack if this flag was set:

```C
    char *stack_end;

    if (boot_params.hdr.loadflags & CAN_USE_HEAP) {
        asm("leal %P1(%%esp),%0"
            : "=r" (stack_end) : "i" (-STACK_SIZE));
```

or in other words `stack_end = esp - STACK_SIZE`.

Then there is the `heap_end` calculation:
```c
    heap_end = (char *)((size_t)boot_params.hdr.heap_end_ptr + 0x200);
```
which means `heap_end_ptr` or `_end` + `512`(`0x200h`). The last check is whether `heap_end` is greater than `stack_end`. If it is then `stack_end` is assigned to `heap_end` to make them equal.

Now the heap is initialized and we can use it using the `GET_HEAP` method. We will see how it is used, how to use it and how the it is implemented in the next posts.

CPU validation
--------------------------------------------------------------------------------

The next step as we can see is cpu validation by `validate_cpu` from [arch/x86/boot/cpu.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/cpu.c).

It calls the [`check_cpu`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/cpucheck.c#L102) function and passes cpu level and required cpu level to it and checks that the kernel launches on the right cpu level.
```c
check_cpu(&cpu_level, &req_level, &err_flags);
if (cpu_level < req_level) {
    ...
    return -1;
}
```
`check_cpu` checks the cpu's flags, presence of [long mode](http://en.wikipedia.org/wiki/Long_mode) in case of x86_64(64-bit) CPU, checks the processor's vendor and makes preparation for certain vendors like turning off SSE+SSE2 for AMD if they are missing, etc.

Memory detection
--------------------------------------------------------------------------------

The next step is memory detection by the `detect_memory` function. `detect_memory` basically provides a map of available RAM to the cpu. It uses different programming interfaces for memory detection like `0xe820`, `0xe801` and `0x88`. We will see only the implementation of **0xE820** here.

Let's look into the `detect_memory_e820` implementation from the [arch/x86/boot/memory.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/memory.c) source file. First of all, the `detect_memory_e820` function initializes the `biosregs` structure as we saw above and fills registers with special values for the `0xe820` call:

```assembly
    initregs(&ireg);
    ireg.ax  = 0xe820;
    ireg.cx  = sizeof buf;
    ireg.edx = SMAP;
    ireg.di  = (size_t)&buf;
```

* `ax` contains the number of the function (0xe820 in our case)
* `cx` register contains size of the buffer which will contain data about memory
* `edx` must contain the `SMAP` magic number
* `es:di` must contain the address of the buffer which will contain memory data
* `ebx` has to be zero.

Next is a loop where data about the memory will be collected. It starts from the call of the `0x15` BIOS interrupt, which writes one line from the address allocation table. For getting the next line we need to call this interrupt again (which we do in the loop). Before the next call `ebx` must contain the value returned previously:

```C
    intcall(0x15, &ireg, &oreg);
    ireg.ebx = oreg.ebx;
```

Ultimately, it does iterations in the loop to collect data from the address allocation table and writes this data into the `e820entry` array:

* start of memory segment
* size  of memory segment
* type of memory segment (which can be reserved, usable and etc...).

You can see the result of this in the `dmesg` output, something like:

```
[    0.000000] e820: BIOS-provided physical RAM map:
[    0.000000] BIOS-e820: [mem 0x0000000000000000-0x000000000009fbff] usable
[    0.000000] BIOS-e820: [mem 0x000000000009fc00-0x000000000009ffff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000000f0000-0x00000000000fffff] reserved
[    0.000000] BIOS-e820: [mem 0x0000000000100000-0x000000003ffdffff] usable
[    0.000000] BIOS-e820: [mem 0x000000003ffe0000-0x000000003fffffff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000fffc0000-0x00000000ffffffff] reserved
```

Keyboard initialization
--------------------------------------------------------------------------------

The next step is the initialization of the keyboard with the call of the [`keyboard_init()`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c#L65) function. At first `keyboard_init` initializes registers using the `initregs` function and calling the [0x16](http://www.ctyme.com/intr/rb-1756.htm) interrupt for getting the keyboard status.
```c
    initregs(&ireg);
    ireg.ah = 0x02;     /* Get keyboard status */
    intcall(0x16, &ireg, &oreg);
    boot_params.kbd_status = oreg.al;
```
After this it calls [0x16](http://www.ctyme.com/intr/rb-1757.htm) again to set repeat rate and delay.
```c
    ireg.ax = 0x0305;   /* Set keyboard repeat rate */
    intcall(0x16, &ireg, NULL);
```

Querying
--------------------------------------------------------------------------------

The next couple of steps are queries for different parameters. We will not dive into details about these queries, but will get back to it in later parts. Let's take a short look at these functions:

The [query_mca](https://github.com/torvalds/linux/blob/master/arch/x86/boot/mca.c#L18) routine calls the [0x15](http://www.ctyme.com/intr/rb-1594.htm) BIOS interrupt to get the machine model number, sub-model number, BIOS revision level, and other hardware-specific attributes:

```c
int query_mca(void)
{
    struct biosregs ireg, oreg;
    u16 len;

    initregs(&ireg);
    ireg.ah = 0xc0;
    intcall(0x15, &ireg, &oreg);

    if (oreg.eflags & X86_EFLAGS_CF)
        return -1;  /* No MCA present */

    set_fs(oreg.es);
    len = rdfs16(oreg.bx);

    if (len > sizeof(boot_params.sys_desc_table))
        len = sizeof(boot_params.sys_desc_table);

    copy_from_fs(&boot_params.sys_desc_table, oreg.bx, len);
    return 0;
}
```

It fills  the `ah` register with `0xc0` and calls the `0x15` BIOS interruption. After the interrupt execution it checks  the [carry flag](http://en.wikipedia.org/wiki/Carry_flag) and if it is set to 1, the BIOS doesn't support [**MCA**](https://en.wikipedia.org/wiki/Micro_Channel_architecture). If carry flag is set to 0, `ES:BX` will contain a pointer to the system information table, which looks like this:

```
Offset  Size    Description
 00h    WORD    number of bytes following
 02h    BYTE    model (see #00515)
 03h    BYTE    submodel (see #00515)
 04h    BYTE    BIOS revision: 0 for first release, 1 for 2nd, etc.
 05h    BYTE    feature byte 1 (see #00510)
 06h    BYTE    feature byte 2 (see #00511)
 07h    BYTE    feature byte 3 (see #00512)
 08h    BYTE    feature byte 4 (see #00513)
 09h    BYTE    feature byte 5 (see #00514)
---AWARD BIOS---
 0Ah  N BYTEs   AWARD copyright notice
---Phoenix BIOS---
 0Ah    BYTE    ??? (00h)
 0Bh    BYTE    major version
 0Ch    BYTE    minor version (BCD)
 0Dh  4 BYTEs   ASCIZ string "PTL" (Phoenix Technologies Ltd)
---Quadram Quad386---
 0Ah 17 BYTEs   ASCII signature string "Quadram Quad386XT"
---Toshiba (Satellite Pro 435CDS at least)---
 0Ah  7 BYTEs   signature "TOSHIBA"
 11h    BYTE    ??? (8h)
 12h    BYTE    ??? (E7h) product ID??? (guess)
 13h  3 BYTEs   "JPN"
 ```

Next we call the `set_fs` routine and pass the value of the `es` register to it. The implementation of `set_fs` is pretty simple:

```c
static inline void set_fs(u16 seg)
{
    asm volatile("movw %0,%%fs" : : "rm" (seg));
}
```

This function contains inline assembly which gets the value of the `seg` parameter and puts it into the `fs` register. There are many functions in [boot.h](https://github.com/torvalds/linux/blob/master/arch/x86/boot/boot.h) like `set_fs`, for example `set_gs`, `fs`, `gs` for reading a value in it etc...

At the end of `query_mca` it just copies the table pointed to by `es:bx` to the `boot_params.sys_desc_table`.

The next step is getting [Intel SpeedStep](http://en.wikipedia.org/wiki/SpeedStep) information by calling the `query_ist` function. First of all it checks the CPU level and if it is correct, calls `0x15` for getting info and saves the result to `boot_params`.

The following [query_apm_bios](https://github.com/torvalds/linux/blob/master/arch/x86/boot/apm.c#L21) function gets [Advanced Power Management](http://en.wikipedia.org/wiki/Advanced_Power_Management) information from the BIOS. `query_apm_bios` calls the `0x15` BIOS interruption too, but with `ah` = `0x53` to check `APM` installation. After the `0x15` execution, `query_apm_bios` functions check the `PM` signature (it must be `0x504d`), carry flag (it must be 0 if `APM` supported) and value of the `cx` register (if it's 0x02, protected mode interface is supported).

Next it calls `0x15` again, but with `ax = 0x5304` for disconnecting the `APM` interface and connecting the 32-bit protected mode interface. In the end it fills `boot_params.apm_bios_info` with values obtained from the BIOS.

Note that `query_apm_bios` will be executed only if `CONFIG_APM` or `CONFIG_APM_MODULE` was set in the configuration file:

```C
#if defined(CONFIG_APM) || defined(CONFIG_APM_MODULE)
    query_apm_bios();
#endif
```

The last is the [`query_edd`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/edd.c#L122) function, which queries `Enhanced Disk Drive` information from the BIOS. Let's look into the `query_edd` implementation.

First of all it reads the [edd](https://github.com/torvalds/linux/blob/master/Documentation/kernel-parameters.txt#L1023) option from the kernel's command line and if it was set to `off` then `query_edd` just returns.

If EDD is enabled, `query_edd` goes over BIOS-supported hard disks and queries EDD information in the following loop:

```C
for (devno = 0x80; devno < 0x80+EDD_MBR_SIG_MAX; devno++) {
    if (!get_edd_info(devno, &ei) && boot_params.eddbuf_entries < EDDMAXNR) {
        memcpy(edp, &ei, sizeof ei);
        edp++;
        boot_params.eddbuf_entries++;
    }
    ...
    ...
    ...
    }
```

where `0x80` is the first hard drive and the value of `EDD_MBR_SIG_MAX` macro is 16. It collects data into the array of [edd_info](https://github.com/torvalds/linux/blob/master/include/uapi/linux/edd.h#L172) structures. `get_edd_info` checks that EDD is present by invoking the `0x13` interrupt with `ah` as `0x41` and if EDD is present, `get_edd_info` again calls the `0x13` interrupt, but with `ah` as `0x48` and `si` containing the address of the buffer where EDD information will be stored.

Conclusion
--------------------------------------------------------------------------------

This is the end of the second part about Linux kernel insides. In the next part we will see video mode setting and the rest of preparations before transition to protected mode and directly transitioning into it.

If you have any questions or suggestions write me a comment or ping me at [twitter](https://twitter.com/0xAX).

**Please note that English is not my first language, And I am really sorry for any inconvenience. If you find any mistakes please send me a PR to [linux-insides](https://github.com/0xAX/linux-internals).**

Links
--------------------------------------------------------------------------------

* [Protected mode](http://en.wikipedia.org/wiki/Protected_mode)
* [Protected mode](http://wiki.osdev.org/Protected_Mode)
* [Long mode](http://en.wikipedia.org/wiki/Long_mode)
* [Nice explanation of CPU Modes with code](http://www.codeproject.com/Articles/45788/The-Real-Protected-Long-mode-assembly-tutorial-for)
* [How to Use Expand Down Segments on Intel 386 and Later CPUs](http://www.sudleyplace.com/dpmione/expanddown.html)
* [earlyprintk documentation](http://lxr.free-electrons.com/source/Documentation/x86/earlyprintk.txt)
* [Kernel Parameters](https://github.com/torvalds/linux/blob/master/Documentation/kernel-parameters.txt)
* [Serial console](https://github.com/torvalds/linux/blob/master/Documentation/serial-console.txt)
* [Intel SpeedStep](http://en.wikipedia.org/wiki/SpeedStep)
* [APM](https://en.wikipedia.org/wiki/Advanced_Power_Management)
* [EDD specification](http://www.t13.org/documents/UploadedDocuments/docs2004/d1572r3-EDD3.pdf)
* [TLDP documentation for Linux Boot Process](http://www.tldp.org/HOWTO/Linux-i386-Boot-Code-HOWTO/setup.html) (old)
* [Previous Part](linux-bootstrap-1.md)
