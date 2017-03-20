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

2. 만약 커널이 이미 지난 스타일의 명령 라인 프로토콜에 의해 로드되었다면 커널 명령 라인을 업데이트 한다.

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

그것은 명령 라인에 `earlyprintk` 옵션이 있는지 찾고, 만약 검색이 성공적이었다면, 그것은 포트 주소와 시리얼 포트의 baud rate 를 파싱하고 시리얼 포트를 초기화한다. `earlyprintk` 와 연관된 명령 라인 옵션의 값을 아래와 같은 형식으로 찾을 수 있다.:

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
    memset(reg, 0, sizeof *reg);
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

힙 초기화
--------------------------------------------------------------------------------

[header.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S) 에서 스택과 bss 영역이 준비된 이후에(이전 [파트](linux-bootstrap-1.md) 참조), 커널은 [`init_heap`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c#L116) 함수를 통해 [힙](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c#L116)을 초기화 해야 한다.

`init_heap` 에서 우선 커널 설정 헤더에 [`loadflags`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L321) 로 부터 [`CAN_USE_HEAP`](https://github.com/torvalds/linux/blob/master/arch/x86/include/uapi/asm/bootparam.h#L21) 플래그를 확인하고 만약 이 플래그가 설정되어 있다면 스택의 마지막 위치를 계산한다:

```C
    char *stack_end;

    if (boot_params.hdr.loadflags & CAN_USE_HEAP) {
        asm("leal %P1(%%esp),%0"
            : "=r" (stack_end) : "i" (-STACK_SIZE));
```

위의 코드를 표현하면 `stack_end = esp - STACK_SIZE` 가 된다.

다음 `heap_end`를 계산한다.:

```c
    heap_end = (char *)((size_t)boot_params.hdr.heap_end_ptr + 0x200);
```

`heap_end` 는 (`heap_end_ptr` 또는 `_end`) + `512`(`0x200h`) 이다. 마지막으로는 `heap_end` 가 `stack_end` 보다 큰지 확인한다. 만약 `stack_end` 보다 크다면 `heap_end` 를 `stack_end` 와 같게 해준다.

이제 힙이 초기화 되었고 우리는 `GET_HEAP` 메서드를 사용하여 힙을 이용할 수 있다. 우리는 어떻게 사용되는지, 어떻게 사용하는지 그리고 어떻게 구현이 되었는지 살펴 볼 것이다.

CPU 유효성 검사(validation)
--------------------------------------------------------------------------------

다음 단계는 [arch/x86/boot/cpu.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/cpu.c) 에서 `validate_cpu` 로 CPU 유효성 검사를 진행한다.

그것은 [`check_cpu`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/cpucheck.c#L102)  함수를 현재 CPU 레벨과 요구되는 CPU 레벨을 인자로 넘겨 알맞은 CPU 레벨로 커널이 구동될 수 있도록 확인한다.
```c
check_cpu(&cpu_level, &req_level, &err_flags);
if (cpu_level < req_level) {
    ...
    return -1;
}
```

`check_cpu` 는 CPU 의 플래그를 확인하는데, x86_64(64 비트) CPU 의 경우 [long mode](http://en.wikipedia.org/wiki/Long_mode) 가 있는지도 확인해야 한다. 프로세서의 벤더를 확인하고 AMD 칩을 위해 SSE+SSE2 의 옵션을 끄는 것과 같이 특정 벤더를 위한 작업도 마무리한다.

메모리 검출(Memory detection)
--------------------------------------------------------------------------------

다음 단계는 `detect_memory` 함수로 메모리 검출을 진행한다. `detect_memory` 는 기본적으로 가용한 RAM 의 맵을 CPU 에게 제공한다. 그것은 `0xe820`, `0xe801` 그리고 `0x88` 과 같은 메모리 검출을 위한 다른 프로그래밍 인터페이스를 사용한다. 우리는 여기서 **0xE820** 의 구현만 살펴 볼 것이다.

`detect_memory_e820` 의 구현을 [arch/x86/boot/memory.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/memory.c) 에서 볼 수 있다. 제일 먼저, `detect_memory_e820` 함수는 `biosregs` 구조체를 앞서 봤듯이 초기화 하고 `0xe820` 호출을 위한 특별한 값들을 레지스터에 채운다.:

```assembly
    initregs(&ireg);
    ireg.ax  = 0xe820;
    ireg.cx  = sizeof buf;
    ireg.edx = SMAP;
    ireg.di  = (size_t)&buf;
```

* `ax` 함수의 숫자를 포함한다.(우리 경우 0xe820)
* `cx` 레지스터는 메모리 테이터를 갖고 있는 버퍼의 크기를 가진다.
* `edx` 는 반드시 `SMAP`(534D4150h) 매직 넘버를 가져야 한다.
* `es:di` 메모리 데이터가 있는 버퍼의 주소를 가진다.
* `ebx` 는 0 여야 한다.

다음에는 루프를 통해 메모리를 맵 정보를 모은다. 주소 할당 테이블로 부터 한 라인을 쓰는 `0x15` BIOS 인터럽트 호출로 부터 시작한다. 다음 라인의 정보를 얻기 위해 우리는 인터럽트를 다시 호출 할 필요가 있다.(우리는 루프 내에 있다.) 다음 `ebx` 호출 전에 `ebx`는 이전에 반환된 값을 갖고 있어야 한다.:

```C
    intcall(0x15, &ireg, &oreg);
    ireg.ebx = oreg.ebx;
```

궁극적으로, 그것은 주소 할당 테이블로 부터 데이터를 루프를 돌면서 모으고 `e820entry` 배열에 데이터를 써준다.:

* 메모리 세그먼트의 시작
* 메로리 세그먼트의 크기
* 메모리 세그먼트의 타입 (예약된, 가용한(usable) 등.)

우리는 이 결과를 `dmesg` 출력을 통해 e820 맵을 확인할 수 있다:

```
[    0.000000] e820: BIOS-provided physical RAM map:
[    0.000000] BIOS-e820: [mem 0x0000000000000000-0x000000000009fbff] usable
[    0.000000] BIOS-e820: [mem 0x000000000009fc00-0x000000000009ffff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000000f0000-0x00000000000fffff] reserved
[    0.000000] BIOS-e820: [mem 0x0000000000100000-0x000000003ffdffff] usable
[    0.000000] BIOS-e820: [mem 0x000000003ffe0000-0x000000003fffffff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000fffc0000-0x00000000ffffffff] reserved
```

키보드 초기화
--------------------------------------------------------------------------------

다음 단계는 [`keyboard_init()`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c#L65) 함수의 호출로 키보드를 초기화한다. `keyboard_init` 는 `initregs` 함수를 사용해서 레지스터를 초기화하고 키보드 상태를 얻기 위해 [0x16](http://www.ctyme.com/intr/rb-1756.htm) 인터럽트를 호출한다.
```c
    initregs(&ireg);
    ireg.ah = 0x02;     /* Get keyboard status */
    intcall(0x16, &ireg, &oreg);
    boot_params.kbd_status = oreg.al;
```
이 다음 키보드 반복 속도(keyboard repeat rate) 와 delay 를 설정하기 위해 [0x16](http://www.ctyme.com/intr/rb-1757.htm) 를 다시 호출한다.
```c
    ireg.ax = 0x0305;   /* Set keyboard repeat rate */
    intcall(0x16, &ireg, NULL);
```

Querying(질의)
--------------------------------------------------------------------------------

다음 몇 단계는 다른 인자들을 위해 질의 하는 것이다. 우리는 질의(query)에 대해 자세히 다루지 않을 것이지만, 이후 나오는 파트에서 다시 살펴 보자. 이 함수에 대해 간단히 살펴보자:

[query_mca](https://github.com/torvalds/linux/blob/master/arch/x86/boot/mca.c#L18) 루틴은 메신 모델 번호, 서브 모델 번호, BIOS 리비전 레벨 그리고 하드웨어 특성 값을 얻기 위해 [0x15](http://www.ctyme.com/intr/rb-1594.htm) BIOS 인터럽트를 호출한다.:

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

`ah` 레지스터를 `0xc0` 으로 채우고, `0x15` BIOS 인터럽트 호출을 한다. 인터럽트 실행은 [carry flag](http://en.wikipedia.org/wiki/Carry_flag) 를 확인하고 만약 그것이 1 이면, BIOS 는 [**MCA**](https://en.wikipedia.org/wiki/Micro_Channel_architecture) 을 지원하지 않는다는 것을 알수 있다. 만약 carry 플래그가 0 이면, `ES:BX` 는 아래와 같이 시스템 정보 테이블를 가리키는 주소를 가질 것이다.:

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

다음 우리는 `set_fs` 루틴을 호출하고 `es` 레지스터의 값을 전달한다. `set_fs`의 구현은 꽤나 간단하다.:

```c
static inline void set_fs(u16 seg)
{
    asm volatile("movw %0,%%fs" : : "rm" (seg));
}
```

이 함수는 `seg` 인자의 값을 얻기 위한 인라인 어셈블리를 가지고 그 값을 `fs` 레지스터로 넣어준다. `set_fs` 와 같은 많은 함수들이 [boot.h](https://github.com/torvalds/linux/blob/master/arch/x86/boot/boot.h) 파일에 있다. 예를 들어 레지스터 값을 읽기 위해 `set_fs`, `fs`, `gs` 가 있다.

`query_mcs` 의 마지막에는 `es:bx` 에 의해 알고 있는 테이블을 `boot_params.sys_desc_table`로 복사한다.

다음 단계는 `query_ist` 함수를 통해 [Intel SpeedStep](http://en.wikipedia.org/wiki/SpeedStep) 정보를 얻는 것이다. 그것은 CPU 레벨을 확인하고 만약 맞다면, 정보를 얻기 위해 `0x15` 를 호출하고 그 결과를 `boot_params` 에 저장한다.

다음 오는 [query_apm_bios](https://github.com/torvalds/linux/blob/master/arch/x86/boot/apm.c#L21) 함수는 [Advanced Power Management](http://en.wikipedia.org/wiki/Advanced_Power_Management) 정보을 BIOS 로 부터 얻는다. `query_apm_bios` 는 `0x15` BIOS 인터럽트 호출을 `APM` 설치 확인 하기 위해 `ah` = `0x53` 과 함께 호출한다. `0x15` 실행 이후에, `query_apm_bios` 함수는 `PM` 시그너처(`0x504d`)를 확인하고, carry 플래그(이것은 `AMP` 가 지원된다면, 0 이어야 한다) 그리고 `cx` 레지스터의 값(만약 0x2라면 보호 모드 인터페이스가 지원된다.)을 확인한다.

다음은 `AMP` 인터페이스와 연결을 끊고 32 비트 보호 모드 인터페이스를 연결하기 위해 `ax = 0x5304` 함께 `0x15` 호출을 다시 한다. 마지막에는 BIOS 로 부터 얻어진 값들을 `boot_params.apm_bios_info` 채운다.

만약 커널 설정에 `CONFIG_APM` 또는 `CONFIG_APM_MODULE` 이 설정되어 있다면, `query_apm_bios` 가 실행될 것이다.:

```C
#if defined(CONFIG_APM) || defined(CONFIG_APM_MODULE)
    query_apm_bios();
#endif
```

마지막은 BIOS 로 부터 `Enhanced Disk Drive` 정보를 얻어오기 위한 [`query_edd`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/edd.c#L122) 함수 이다. `query_edd` 함수 구현을 살펴보자.

무엇보다더 커널 명령 라인으로 부터 [edd](https://github.com/torvalds/linux/blob/master/Documentation/kernel-parameters.txt#L1023) 옵션을 읽어서 만약 `off` 로 설정되어 있다면 `query_edd` 는 아무것도 하지 않고 종료할 것이다.

만약 EDD 가 활성화 되어 있다면, `query_edd` 는 BIOS 지원하드 디스크를 순회하며 EDD 정보를 아래와 같이 가져온다.:

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

`0x80` 은 처음 하드 드라이브이고 `EDD_MBR_SIG_MAX` 매크로의 값은 16이다. 그것은 데이터를 [edd_info](https://github.com/torvalds/linux/blob/master/include/uapi/linux/edd.h#L172) 구조체의 배열을 이용해서 모은다. `get_edd_info` 는 `ah` = `0x41` 과 함께 `0x13` 인터럽트 실행을 통해 EDD 가 있는지 확인하고, EDD 가 있다면, `get_edd_info` 는 `ah` = `0x48` 과 `si` 와 함께 `0x13` 인터럽트를 다시 호출하면, 설정된 버퍼 주소에 EDD 정보가 저장될 것이다.

결론
--------------------------------------------------------------------------------

이문서는 리눅스 커널 인사이드에 관련된 두 번째 파트의 끝이다. 다음 파트는 비디오 모드 설정과 보호 모드 전에 남은 준비를 무리리하면 보호 모드로 전환된다.

당신이 어떤 질문이나 제안이 있다면 [twitter](https://twitter.com/0xAX) - 원저자 에게 알려주길 바란다.

**나는 영어권의 사람이 아니고 이런 것에 대해 매우 미안해 하고 있다. 만약 어떤 실수를 발견한다면, 나에게 PR을 [linux-insides](https://github.com/0xAX/linux-internals)을 보내줘**

링크
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
