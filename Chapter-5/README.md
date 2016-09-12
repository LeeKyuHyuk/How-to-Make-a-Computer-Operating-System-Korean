## 5장: x86 아키텍처를 관리하기 위한 기본 클래스

이제 우리는 어떻게 C++로 짜인 커널을 컴파일하고, GRUB를 사용하여 바이너리를 부팅하는 방법을 알고 있습니다. 우리는 C/C++로 재밌는 작업을 시작할 수 있습니다.

#### 콘솔 화면에 출력하기

우리는 VGA 기본 모드 (03h)를 사용하여 사용자가 텍스트를 출력할 수 있게 할 것입니다. 화면은 비디오 메모리를 사용하여 0xB8000에 직접 접근할 수 있습니다. 화면의 해상도는 80x25이고 화면의 각 문자는 2바이트(하나의 문자 코드와 하나의 스타일 플래그)로 정의됩니다. 이것은 비디오 메모리의 전체 크기는 4000바이트(80바이트*25바이트*2바이트)라는 것을 의미합니다.

입출력 클래스 ([io.cc](https://github.com/LeeKyuHyuk/How-to-Make-a-Computer-Operating-System-Korean/blob/master/src/kernel/arch/x86/io.cc)),:
* **x,y**: 화면의 커서 위치를 정의
* **real_screen**: 비디오 메모리 포인터를 정의
* **putc(char c)**: 문자를 화면에 출력하고 커서 위치를 관리
* **printf(char* s, ...)**: 문자열 출력

우리는 화면에 문자를 출력하고 (x,y) 위치를 갱신하기 위해 **putc** 메서드를 [IO Class](https://github.com/LeeKyuHyuk/How-to-Make-a-Computer-Operating-System-Korean/blob/master/src/kernel/arch/x86/io.cc)에 추가해야 합니다.

```cpp
/* put a byte on screen */
void Io::putc(char c){
	kattr = 0x07;
	unsigned char *video;
	video = (unsigned char *) (real_screen+ 2 * x + 160 * y);
	// newline
	if (c == '\n') {
		x = 0;
		y++;
	// back space
	} else if (c == '\b') {
		if (x) {
			*(video + 1) = 0x0;
			x--;
		}
	// horizontal tab
	} else if (c == '\t') {
		x = x + 8 - (x % 8);
	// carriage return
	} else if (c == '\r') {
		x = 0;
	} else {
		*video = c;
		*(video + 1) = kattr;

		x++;
		if (x > 79) {
			x = 0;
			y++;
		}
	}
	if (y > 24)
		scrollup(y - 24);
}
```

유용하고 자주 사용하는 메서드도 추가합니다: [printf](https://github.com/LeeKyuHyuk/How-to-Make-a-Computer-Operating-System-Korean/blob/master/src/kernel/arch/x86/io.cc#L154)

```cpp
/* put a string in screen */
void Io::print(const char *s, ...){
	va_list ap;

	char buf[16];
	int i, j, size, buflen, neg;

	unsigned char c;
	int ival;
	unsigned int uival;

	va_start(ap, s);

	while ((c = *s++)) {
		size = 0;
		neg = 0;

		if (c == 0)
			break;
		else if (c == '%') {
			c = *s++;
			if (c >= '0' && c <= '9') {
				size = c - '0';
				c = *s++;
			}

			if (c == 'd') {
				ival = va_arg(ap, int);
				if (ival < 0) {
					uival = 0 - ival;
					neg++;
				} else
					uival = ival;
				itoa(buf, uival, 10);

				buflen = strlen(buf);
				if (buflen < size)
					for (i = size, j = buflen; i >= 0;
					     i--, j--)
						buf[i] =
						    (j >=
						     0) ? buf[j] : '0';

				if (neg)
					print("-%s", buf);
				else
					print(buf);
			}
			 else if (c == 'u') {
				uival = va_arg(ap, int);
				itoa(buf, uival, 10);

				buflen = strlen(buf);
				if (buflen < size)
					for (i = size, j = buflen; i >= 0;
					     i--, j--)
						buf[i] =
						    (j >=
						     0) ? buf[j] : '0';

				print(buf);
			} else if (c == 'x' || c == 'X') {
				uival = va_arg(ap, int);
				itoa(buf, uival, 16);

				buflen = strlen(buf);
				if (buflen < size)
					for (i = size, j = buflen; i >= 0;
					     i--, j--)
						buf[i] =
						    (j >=
						     0) ? buf[j] : '0';

				print("0x%s", buf);
			} else if (c == 'p') {
				uival = va_arg(ap, int);
				itoa(buf, uival, 16);
				size = 8;

				buflen = strlen(buf);
				if (buflen < size)
					for (i = size, j = buflen; i >= 0;
					     i--, j--)
						buf[i] =
						    (j >=
						     0) ? buf[j] : '0';

				print("0x%s", buf);
			} else if (c == 's') {
				print((char *) va_arg(ap, int));
			}
		} else
			putc(c);
	}

	return;
}
```

#### 어셈블리 인터페이스

어셈블리에는 많은 명령이 존재하지만 C(cli, sti, in, out 같은)와 일치하지 않습니다. 그래서 우리는 이 명령에 대한 인터페이스가 필요합니다.

C에서는 우리는 "asm()" 지시문을 사용하여 어셈블리를 포함할 수 있습니다. [GCC](https://en.wikipedia.org/wiki/GNU_Compiler_Collection)는 [GAS](https://en.wikipedia.org/wiki/GNU_Assembler)를 어셈블리 컴파일 할 때 사용합니다.

**주의:** GAS는 AT&T 문법을 사용합니다.

```cpp
/* output byte */
void Io::outb(u32 ad, u8 v){
	asmv("outb %%al, %%dx" :: "d" (ad), "a" (v));;
}
/* output word */
void Io::outw(u32 ad, u16 v){
	asmv("outw %%ax, %%dx" :: "d" (ad), "a" (v));
}
/* output word */
void Io::outl(u32 ad, u32 v){
	asmv("outl %%eax, %%dx" : : "d" (ad), "a" (v));
}
/* input byte */
u8 Io::inb(u32 ad){
	u8 _v;       \
	asmv("inb %%dx, %%al" : "=a" (_v) : "d" (ad)); \
	return _v;
}
/* input word */
u16	Io::inw(u32 ad){
	u16 _v;			\
	asmv("inw %%dx, %%ax" : "=a" (_v) : "d" (ad));	\
	return _v;
}
/* input word */
u32	Io::inl(u32 ad){
	u32 _v;			\
	asmv("inl %%dx, %%eax" : "=a" (_v) : "d" (ad));	\
	return _v;
}
```
