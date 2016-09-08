## 2장: 개발환경 설정

첫 번째 단계로 Ubuntu 16.04에서 개발 환경을 설정할 것입니다.

### **패키지 설치하기**

```
$ sudo apt install nasm make build-essential grub qemu zip -y
```

### **운영체제 빌드와 테스트**

[Makefile](https://github.com/LeeKyuHyuk/How-to-Make-a-Computer-Operating-System-Korean/blob/master/src/Makefile) 파일에 커널과 사용자 libc 그리고 사용자 영역의 프로그램들의 빌드 규칙이 정의되어있습니다.

```
$ git clone https://github.com/LeeKyuHyuk/How-to-Make-a-Computer-Operating-System-Korean.git
$ cd How-to-Make-a-Computer-Operating-System-Korean/src
```

빌드:

```
make all
```

QEMU를 사용하여 운영체제 테스트:

```
make run
```

QEMU에 대한 문서는 [QEMU Emulator Documentation](http://wiki.qemu.org/download/qemu-doc.html)에서 볼 수 있습니다.

에뮬레이터에서 나가고 싶다면 아래의 단축키를 사용합니다: <kbd>Ctrl</kbd>-<kbd>a</kbd>
