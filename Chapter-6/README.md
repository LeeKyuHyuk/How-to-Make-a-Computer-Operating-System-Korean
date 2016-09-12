## 6장: GDT

GRUB 덕분에, 커널은 더 이상 리얼 모드가 아니고 [보호 모드](http://en.wikipedia.org/wiki/Protected_mode)입니다. 이 모드로 우리는 가상 메모리, 페이징, 안정적인 멀티태스킹으로 마이크로프로세서의 모든 기능을 사용할 수 있습니다.

#### GDT는 무엇인가요?

[GDT](http://en.wikipedia.org/wiki/Global_Descriptor_Table)("Global Descriptor Table")는 서로 다른 메모리 영역을 정의하는데 사용되는 데이터구조 입니다: 기본 주소, 크기 및 액세스 권한을 가지고 있으며 쓰기와 실행도 가능합니다. 이 메모리 영역을 "세그먼트(segments)"라고 합니다.

우리는 GDT를 사용하여 다른 메모리 세그먼트를 정의합니다:

* *"code"*: 커널 코드, 실행 가능한 이진 코드를 저장하는데 사용됩니다.
* *"data"*: 커널 데이터
* *"stack"*: 커널 스택, 커널 실행 중에 발생하는 스택 호출을 저장하기 위해 사용됩니다.
* *"ucode"*: 사용자 코드, 실행 가능한 사용자 이진 코드를 저장하는데 사용됩니다.
* *"udata"*: 사용자 프로그램 데이터
* *"ustack"*: 사용자 스택, 사용자단에서 발생하는 스택 호출을 저장하기 위해 사용됩니다.

#### 어떻게 우리의 GDT를 로드하나요?

GRUB는 GDT를 초기화하지만, GDT는 우리의 커널에 해당되지 않습니다.
GDT는 LGDT 어셈블리 명령어를 사용하여 로드 됩니다. 그것은 GDT 구조체의 위치를 요구합니다:

![GDTR](./gdtr.png)

C 구조체:

```cpp
struct gdtr {
	u16 limite;
	u32 base;
} __attribute__ ((packed));
```

**주의:**  ```__attribute__ ((packed))``` 지시어는 GCC에서 구조체가 가능한 적은 메모리를 사용하도록 신호를 보냅니다. 이 지시어가 없으면 gcc는 메모리 정렬과 메모리 접근을 최적화하기 위해 몇 바이트를 포함하게 됩니다.

이제 우리는 GDT 테이블을 정의하고 LGDT를 사용하여 로드해야 합니다. GDT 테이블은 우리가 원하는 메모리 어디든지 저장될 수 있습니다. 그 주소는 GDTR 레지스트리를 사용하여 프로세스로 신호가 보내져야 합니다.

GDT 테이블은 다음과 같은 세그먼트 구조로 구성됩니다:

![GDTR](./gdtentry.png)

C 구조체:

```cpp
struct gdtdesc {
	u16 lim0_15;
	u16 base0_15;
	u8 base16_23;
	u8 acces;
	u8 lim16_19:4;
	u8 other:4;
	u8 base24_31;
} __attribute__ ((packed));
```

#### GDT 테이블을 어떻게 정의하나요?

우리는 메모리에 우리의 GDT를 정의하고 GDTR 레지스트리를 사용하여 로드해야 합니다.

우리는 주소로 GDT를 저장해야 합니다:

```cpp
#define GDTBASE	0x00000800
```

**init_gdt_desc** 함수는 [x86.cc](https://github.com/LeeKyuHyuk/How-to-Make-a-Computer-Operating-System-Korean/blob/master/src/kernel/arch/x86/x86.cc)에서 GDT 세그먼트 디스크립터를 초기화합니다.

```cpp
void init_gdt_desc(u32 base, u32 limite, u8 acces, u8 other, struct gdtdesc *desc)
{
	desc->lim0_15 = (limite & 0xffff);
	desc->base0_15 = (base & 0xffff);
	desc->base16_23 = (base & 0xff0000) >> 16;
	desc->acces = acces;
	desc->lim16_19 = (limite & 0xf0000) >> 16;
	desc->other = (other & 0xf);
	desc->base24_31 = (base & 0xff000000) >> 24;
	return;
}
```

그리고 **init_gdt** 함수는 GDT를 초기화하고, 아래의 일부 기능은 뒤에서 설명할 멀티태스킹을 위해 사용됩니다.

```cpp
void init_gdt(void)
{
	default_tss.debug_flag = 0x00;
	default_tss.io_map = 0x00;
	default_tss.esp0 = 0x1FFF0;
	default_tss.ss0 = 0x18;

	/* initialize gdt segments */
	init_gdt_desc(0x0, 0x0, 0x0, 0x0, &kgdt[0]);
	init_gdt_desc(0x0, 0xFFFFF, 0x9B, 0x0D, &kgdt[1]);	/* code */
	init_gdt_desc(0x0, 0xFFFFF, 0x93, 0x0D, &kgdt[2]);	/* data */
	init_gdt_desc(0x0, 0x0, 0x97, 0x0D, &kgdt[3]);		/* stack */

	init_gdt_desc(0x0, 0xFFFFF, 0xFF, 0x0D, &kgdt[4]);	/* ucode */
	init_gdt_desc(0x0, 0xFFFFF, 0xF3, 0x0D, &kgdt[5]);	/* udata */
	init_gdt_desc(0x0, 0x0, 0xF7, 0x0D, &kgdt[6]);		/* ustack */

	init_gdt_desc((u32) & default_tss, 0x67, 0xE9, 0x00, &kgdt[7]);	/* descripteur de tss */

	/* initialize the gdtr structure */
	kgdtr.limite = GDTSIZE * 8;
	kgdtr.base = GDTBASE;

	/* copy the gdtr to its memory area */
	memcpy((char *) kgdtr.base, (char *) kgdt, kgdtr.limite);

	/* load the gdtr registry */
	asm("lgdtl (kgdtr)");

	/* initiliaz the segments */
	asm("   movw $0x10, %ax	\n \
            movw %ax, %ds	\n \
            movw %ax, %es	\n \
            movw %ax, %fs	\n \
            movw %ax, %gs	\n \
            ljmp $0x08, $next	\n \
            next:		\n");
}
```
