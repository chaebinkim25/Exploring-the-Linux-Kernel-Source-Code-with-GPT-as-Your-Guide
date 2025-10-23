# 제 2장: 프로세스 관리

## Process Management

### 프로세스란?

프로세스는 실행 중인 프로그램의 인스턴스입니다. 리눅스 커널에서 프로세스는 `task_struct` 구조체로 표현됩니다.

### 프로세스 상태

리눅스에서 프로세스는 다음과 같은 상태를 가질 수 있습니다:

```c
// include/linux/sched.h
#define TASK_RUNNING            0  // 실행 중 또는 실행 대기
#define TASK_INTERRUPTIBLE      1  // 인터럽트 가능한 대기
#define TASK_UNINTERRUPTIBLE    2  // 인터럽트 불가능한 대기
#define __TASK_STOPPED          4  // 정지됨
#define __TASK_TRACED           8  // 추적 중
#define EXIT_ZOMBIE            16  // 좀비 프로세스
#define EXIT_DEAD              32  // 종료된 프로세스
```

**프로세스 상태 다이어그램**:
```
     [생성]
       ↓
   TASK_RUNNING ←→ TASK_INTERRUPTIBLE
       ↓                   ↓
   [실행 중]         TASK_UNINTERRUPTIBLE
       ↓                   ↓
   EXIT_ZOMBIE    ←───────┘
       ↓
   EXIT_DEAD
```

### task_struct 구조체

프로세스를 나타내는 핵심 데이터 구조입니다.

```c
// include/linux/sched.h (간략화된 버전)
struct task_struct {
    /* 프로세스 상태 */
    volatile long state;
    
    /* 스케줄링 정보 */
    int prio;                    // 우선순위
    int static_prio;             // 정적 우선순위
    int normal_prio;             // 일반 우선순위
    unsigned int policy;         // 스케줄링 정책
    
    /* 프로세스 식별자 */
    pid_t pid;                   // 프로세스 ID
    pid_t tgid;                  // 스레드 그룹 ID
    
    /* 프로세스 관계 */
    struct task_struct *parent;  // 부모 프로세스
    struct list_head children;   // 자식 프로세스 리스트
    struct list_head sibling;    // 형제 프로세스 리스트
    
    /* 메모리 관리 */
    struct mm_struct *mm;        // 메모리 디스크립터
    struct mm_struct *active_mm;
    
    /* 파일 시스템 정보 */
    struct fs_struct *fs;        // 파일 시스템 정보
    struct files_struct *files;  // 열린 파일 테이블
    
    /* 시그널 처리 */
    struct signal_struct *signal;
    struct sighand_struct *sighand;
    
    /* CPU 시간 */
    u64 utime;                   // 유저 모드 시간
    u64 stime;                   // 커널 모드 시간
    
    /* 스택 */
    void *stack;                 // 커널 스택
    
    /* 실행 정보 */
    struct thread_struct thread; // 아키텍처 특정 정보
    
    // ... 더 많은 필드들
};
```

### 프로세스 생성

#### fork() 시스템 콜

새로운 프로세스를 생성하는 가장 기본적인 방법입니다.

**사용자 공간 코드**:
```c
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>

int main() {
    pid_t pid = fork();
    
    if (pid < 0) {
        // 에러 발생
        perror("fork failed");
        return 1;
    } else if (pid == 0) {
        // 자식 프로세스
        printf("Child process: PID=%d, Parent PID=%d\n", 
               getpid(), getppid());
    } else {
        // 부모 프로세스
        printf("Parent process: PID=%d, Child PID=%d\n", 
               getpid(), pid);
    }
    
    return 0;
}
```

**커널 내부 구현 (간략화)**:
```c
// kernel/fork.c

SYSCALL_DEFINE0(fork)
{
    return _do_fork(SIGCHLD, 0, 0, NULL, NULL, 0);
}

/*
 * _do_fork() - 실제 프로세스 생성 작업
 */
long _do_fork(unsigned long clone_flags,
              unsigned long stack_start,
              unsigned long stack_size,
              int __user *parent_tidptr,
              int __user *child_tidptr,
              unsigned long tls)
{
    struct task_struct *p;
    int trace = 0;
    long nr;
    
    // 1. task_struct 복사
    p = copy_process(clone_flags, stack_start, stack_size,
                     parent_tidptr, child_tidptr, NULL, trace, tls, NUMA_NO_NODE);
    
    if (!IS_ERR(p)) {
        // 2. PID 할당
        struct pid *pid;
        pid = get_task_pid(p, PIDTYPE_PID);
        nr = pid_vnr(pid);
        
        // 3. 자식 프로세스 깨우기
        wake_up_new_task(p);
        
        put_pid(pid);
    }
    
    return nr;
}
```

#### copy_process() 함수

`copy_process()` 함수는 부모 프로세스를 복사하여 새로운 프로세스를 만듭니다:

```c
static struct task_struct *copy_process(...)
{
    struct task_struct *p;
    
    // 1. task_struct 할당
    p = dup_task_struct(current);
    
    // 2. 자식 프로세스 초기화
    // - 상태를 TASK_RUNNING으로 설정
    // - PID 할당
    // - 부모-자식 관계 설정
    
    // 3. 리소스 복사
    // - 메모리 공간 (copy_mm)
    // - 파일 디스크립터 (copy_files)
    // - 시그널 핸들러 (copy_sighand)
    // - 네임스페이스 (copy_namespaces)
    
    // 4. 스케줄러 초기화
    sched_fork(p);
    
    return p;
}
```

### 프로세스 스케줄링

리눅스 커널은 CFS(Completely Fair Scheduler)를 사용합니다.

#### CFS (Completely Fair Scheduler)

CFS는 모든 프로세스에게 공정한 CPU 시간을 제공하는 것을 목표로 합니다.

**핵심 개념**:
- **vruntime**: 가상 실행 시간 (virtual runtime)
- **Red-Black Tree**: vruntime을 기준으로 프로세스를 정렬
- **nice 값**: 프로세스 우선순위 (-20 ~ +19)

```c
// kernel/sched/fair.c

struct cfs_rq {
    struct rb_root_cached tasks_timeline;  // Red-Black Tree
    struct rb_node *rb_leftmost;           // 최소 vruntime 노드
    unsigned int nr_running;               // 실행 가능한 태스크 수
    u64 min_vruntime;                      // 최소 vruntime
};

struct sched_entity {
    struct load_weight load;
    struct rb_node run_node;
    u64 vruntime;                          // 가상 실행 시간
    u64 sum_exec_runtime;                  // 총 실행 시간
    u64 prev_sum_exec_runtime;
};
```

#### 스케줄링 알고리즘

```c
// 다음에 실행할 프로세스 선택
static struct task_struct *pick_next_task_fair(struct rq *rq)
{
    struct sched_entity *se;
    struct cfs_rq *cfs_rq = &rq->cfs;
    
    // Red-Black Tree의 최좌측 노드 (최소 vruntime)
    se = __pick_first_entity(cfs_rq);
    
    return task_of(se);
}

// vruntime 업데이트
static void update_curr(struct cfs_rq *cfs_rq)
{
    struct sched_entity *curr = cfs_rq->curr;
    u64 now = rq_clock_task(rq_of(cfs_rq));
    u64 delta_exec;
    
    delta_exec = now - curr->exec_start;
    
    // vruntime 증가
    curr->vruntime += calc_delta_fair(delta_exec, curr);
}
```

### 프로세스 간 통신 (IPC)

#### 1. 파이프 (Pipe)

```c
#include <stdio.h>
#include <unistd.h>
#include <string.h>

int main() {
    int pipefd[2];
    char buffer[100];
    
    pipe(pipefd);  // 파이프 생성
    
    if (fork() == 0) {
        // 자식 프로세스: 읽기
        close(pipefd[1]);
        read(pipefd[0], buffer, sizeof(buffer));
        printf("Child received: %s\n", buffer);
        close(pipefd[0]);
    } else {
        // 부모 프로세스: 쓰기
        close(pipefd[0]);
        const char *msg = "Hello from parent!";
        write(pipefd[1], msg, strlen(msg) + 1);
        close(pipefd[1]);
    }
    
    return 0;
}
```

#### 2. 시그널 (Signal)

```c
#include <stdio.h>
#include <signal.h>
#include <unistd.h>

void signal_handler(int signum) {
    printf("Received signal %d\n", signum);
}

int main() {
    // 시그널 핸들러 등록
    signal(SIGINT, signal_handler);
    
    printf("Press Ctrl+C to send SIGINT\n");
    
    while (1) {
        sleep(1);
    }
    
    return 0;
}
```

#### 3. 공유 메모리 (Shared Memory)

```c
#include <stdio.h>
#include <sys/shm.h>
#include <string.h>

int main() {
    key_t key = 1234;
    int shmid;
    char *data;
    
    // 공유 메모리 생성
    shmid = shmget(key, 1024, 0644 | IPC_CREAT);
    
    // 공유 메모리 연결
    data = shmat(shmid, NULL, 0);
    
    // 데이터 쓰기
    strcpy(data, "Hello, Shared Memory!");
    
    // 공유 메모리 분리
    shmdt(data);
    
    return 0;
}
```

### 프로세스 종료

#### exit() 시스템 콜

```c
// kernel/exit.c

SYSCALL_DEFINE1(exit, int, error_code)
{
    do_exit((error_code & 0xff) << 8);
}

void do_exit(long code)
{
    struct task_struct *tsk = current;
    
    // 1. 리소스 해제
    exit_mm(tsk);        // 메모리 해제
    exit_files(tsk);     // 파일 디스크립터 닫기
    exit_fs(tsk);        // 파일 시스템 정보 해제
    
    // 2. 자식 프로세스 재부모화 (reparenting)
    exit_notify(tsk, group_dead);
    
    // 3. 상태를 EXIT_ZOMBIE로 변경
    tsk->exit_state = EXIT_ZOMBIE;
    
    // 4. 스케줄러 호출 (이 프로세스는 다시 실행되지 않음)
    schedule();
}
```

### 프로세스 모니터링

#### /proc 파일 시스템

```bash
# 프로세스 정보 확인
cat /proc/[PID]/status

# 프로세스의 메모리 맵
cat /proc/[PID]/maps

# 프로세스의 열린 파일
ls -l /proc/[PID]/fd

# 프로세스의 명령행
cat /proc/[PID]/cmdline
```

#### ps 명령어

```bash
# 모든 프로세스 표시
ps aux

# 프로세스 트리 표시
ps auxf

# 특정 프로세스 정보
ps -p [PID] -o pid,ppid,cmd,stat,pri,nice
```

### GPT 활용 예제

1. **프로세스 생성 과정 이해**:
   ```
   "리눅스에서 fork() 시스템 콜이 호출될 때 커널 내부에서 일어나는 
   일을 단계별로 자세히 설명해주세요"
   ```

2. **스케줄러 분석**:
   ```
   "CFS 스케줄러의 vruntime 계산 방식을 예제와 함께 설명해주세요"
   ```

3. **디버깅 도움**:
   ```
   "fork() 후 자식 프로세스가 부모의 파일 디스크립터를 상속받는 
   메커니즘을 코드와 함께 설명해주세요"
   ```

### 실습 과제

1. fork()를 사용하여 여러 자식 프로세스를 생성하는 프로그램 작성
2. pipe()를 사용하여 부모-자식 간 통신하는 프로그램 작성
3. 시그널 핸들러를 구현하여 SIGINT, SIGTERM 처리
4. /proc 파일 시스템을 탐색하여 프로세스 정보 수집
5. nice 명령어로 프로세스 우선순위 변경 실험

### 다음 장에서는...

다음 장에서는 메모리 관리를 자세히 살펴보고, 가상 메모리와 페이징 시스템이 어떻게 동작하는지 알아봅니다.

---

[← 이전: 커널 아키텍처 개요](01-kernel-architecture.md) | [다음: 메모리 관리 →](03-memory-management.md)
