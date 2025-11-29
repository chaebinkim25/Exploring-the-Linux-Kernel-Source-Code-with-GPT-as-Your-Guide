# GPT와 함께 떠나는 리눅스 커널 소스 코드 여행

## Exploring the Linux Kernel Source Code with GPT as Your Guide

프로세스, 메모리, 스케줄러, 그리고 커널의 심장 속으로

AI와 인간이 함께 푸는 커널의 비밀

리눅스 커널을 이해하고 싶은 개발자들을 위한 전자책입니다. GPT를 활용하여 복잡한 커널 코드를 효율적으로 학습하는 방법을 제시합니다.

[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)

## 📚 전자책 내용

이 전자책은 리눅스 커널의 핵심 서브시스템을 체계적으로 다룹니다:

1. **[여행을 떠나기 전에](docs/00-introduction.md)** - 커널 탐험의 준비
   - 리눅스 커널은 무엇인가
   - 운영체제의 구조 복습: 유저 모드 vs 커널 모드
   - 커널 소스 트리 한눈에 보기 (`/init`, `/kernel`, `/mm`, `/fs`, `/drivers`)
   - 빌드 환경 구축 (WSL, Ubuntu, QEMU, gdb)
   - "Hello Kernel!": 커널 모듈 첫 빌드 & 로드

2. **[프로세스의 비다](docs/02-process-management.md)** - `task_struct`의 항해
   - 프로세스 생성: `fork()`, `clone()`, `copy_process()`
   - 스케줄링의 비밀: CFS (Completely Fair Scheduler)
   - 컨텍스트 스위칭, runqueue, tickless cheduling
   - 프로세스 종료와 시그널 처리
   - `ps`, `top`, `/proc`을 통한 내부 구조 관찰
     
3. **[메모리 행성](docs/03-memory-management.md)** - `mm_struct`와 페이지의 세계
   - 가상 메모리의 구조와 페이지 테이블
   - 물리 메모리 관리: buddy allocator
   - slab/slub/slob allocator 비교
   - demand paging과 page fault 처리
   - `vm_area_struct` 실제 분석
     
4. **[파일 시스템 은하](docs/04-file-systems.md)** - `inode`와 `dentry` 탐사
   - `inode` 구조체와 파일의 정체
   - `ext4` 내부 동작
   - VFS(Virtual File System) 추상화 구조
   - 시스템 콜 흐름 따라가기 (`open() → sys_open() → vfs_open()`)

5. **[장치 드라이버의 정글](docs/05-device-drivers.md)** - 하드웨어로 향하는 길
   - 문자 장치 vs 블록 장치
   - `file_operations`와 커널 인터페이스
   - `/dev`와 `udev`의 비밀
   - 간단한 커널 모듈 드라이버 작성
   - IRQ와 인터럽트 핸들링의 흐름

6. **[커널의 심장](docs/06-schedular.md)** - 스케줄러, 인터럽트, 동기화
   - 인터럽트 컨텍스트 vs 프로세스 컨텍스트
   - softirq, tasklet, workqueue
   - 스핀락, 세마포어, mutex
   - preemption과 SMP 환경의 도전
   - RCU(Read-Copy-Update)와 최신 커널 병렬화 기법
  
7. **커널 디버깅과 해킹** - 커널을 "읽는" 기술
   - `printk()`, `dmesg`, `ftrace`, `perf`, `kprobe`
   - gdb + QEMU로 커널 내부 탐험
   - crash dump 분석 (`/proc/kcore`)
   - CVE 사례로 보는 커널 취약점 이해
   - "리눅스 커널 워게임" 미션 (실습형 챕터)

8. **우주 끝의 커널** - 최신 기술과 철학
   - eBPF, io_uring, Rust for Linux
   - LTS vs mainline 커널의 생태
   - Linus의 커밋 철학과 커뮤니티 운영
   - 커널 해커들의 개발 문화
   - "커널을 이해하는 법" - 인간과 AI의 협업

9. **[네트워킹](docs/06-networking.md)** - 네트워크 스택과 소켓 프로그래밍
10. **[결론 및 참고 자료](docs/07-conclusion.md)** - 추가 학습 자료와 경력 개발

📖 **[전자책 읽기 시작하기 →](docs/index.md)**

## 🎯 대상 독자

- 리눅스 커널에 관심 있는 개발자
- 시스템 프로그래밍을 배우고 싶은 학생
- 오픈소스 기여를 준비하는 개발자
- 운영체제 내부 동작을 이해하고 싶은 엔지니어

## 💡 특징

- **실제 코드 중심**: 리눅스 커널의 실제 소스 코드를 분석합니다
- **GPT 활용 가이드**: GPT를 활용하여 효율적으로 학습하는 방법을 제시합니다
- **실습 과제**: 각 장마다 실습 과제를 포함합니다
- **한글/영어 병행**: 한글로 작성되었으며 주요 용어는 영어를 병기합니다

## 🚀 시작하기

### 사전 요구 사항

- C 프로그래밍 언어에 대한 기본 이해
- 리눅스 명령어에 대한 기본 지식
- Git 사용 경험

### 개발 환경 설정

```bash
# 필수 패키지 설치 (Ubuntu/Debian)
sudo apt-get update
sudo apt-get install build-essential libncurses-dev bison flex libssl-dev libelf-dev

# 리눅스 커널 소스 다운로드
git clone https://github.com/torvalds/linux.git
cd linux

# 커널 버전 확인
git tag -l | tail
```

## 📖 사용 방법

1. **순차적 학습**: [전자책](docs/index.md)의 장을 순서대로 읽으며 학습합니다
2. **실습**: 각 장의 예제 코드를 직접 실행하고 수정해봅니다
3. **GPT 활용**: 이해가 안 되는 부분은 GPT에게 질문합니다
4. **커뮤니티 참여**: GitHub Issues를 통해 질문하고 토론합니다

## 🤝 기여하기

이 프로젝트는 오픈소스입니다! 기여를 환영합니다.

1. 이 저장소를 Fork 합니다
2. 새로운 브랜치를 생성합니다 (`git checkout -b feature/amazing-feature`)
3. 변경 사항을 커밋합니다 (`git commit -m 'Add some amazing feature'`)
4. 브랜치에 푸시합니다 (`git push origin feature/amazing-feature`)
5. Pull Request를 생성합니다

### 기여 가이드라인

- 오타나 문법 오류 수정
- 예제 코드 개선
- 새로운 내용 추가
- 번역 개선
- 실습 과제 추가

## 📝 라이선스

이 프로젝트는 GNU General Public License v3.0 라이선스를 따릅니다. 자세한 내용은 [LICENSE](LICENSE) 파일을 참조하세요.

## 🔗 참고 자료

- [Linux Kernel Documentation](https://www.kernel.org/doc/html/latest/)
- [Kernel Newbies](https://kernelnewbies.org/)
- [LWN.net](https://lwn.net/)
- [Bootlin Elixir](https://elixir.bootlin.com/linux/latest/source)

## 📬 연락처

질문이나 제안이 있으시면 [GitHub Issues](https://github.com/chaebinkim25/Exploring-the-Linux-Kernel-Source-Code-with-GPT-as-Your-Guide/issues)를 통해 연락해주세요.

## ⭐ 지원하기

이 프로젝트가 도움이 되셨다면 ⭐️ 스타를 눌러주세요!

---

**Happy Kernel Hacking! 🐧**
