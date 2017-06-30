인터럽트와 인터럽트 핸들링 Part 1.
================================================================================

소개
--------------------------------------------------------------------------------

우리는 이전 챕터에서 리눅스 커널 초기화의 가장 먼저 일어나는 [단계](https://github.com/daeseokyoun/linux-insides/blob/korean-trans/Initialization/README.md)들과 첫 프로세스인 `init` 프로세스의 실행하는 과정까지 모두 살펴 보았다. 그렇다. 우리는 다양한 커널 서브 시스템과 관련된 몇몇의 초기화 단계를 보았다. 하지만, 이 서브 시스템들의 깊이 있는 세부 사항은 보질 못했다. 이 챕터에서는, 우리는 다양한 커널 서브 시스템이 어떻게 동작하고 구현되어 있는지 살펴 볼 것이다. 제목에서 벌써 알 수 있듯이, 우리가 살펴 볼 첫 서브시스템은 [인터럽트](http://en.wikipedia.org/wiki/Interrupt) 이다.

인터럽트?
--------------------------------------------------------------------------------

우리는 이미 이 책에서 `인터럽트(interrupt)` 라는 단어를 많이 봐왔다. 게다가 인터럽트 핸들러들의 몇가지 샘플도 보았다. 현재 챕터에서는 아래과 같은 질문에서 시작할 것이다.

* `인터럽트` 는 무엇인가?
* `인터럽트 핸들러는 무엇을 하는가`?

이 질문에 대한 내용을 먼저 보고 `인터럽트` 에 대한 상세 내용 및 리눅스 커널이 어떻게 그것들을 처리하는지 살펴 볼 것이다.

우리가 `인터럽트`라는 단어를 접할 때, 가장 먼저 떠오르는 질문은 `인터럽트가 무엇인가?` 일 것이다. 인터럽트는 소프트웨어나 하드웨어가 CPU 주목(attention)을 받을 필요가 있을 때, 알리는 `이벤트(event)` 같은 것이다. 예를 들어, 우리가 키보드의 버튼 하나를 눌렀을 때, 우리는 어떤 일이 일어나길 기대하는 것인가? 운영체제나 컴퓨터가 이 다음에는 무엇을 해야 하는 것인가? 문제를 단순화 하기 위해, 각 주변 장치들은 CPU 에 연결된 인터럽트 라인을 갖고 있다고 가정하자. 하나의 장치는 인터럽트 신호를 CPU 에게 보내기 위해 라인을 사용할 수 있다. 그렇지만, 인터럽트들은 CPU 에 직접적으로 신호를 전달하지 않는다. 예전 기기는 여러 장치로 부터 들어온 여러 개의 인터럽트 요청을 순서대로 처리하기 위한 [PIC](http://en.wikipedia.org/wiki/Programmable_Interrupt_Controller) 가 있었다. 요즈음 나오는 기기들은 `APIC` 라고 일반적으로 알려진 [Advanced Programmable Interrupt Controller](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller) 가 있다. `APIC` 는 두개의 분리된 장치들로 구성된다.:

* `지역 APIC`
* `I/O APIC`

`지역(local) APIC` 는 각 CPU 코어에 위치한다. 이 지역 APIC 는 CPU 특화된 인터럽트 구성을 처리하기 위한 책임이 있다. 지역 APIC 는 대게 APIC-timer, 열 감지 센서 그리고 각 지역적으로 연결된 I/O 장치들로 부터 인터럽트를 관리하기 위해 사용된다.

두 번째 `I/O APIC` 는 멀티 프로세서 인터럽트 관리를 제공한다. 여러 CPU 코어들 중에 외부 인터럽트를 처리할 수 있도록 한다. 조금 더 자세한 내용은 이 파트의 뒷 부분에 추가적으로 살펴 볼 것이다. 당신이 이해한대로, 인터럽트는 언제든 일어날 수 있다. 인터럽트가 발생하면, 운영체는 즉시 그 인터럽트를 처리해야 한다. 그렇지만, `인터럽트를 처리한다`라는 의미는 무엇인가? 인터럽트가 발생하면 운영체제는 아래의 단계 수행을 보장한다.:

* 커널은 반드시 현재 수행 중인 프로세스의 실행을 멈추어야 한다. (현재 태스크를 선점);
* 커널은 인터럽트 핸들러를 찾아야 하고 제어를 넘겨줘야 한다. (인터럽트 핸들러의 실행);
* 인터럽트 핸들러의 실행이 완료되면, 멈춰진 프로세스의 실행을 재개한다.

물론 인터럽트를 처리하는 과정에 많은 복잡한 것들이 있다. 하지만 위의 3 단계는 그 과정의 기본적인 틀이라고 할 수 있다.

인터럽트 핸들러의 각 주소들은 `Interrupt Descriptor Table` 또는 `IDT` 가 가리키는 특별한 위치에서 관리되고 있다. 프로세서는 인터럽트나 예외의 타잎을 인지하기 위해 고유의 번호를 사용한다. 이 고유한 번호를 `벡터 번호(vector number)` 라고 불린다. 이 벡터 번호는 `IDT` 내에 있는 인덱스이다. 벡터 번호는 `0` 에서 `255` 까지만 사용될 수 있는 제한이 있다. 당신이 리눅스 커널 내에서 아래와 같이 범위 확인을 하는 것을 볼 수 있다.:

```C
BUG_ON((unsigned)n > 0xFF);
```

이 확인 코드는 리눅스 커널에서 인터럽트 설정과 관련된 코드에서 쉽게 찾을 수 있다.(예를 들면, [arch/x86/include/asm/desc.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/desc.h)에 `set_intr_gate`, `void set_system_intr_gate`) 첫 32 개의 벡터 번호인 `0` 에서 `31`은 프로세서에 의해 예약되어 있고 아키텍처에서 정의된 예외와 인터럽트 처리를 위해 사용된다. 이 벡터 번호들을 기술한 테이블은 리눅스 초기화 과정 part 2 인 [초기 인터럽트와 예외 처리](https://github.com/daeseokyoun/linux-insides/blob/korean-trans/Initialization/linux-initialization-2.md) 에서 찾아볼 수 있다. `32` 에서 `255`까지의 벡터 번호는 사용자 정의된 인터럽트를 나타내고 프로세서에 의해 예약되어 있지 않다. 이 인터럽트들은 일반적으로 외부 I/O 장치들에게 할당되어 그 장치들을 활성화 시키고 프로세서에게 인터럽트를 보내기 위해 사용된다.

이제 인터럽트의 형태에 대해 알아보자. 일반적으로, 인터럽트들은 2 개의 주요 클래스로 나뉜다.:

* 외부 혹은 하드웨어에서 발생한 인터럽트
* 소프트웨어에서 발생한 인터럽트 (Software-generated interrupts)

외부 인터럽트는 `Local APIC` 로 부터 받던가 `Local APIC` 에 연결된 프로세서의 핀들로 부터 받는다. 소프트웨어에서 발생한 인터럽트는 프로세서 자체에 예외적인 상태에 의해 발생한다.(때때로 특별한 아키텍처 특화된 명령어들을 사용하여 발생시킨다.) 예외적인 상태에 대한 가장 공통의 예제는 `division by zero(0으로 나누기)` 이다. 다른 예제는 `syscall` 명령어로 프로그램을 종료하는 것이다.

이미 언급했지만, 인터럽트는 코드와 CPU 가 더이상 제어할 수 없는 상태를 위해 언제든지 발생할 수 있다. 다른 말로는, 예외는 프로그램 실행과 `동기적`이고 아래 3가지 분류로 나뉠 수 있다.:


* `Faults`
* `Traps`
* `Aborts`

`fault` 는 "faulty" 명령어(고쳐질 수 있다)의 실행 전에 보고된 예외이다. 수정되었다면, 인터럽트된 프로그램은 계속 재개할 수 있다.

`trap` 은 `trap` 명령어어 실행 바로 다음에 보고되는 예외이다. 트랩은 `fault` 가 했던대로 인터럽트되었던 프로그램이 재개할 수도 있다.

`abort` 는 예외가 발생한 정확한 명령어를 항상 보고 받을 수 없고 인터럽트된 프로그램이 재개 될 수 없는 예외이다.

또한 인터럽트는 `maskable` 과 `non-maskable` 로 분류될 수 있다고 이전 [파트](https://github.com/daeseokyoun/linux-insides/blob/korean-trans/Booting/linux-bootstrap-2.md) 에서 살펴 보았다. Maskable(마스킹 가능한) 인터럽트는 `x86_64`에서 `sti` 와 `cli` 명령어로 블락킹할 수 있는 인터럽트이다. 리눅스 커널 소스 코드에서 마스킹하는 것을 볼 수 있다.:

```C
static inline void native_irq_disable(void)
{
        asm volatile("cli": : :"memory");
}
```

와

```C
static inline void native_irq_enable(void)
{
        asm volatile("sti": : :"memory");
}
```

이 두 명령어들은 인터럽트 레지스터 내의 `IF` 플래그를 수정한다. `sti` 명령어는 `IF` 플래그를 설정하고 `cli` 명령어는 그 플래그를 클리어 한다. 넌마스커블 인터럽트들은 이 플래그로 블락킹되지 않는다. 대게 하드웨어의 어떤 실패는 넌마스커블 인터럽트 같은 것으로 맵핑되어 있다.

만약 여러 예외와 인터럽트가 동시에 발생한다면, 프로세서는 미리 정의된 우선순위에 따라 그것들을 처리한다. 우리는 우선순위를 아래 테이블내에서 높은 것에서 부터 가장 낮은 순대로 정해진 것을 볼 수 있다.:

```
+----------------------------------------------------------------+
|              |                                                 |
|   Priority   | Description                                     |
|              |                                                 |
+--------------+-------------------------------------------------+
|              | Hardware Reset and Machine Checks               |
|     1        | - RESET                                         |
|              | - Machine Check                                 |
+--------------+-------------------------------------------------+
|              | Trap on Task Switch                             |
|     2        | - T flag in TSS is set                          |
|              |                                                 |
+--------------+-------------------------------------------------+
|              | External Hardware Interventions                 |
|              | - FLUSH                                         |
|     3        | - STOPCLK                                       |
|              | - SMI                                           |
|              | - INIT                                          |
+--------------+-------------------------------------------------+
|              | Traps on the Previous Instruction               |
|     4        | - Breakpoints                                   |
|              | - Debug Trap Exceptions                         |
+--------------+-------------------------------------------------+
|     5        | Nonmaskable Interrupts                          |
+--------------+-------------------------------------------------+
|     6        | Maskable Hardware Interrupts                    |
+--------------+-------------------------------------------------+
|     7        | Code Breakpoint Fault                           |
+--------------+-------------------------------------------------+
|     8        | Faults from Fetching Next Instruction           |
|              | Code-Segment Limit Violation                    |
|              | Code Page Fault                                 |
+--------------+-------------------------------------------------+
|              | Faults from Decoding the Next Instruction       |
|              | Instruction length > 15 bytes                   |
|     9        | Invalid Opcode                                  |
|              | Coprocessor Not Available                       |
|              |                                                 |
+--------------+-------------------------------------------------+
|     10       | Faults on Executing an Instruction              |
|              | Overflow                                        |
|              | Bound error                                     |
|              | Invalid TSS                                     |
|              | Segment Not Present                             |
|              | Stack fault                                     |
|              | General Protection                              |
|              | Data Page Fault                                 |
|              | Alignment Check                                 |
|              | x87 FPU Floating-point exception                |
|              | SIMD floating-point exception                   |
|              | Virtualization exception                        |
+--------------+-------------------------------------------------+
```

이제 우리는 인터럽트와 예외의 다양한 타입에 관련해서 알아보기 위해 좀 더 실용적인 부분으로 넘어가보도록 하자. 우리는 `Interrupt Descriptor Table` 의 설명에서 부터 시작하려고 한다. 앞절에서 설명했듯이, `IDT`는 인터럽트와 예외 처리 핸들러의 엔트리 포인트를 저장한다. `IDT`는 이미 살펴본 [파트](https://github.com/daeseokyoun/linux-insides/blob/korean-trans/Booting/linux-bootstrap-2.md) 에서 `Global Descriptor Table` 와 비슷한 구조로 되어 있다. 물론 다른 점이 있다. `descriptors` 대신에, `IDT` 엔트리들은 `gates(게이트)` 라고 불린다. 그것은 `x86`에서 아래의 게이트들 중 하나로 구성된다.:

* Interrupt gates (인터럽트 게이트)
* Task gates (태스크 게이트)
* Trap gates. (트랩 게이트)

단지 [long mode](http://en.wikipedia.org/wiki/Long_mode) 인터럽트 게이트와 트랩 게이트는 `x86_64` 에서 찾아볼 수 있다. `Global Descriptor Table` 와 같이 `Interrupt Descriptor table`는 `x86`에서는 8 바이트 게이트의 배열이고 `x86_64` 에서는 16 바이트 게이트의 배열이다. [커널 부팅 과정 part 2](https://github.com/daeseokyoun/linux-insides/blob/korean-trans/Booting/linux-bootstrap-2.md) 의 내용 중에 `Global Descriptor Table` 의 첫번째 요소는 `NULL` 디스크립터를 포함해야 한다는 것이다. `Global Descriptor Table` 다르게 `Interrupt Descriptor Table`는 하나의 게이트를 갖고 있다.(꼭 그래야 하는 것은 아니다.) 예를 들어, [protected mode](http://en.wikipedia.org/wiki/Protected_mode) 전환과정에서 인터럽트 디스크립터 테이블을 `NULL` 게이트로 로드했다는 것을 [커널 부팅 과정 part 3](https://github.com/daeseokyoun/linux-insides/blob/korean-trans/Booting/linux-bootstrap-3.md) 에서 확인 할 수 있을 것이다.:

```C
/*
 * Set up the IDT
 */
static void setup_idt(void)
{
	static const struct gdt_ptr null_idt = {0, 0};
	asm volatile("lidtl %0" : : "m" (null_idt));
}
```

이 함수는 [arch/x86/boot/pm.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/pm.c)에 구현되어 있다. `Interrupt Descriptor table` 는 선형 주소 공간 어디든지 로드될 수 있고, 베이스 주소는 `x86` 의 경우에 8 바이트로 정렬되어 있어야 하고 `x86_64` 는 16 바이트로 정렬(aligned) 되어 있어야 한다. `IDT` 의 베이스 주소는 특별한 레지스터인 `IDTR`에 저장되어 있다. 여기서 `x86` 호환 프로세서에서 `IDTR` 레지스터를 수정하기 위한 두 개의 명령어들이 있다.:

* `LIDT`
* `SIDT`

첫 번째 명령어인 `LIDT` 는 특정의 `IDTR` 레지스터에 `IDT`의 베이스 주소를 로드하기 위해 사용된다. 두 번째 명령어인 `SIDT`는 `IDTR`의 내용을 읽고 특정 피연산자에 저장하기 위해 사용된다. `IDTR`은 `x86`에서 48 비트이고 아래의 정보를 포함한다.:

```
+-----------------------------------+----------------------+
|                                   |                      |
|     Base address of the IDT       |   Limit of the IDT   |
|                                   |                      |
+-----------------------------------+----------------------+
47                                16 15                    0
```

`setup_idt` 의 구현을 보면, `null_idt` 를 준비했고 그것을 `lidt` 명령어로 `IDTR` 에 로드했다. `null_idt`는 아래와 같이 정의된 `gdt_ptr` 을 갖는다는 것을 기억하자.:

```C
struct gdt_ptr {
        u16 len;
        u32 ptr;
} __attribute__((packed));
```

여기서 2 바이트와 4 바이트로 된 (총 48 비트) 두개의 필드를 갖고 있는 구조체의 정의를 볼 수 있다. 이제 `IDT` 엔트리 구조체를 살펴보자. `IDT` 엔트리 구조체는 `x86_64` 에서 게이트라고 불리는 16 바이트 엔트리들의 배열이다. 그것들은 아래와 같은 구조로 되어 있다.:

```
127                                                                             96
+-------------------------------------------------------------------------------+
|                                                                               |
|                                Reserved                                       |
|                                                                               |
+--------------------------------------------------------------------------------
95                                                                              64
+-------------------------------------------------------------------------------+
|                                                                               |
|                               Offset 63..32                                   |
|                                                                               |
+-------------------------------------------------------------------------------+
63                               48 47      46  44   42    39             34    32
+-------------------------------------------------------------------------------+
|                                  |       |  D  |   |     |      |   |   |     |
|       Offset 31..16              |   P   |  P  | 0 |Type |0 0 0 | 0 | 0 | IST |
|                                  |       |  L  |   |     |      |   |   |     |
 -------------------------------------------------------------------------------+
31                                   16 15                                      0
+-------------------------------------------------------------------------------+
|                                      |                                        |
|          Segment Selector            |                 Offset 15..0           |
|                                      |                                        |
+-------------------------------------------------------------------------------+
```

IDT에 인덱스를 형성하기 위해 프로세서는 예외 또는 인터럽트 벡터를 16 으로 확장한다. 프로세서는 `call` 명령어가 처리하는 과정처럼 예외들과 인터럽트들의 발생을 처리한다. 하나의 프로세서는 인터럽트나 예외의 고유 번호(`벡터 번호`)를 갖고 그 번호를 필요한 `Interrupt Descriptor Table` 엔트리를 찾기 위해 인덱스로 사용한다. 이제 `IDT` 엔트리 항목들을 살펴보자.

보시다시피, 위의 다이어그램에서 `IDT` 엔트리는 아래와 같은 항목들로 구성되어 있다.:

* `0-15` 비트  - 인터럽트 핸들러 엔트리 포인트의 베이스 주소로 프로세서에 의해 사용되어지는 세그먼트 셀렉터로 부터의 오프셋
* `16-31` 비트 - 인터럽트 핸들러의 엔트리 포인트를 갖고 있는 세그먼트 셀렉터의 베이스 주소
* `IST` - `x86_64` 에서 새로운 특별한 메커니즘, 나중에 설명
* `DPL` - 디스크립터 특권 레벨 (Descriptor Privilege Level)
* `P` - 세그먼트 존재 유무 플래그 (Segment Present flag)
* `48-63` 비트 - 핸들러 베이스 주소의 두 번째 부분
* `64-95` 비트 - 핸들러 베이스 주소의 세 번째 부분
* `96-127` 비트 - CPU 에 의해 예약된 마지막 비트들

그리고 마지막 `Type` 항목은 `IDT` 엔트리의 타입을 기술한다. 인터럽트를 위한 3 가지의 서로 다른 종류의 핸들러가 있다.:

* 인터럽트 게이트 Interrupt gate
* 트랩 게이트 Trap gate
* 태스크 게이트 Task gate

`IST` 또는 `Interrupt Stack Table(인터럽트 스택 테이블)` 은 `x86_64` 에서 새로운 매커니즘이다. 그것은 기존 스택-스위치 매커니즘을 대체하여 사용된다. 이전에 `x86` 아키텍처는 인터럽트에 대응하여 스택 프레임을 자동으로 전환하는 메커니즘을 제공했다. `IST`는 `x86` 스택 스위칭 모드의 수정된 버전이다. 이 매커니즘은 활성화 될 때 스택을 무조건적으로 전환하고 특정 인터럽트와 연관된 `IDT` 엔트리에 있는 어떤 인터럽트를 위해 활성화 될 수 있다.(차차 알아보자.) 이것을 봤을 때, 모든 인터럽트를 위해 `IST` 가 필요하지는 않다는 것이다. 어떤 인터럽트들은 기존 스택 스위치 모드를 사용해서 진행할 수 있다는 것이다. `IST` 메커니즘은 프로세스 관련된 정보를 담고 있는 특별한 구조체인 [Task State Segment](http://en.wikipedia.org/wiki/Task_state_segment)/`TSS` 내에 7 개까지 `IST` 포인터를 제공한다. `TSS` 는 리눅스 커널에서 예외/인터럽트 처리 되는 동안 스택 전환을 위해 사용된다.

`Interrupt Descriptor Table` 는 `gate_desc` 구조체의 배열에 의해 표현된다.:

```C
extern gate_desc idt_table[];
```

`gate_desc` 은:

```C
#ifdef CONFIG_X86_64
...
...
...
typedef struct gate_struct64 gate_desc;
...
...
...
#endif
```

그리고 `gate_struct64` 는 아래와 같이 구성되어 있다.:

```C
struct gate_struct64 {
        u16 offset_low;
        u16 segment;
        unsigned ist : 3, zero0 : 5, type : 5, dpl : 2, p : 1;
        u16 offset_middle;
        u32 offset_high;
        u32 zero1;
} __attribute__((packed));
```

각 활동중인 쓰레드는 `x86_64` 아키텍처를 위해 리눅스 커널에서 큰 스택을 갖고 있다. 스택의 크기는 `THREAD_SIZE` 정의되어 있고 아래의 값을 가진다.:

```C
#define PAGE_SHIFT      12
#define PAGE_SIZE       (_AC(1,UL) << PAGE_SHIFT)
...
...
...
#define THREAD_SIZE_ORDER       (2 + KASAN_STACK_ORDER)
#define THREAD_SIZE  (PAGE_SIZE << THREAD_SIZE_ORDER)
```

`PAGE_SIZE` 는 `4096` 바이트이고 `THREAD_SIZE_ORDER` 는 `KASAN_STACK_ORDER` 에 의존적이다. 이미 봤듯이, `KASAN_STACK`는 `CONFIG_KASAN` 커널 구성 파라미터에 의존적이고 아래와 같이 정의되어 있다.:

```C
#ifdef CONFIG_KASAN
    #define KASAN_STACK_ORDER 1
#else
    #define KASAN_STACK_ORDER 0
#endif
```

`KASan` 은 런타임 메모리 [디버커](http://lwn.net/Articles/618180/)이다. `CONFIG_KASAN` 구성 옵션이 비활성화 되어 있다면, `THREAD_SIZE` 는 `16384` 가 될 것이고, 활성화 되어 있다면, `32768` 이 될 것이다. 이 스택들은 쓰레드가 살아 있거나 좀비 상태인 것을 위한 유용한 정보를 포함한다. 반면에 사용자 영역에 있는 쓰레드는 커널 스택은 스택 바닥에 있는 `thread_info` 구조체를 위한 것을 제외하고는 비어 있다.(이 구조체에 자세한 정보는 [리눅스 커널 초기화 과정 part 4](https://github.com/daeseokyoun/linux-insides/blob/korean-trans/Initialization/linux-initialization-4.md) 에서 확인 가능하다.) 활성화되어 있거나 좀비 쓰레드들은 단지 자신의 스택을 갖고 있는 쓰레드만은 아니다. 거기에는 각 가용한 CPU 와 연관된 특수한 스택들도 존재한다. 이 스택들은 커널이 CPU 에서 실행되면 활성화 된다. 사용자 영역이 CPU 에서 실행될 때, 이런 경우 스택들은 유용한 정보를 담고 있지 않다. 각 CPU 는 몇몇의 특별한 per-cpu 스택을 갖고 있다. 그 중 첫번째는 외부 하드웨어 인터럽트를 위해 사용되는 `인터럽트 스택`이 있다. 그것의 크기는 아래에 의해 결정된다.:

```C
#define IRQ_STACK_ORDER (2 + KASAN_STACK_ORDER)
#define IRQ_STACK_SIZE (PAGE_SIZE << IRQ_STACK_ORDER)
```

그것은 `16384` 바이트이다. per-cpu 인터럽트 스택은 `x86_64` 를 위한 리눅스 커널에서 `irq_stack_union` union 구조체에 의해 표현된다.:

```C
union irq_stack_union {
	char irq_stack[IRQ_STACK_SIZE];

    struct {
		char gs_base[40];
		unsigned long stack_canary;
	};
};
```

첫 `irq_stack` 이라는 항목은 16 킬로바이트인 배열이다. 또한 당신은 `irq_stack_union` 구조체를 포함한다는 것을 볼 수 있는데, 이는 두 개의 항목들이 있다.:

* `gs_base` - `gs` 레지스터는 항상 `irqstack` union 의 바닥을 가리키고 있다. `x86_64` 에서는, `gs` 레지스터는 per-cpu 영역과 스택 카나리(canary)를 공유한다. (`per-cpu` 변수에 관해 더 자세히 알고 싶다면, 이를 위한 특별한 [파트](https://github.com/daeseokyoun/linux-insides/blob/korean-trans/Concepts/cpumask.md)를 확인하자.) 모든 per-cpu 심볼들은 기본적으로 0으로 채워져 있고, `gs`는 per-cpu 영역의 베이스를 가리키고 있다. 당신은 이미 롱 모드에서 사용되지 않는 [segmented memory model](http://en.wikipedia.org/wiki/Memory_segmentation) 에 대해 알고 있을 것이다. 하지만 우리는 두 개의 세그먼트 레지스터를 위한 베이스 주소를 설정해야 한다. - `fs` 와 [Model specific registers](http://en.wikipedia.org/wiki/Model-specific_register) 와 함께 `gs`이다. 그리고 이런 레지스터들은 주소 포함 레지스터로 사용될 수 있다. 만얀 당신이 리눅스 커널 초기화 과정 [part 1](https://github.com/daeseokyoun/linux-insides/blob/korean-trans/Initialization/linux-initialization-1.md) 을 봤다면, `gs` 레지스터는 아래와 같이 설정된다는 것을 봤을 것이다.:

```assembly
	movl	$MSR_GS_BASE,%ecx
	movl	initial_gs(%rip),%eax
	movl	initial_gs+4(%rip),%edx
	wrmsr
```

`initial_gs` 는 `irq_stack_union` 를 가리킨다.:

```assembly
GLOBAL(initial_gs)
.quad	INIT_PER_CPU_VAR(irq_stack_union)
```

* `stack_canary` - 인터럽트 스택을 위한 [Stack canary](http://en.wikipedia.org/wiki/Stack_buffer_overflow#Stack_canaries) 는 스택이 덮어쓰여지는지 확인하기 위한 `스택 보호자(stack protector)` 이다. `gs_base` 는 40 바이트 배열인것을 기억하자. `GCC`는 `gs` 의 베이스로 부터 고정된 오프셋에 스택 카나리(canary) 를 위치 시킬 것이다. 그 오프셋의 값은 `x86_64`에서 `40` 이고, `x86`에서는 `20` 이 될 것이다.

`irq_stack_union` 는 `percpu` 영역에 있는 첫번째 자료이고, `System.map` 에서 그것을 볼 수 있다.:

```
0000000000000000 D __per_cpu_start
0000000000000000 D irq_stack_union
0000000000004000 d exception_stacks
0000000000009000 D gdt_page
...
...
...
```

코드에서 그것의 선언을 볼 수 있다.:

```C
DECLARE_PER_CPU_FIRST(union irq_stack_union, irq_stack_union) __visible;
```

이제, `irq_stack_union` 의 초기화를 살펴 볼 시간이다. `irq_stack_union` 정의 외에, 우리는 [arch/x86/include/asm/processor.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/processor.h) 에서 아래와 같은 per-cpu 변수들의 선언들을 볼 수 있다.:

```C
DECLARE_PER_CPU(char *, irq_stack_ptr);
DECLARE_PER_CPU(unsigned int, irq_count);
```

첫번째인 `irq_stack_ptr`는 이름에서 알 수 있듯이, 스택의 꼭대기를 가키는 포인터 변수이다. 두 번째 `irq_count`는 만약 CPU 가 이미 인터럽트 스택에 있는지 혹은 아닌지를 확인하기 위해 사용된다. `irq_stack_ptr` 의 초기화는 [arch/x86/kernel/setup_percpu.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/setup_percpu.c) 에 `setup_per_cpu_areas` 함수에 있다.:

```C
void __init setup_per_cpu_areas(void)
{
...
...
#ifdef CONFIG_X86_64
for_each_possible_cpu(cpu) {
    ...
    ...
    ...
    per_cpu(irq_stack_ptr, cpu) =
            per_cpu(irq_stack_union.irq_stack, cpu) +
            IRQ_STACK_SIZE - 64;
    ...
    ...
    ...
#endif
...
...
}
```

여기서 모든 CPU 들을 하나씩 살펴 보면서 `irq_stack_ptr` 를 설정한다. 이것은 인터럽트 스택의 꼭대기에서 `64`를 뺀것과 같다. 왜 `64` 냐고(나중에 알아보자)? [arch/x86/kernel/cpu/common.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/cpu/common.c) 소스 코드 파일에 다음과 같은 것을 볼 수 있다.:

```C
void load_percpu_segment(int cpu)
{
        ...
        ...
        ...
        loadsegment(gs, 0);
        wrmsrl(MSR_GS_BASE, (unsigned long)per_cpu(irq_stack_union.gs_base, cpu));
}
```

그리고 우리가 이미 알고 있는 내용인 `gs` 레지스터는 인터럽트 스택의 바닥을 가리키도록 한다.

```assembly
	movl	$MSR_GS_BASE,%ecx
	movl	initial_gs(%rip),%eax
	movl	initial_gs+4(%rip),%edx
	wrmsr

	GLOBAL(initial_gs)
	.quad	INIT_PER_CPU_VAR(irq_stack_union)
```

여기서 우리는 `ecx` 레지스터에 의해 가리켜지는 [Model specific register](http://en.wikipedia.org/wiki/Model-specific_register) 에 `edx:eax` 로 부터 데이터를 로드하는 `wrmsr` 명령어를 볼 수 있다. 우리의 경우 모델 특화된 레지스터(Model specific register)는 `gs` 레지스터가 가리키는 메모리 세그먼트의 베이스 주소를 포함하는 `MSR_GS_BASE` 이다. `edx:eax` 는 여기서의 `irq_stack_union` 베이스 주소인 `initial_gs`의 주소를 가리킨다.

우리는 이미 `x86_64` 이 `Interrupt Stack Table` 혹은 `IST` 라 불리는 특징을 갖고 있고 이 특징은 넌-마스커블 인터럽트, [double fault](http://blog.daum.net/tlos6733/169) 이벤트를 위한 새로운 스택으로 전환할 수 있는 기능을 제공한다. 여기에 7 개의 per-cpu `IST` 엔트리가 있다. 그 중 몇개를 살펴 보면,:

* `DOUBLEFAULT_STACK`
* `NMI_STACK`
* `DEBUG_STACK`
* `MCE_STACK`

혹은

```C
#define DOUBLEFAULT_STACK 1
#define NMI_STACK 2
#define DEBUG_STACK 3
#define MCE_STACK 4
```

모든 `IST`로 새로운 스택으로 전환되는 인터럽트-게이트 기술자(descriptors)들은 `set_intr_gate_ist` 함수에서 초기화 된다. 예를 들어,:

```C
set_intr_gate_ist(X86_TRAP_NMI, &nmi, NMI_STACK);
...
...
...
set_intr_gate_ist(X86_TRAP_DF, &double_fault, DOUBLEFAULT_STACK);
```

여기서 `&nmi` 와 `&double_fault` 는 주어진 인터럽트 핸들러들의 엔트리 주소들이다.:

```C
asmlinkage void nmi(void);
asmlinkage void double_fault(void);
```

[arch/x86/kernel/entry_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/entry_64.S)에 정의되어 있다.

```assembly
idtentry double_fault do_double_fault has_error_code=1 paranoid=2
...
...
...
ENTRY(nmi)
...
...
...
END(nmi)
```

인터럽트나 예외가 발생하면, 새로운 `ss` 선택자(selector)는 `NULL`로 강제되고 `ss` 선택자(selector)의 `rpl` 항목은 새로운 `cpl`로 설정된다. 이전 `ss`, `rsp`, 레지스터 플래그, `cs`, `rip`들은 새로운 스택에 넣어진다. 64 비트 모드에서는 인터럽트 스택 프레임 크기는 8 바이트로 고정되어 있기에 다음과 같은 스택을 가질 수 있다.:

```
+---------------+
|               |
|      SS       | 40
|      RSP      | 32
|     RFLAGS    | 24
|      CS       | 16
|      RIP      | 8
|   Error code  | 0
|               |
+---------------+
```

만약 인터럽트 게이트내에 `IST` 항목이 `0`이 아니라면, 우리는 `rsp`를 통해 `IST` 포인터를 읽을 수 있다. 만약 인터럽트 벡터 번호가 그것과 연관된 에러 코드를 갖고 있다면, 우리는 스택에 그 에러 코드를 넣을 것이다. 만약 인터럽트 벡터 번호에 에러 코드가 없다면, 우리는 더미 에러 코드를 스택에 넣을 것이다. 우리는 스택 일관성을 유지하기 위해 필요한 것이다. 다음은, 세그먼트-선택자 항목을 게이트 기술자로 부터 CS 레지스터로 로드하고 타덱 코드 세그먼트가 `21` 비트를 확인하여 64 비트 모드 코드 세그먼트인지 반드시 확인해야 한다. 예를 들면, `Global Descriptor Table` 에 있는 `L` 비트다. 마지막으로 우리는 게이트 기술자로 부터 오프셋 항목을 인터럽트 핸들러의 엔트리 포인터가 될 것인 `rip`로 로드한다. 이후에는 인터럽트 핸들러가 실행을 시작할 수 있고 인터럽트 핸들러가 그것의 실행을 완료 했을 때, 그것은 `iret` 명령어로 인터럽트된 프로세스로 제어권을 반드시 넘겨줘야 한다. `iret` 명령어는 무조건적으로 스택포인터(`ss:rsp`)를 인터럽트 당한 프로세스의 스택으로 복구해주고 `cpl` 변경에 의존적이지 않다.

끝~

결론
--------------------------------------------------------------------------------

리눅스 커널에서 `Interrupts and Interrupt Handling` 의 첫번째 파트가 끝이 났다. 우리는 특정 이론을 다루었고 인터럽트와 예외 처리와 연관된 것들의 초기화 첫 단계를 살펴 보았다. 다음 파트에서는 인터럽트들과 인터럽트 처리의 관련해서 더 많이 다루어 보도록 하겠다.

어떤 질문이나 제안이 있다면, twitter [0xAX](https://twitter.com/0xAX), [email](anotherworldofworld@gmail.com) 또는 [issue](https://github.com/0xAX/linux-insides/issues/new) 를 만들어 주길 바란다.

**나는 영어권의 사람이 아니고 이런 것에 대해 매우 미안해 하고 있다. 만약 어떤 실수를 발견한다면, 나에게 PR을 [linux-insides](https://github.com/0xAX/linux-internals)을 보내줘**

Links
--------------------------------------------------------------------------------

* [PIC](http://en.wikipedia.org/wiki/Programmable_Interrupt_Controller)
* [Advanced Programmable Interrupt Controller](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller)
* [protected mode](http://en.wikipedia.org/wiki/Protected_mode)
* [long mode](http://en.wikipedia.org/wiki/Long_mode)
* [kernel stacks](https://www.kernel.org/doc/Documentation/x86/kernel-stacks)
* [Task State Segment](http://en.wikipedia.org/wiki/Task_state_segment)
* [segmented memory model](http://en.wikipedia.org/wiki/Memory_segmentation)
* [Model specific registers](http://en.wikipedia.org/wiki/Model-specific_register)
* [Stack canary](http://en.wikipedia.org/wiki/Stack_buffer_overflow#Stack_canaries)
* [Previous chapter](http://0xax.gitbooks.io/linux-insides/content/Initialization/index.html)
