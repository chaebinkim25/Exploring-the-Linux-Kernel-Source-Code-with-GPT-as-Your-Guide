# 제 1장: 커널 아키텍처 개요

## Kernel Architecture Overview

### 리눅스 커널 아키텍처

리눅스 커널은 모놀리식(Monolithic) 아키텍처를 기반으로 하지만, 모듈화를 통해 유연성을 제공합니다.

```
┌─────────────────────────────────────────────────────────┐
│                    User Space                            │
│  (Applications, Libraries, Shell, etc.)                  │
└─────────────────────────────────────────────────────────┘
                          │
                   System Calls
                          │
┌─────────────────────────────────────────────────────────┐
│                    Kernel Space                          │
│  ┌───────────────────────────────────────────────────┐  │
│  │         System Call Interface                      │  │
│  └───────────────────────────────────────────────────┘  │
│  ┌───────────┬───────────┬───────────┬───────────────┐  │
│  │  Process  │  Memory   │   File    │    Network    │  │
│  │Management │Management │  System   │     Stack     │  │
│  └───────────┴───────────┴───────────┴───────────────┘  │
│  ┌───────────────────────────────────────────────────┐  │
│  │           Device Drivers                          │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
                          │
┌─────────────────────────────────────────────────────────┐
│                     Hardware                             │
│  (CPU, Memory, Disk, Network, etc.)                      │
└─────────────────────────────────────────────────────────┘
```

### 커널의 주요 서브시스템

#### 1. 프로세스 관리 (Process Management)

프로세스의 생성, 스케줄링, 종료를 관리합니다.

**핵심 파일 위치**:
- `kernel/sched/`: 스케줄러 코드
- `kernel/fork.c`: 프로세스 생성
- `kernel/exit.c`: 프로세스 종료

**주요 구조체**:
```c
// include/linux/sched.h
struct task_struct {
    volatile long state;    // 프로세스 상태
    void *stack;           // 커널 스택
    int prio;              // 우선순위
    struct mm_struct *mm;  // 메모리 디스크립터
    struct files_struct *files;  // 파일 디스크립터
    // ... 더 많은 필드들
};
```

#### 2. 메모리 관리 (Memory Management)

물리 메모리와 가상 메모리를 관리합니다.

**핵심 파일 위치**:
- `mm/memory.c`: 메모리 관리 핵심 기능
- `mm/vmalloc.c`: 가상 메모리 할당
- `mm/page_alloc.c`: 페이지 할당자

**주요 개념**:
- 페이징(Paging)
- 가상 메모리(Virtual Memory)
- 메모리 맵핑(Memory Mapping)
- 스왑(Swap)

#### 3. 파일 시스템 (File System)

파일과 디렉토리 관리를 담당합니다.

**핵심 파일 위치**:
- `fs/`: 파일 시스템 구현
- `fs/ext4/`: ext4 파일 시스템
- `fs/proc/`: /proc 파일 시스템
- `fs/sysfs/`: /sys 파일 시스템

**VFS (Virtual File System)**:
```c
// include/linux/fs.h
struct file_operations {
    ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
    ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
    int (*open) (struct inode *, struct file *);
    int (*release) (struct inode *, struct file *);
    // ... 더 많은 연산들
};
```

#### 4. 장치 드라이버 (Device Drivers)

하드웨어 장치와의 통신을 담당합니다.

**핵심 파일 위치**:
- `drivers/`: 모든 장치 드라이버
- `drivers/char/`: 문자 장치 드라이버
- `drivers/block/`: 블록 장치 드라이버
- `drivers/net/`: 네트워크 장치 드라이버

**장치 분류**:
- **문자 장치(Character Device)**: 순차적 접근 (키보드, 마우스)
- **블록 장치(Block Device)**: 랜덤 접근 (하드디스크, SSD)
- **네트워크 장치(Network Device)**: 네트워크 인터페이스

#### 5. 네트워킹 스택 (Network Stack)

네트워크 프로토콜과 통신을 관리합니다.

**핵심 파일 위치**:
- `net/`: 네트워킹 코드
- `net/ipv4/`: IPv4 구현
- `net/ipv6/`: IPv6 구현
- `net/core/`: 네트워크 핵심 기능

**프로토콜 스택**:
```
Application Layer (응용 계층)
    ↓
Transport Layer (전송 계층: TCP/UDP)
    ↓
Network Layer (네트워크 계층: IP)
    ↓
Link Layer (링크 계층: Ethernet)
    ↓
Physical Layer (물리 계층: Hardware)
```

### 시스템 콜 인터페이스

사용자 공간과 커널 공간을 연결하는 인터페이스입니다.

**시스템 콜 예제**:
```c
// 사용자 공간에서
#include <unistd.h>
#include <sys/types.h>

pid_t pid = fork();  // 새 프로세스 생성
```

**커널 내부에서**:
```c
// kernel/fork.c
SYSCALL_DEFINE0(fork)
{
    return _do_fork(SIGCHLD, 0, 0, NULL, NULL, 0);
}
```

### 커널 모듈 시스템

리눅스 커널은 모듈을 통해 동적으로 기능을 추가하거나 제거할 수 있습니다.

**간단한 커널 모듈 예제**:
```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>

static int __init hello_init(void)
{
    printk(KERN_INFO "Hello, Kernel!\n");
    return 0;
}

static void __exit hello_exit(void)
{
    printk(KERN_INFO "Goodbye, Kernel!\n");
}

module_init(hello_init);
module_exit(hello_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Your Name");
MODULE_DESCRIPTION("A Simple Hello World Module");
```

**모듈 관리 명령어**:
```bash
# 모듈 로드
sudo insmod hello.ko

# 로드된 모듈 확인
lsmod | grep hello

# 모듈 언로드
sudo rmmod hello

# 모듈 정보 확인
modinfo hello.ko
```

### 커널 디버깅

#### printk를 사용한 디버깅

```c
printk(KERN_DEBUG "Debug message\n");
printk(KERN_INFO "Info message\n");
printk(KERN_WARNING "Warning message\n");
printk(KERN_ERR "Error message\n");
```

#### 로그 레벨
- `KERN_EMERG`: 시스템이 사용 불가능
- `KERN_ALERT`: 즉시 조치 필요
- `KERN_CRIT`: 치명적 상황
- `KERN_ERR`: 에러
- `KERN_WARNING`: 경고
- `KERN_NOTICE`: 정상이지만 중요한 상황
- `KERN_INFO`: 정보성 메시지
- `KERN_DEBUG`: 디버그 메시지

### GPT를 활용한 코드 분석 팁

1. **함수 흐름 이해하기**:
   ```
   "리눅스 커널의 do_fork() 함수가 어떻게 동작하는지 단계별로 설명해주세요"
   ```

2. **구조체 분석하기**:
   ```
   "task_struct 구조체의 주요 필드들과 그 역할을 설명해주세요"
   ```

3. **코드 찾기**:
   ```
   "리눅스 커널에서 프로세스 스케줄링을 담당하는 주요 함수들이 있는 파일 경로를 알려주세요"
   ```

### 실습 과제

1. 간단한 "Hello World" 커널 모듈을 작성하고 로드해보세요
2. `dmesg` 명령어로 커널 로그를 확인해보세요
3. `/proc` 디렉토리를 탐색하고 시스템 정보를 확인해보세요
4. GPT에게 관심 있는 커널 서브시스템에 대해 질문해보세요

### 다음 장에서는...

다음 장에서는 프로세스 관리를 자세히 살펴보고, 프로세스가 어떻게 생성되고 스케줄링되는지 알아봅니다.

---

[← 이전: 소개](00-introduction.md) | [다음: 프로세스 관리 →](02-process-management.md)
