## 4장: OS와 C++ 런타임의 백본

#### C++ 커널 런타임

커널은 C로 작성될 수 있는 것처럼 C++로도 작성될 수 있습니다. 하지만 C++([런타임 지원](https://ko.wikipedia.org/wiki/%EB%9F%B0%ED%83%80%EC%9E%84_%EB%9D%BC%EC%9D%B4%EB%B8%8C%EB%9F%AC%EB%A6%AC), [생성자](https://ko.wikipedia.org/wiki/%EC%83%9D%EC%84%B1%EC%9E%90), 기타)를 사용함으로써 얻을 수 있는 약간의 잠재 위협에 대한 예외 상황도 있습니다.

컴파일러는 모든 필요한 C++ 런타임 지원을 기본적으로 사용할 수 있음을 가정합니다. 하지만 libsupc++를 우리가 만든 C++ 커널에 링크하지 않습니다. 우리는 기본적인 함수가 필요한데 [cxx.cc](https://github.com/LeeKyuHyuk/How-to-Make-a-Computer-Operating-System-Korean/blob/master/src/kernel/runtime/cxx.cc) 파일에서 찾을 수 있습니다.

**주의:** `new`와 `delete` 오퍼레이터는 [가상 메모리](https://ko.wikipedia.org/wiki/%EA%B0%80%EC%83%81_%EB%A9%94%EB%AA%A8%EB%A6%AC)와 [페이징](https://ko.wikipedia.org/wiki/%ED%8E%98%EC%9D%B4%EC%A7%95)이 초기화되기 전에 사용될 수 없습니다.

#### C/C++ 기본 함수

커널 코드는 표준 라이브러리 함수를 사용할 수 없습니다. 그래서 우리는 몇몇 기본적인 함수(메모리 관리, 문자열)를 추가해야 할 필요가 있습니다:

```cpp
void 	itoa(char *buf, unsigned long int n, int base);

void *	memset(char *dst,char src, int n);
void *	memcpy(char *dst, char *src, int n);

int 	strlen(char *s);
int 	strcmp(const char *dst, char *src);
int 	strcpy(char *dst,const char *src);
void 	strcat(void *dest,const void *src);
char *	strncpy(char *destString, const char *sourceString,int maxLength);
int 	strncmp( const char* s1, const char* s2, int c );
```

이 함수들은  [string.cc](https://github.com/LeeKyuHyuk/How-to-Make-a-Computer-Operating-System-Korean/blob/master/src/kernel/runtime/string.cc), [memory.cc](https://github.com/LeeKyuHyuk/How-to-Make-a-Computer-Operating-System-Korean/blob/master/src/kernel/runtime/memory.cc), [itoa.cc](https://github.com/LeeKyuHyuk/How-to-Make-a-Computer-Operating-System-Korean/blob/master/src/kernel/runtime/itoa.cc)에 정의되어 있습니다.

#### C 자료형

다음 단계에서 우리가 코드에서 사용할 여러 가지 자료형을 정의할 것입니다. 우리가 사용할 대부분의 변수는 unsigned가 될 것입니다. 이는 모든 비트가 정수를 저장하기 위해 사용되는 것을 의미합니다. Signed 변수는 자신의 기호를 표시하기 위해 자신의 첫 번째 비트를 사용합니다.

```cpp
typedef unsigned char 	u8;
typedef unsigned short 	u16;
typedef unsigned int 	u32;
typedef unsigned long long 	u64;

typedef signed char 	s8;
typedef signed short 	s16;
typedef signed int 		s32;
typedef signed long long	s64;
```

#### 커널 컴파일

커널 컴파일은 리눅스 실행파일을 컴파일하는 거와 같지 않습니다. 표준 라이브러리를 사용할 수 없으며 시스템에 대한 의존성을 가지고 있지 않습니다.

[Makefile](https://github.com/LeeKyuHyuk/How-to-Make-a-Computer-Operating-System-Korean/blob/master/src/kernel/Makefile)에 커널 컴파일과 링크 과정이 정의되어 있습니다.

x86 아키텍처의 경우, 아래의 인수가 gcc/g++/ld에서 사용됩니다:

```
# Linker
LD=ld
LDFLAG= -melf_i386 -static  -L ./  -T ./arch/$(ARCH)/linker.ld

# C++ compiler
SC=g++
FLAG= $(INCDIR) -g -O2 -w -trigraphs -fno-builtin  -fno-exceptions -fno-stack-protector -O0 -m32  -fno-rtti -nostdlib -nodefaultlibs

# Assembly compiler
ASM=nasm
ASMFLAG=-f elf -o
```
