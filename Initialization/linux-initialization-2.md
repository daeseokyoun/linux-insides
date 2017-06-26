커널 초기화. Part 2.
================================================================================

초기 인터럽트와 예외 처리
--------------------------------------------------------------------------------

이전 [파트](https://github.com/daeseokyoun/linux-insides/blob/master/Initialization/linux-initialization-1.md) 에서 우리는 초기 인터럽트 핸들러의 설정 전까지 알아 보았다. 이제 압축 해제된 리눅스 커널에서 수행중에 있으며, 우리는 초기 부팅을 위한 기본 [페이징](https://en.wikipedia.org/wiki/Page_table) 구조체를 가지고, 우리의 현재 목표는 메일 커널 코드가 수행되기 전에 초기 준비 작업에 대한 내용을 마무리 하는 것이다.

이에 대한 내용은 이전 [파트](https://github.com/daeseokyoun/linux-insides/blob/master/Initialization/linux-initialization-1.md)에서 부터 시작되었다. 현재 파트에서 계속 이어 나가 인터럽트와 예외 처리에 대해 알아보자.

우리가 지난 파트에서 마지막으로 봤던 코드는 아래와 같다.:

```C
for (i = 0; i < NUM_EXCEPTION_VECTORS; i++)
	set_intr_gate(i, early_idt_handler_array[i]);
```

[arch/x86/kernel/head64.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head64.c) 소스 코드에 있는 내용이다. 하지만 이 코드를 정리하기 전에 우리는 인터럽트와 핸들러에 관련해서 먼저 알아 봐야 한다.

이론들.
--------------------------------------------------------------------------------

인터럽트는 소프트웨어나 하드웨어가 CPU 에게 event 를 보내는 것이다. 예를 들어, 사용자가 키보드의 키를 누르는 것과 같다. 인터럽트가 발생하면, CPU 가 현재 태스크 수행을 멈추고 [인터럽트 핸들러](https://en.wikipedia.org/wiki/Interrupt_handler)로 불리는 특별한 루틴으로 제어권을 넘기게 된다. 인터럽트 핸들러의 수행이 완료되면 이전 멈추었던 태스크 수행 상태로 다시 돌아와야 한다. 우리는 인터럽트를 세가지 형태로 구분 할 수 있다:

* 소프트웨어 인터럽트 - 커널에서 시스템 콜과 같은 이벤트가 발생하는 경우를 말한다.
* 하드웨어 인터럽트 - 키보드에서 키가 눌렸을 때와 같은 하드웨어 이벤트를 말한다.
* Exceptions - 잘못된 메모리 접근이나, 0으로 나누는 것과 같이 CPU 가 에러를 발견하여 CPU 에 생성되는 인터럽트 이다.

모든 인터럽트와 예외들은 `vector number` 라 불리는 수만큼 할당이 된다. `Vector number` 는 `0` 에서 `255` 사이에 어떤 수도 될 수 있다. 여기는 일반적으로 예외 처리를 위해 첫 벡터 넘버로 `32`를 사용하고, 벡터 번호 `32` 부터 `255` 까지 사용자가 정의하여 사용할 수 있게 된다. 우리는 그것을 아래 `NUM_EXCEPTION_VECTORS` 정의로 부터 알 수 있다:

```C
#define NUM_EXCEPTION_VECTORS 32
```

CPU 는 `Interrupt Descriptor Table(인터럽트 디스크립션 테이블)`에서 벡터 넘버를 인덱스로 사용한다. CPU 는 [APIC](http://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller) 또는 그 핀을 통해서 인터럽트를 받는다. 아래 테이블 에서 `0-31` 예외들을 볼 수 있다.:

```
----------------------------------------------------------------------------------------------
|Vector|Mnemonic|Description         |Type |Error Code|Source                   |
----------------------------------------------------------------------------------------------
|0     | #DE    |Divide Error        |Fault|NO        |DIV and IDIV                          |
|---------------------------------------------------------------------------------------------
|1     | #DB    |Reserved            |F/T  |NO        |                                      |
|---------------------------------------------------------------------------------------------
|2     | ---    |NMI                 |INT  |NO        |external NMI                          |
|---------------------------------------------------------------------------------------------
|3     | #BP    |Breakpoint          |Trap |NO        |INT 3                                 |
|---------------------------------------------------------------------------------------------
|4     | #OF    |Overflow            |Trap |NO        |INTO  instruction                     |
|---------------------------------------------------------------------------------------------
|5     | #BR    |Bound Range Exceeded|Fault|NO        |BOUND instruction                     |
|---------------------------------------------------------------------------------------------
|6     | #UD    |Invalid Opcode      |Fault|NO        |UD2 instruction                       |
|---------------------------------------------------------------------------------------------
|7     | #NM    |Device Not Available|Fault|NO        |Floating point or [F]WAIT             |
|---------------------------------------------------------------------------------------------
|8     | #DF    |Double Fault        |Abort|YES       |An instruction which can generate NMI |
|---------------------------------------------------------------------------------------------
|9     | ---    |Reserved            |Fault|NO        |                                      |
|---------------------------------------------------------------------------------------------
|10    | #TS    |Invalid TSS         |Fault|YES       |Task switch or TSS access             |
|---------------------------------------------------------------------------------------------
|11    | #NP    |Segment Not Present |Fault|NO        |Accessing segment register            |
|---------------------------------------------------------------------------------------------
|12    | #SS    |Stack-Segment Fault |Fault|YES       |Stack operations                      |
|---------------------------------------------------------------------------------------------
|13    | #GP    |General Protection  |Fault|YES       |Memory reference                      |
|---------------------------------------------------------------------------------------------
|14    | #PF    |Page fault          |Fault|YES       |Memory reference                      |
|---------------------------------------------------------------------------------------------
|15    | ---    |Reserved            |     |NO        |                                      |
|---------------------------------------------------------------------------------------------
|16    | #MF    |x87 FPU fp error    |Fault|NO        |Floating point or [F]Wait             |
|---------------------------------------------------------------------------------------------
|17    | #AC    |Alignment Check     |Fault|YES       |Data reference                        |
|---------------------------------------------------------------------------------------------
|18    | #MC    |Machine Check       |Abort|NO        |                                      |
|---------------------------------------------------------------------------------------------
|19    | #XM    |SIMD fp exception   |Fault|NO        |SSE[2,3] instructions                 |
|---------------------------------------------------------------------------------------------
|20    | #VE    |Virtualization exc. |Fault|NO        |EPT violations                        |
|---------------------------------------------------------------------------------------------
|21-31 | ---    |Reserved            |INT  |NO        |External interrupts                   |
----------------------------------------------------------------------------------------------
```

CPU 는 인터럽트에 대한 반응을 하기 위해 특별한 구조체인 인터럽트 디스크립션 테이블(IDT) 를 사용한다. IDT 는 글로벌 디스크립션 테이블 처럼 8 바이트 디스크립션을 갖는 배열이다. (IDT 의 엔트리들은 `gates` 라 불린다.) CPU 는 IDT 엔트리의 인덱스를 찾기 위해 벡터 번호에 8을 곱한다. 하지만, 64 비트 모드에서는 IDT 는 16 바이트 디스크립션 테이블 배열이므로, CPU 는 벡터 번호에 16 을 곱해야 한다. 우리는 이전 파트로 부터 CPU 는 글로벌 디스크립션 테이블에 있는 특별한 `GDTR` 레지스터를 사용한다는 것을 기억한다면, CPU 는 또한 인터럽트 디스크립션 테이블을 위해 `IDTR` 이라는 특별한 레지스터를 사용하고 `lidt` 명령어를 사용하여 인터럽트 디스크립션 테이블의 베이스 주소를 이 레지스터로 로딩한다.

64 비트 모드 IDT 엔트리는 아래 처럼 생겼다:

```
127                                                                             96
 --------------------------------------------------------------------------------
|                                                                               |
|                                Reserved                                       |
|                                                                               |
 --------------------------------------------------------------------------------
95                                                                              64
 --------------------------------------------------------------------------------
|                                                                               |
|                               Offset 63..32                                   |
|                                                                               |
 --------------------------------------------------------------------------------
63                               48 47      46  44   42    39             34    32
 --------------------------------------------------------------------------------
|                                  |       |  D  |   |     |      |   |   |     |
|       Offset 31..16              |   P   |  P  | 0 |Type |0 0 0 | 0 | 0 | IST |
|                                  |       |  L  |   |     |      |   |   |     |
 --------------------------------------------------------------------------------
31                                   15 16                                      0
 --------------------------------------------------------------------------------
|                                      |                                        |
|          Segment Selector            |                 Offset 15..0           |
|                                      |                                        |
 --------------------------------------------------------------------------------
```

Where:

* `Offset` - 인터럽트 핸들러의 엔트리 포인트에서의 오프셋
* `DPL` -    디스크립터 특권 레벨
* `P` -      세그먼트가 "Present" 상태인지 확인 하는 플래그
* `Segment selector` - GDT 나 LDT 에서 코드 세그먼트 셀렉터
* `IST` -    인터럽트 핸들링을 위해 새로운 스택으로 전환할 수 있도록 하는 기능을 제공

그리고 마지막 `Type` 필드에 대해 살펴보자. 인터럽트를 위한 핸들러의 3가지 타입을 기술한다.:

* 태스크 티스크립터
* 인터럽트 디스크립터
* 트랩 디스크립터

인터럽트와 트랩 디스크립션들은 인터럽트 핸들러의 엔트리 포인터를 가지고 있다. 단지 이 두 타입의 차이점은 어떻게 `IF` 플래그를 처리하는냐에 있다. 만약 인터럽트 핸들러가 인터럽트 게이트를 통해 접근했다면, CPU 는 현재 인터럽트 핸들러가 수행중 동안 다른 인터럽트가 발생하지 않도록 `IF` 플래그를 클리어 한다. 이후 현재 인터럽트 핸들러가 수행이 끝나면, CPU 는 `IF` 플래그를 `iret` 명령어와 함께 다시 설정해준다.

예약된 인터럽트 게이트에서 다른 비트들은 0이여야 한다. 이제 CPU 가 어떻게 인터럽트를 처리하는지 보자:

* CPU 플래그 레지스터(`CS`)를 저장하고, 명령 포인터(IP, Instruction Pointer) 를 스택에 넣는다.
* 만약 인터럽트가 에러코드를 발생시키면(`#PF`와 같은), CPU 는 명령 포인터 다음에 에러 코드를 저장한다.
* 인터럽트 핸들러가 실행되고 나면, `iret` 명령어로 복귀한다.

이제 코드로 돌아가보자.

IDT 를 채우고 로드
--------------------------------------------------------------------------------

우리는 아래의 코드까지 살펴보고 멈추었었다.

```C
for (i = 0; i < NUM_EXCEPTION_VECTORS; i++)
	set_intr_gate(i, early_idt_handler_array[i]);
```

여기 루프 내에서 `set_intr_gate` 호출을 볼 수 있다. 이 함수의 인자는 두 개인데:

* `벡터 번호` 또는 인터럽트 번호
* IDT 핸들러의 주소

그리고 인터럽트 게이트를 `&idt_desc` 에 대응되는 `IDT` 테이블에 넣는다. 제일 먼저 `early_idt_handler_array` 배열을 보자. 이 배열은 [arch/x86/include/asm/segment.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/segment.h) 헤더 파일에 정의되어 있고 첫 `32` 예외 처리의 주소를 담고 있다.:

```C
#define EARLY_IDT_HANDLER_SIZE   9
#define NUM_EXCEPTION_VECTORS	32

extern const char early_idt_handler_array[NUM_EXCEPTION_VECTORS][EARLY_IDT_HANDLER_SIZE];
```

`early_idt_handler_array` 는 9 바이트의 크기의 예외 엔트리 포인트의 주소들을 포함하여 총 `288` 바이트가 된다. 이 배열의 각 9 바이트의 구성을 살펴 보면, 만약 예외가 제공되지 않는다면 dummy 에러 코드를 넣기 위해 추가적인 2 바이트가 있고, 또 다른 2 바이트 명령어로 스택에 벡터 번호를 넣는다. 마지막 5바이트는 일반적인 예외 처리 코드로 `jump` 하기 위한 명령어이다.

이미 봤듯이, 루프내에서 첫 32 개의 `IDT` 엔트리를 채웠다. 그 이유는 초기 설정하는 동안 인터럽트들이 비활성화 되어 있었고, `32` 보다 큰 벡터들을 위한 인터럽트 핸들러를 설정할 필요가 없었기 때문이다.  `early_idt_handler_array` 배열은 일반적인 IDT 핸들러로 구성되어 있고 그 정의는 [arch/x86/kernel/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head_64.S) 어셈블리 파일에서 찾아 볼 수 있다. 지금은 이 부분을 건너뛰고 `set_intr_gate` 매크로를 살펴 볼 것이다.

`set_intr_gate` 매크로는 [arch/x86/include/asm/desc.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/desc.h) 헤더 파일에 정의되어 있고 아래 처럼 생겼다:

```C
#define set_intr_gate(n, addr)                         \
         do {                                                            \
                 BUG_ON((unsigned)n > 0xFF);                             \
                 _set_gate(n, GATE_INTERRUPT, (void *)addr, 0, 0,        \
                           __KERNEL_CS);                                 \
                 _trace_set_gate(n, GATE_INTERRUPT, (void *)trace_##addr,\
                                 0, 0, __KERNEL_CS);                     \
         } while (0)
```

인터럽트 번호를 첫번째로 인자로 들어가고 `BUG_ON` 매크로로 `255` 보다 큰지 확인한다. 우리는 여기서 인터럽트는 모두 `256`개만 가질 수 있다. 이 다음에, 그것은 `IDT` 로 인터럽트 게이트의 주소를 쓰는 `_set_gate` 함수를 호출한다.:

```C
static inline void _set_gate(int gate, unsigned type, void *addr,
	                         unsigned dpl, unsigned ist, unsigned seg)
{
         gate_desc s;
         pack_gate(&s, type, (unsigned long)addr, dpl, ist, seg);
         write_idt_entry(idt_table, gate, &s);
         write_trace_idt_entry(gate, &s);
}
```

`_set_gate` 함수의 시작을 보면 `gate_desc` 구조체를 주어진 값들로 채우는 `pack_gate` 함수의 호출을 볼 수 있다:

```C
static inline void pack_gate(gate_desc *gate, unsigned type, unsigned long func,
                             unsigned dpl, unsigned ist, unsigned seg)
{
        gate->offset_low        = PTR_LOW(func);
        gate->segment           = __KERNEL_CS
        gate->ist               = ist;
        gate->p                 = 1;
        gate->dpl               = dpl;
        gate->zero0             = 0;
        gate->zero1             = 0;
        gate->type              = type;
        gate->offset_middle     = PTR_MIDDLE(func);
        gate->offset_high       = PTR_HIGH(func);
}
```

내가 위에서 언급했듯이, 우리는 이 함수에서 게이트 디스크립터를(gate descriptor) 채운다. 메인 루프(인터럽트 엔트리 포인트의 주소)에서 얻은 주소와 함께 인터럽트 핸들러의 주소의 3 부분(low, mid, high)을 채운다. 아래 3개의 매크들을 사용하여 주소를 3 부분으로 분리한다.:

```C
#define PTR_LOW(x) ((unsigned long long)(x) & 0xFFFF)
#define PTR_MIDDLE(x) (((unsigned long long)(x) >> 16) & 0xFFFF)
#define PTR_HIGH(x) ((unsigned long long)(x) >> 32)
```

`PTR_LOW` 매크로로 주소의 첫 '2' 바이트를 얻어온다. `PTR_MIDDLE` 로는 그 다음의 '2' 바이트를 얻을 수 있고, 마지막으로 `PTR_HIGH` 로는 주소의 마지막 `4`바이트를 얻을 수 있다. 다음은 인터럽트 핸들러를 위한 세그먼트 셀렉터를 설정한다. 그것은 우리의 커널 코드 세그먼트인 `__KERNEL_CS` 이다. 다음 단계는 `Interrupt Stack Table` 와 `Descriptor Privilege Level`(가장 높은 특권 레벨로)을 0으로 채운다. 그리고 마지막으로 우리는 `GAT_INTERRUPT` 타입을 설정한다.

Now we have filled IDT entry and we can call `native_write_idt_entry` function which just copies filled `IDT` entry to the `IDT`:

```C
static inline void native_write_idt_entry(gate_desc *idt, int entry, const gate_desc *gate)
{
        memcpy(&idt[entry], gate, sizeof(*gate));
}
```

메인 루프가 마무리되면, 우리는 `gate_desc` 구조체의 `idt_table` 배열을 채우게 될 것이고, 아래의 코드가 호출되면 `Interrupt Descriptor table` 를 로드할 수 있다:

```C
load_idt((const struct desc_ptr *)&idt_descr);
```

`idt_descr` 은 아래에서 확인:

```C
struct desc_ptr idt_descr = { NR_VECTORS * 16 - 1, (unsigned long) idt_table };
```

그리고 `load_idt`는 단지 `lidt` 명령어를 실행한다:

```C
asm volatile("lidt %0"::"m" (*dtr));
```

당신은 `_set_gate` 와 다른 함수에서 `_trace_*` 함수들의 호출을 볼 수 있을 것이다.  이것은 `_set_gate` 와 `IDT` 게이트를 채우는 것은 동일하나 한가지 다른 점을 갖고 있다. 그것은 트레이스 포인트(tracepoints) 를 위해(나중에 다른 파트에서 살펴보자) `Interrupt Descriptor Table` 을 위한 `idt_table` 대신에 `trace_idt_table` 를 사용한다.

좋다, 이제 우리는 `Interrupt Descriptor Table`을 채우고 로드했다. 우리는 이제 어떻게 CPU 가 인터럽트를 핸들링 하는지 알아보자.

초기 인터럽트 핸들러
--------------------------------------------------------------------------------

위에서 봤듯이, 우리는 `early_idt_handler_array` 의 주소로 `IDT`를 채웠다. 우리는 [arch/x86/kernel/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head_64.S) 어셈블리 파일에서 그것을 찾아 볼 수 있다:

```assembly
	.globl early_idt_handler_array
early_idt_handlers:
	i = 0
	.rept NUM_EXCEPTION_VECTORS
	.if (EXCEPTION_ERRCODE_MASK >> i) & 1
	pushq $0
	.endif
	pushq $i
	jmp early_idt_handler_common
	i = i + 1
	.fill early_idt_handler_array + i*EARLY_IDT_HANDLER_SIZE - ., 1, 0xcc
	.endr
```

여기서 우리는 첫 `32` 예외를 위해 인터럽트 핸들러 생성을 볼 수 있다. 만약 예외가 에러 코드를 가진다면 아무것도 하지 않고, 예외가 에러 코드를 반환하면, 우리는 스택에 0을 넣을 것이다. 이것은 스택을 0으로 균일한 값을 가지도록 하는 것이다. 이후에, 예외 번호를 스택에 넣고 지금 사용되는 일반적인 인터럽트 핸들러인 `early_idt_handler_array`로 점프할 것이다. 위에 코드를 보면, `early_idt_handler_array` 의 매 9 바이트는
추가적인 에러 코드, `벡터 번호`와 jump 명령어를 스택에 넣는다. 우리는 `objdump` 유틸의 출력으로 확인 할 수 있다.:
```
$ objdump -D vmlinux
...
...
...
ffffffff81fe5000 <early_idt_handler_array>:
ffffffff81fe5000:       6a 00                   pushq  $0x0
ffffffff81fe5002:       6a 00                   pushq  $0x0
ffffffff81fe5004:       e9 17 01 00 00          jmpq   ffffffff81fe5120 <early_idt_handler_common>
ffffffff81fe5009:       6a 00                   pushq  $0x0
ffffffff81fe500b:       6a 01                   pushq  $0x1
ffffffff81fe500d:       e9 0e 01 00 00          jmpq   ffffffff81fe5120 <early_idt_handler_common>
ffffffff81fe5012:       6a 00                   pushq  $0x0
ffffffff81fe5014:       6a 02                   pushq  $0x2
...
...
...
```

위의 내용을 보면, CPU 는 플래그(flag) 레지스터인 `CS` 와 `RIP`를 스택에 넣는다. 그래서 `early_idt_handler` 실행되기 전에, 스택은 아래의 데이터를 포함한다:

```
|--------------------|
| %rflags            |
| %cs                |
| %rip               |
| rsp --> error code |
|--------------------|
```

이제 `early_idt_handler_common` 구현을 살펴보자. 그것은 [arch/x86/kernel/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head_64.S#L343) 어셈블리 파일에 위치 하고 있으며, 제일 먼저 [NMI](http://en.wikipedia.org/wiki/Non-maskable_interrupt) 를 확인하다. 우리는 그것을 처리할 필요 없고, `early_idt_handler_common` 에서는 그냥 그것을 무시한다.:

```assembly
	cmpl $2,(%rsp)
	je .Lis_nmi
```

`is_nmi` 는:

```assembly
is_nmi:
	addq $16,%rsp
	INTERRUPT_RETURN
```

스택으로 부터 에러코드와 벡터 번호를 버리고 `iretq` 명령어를 확장해주는 `INTERRUPT_RETURN` 를 호출한다. 벡터 번호와 `NMI` 가 아니고 벡터 번호가 확인 되고 나서, `early_idt_handler_common`내에서 재귀 호출을 방지하기 위해 `early_recursion_flag` 를 확인하고, 만약 맞게 되었다면 범용 레지스터를 스택에 저장한다.:

```assembly
	pushq %rax
	pushq %rcx
	pushq %rdx
	pushq %rsi
	pushq %rdi
	pushq %r8
	pushq %r9
	pushq %r10
	pushq %r11
```

인터럽트 핸들러로 부터 복귀할 때, 레지스터들의 잘못된 값이 들어가는 것을 막기 위해 이렇게 할 필요가 있다. 이 후에, 스택에 있는 세그먼트 셀렉터를 확인한다.:

```assembly
	cmpl $__KERNEL_CS,96(%rsp)
	jne 11f
```

이것은 커널 코드 세그먼트와 반드시 같아야 하고, 만약 그렇지 않은 경우, 라벨 `11` 로 점프하여 `PANIC` 메세지와 함께 스택 덤프 데이터를 출력한다.

코드 세그먼트가 확인되면, 벡터 번호를 확인하고 만약 그것이 `#PF`, 즉 [페이지 폴트](https://en.wikipedia.org/wiki/Page_fault) 라면, 우리는 `cr2` 의 값을 `rdi` 레지스터에 넣고, `early_make_pgtable` 함수를 호출한다.:

```assembly
	cmpl $14,72(%rsp)
	jnz 10f
	GET_CR2_INTO(%rdi)
	call early_make_pgtable
	andl %eax,%eax
	jz 20f
```

만약 벡터 번호가 `#PF` 가 아니라면, 우리는 범용 레지스터를 스택에 저장한다:

```assembly
	popq %r11
	popq %r10
	popq %r9
	popq %r8
	popq %rdi
	popq %rsi
	popq %rdx
	popq %rcx
	popq %rax
```

그리고 `iret` 을 통해 핸들러로 부터 종료한다.

여기까지가 첫 인터럽트 핸들러의 마지막이다. 이것은 매우 초기의 인터럽트 핸들러이니, 페이지 폴트만 처리 가능하다. 우리는 다른 인터럽트를 위한 핸들러들을 살펴 볼 것이지만, 먼저 페이지 폴트 핸들러에 대해 조금 더 살펴보자.

페이지 폴트 처리
--------------------------------------------------------------------------------

페이지 폴트 확인을 위한 인터럽트 번호를 확인하는 초기 인터럽트 핸들러를 보았고, 핸들러가 호출된다면, 새로운 페이지 테이블을 만들기 위해 `early_make_pgtable` 를 호출한다. 우리는 지금 단계에서 `4G` 위쪽에 커널을 로드하고 `boot_params` 를 접근하기 위해 `#PF` 핸들러를 가져야 한다.

`early_make_pgtable` 의 구현은 [arch/x86/kernel/head64.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head64.c) 에서 찾을 수 있고, 그것은 하나의 인자를 받는다. 그 인자는 `cr2` 레지스터의 주소이며, 페이지 폴트를 야기한다:

```C
int __init early_make_pgtable(unsigned long address)
{
	unsigned long physaddr = address - __PAGE_OFFSET;
	unsigned long i;
	pgdval_t pgd, *pgd_p;
	pudval_t pud, *pud_p;
	pmdval_t pmd, *pmd_p;
	...
	...
	...
}
```

`*var_t` 로 시작하는 타입의 변수들의 선언으로 시작한다. 이 모든 타입은 모두 아래와 같다:

```C
typedef unsigned long   pgdval_t;
```

`*_t` 타입과 함께 수행할 것이다, 예를 들면 `pgd_t` 와 같은 것이다. [arch/x86/include/asm/pgtable_types.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/pgtable_types.h) 에 이런 타입들이 선언되어 있고 아래의 구조체로 만들어 놓는다:

```C
typedef struct { pgdval_t pgd; } pgd_t;
```

예를 들면,

```C
extern pgd_t early_level4_pgt[PTRS_PER_PGD];
```

여기에 초기에 최상위 레벨의 `pgd_t` 타입의 배열과 low 레벨 페이지 엔트리를 가리키는 `pgd` 로 구성되는 페이지 테이블 디렉토리를 표현하는 `early_level4_pgt` 가 있다.

유효하지 않는 주소가 없다고 확인된 후에, 우리는 `#PF` 주소를 포함하는 페이지 글러벌 디렉토리(Page Global Directory) 엔트리의 주소를 얻고 그 주소를 `pgd` 변수에 넣어준다.:

```C
pgd_p = &early_level4_pgt[pgd_index(address)].pgd;
pgd = *pgd_p;
```

다음 단계로는 `pgd`를 확인하는 것인데, 만약 그것이 알맞은 페이지 글로벌 디렉토리 엔트리를 갖고 있다면, 글로벌 디렉토리 엔트리의 물리주소를 `pud_p` 에 넣어준다:

```C
pud_p = (pudval_t *)((pgd & PTE_PFN_MASK) + __START_KERNEL_map - phys_base);
```

`PTE_PFN_MASK` 는 매크로 이다:

```C
#define PTE_PFN_MASK            ((pteval_t)PHYSICAL_PAGE_MASK)
```

이것을 풀어보면:

```C
(~(PAGE_SIZE-1)) & ((1 << 46) - 1)
```

or

```
0b1111111111111111111111111111111111111111111111
```

이것은 46 비트의 페이 프레임 마스크 값이다.

만약 `pgd` 가 제대로된 주소를 갖고 있지 않다면, 우리는 `next_early_pgt` 가 `EARLY_DYNAMIC_PAGE_TABLES`(64) 보다 크지 않는지 확인해야 하고 요구가 있을 때 새로운 페이지 테이블을 설정하기 위해 고정된 크기의 버퍼들이 있는지도 확인해야 한다. 만약 `next_early_pgt` 가 `EARLY_DYNAMIC_PAGE_TABLES` 보다 크다면, 페이지 테이블을 리셋하고 다시 시작해야 한다. 만약 `next_early_pgt`이 `EARLY_DYNAMIC_PAGE_TABLES` 보다 작다면, 우리는 현재 동적 페이지 테이블을 가리키고 있게 하는 새로우 PUD(page upper directory)를 만들고 그것의 물리 주소를  `_KERPG_TABLE`의 접근권한과 함께 페이지 글로벌 디렉토리(PGD)로 써준다.:

```C
if (next_early_pgt >= EARLY_DYNAMIC_PAGE_TABLES) {
	reset_early_page_tables();
    goto again;
}

pud_p = (pudval_t *)early_dynamic_pgts[next_early_pgt++];
for (i = 0; i < PTRS_PER_PUD; i++)
	pud_p[i] = 0;
*pgd_p = (pgdval_t)pud_p - __START_KERNEL_map + phys_base + _KERNPG_TABLE;
```

PUD(Page Upper Direcotry)의 주소를 고친 다음에는:

```C
pud_p += pud_index(address);
pud = *pud_p;
```

다음 단계에서는 이전에 했던 같은 행동을 한다, 하지만 PMD(Page Middle Directory) 와 함께 진행한다. 마지막으로는 커널 text + data 가상 주소의 맵을 포함하는 PMD 를 수정한다:

```C
pmd = (physaddr & PMD_MASK) + early_pmd_flags;
pmd_p[pmd_index(address)] = pmd;
```

페이지 폴트 핸들러가 마무리 되고 동작한다면, 그 결과로 `early_level4_pgt`가 유효한 주소를 가리키는 엔트리들을 포함하게 된다.

결론
--------------------------------------------------------------------------------

리눅스 커널 초기화와 관련된 두 번째 파트가 끝이 났다. 어떤 질문이나 제안이 있다면, twitter [0xAX](https://twitter.com/0xAX), [email](anotherworldofworld@gmail.com) 또는 [issue](https://github.com/0xAX/linux-insides/issues/new) 를 만들어 주길 바란다. 다음 파트에서는 커널 엔트리 포인트인 `start_kernel` 함수 전에 일어나는 모든 단계를 살펴보도록 하자.

**나는 영어권의 사람이 아니고 이런 것에 대해 매우 미안해 하고 있다. 만약 어떤 실수를 발견한다면, 나에게 PR을 [linux-insides](https://github.com/0xAX/linux-internals)을 보내줘**

링크
--------------------------------------------------------------------------------

* [GNU assembly .rept](https://sourceware.org/binutils/docs-2.23/as/Rept.html)
* [APIC](http://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller)
* [NMI](http://en.wikipedia.org/wiki/Non-maskable_interrupt)
* [Page table](https://en.wikipedia.org/wiki/Page_table)
* [Interrupt handler](https://en.wikipedia.org/wiki/Interrupt_handler)
* [Page Fault](https://en.wikipedia.org/wiki/Page_fault),
* [Previous part](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-1.html)
