# 제 3장: 메모리 관리

## Memory Management

### 메모리 관리 개요

리눅스 커널의 메모리 관리 서브시스템은 물리 메모리와 가상 메모리를 효율적으로 관리합니다.

### 주요 개념

#### 1. 가상 메모리 (Virtual Memory)

각 프로세스는 독립적인 가상 주소 공간을 가집니다.

```
가상 주소 공간 (64-bit 시스템)
┌─────────────────────────────┐ 0xFFFFFFFFFFFFFFFF
│     Kernel Space            │
│     (커널 영역)              │
├─────────────────────────────┤ 0x00007FFFFFFFFFFF
│     Stack                   │ (스택)
│         ↓                   │
│                             │
│     (Free Space)            │
│                             │
│         ↑                   │
│     Heap                    │ (힙)
├─────────────────────────────┤
│     BSS Segment             │ (초기화되지 않은 데이터)
├─────────────────────────────┤
│     Data Segment            │ (초기화된 데이터)
├─────────────────────────────┤
│     Text Segment            │ (코드)
└─────────────────────────────┘ 0x0000000000000000
```

#### 2. 페이징 (Paging)

메모리를 고정 크기의 페이지로 나누어 관리합니다.

**일반적인 페이지 크기**:
- x86: 4KB
- x86-64: 4KB (기본), 2MB, 1GB (huge pages)
- ARM: 4KB, 64KB

```c
// include/linux/mm_types.h

struct page {
    unsigned long flags;          // 페이지 플래그
    atomic_t _refcount;          // 참조 카운트
    atomic_t _mapcount;          // 매핑 카운트
    unsigned long private;        // 사용자 데이터
    struct address_space *mapping; // 페이지 소유자
    pgoff_t index;               // 오프셋
    struct list_head lru;        // LRU 리스트
    // ...
};
```

### 메모리 디스크립터 (mm_struct)

각 프로세스의 메모리 레이아웃을 기술합니다.

```c
// include/linux/mm_types.h

struct mm_struct {
    struct vm_area_struct *mmap;     // VMA 리스트
    struct rb_root mm_rb;            // VMA Red-Black Tree
    
    unsigned long start_code, end_code;      // 코드 영역
    unsigned long start_data, end_data;      // 데이터 영역
    unsigned long start_brk, brk;            // 힙 영역
    unsigned long start_stack;               // 스택 시작
    unsigned long arg_start, arg_end;        // 인자 영역
    unsigned long env_start, env_end;        // 환경 변수 영역
    
    unsigned long total_vm;          // 총 페이지 수
    unsigned long locked_vm;         // 잠긴 페이지 수
    unsigned long pinned_vm;         // 핀된 페이지 수
    
    pgd_t *pgd;                      // 페이지 전역 디렉토리
    
    atomic_t mm_users;               // 사용자 수
    atomic_t mm_count;               // 참조 카운트
    
    // ...
};
```

### VMA (Virtual Memory Area)

프로세스의 가상 메모리 영역을 나타냅니다.

```c
// include/linux/mm_types.h

struct vm_area_struct {
    unsigned long vm_start;          // VMA 시작 주소
    unsigned long vm_end;            // VMA 끝 주소
    
    struct vm_area_struct *vm_next;  // 다음 VMA
    struct vm_area_struct *vm_prev;  // 이전 VMA
    
    struct rb_node vm_rb;            // Red-Black Tree 노드
    
    struct mm_struct *vm_mm;         // 소속 mm_struct
    
    pgprot_t vm_page_prot;          // 접근 권한
    unsigned long vm_flags;          // 플래그
    
    struct file *vm_file;            // 매핑된 파일
    unsigned long vm_pgoff;          // 파일 오프셋
    
    const struct vm_operations_struct *vm_ops;  // 연산
    
    // ...
};
```

**VMA 플래그 예제**:
```c
#define VM_READ     0x00000001  // 읽기 가능
#define VM_WRITE    0x00000002  // 쓰기 가능
#define VM_EXEC     0x00000004  // 실행 가능
#define VM_SHARED   0x00000008  // 공유 매핑
#define VM_GROWSDOWN 0x00000100 // 스택 (아래로 성장)
#define VM_GROWSUP  0x00000200  // 힙 (위로 성장)
```

### 페이지 테이블

가상 주소를 물리 주소로 변환하는 구조입니다.

#### 4-Level 페이지 테이블 (x86-64)

```
Virtual Address (64-bit)
┌────┬────┬────┬────┬────────────┐
│PGD │PUD │PMD │PTE │   Offset   │
└────┴────┴────┴────┴────────────┘
  9     9    9    9       12 bits

변환 과정:
1. PGD (Page Global Directory)
2. PUD (Page Upper Directory)  
3. PMD (Page Middle Directory)
4. PTE (Page Table Entry)
5. Physical Address = PTE + Offset
```

```c
// 페이지 테이블 워킹 예제
pgd_t *pgd;
pud_t *pud;
pmd_t *pmd;
pte_t *pte;

// 1. PGD 엔트리 가져오기
pgd = pgd_offset(mm, address);
if (pgd_none(*pgd) || pgd_bad(*pgd))
    return NULL;

// 2. PUD 엔트리 가져오기
pud = pud_offset(pgd, address);
if (pud_none(*pud) || pud_bad(*pud))
    return NULL;

// 3. PMD 엔트리 가져오기
pmd = pmd_offset(pud, address);
if (pmd_none(*pmd) || pmd_bad(*pmd))
    return NULL;

// 4. PTE 엔트리 가져오기
pte = pte_offset_map(pmd, address);
```

### 페이지 할당자

커널은 여러 페이지 할당 메커니즘을 제공합니다.

#### 1. Buddy System

2의 거듭제곱 크기로 메모리를 할당합니다.

```c
// mm/page_alloc.c

// 페이지 할당
struct page *alloc_pages(gfp_t gfp_mask, unsigned int order)
{
    return alloc_pages_current(gfp_mask, order);
}

// 단일 페이지 할당
#define alloc_page(gfp_mask) alloc_pages(gfp_mask, 0)

// 페이지 해제
void __free_pages(struct page *page, unsigned int order);

#define __free_page(page) __free_pages((page), 0)
```

**GFP 플래그**:
```c
#define GFP_KERNEL    // 커널 내부 할당 (슬립 가능)
#define GFP_ATOMIC    // 원자적 할당 (슬립 불가)
#define GFP_USER      // 사용자 공간 할당
#define GFP_HIGHUSER  // 상위 메모리 사용자 할당
#define GFP_DMA       // DMA 가능 메모리
```

#### 2. Slab Allocator

작은 객체를 효율적으로 할당합니다.

```c
// include/linux/slab.h

// 캐시 생성
struct kmem_cache *kmem_cache_create(
    const char *name,       // 캐시 이름
    size_t size,           // 객체 크기
    size_t align,          // 정렬
    unsigned long flags,   // 플래그
    void (*ctor)(void *)   // 생성자
);

// 객체 할당
void *kmem_cache_alloc(struct kmem_cache *cachep, gfp_t flags);

// 객체 해제
void kmem_cache_free(struct kmem_cache *cachep, void *objp);

// 캐시 파괴
void kmem_cache_destroy(struct kmem_cache *cachep);
```

**사용 예제**:
```c
// 캐시 생성
struct kmem_cache *my_cache;
my_cache = kmem_cache_create("my_cache", 
                             sizeof(struct my_struct),
                             0, SLAB_HWCACHE_ALIGN, NULL);

// 객체 할당
struct my_struct *obj;
obj = kmem_cache_alloc(my_cache, GFP_KERNEL);

// 사용...

// 객체 해제
kmem_cache_free(my_cache, obj);

// 캐시 파괴
kmem_cache_destroy(my_cache);
```

#### 3. kmalloc/kfree

일반적인 커널 메모리 할당입니다.

```c
// 메모리 할당
void *kmalloc(size_t size, gfp_t flags);

// 0으로 초기화된 메모리 할당
void *kzalloc(size_t size, gfp_t flags);

// 메모리 해제
void kfree(const void *objp);
```

### 페이지 폴트 (Page Fault)

프로세스가 매핑되지 않은 메모리에 접근하면 페이지 폴트가 발생합니다.

```c
// mm/memory.c

/*
 * 페이지 폴트 핸들러
 */
vm_fault_t handle_mm_fault(struct vm_area_struct *vma,
                          unsigned long address,
                          unsigned int flags)
{
    vm_fault_t ret;
    
    // VMA가 유효한지 확인
    if (unlikely(is_vm_hugetlb_page(vma)))
        ret = hugetlb_fault(vma->vm_mm, vma, address, flags);
    else
        ret = __handle_mm_fault(vma, address, flags);
    
    return ret;
}

static vm_fault_t __handle_mm_fault(struct vm_area_struct *vma,
                                    unsigned long address,
                                    unsigned int flags)
{
    struct vm_fault vmf = {
        .vma = vma,
        .address = address & PAGE_MASK,
        .flags = flags,
    };
    
    // 1. 페이지 테이블 엔트리 찾기
    // 2. 페이지가 없으면 할당
    // 3. 페이지 테이블 업데이트
    // 4. TLB 무효화
    
    return handle_pte_fault(&vmf);
}
```

### COW (Copy-on-Write)

fork() 시 메모리를 즉시 복사하지 않고, 쓰기 시에만 복사합니다.

```c
/*
 * COW 페이지 폴트 처리
 */
static vm_fault_t do_wp_page(struct vm_fault *vmf)
{
    struct vm_area_struct *vma = vmf->vma;
    
    // 1. 페이지가 공유되는지 확인
    if (page_mapcount(vmf->page) == 1) {
        // 단독 소유: 그냥 쓰기 가능하게 변경
        wp_page_reuse(vmf);
        return VM_FAULT_WRITE;
    }
    
    // 2. 공유 중: 새 페이지 할당
    return wp_page_copy(vmf);
}
```

**COW 예제**:
```c
#include <stdio.h>
#include <unistd.h>
#include <sys/mman.h>

int main() {
    int *shared = mmap(NULL, sizeof(int),
                      PROT_READ | PROT_WRITE,
                      MAP_SHARED | MAP_ANONYMOUS, -1, 0);
    
    *shared = 100;
    
    if (fork() == 0) {
        // 자식 프로세스
        printf("Child before write: %d\n", *shared);
        *shared = 200;  // COW 발생
        printf("Child after write: %d\n", *shared);
    } else {
        // 부모 프로세스
        sleep(1);
        printf("Parent: %d\n", *shared);
    }
    
    munmap(shared, sizeof(int));
    return 0;
}
```

### mmap() 시스템 콜

파일이나 장치를 메모리에 매핑합니다.

```c
#include <sys/mman.h>

void *mmap(void *addr,      // 시작 주소 (NULL = 커널이 선택)
           size_t length,   // 길이
           int prot,        // 보호 플래그
           int flags,       // 매핑 플래그
           int fd,          // 파일 디스크립터
           off_t offset);   // 파일 오프셋

int munmap(void *addr, size_t length);
```

**prot 플래그**:
- `PROT_READ`: 읽기 가능
- `PROT_WRITE`: 쓰기 가능
- `PROT_EXEC`: 실행 가능
- `PROT_NONE`: 접근 불가

**flags**:
- `MAP_SHARED`: 공유 매핑
- `MAP_PRIVATE`: 사적 매핑 (COW)
- `MAP_ANONYMOUS`: 익명 매핑
- `MAP_FIXED`: 고정 주소

**사용 예제**:
```c
#include <stdio.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <sys/stat.h>

int main() {
    int fd;
    struct stat sb;
    char *mapped;
    
    // 파일 열기
    fd = open("example.txt", O_RDONLY);
    fstat(fd, &sb);
    
    // 메모리 매핑
    mapped = mmap(NULL, sb.st_size, PROT_READ,
                  MAP_PRIVATE, fd, 0);
    
    // 파일 내용 읽기 (메모리처럼)
    printf("%.*s\n", (int)sb.st_size, mapped);
    
    // 언매핑
    munmap(mapped, sb.st_size);
    close(fd);
    
    return 0;
}
```

### 스왑 (Swap)

물리 메모리가 부족할 때 디스크를 사용합니다.

```bash
# 스왑 정보 확인
free -h
swapon --show

# 스왑 추가
sudo fallocate -l 1G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# 스왑 제거
sudo swapoff /swapfile
```

### 메모리 정보 확인

```bash
# 전체 메모리 상태
cat /proc/meminfo

# 특정 프로세스의 메모리 사용량
cat /proc/[PID]/status | grep -i vm
cat /proc/[PID]/smaps

# 메모리 맵 확인
cat /proc/[PID]/maps

# 시스템 전체 메모리 사용량
vmstat 1
```

### Huge Pages

큰 페이지를 사용하여 TLB 미스를 줄입니다.

```bash
# Huge pages 설정
echo 20 > /proc/sys/vm/nr_hugepages

# Huge pages 정보
cat /proc/meminfo | grep -i huge

# Transparent Huge Pages (THP)
cat /sys/kernel/mm/transparent_hugepage/enabled
```

**프로그램에서 사용**:
```c
#include <sys/mman.h>

void *addr = mmap(NULL, 2 * 1024 * 1024,  // 2MB
                  PROT_READ | PROT_WRITE,
                  MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB,
                  -1, 0);
```

### GPT 활용 예제

1. **페이지 테이블 워킹 이해**:
   ```
   "x86-64에서 가상 주소 0x7fffffffe000을 물리 주소로 변환하는 
   과정을 단계별로 설명해주세요"
   ```

2. **메모리 할당 비교**:
   ```
   "kmalloc, vmalloc, __get_free_pages의 차이점과 각각을 
   언제 사용해야 하는지 설명해주세요"
   ```

3. **COW 메커니즘**:
   ```
   "리눅스의 Copy-on-Write 메커니즘이 fork()와 어떻게 
   연동되어 메모리를 절약하는지 예제와 함께 설명해주세요"
   ```

### 실습 과제

1. mmap()을 사용하여 파일을 메모리에 매핑하고 읽는 프로그램 작성
2. fork() 후 COW 동작을 확인하는 프로그램 작성
3. /proc/[PID]/maps를 파싱하여 프로세스의 메모리 레이아웃 시각화
4. 공유 메모리를 사용하는 간단한 IPC 프로그램 작성
5. kmalloc과 vmalloc을 사용하는 간단한 커널 모듈 작성

### 다음 장에서는...

다음 장에서는 파일 시스템을 자세히 살펴보고, VFS와 실제 파일 시스템 구현을 알아봅니다.

---

[← 이전: 프로세스 관리](02-process-management.md) | [다음: 파일 시스템 →](04-file-systems.md)
