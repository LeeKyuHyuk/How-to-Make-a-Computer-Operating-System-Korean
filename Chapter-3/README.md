## 3장: GRUB를 통한 첫 부팅

#### 부팅은 어떤 과정으로 진행되나요?

x86 기반 컴퓨터가 켜질 때, 커널의 'main' 루틴 (`kmain()`)으로 제어가 전송될 때 얻을 수 있는 복잡한 경로를 시작합니다. 이 강좌에서는, 우리는 오직 BIOS 부팅 체계를 고려합니다 UEFI는 고려하지 않습니다.

BIOS의 부팅 순서: RAM 감지 -> 하드웨어 감지/초기화 -> 부팅 순서

우리에게 가장 중요한 단계는 '부팅 순서'입니다. 이것은 BIOS가 초기화를 완료할 때 부트로더 프로세스의 다음 단계로 제어를 전달하려고 시도합니다.

'부팅 순서' 단계에 BIOS는 '부팅 장치'를 결정하려고 합니다. 우리의 운영체제는 하드디스크로 부팅 (CD 또는 USB 메모리 장치로 부팅되게도 가능합니다) 할 것입니다. 부트섹터의 511과 512 오프셋에서 유효한 서명인 `0x55` 와 `0xAA` 바이트를 가지고 있는 경우 장치가 부팅 가능하다고 판단합니다 (이 매직 바이트를 마스터 부트 레코드라고 부르기도 하며, MBR이라고도 합니다). 이 서명은 0b1010101001010101(이진법)으로 표시됩니다. 비트 패턴을 규칙적이면서 주기적으로 크기와 방향을 바꾸는 것이 특정 장애(디바이스 또는 컨트롤러)로부터 보호할 것이라고 생각합니다. 만약 이 패턴이 `0x00`이거나 깨진 경우에는 디바이스가 부팅이 불가능하다고 판단합니다.

BIOS는 각 디바이스에 있는 부트섹터의 맨 앞의 512 바이트를 로드합니다. 유효한 서명 바이트가 발견되면 `0x7C00` 메모리 주소로 복사한 후 `0x7C00` 메모리 주소로부터 코드를 실행하도록 합니다.

이 과정을 통해 CPU가 16비트 리얼 모드로 작동되고 있다면, 이는 하위 호환성을 유지하기 위한 x86 CPU의 기본 상태입니다. 커널에서 32비트 명령을 실행하기 위해서는 부트로더는 CPU를 보호 모드로 전환해야 합니다.

#### GRUB는 무엇인가요?

> GNU GRUB(대개 GRUB)은 GNU 프로젝트의 부트로더이다. 대부분 운영 체제의 커널을 불러올 수 있으며, 인자를 넘겨 줄 수도 있다.

GRUB는 기계(부트로더)에 의해 가장 먼저 부팅되며, 하드디스크에 저장된 커널의 로딩을 간단하게 해줍니다.

#### 왜 GRUB를 사용해야 하나요?

* GRUB는 사용하기 간편합니다
* 32비트 커널을 16비트 코드 없이 간편하게 로드할 수 있습니다
* 멀티부팅이 가능합니다
* 쉽게 메모리에 외부 모듈을 로드할 수 있습니다

#### 어떻게 GRUB를 사용하나요?

GRUB는 멀티부팅 사양을 사용하며, 실행 바이너리는 32비트이어야 하며 특별한 헤더(멀티부팅 헤더)를 맨 앞의 8192 바이트에 포함해야 합니다. 우리의 커널은 ELF("Executable and Linkable Format",  유닉스 계열 시스템들의 표준 바이너리 파일 형식) 실행 파일이 될 것입니다.

우리가 만들 커널의 첫 번째 부팅 순서는 어셈블리로 작성되어 있습니다: [start.asm](https://github.com/LeeKyuHyuk/How-to-Make-a-Computer-Operating-System-Korean/blob/master/src/kernel/arch/x86/start.asm) 그리고 우리는 실행 구조를 정의하기 위해 링커 파일을 사용하고 있습니다: [linker.ld](https://github.com/LeeKyuHyuk/How-to-Make-a-Computer-Operating-System-Korean/blob/master/src/kernel/arch/x86/linker.ld).

이 부팅 과정에  C++ 런타임 중 일부는 초기화될 것입니다. 이 부분에 대해서는 다음 장에서 설명하도록 하겠습니다.

멀티부트 헤더의 구조:

```cpp
struct multiboot_info {
	u32 flags;
	u32 low_mem;
	u32 high_mem;
	u32 boot_device;
	u32 cmdline;
	u32 mods_count;
	u32 mods_addr;
	struct {
		u32 num;
		u32 size;
		u32 addr;
		u32 shndx;
	} elf_sec;
	unsigned long mmap_length;
	unsigned long mmap_addr;
	unsigned long drives_length;
	unsigned long drives_addr;
	unsigned long config_table;
	unsigned long boot_loader_name;
	unsigned long apm_table;
	unsigned long vbe_control_info;
	unsigned long vbe_mode_info;
	unsigned long vbe_mode;
	unsigned long vbe_interface_seg;
	unsigned long vbe_interface_off;
	unsigned long vbe_interface_len;
};
```

```mbchk kernel.elf``` 명령어로 kernel.elf 파일이 멀티 부트 커널 포맷인지 확인할 수 있습니다.  또한  ```nm -n kernel.elf``` 명령어로 ELF 바이너리에 있는 다른 오브젝트의 오프셋의 유효성을 검사합니다.

#### 커널과 GRUB를 위한 디스크 이미지 생성

[diskimage.sh](https://github.com/LeeKyuHyuk/How-to-Make-a-Computer-Operating-System-Korean/blob/master/src/sdk/diskimage.sh) 스크립트로 QEMU에서 사용할 하드 디스크 이미지를 생성할 수 있습니다.

첫 번째 단계로 하드 디스크 이미지 (c.img)를 qemu-img를 사용하여 생성합니다:

```
qemu-img create c.img 2M
```

fdisk를 사용하여 디스크 파티션을 설정합니다:

```bash
fdisk ./c.img

# 전문가 명령으로 전환
> x

# 실린더 넘버를 변경 (1-1048576)
> c
> 4

# 헤더 넘버를 변경 (1-256, 기본값 16):
> h
> 16

# 섹터/트랙 넘버를 변경 (1-63, 기본값 63)
> s
> 63

# 메인 메뉴로 돌아가기
> r

# 새로운 파티션을 추가
> n

# primary 파티션으로 선택
> p

# 파티션 넘버를 선택
> 1

# first sector 선택 (1-4, 기본값 1)
> 1

# last sector 선택, +실린더 또는 +크기{K,M,G} (1-4, 기본값 4)
> 4

# 부팅가능 플래그로 전환
> a

# 첫번째 파티션을 부팅가능 플래그로 선택
> 1

# 테이블을 디스크에 기록하고 종료
> w
```

우리는 이제 losetup을 사용하여 loop-device에 생성된 파티션을 첨부해야 합니다. 이 파일은 블록 디바이스 같은 거에 액세스할 수 있습니다. 파티션의 오프셋은 아래와 같이 계산합니다: **offset= start_sector * bytes_by_sector**.

```fdisk -l -u c.img```를 사용하여 얻을 수 있습니다: 63 * 512 = 32256.

```bash
losetup -o 32256 /dev/loop1 ./c.img
```

새로운 디바이스에서 EXT2 파일시스템을 사용할 수 있게 생성합니다:

```bash
mke2fs /dev/loop1
```

파일들을 마운트 한 디스크에 복사합니다:

```bash
mount  /dev/loop1 /mnt/
cp -R bootdisk/* /mnt/
umount /mnt/
```

디스크에 GRUB를 설치합니다:

```bash
grub --device-map=/dev/null << EOF
device (hd0) ./c.img
geometry (hd0) 4 16 63
root (hd0,0)
setup (hd0)
quit
EOF
```

그리고 마지막으로 loop device를 분리합니다:

```bash
losetup -d /dev/loop1
```

#### 참조

* [GNU GRUB on Wikipedia](http://en.wikipedia.org/wiki/GNU_GRUB)
* [Multiboot specification](https://www.gnu.org/software/grub/manual/multiboot/multiboot.html)
