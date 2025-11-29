# 1장: 커널 여행의 첫걸음 - Hello, Kernel World!

> _“이 여정은 단순한 명령어의 나열이 아니다.  
> 시스템의 심장박동을 직접 느끼는 일이다.”_

---

## 1.1 우리는 왜 커널을 탐험하는가

**나:** GPT, 커널이란 정확히 뭐야?

**GPT:** "운영체제의 심장입니다. 
모든 프로그램이 실행될 때,
CPU와 메모리, 파일, 장치 사이를 중재하죠."

커널(Kernel)은 **운영체제의 핵심 부분**으로, 
하드웨어와 유저 프로그램(User Space) 사이에서
다리 역할을 합니다. 

### 커널이 하는 일

- 프로세스 관리 (Process Management)
- 메모리 관리 (Memory Management)
- 파일 시스템 (File Systems)
- 장치 제어 (Device Drivers)
- 네트워킹 (Networking)
- 시스템 호출 인터페이스 (System Call Interface)

> 💬 **GPT의 코멘트:**  
> “커널은 눈에 보이지 않지만,  
> 당신의 모든 명령은 그를 거쳐 갑니다.”

---

### 1.2 리눅스 커널의 세계지도

리눅스 커널 소스 트리는 거대한 도시와 같습니다.
각 디렉토리에는 역할이 분명히 나누어져 있죠. 

```
/init → 부팅 초기화 코드 (init/main.c)
/kernel → 스케줄러, 시그널, 프로세스 관리
/mm → 메모리 관리
/fs → 파일 시스템 (VFS, ext4 등)
/drivers → 장치 드라이버
/net → 네트워킹
/include → 전역 헤더 파일
/Documentation → 문서
/arch → CPU 종류별 코드
```

📘 **비유하자면:**  
- `init/`은 입국심사대,  
- `mm/`은 메모리 행성,  
- `fs/`는 데이터의 바다,  
- `drivers/`는 하드웨어의 정글입니다.

> 💬 **GPT의 코멘트:**  
> “이 지도를 익히면,  
> 나중에 특정 기능이 ‘어디에 살고 있는지’  
> 직감적으로 찾을 수 있게 됩니다.”

---
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
