# 제 0장: 소개

## Introduction to Linux Kernel

### 리눅스 커널이란?

리눅스 커널은 운영체제의 핵심 부분으로, 하드웨어와 소프트웨어 간의 중재자 역할을 합니다. 1991년 리누스 토발즈(Linus Torvalds)가 처음 개발한 이후, 전 세계 개발자들의 기여로 성장해왔습니다.

### 왜 리눅스 커널을 공부해야 하는가?

1. **운영체제의 작동 원리 이해**: 컴퓨터 시스템이 어떻게 동작하는지 깊이 있게 이해할 수 있습니다
2. **시스템 프로그래밍 능력 향상**: 저수준 프로그래밍 기술을 향상시킬 수 있습니다
3. **성능 최적화**: 시스템 성능을 최적화하는 방법을 배울 수 있습니다
4. **오픈소스 기여**: 세계에서 가장 큰 오픈소스 프로젝트에 기여할 수 있습니다

### 리눅스 커널의 주요 구성 요소

```
Linux Kernel
├── Process Management (프로세스 관리)
├── Memory Management (메모리 관리)
├── File Systems (파일 시스템)
├── Device Drivers (장치 드라이버)
├── Networking (네트워킹)
└── System Calls (시스템 콜)
```

### 커널 소스 코드 구조

리눅스 커널 소스 코드는 다음과 같은 주요 디렉토리로 구성되어 있습니다:

- **arch/**: 아키텍처별 코드 (x86, ARM, etc.)
- **kernel/**: 핵심 커널 코드
- **mm/**: 메모리 관리
- **fs/**: 파일 시스템
- **drivers/**: 장치 드라이버
- **net/**: 네트워킹 스택
- **include/**: 헤더 파일
- **Documentation/**: 문서

### GPT를 활용한 코드 탐색

GPT(Generative Pre-trained Transformer)를 활용하면 복잡한 커널 코드를 더 쉽게 이해할 수 있습니다:

1. **코드 설명 요청**: 특정 함수나 코드 블록의 의미를 물어볼 수 있습니다
2. **디버깅 지원**: 버그를 찾거나 코드의 문제점을 파악하는 데 도움을 받을 수 있습니다
3. **학습 가이드**: 단계별로 학습할 내용을 추천받을 수 있습니다
4. **코드 예제**: 특정 개념을 설명하는 예제 코드를 생성할 수 있습니다

### 이 책의 활용 방법

1. **순차적 학습**: 장을 순서대로 읽으며 체계적으로 학습합니다
2. **실습 중심**: 각 장의 예제를 직접 실행하고 수정해봅니다
3. **GPT 활용**: 이해가 안 되는 부분은 GPT에게 질문합니다
4. **커뮤니티 참여**: GitHub Issues를 통해 질문하고 토론합니다

### 개발 환경 설정

리눅스 커널을 학습하기 위해서는 다음과 같은 환경이 필요합니다:

#### 필수 도구

```bash
# Ubuntu/Debian 기반
sudo apt-get update
sudo apt-get install build-essential libncurses-dev bison flex libssl-dev libelf-dev

# 커널 소스 다운로드
git clone https://github.com/torvalds/linux.git
cd linux

# 브랜치 확인
git branch -a
```

#### 추천 도구

- **Code Editor**: Visual Studio Code, Vim, Emacs
- **Debugger**: GDB, KGDB
- **Virtual Machine**: QEMU, VirtualBox
- **Documentation Tools**: Doxygen, Sphinx

### 다음 장에서는...

다음 장에서는 리눅스 커널의 전체 아키텍처를 살펴보고, 각 구성 요소가 어떻게 상호작용하는지 알아봅니다.

---

[다음: 커널 아키텍처 개요 →](01-kernel-architecture.md)
