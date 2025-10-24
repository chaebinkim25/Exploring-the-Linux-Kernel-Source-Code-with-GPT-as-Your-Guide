# 제 4장: 파일 시스템

## File Systems

### 파일 시스템 개요

리눅스 파일 시스템은 데이터를 계층적으로 저장하고 관리합니다. 리눅스는 다양한 파일 시스템을 지원하며, VFS(Virtual File System)를 통해 통합된 인터페이스를 제공합니다.

### VFS (Virtual File System)

VFS는 다양한 파일 시스템에 대한 추상화 계층을 제공합니다.

```
┌─────────────────────────────────────────┐
│        User Space Applications          │
└─────────────────────────────────────────┘
                  │
        (System Calls: open, read, write, etc.)
                  │
┌─────────────────────────────────────────┐
│       VFS (Virtual File System)         │
│   (공통 인터페이스 및 추상화)              │
└─────────────────────────────────────────┘
      │           │           │         │
┌──────────┐ ┌──────────┐ ┌──────┐ ┌──────┐
│   ext4   │ │   XFS    │ │ NFS  │ │ FAT  │
└──────────┘ └──────────┘ └──────┘ └──────┘
      │           │           │         │
└─────────────────────────────────────────┘
│          Block Device Layer             │
└─────────────────────────────────────────┘
```

### VFS의 주요 객체

#### 1. superblock (슈퍼블록)

파일 시스템 전체에 대한 정보를 담고 있습니다.

```c
// include/linux/fs.h

struct super_block {
    struct list_head s_list;           // 슈퍼블록 리스트
    dev_t s_dev;                       // 장치 식별자
    unsigned char s_blocksize_bits;    // 블록 크기 (비트)
    unsigned long s_blocksize;         // 블록 크기
    loff_t s_maxbytes;                // 최대 파일 크기
    
    struct file_system_type *s_type;   // 파일 시스템 타입
    const struct super_operations *s_op;  // 슈퍼블록 연산
    
    unsigned long s_flags;             // 마운트 플래그
    unsigned long s_magic;             // 매직 넘버
    
    struct dentry *s_root;             // 루트 디렉토리
    
    struct list_head s_inodes;         // inode 리스트
    struct list_head s_dirty;          // 더티 inode
    
    // ...
};

struct super_operations {
    struct inode *(*alloc_inode)(struct super_block *sb);
    void (*destroy_inode)(struct inode *);
    
    void (*dirty_inode)(struct inode *, int flags);
    int (*write_inode)(struct inode *, struct writeback_control *);
    void (*drop_inode)(struct inode *);
    void (*evict_inode)(struct inode *);
    
    void (*put_super)(struct super_block *);
    int (*sync_fs)(struct super_block *sb, int wait);
    int (*statfs)(struct dentry *, struct kstatfs *);
    
    // ...
};
```

#### 2. inode (아이노드)

파일의 메타데이터를 저장합니다.

```c
// include/linux/fs.h

struct inode {
    umode_t i_mode;              // 파일 타입과 권한
    unsigned short i_opflags;    // 연산 플래그
    kuid_t i_uid;               // 소유자 UID
    kgid_t i_gid;               // 그룹 GID
    unsigned int i_flags;       // 파일 시스템 플래그
    
    const struct inode_operations *i_op;     // inode 연산
    struct super_block *i_sb;                // 슈퍼블록
    struct address_space *i_mapping;         // 페이지 캐시
    
    unsigned long i_ino;        // inode 번호
    
    dev_t i_rdev;              // 장치 번호 (장치 파일용)
    loff_t i_size;             // 파일 크기
    
    struct timespec64 i_atime;  // 접근 시간
    struct timespec64 i_mtime;  // 수정 시간
    struct timespec64 i_ctime;  // 변경 시간
    
    unsigned short i_bytes;     // 사용 중인 바이트
    u8 i_blkbits;              // 블록 크기 비트
    blkcnt_t i_blocks;         // 블록 수
    
    // 파일 타입별 데이터
    union {
        struct pipe_inode_info *i_pipe;
        struct block_device *i_bdev;
        struct cdev *i_cdev;
        char *i_link;
        unsigned i_dir_seq;
    };
    
    // ...
};

struct inode_operations {
    struct dentry *(*lookup)(struct inode *, struct dentry *, unsigned int);
    int (*create)(struct inode *, struct dentry *, umode_t, bool);
    int (*link)(struct dentry *, struct inode *, struct dentry *);
    int (*unlink)(struct inode *, struct dentry *);
    int (*mkdir)(struct inode *, struct dentry *, umode_t);
    int (*rmdir)(struct inode *, struct dentry *);
    int (*rename)(struct inode *, struct dentry *,
                 struct inode *, struct dentry *, unsigned int);
    int (*setattr)(struct dentry *, struct iattr *);
    int (*getattr)(const struct path *, struct kstat *, u32, unsigned int);
    
    // ...
};
```

#### 3. dentry (디렉토리 엔트리)

파일 이름과 inode를 연결합니다.

```c
// include/linux/dcache.h

struct dentry {
    unsigned int d_flags;           // 플래그
    seqcount_t d_seq;              // 시퀀스 카운트
    struct hlist_bl_node d_hash;   // 해시 테이블
    struct dentry *d_parent;       // 부모 디렉토리
    struct qstr d_name;            // 파일 이름
    struct inode *d_inode;         // 연결된 inode
    
    unsigned char d_iname[DNAME_INLINE_LEN];  // 짧은 이름
    
    struct lockref d_lockref;      // 참조 카운트
    const struct dentry_operations *d_op;
    struct super_block *d_sb;      // 슈퍼블록
    
    union {
        struct list_head d_lru;    // LRU 리스트
        wait_queue_head_t *d_wait;
    };
    
    struct list_head d_child;      // 자식 리스트
    struct list_head d_subdirs;    // 서브디렉토리
    
    // ...
};

struct dentry_operations {
    int (*d_revalidate)(struct dentry *, unsigned int);
    int (*d_weak_revalidate)(struct dentry *, unsigned int);
    int (*d_hash)(const struct dentry *, struct qstr *);
    int (*d_compare)(const struct dentry *, unsigned int, const char *, const struct qstr *);
    int (*d_delete)(const struct dentry *);
    int (*d_init)(struct dentry *);
    void (*d_release)(struct dentry *);
    void (*d_iput)(struct dentry *, struct inode *);
    
    // ...
};
```

#### 4. file (파일 객체)

열린 파일을 나타냅니다.

```c
// include/linux/fs.h

struct file {
    union {
        struct llist_node fu_llist;
        struct rcu_head fu_rcuhead;
    } f_u;
    
    struct path f_path;                    // 경로
    struct inode *f_inode;                 // inode
    const struct file_operations *f_op;    // 파일 연산
    
    spinlock_t f_lock;
    atomic_long_t f_count;                 // 참조 카운트
    unsigned int f_flags;                  // 파일 플래그
    fmode_t f_mode;                        // 파일 모드
    
    loff_t f_pos;                         // 파일 위치
    
    struct fown_struct f_owner;
    const struct cred *f_cred;
    
    void *private_data;                    // 파일 시스템 전용 데이터
    
    // ...
};

struct file_operations {
    struct module *owner;
    loff_t (*llseek)(struct file *, loff_t, int);
    ssize_t (*read)(struct file *, char __user *, size_t, loff_t *);
    ssize_t (*write)(struct file *, const char __user *, size_t, loff_t *);
    ssize_t (*read_iter)(struct kiocb *, struct iov_iter *);
    ssize_t (*write_iter)(struct kiocb *, struct iov_iter *);
    
    int (*open)(struct inode *, struct file *);
    int (*release)(struct inode *, struct file *);
    int (*fsync)(struct file *, loff_t, loff_t, int datasync);
    
    long (*unlocked_ioctl)(struct file *, unsigned int, unsigned long);
    long (*compat_ioctl)(struct file *, unsigned int, unsigned long);
    
    int (*mmap)(struct file *, struct vm_area_struct *);
    
    // ...
};
```

### 파일 시스템 타입

#### ext4 파일 시스템

리눅스에서 가장 널리 사용되는 파일 시스템입니다.

**주요 특징**:
- 저널링 지원
- 대용량 파일 및 파일 시스템 지원
- Extent 기반 할당
- 온라인 defragmentation

```c
// fs/ext4/ext4.h

struct ext4_inode {
    __le16 i_mode;          // 파일 모드
    __le16 i_uid;          // 소유자 UID
    __le32 i_size_lo;      // 파일 크기 하위 32비트
    __le32 i_atime;        // 접근 시간
    __le32 i_ctime;        // 생성 시간
    __le32 i_mtime;        // 수정 시간
    __le32 i_dtime;        // 삭제 시간
    __le16 i_gid;          // 그룹 ID
    __le16 i_links_count;  // 하드 링크 수
    __le32 i_blocks_lo;    // 블록 수
    __le32 i_flags;        // 파일 플래그
    
    __le32 i_block[EXT4_N_BLOCKS];  // 블록 포인터
    
    // ...
};
```

**디스크 레이아웃**:
```
┌──────────────┐
│  Boot Block  │ (1024 bytes)
├──────────────┤
│  Superblock  │ (블록 그룹 0)
├──────────────┤
│ Group Desc.  │
├──────────────┤
│ Block Bitmap │
├──────────────┤
│ Inode Bitmap │
├──────────────┤
│ Inode Table  │
├──────────────┤
│ Data Blocks  │
├──────────────┤
│     ...      │ (블록 그룹 1, 2, ...)
└──────────────┘
```

### 파일 시스템 연산

#### 파일 열기

```c
// fs/open.c

SYSCALL_DEFINE3(open, const char __user *, filename,
                int, flags, umode_t, mode)
{
    return do_sys_open(AT_FDCWD, filename, flags, mode);
}

long do_sys_open(int dfd, const char __user *filename, 
                 int flags, umode_t mode)
{
    struct open_how how = build_open_how(flags, mode);
    return do_sys_openat2(dfd, filename, &how);
}

static long do_sys_openat2(int dfd, const char __user *filename,
                           struct open_how *how)
{
    struct open_flags op;
    int fd = build_open_flags(how, &op);
    struct filename *tmp;
    
    if (fd)
        return fd;
    
    // 1. 파일 이름 검색
    tmp = getname(filename);
    
    // 2. 파일 디스크립터 할당
    fd = get_unused_fd_flags(how->flags);
    
    // 3. 파일 열기
    struct file *f = do_filp_open(dfd, tmp, &op);
    
    // 4. 파일 디스크립터 설치
    fd_install(fd, f);
    
    putname(tmp);
    return fd;
}
```

#### 경로 탐색 (Path Lookup)

```c
// fs/namei.c

/*
 * 경로명을 dentry로 변환
 */
int path_lookupat(struct nameidata *nd, unsigned flags, struct path *path)
{
    const char *s = nd->name->name;
    int err;
    
    // 절대 경로면 루트부터, 상대 경로면 현재 디렉토리부터
    if (*s == '/') {
        set_root(nd);
        nd->path = nd->root;
    } else {
        nd->path = nd->pwd;
    }
    
    // 경로 구성 요소들을 순회
    while (*s) {
        // 각 구성 요소를 찾기
        err = walk_component(nd, 0);
        if (err)
            break;
    }
    
    return err;
}
```

#### 파일 읽기

```c
// fs/read_write.c

SYSCALL_DEFINE3(read, unsigned int, fd,
                char __user *, buf, size_t, count)
{
    struct fd f = fdget_pos(fd);
    ssize_t ret = -EBADF;
    
    if (f.file) {
        loff_t pos = file_pos_read(f.file);
        ret = vfs_read(f.file, buf, count, &pos);
        if (ret >= 0)
            file_pos_write(f.file, pos);
        fdput_pos(f);
    }
    
    return ret;
}

ssize_t vfs_read(struct file *file, char __user *buf,
                size_t count, loff_t *pos)
{
    ssize_t ret;
    
    // 읽기 권한 확인
    if (!(file->f_mode & FMODE_READ))
        return -EBADF;
    
    // 파일 연산 확인
    if (!file->f_op->read && !file->f_op->read_iter)
        return -EINVAL;
    
    // 실제 읽기
    if (file->f_op->read)
        ret = file->f_op->read(file, buf, count, pos);
    else if (file->f_op->read_iter)
        ret = new_sync_read(file, buf, count, pos);
    
    return ret;
}
```

### 페이지 캐시 (Page Cache)

파일 I/O 성능을 향상시키기 위한 캐시입니다.

```c
// include/linux/fs.h

struct address_space {
    struct inode *host;                    // 소유 inode
    struct xarray i_pages;                 // 페이지 트리
    gfp_t gfp_mask;                       // 메모리 할당 플래그
    atomic_t i_mmap_writable;
    
    struct rb_root_cached i_mmap;         // private 매핑
    unsigned long nrpages;                // 페이지 수
    
    const struct address_space_operations *a_ops;
    
    // ...
};

struct address_space_operations {
    int (*writepage)(struct page *page, struct writeback_control *wbc);
    int (*readpage)(struct file *, struct page *);
    
    int (*writepages)(struct address_space *, struct writeback_control *);
    int (*set_page_dirty)(struct page *page);
    
    int (*readpages)(struct file *filp, struct address_space *mapping,
                    struct list_head *pages, unsigned nr_pages);
    
    sector_t (*bmap)(struct address_space *, sector_t);
    void (*invalidatepage)(struct page *, unsigned int, unsigned int);
    int (*releasepage)(struct page *, gfp_t);
    
    // ...
};
```

### 특수 파일 시스템

#### 1. /proc 파일 시스템

커널 정보와 프로세스 정보를 제공합니다.

```bash
# CPU 정보
cat /proc/cpuinfo

# 메모리 정보
cat /proc/meminfo

# 프로세스 정보
cat /proc/[PID]/status
cat /proc/[PID]/cmdline
cat /proc/[PID]/environ
```

**간단한 /proc 엔트리 생성**:
```c
#include <linux/proc_fs.h>
#include <linux/seq_file.h>

static int my_proc_show(struct seq_file *m, void *v)
{
    seq_printf(m, "Hello from /proc!\n");
    return 0;
}

static int my_proc_open(struct inode *inode, struct file *file)
{
    return single_open(file, my_proc_show, NULL);
}

static const struct file_operations my_proc_fops = {
    .owner = THIS_MODULE,
    .open = my_proc_open,
    .read = seq_read,
    .llseek = seq_lseek,
    .release = single_release,
};

static int __init my_init(void)
{
    proc_create("my_entry", 0, NULL, &my_proc_fops);
    return 0;
}
```

#### 2. /sys 파일 시스템 (sysfs)

커널 객체와 속성을 나타냅니다.

```bash
# 장치 정보
ls /sys/devices/

# 블록 장치
ls /sys/block/

# 네트워크 인터페이스
ls /sys/class/net/
```

#### 3. tmpfs

RAM 기반 임시 파일 시스템입니다.

```bash
# tmpfs 마운트
sudo mount -t tmpfs -o size=1G tmpfs /mnt/ramdisk

# 사용량 확인
df -h /mnt/ramdisk
```

### 파일 시스템 마운트

```bash
# 파일 시스템 마운트
sudo mount -t ext4 /dev/sda1 /mnt/data

# 마운트 옵션
sudo mount -o ro,noexec /dev/sdb1 /mnt/readonly

# 마운트 확인
mount | grep /mnt

# 언마운트
sudo umount /mnt/data
```

### GPT 활용 예제

1. **VFS 레이어 이해**:
   ```
   "리눅스 VFS의 inode, dentry, file 구조체가 어떻게 연결되어 
   파일 시스템을 추상화하는지 설명해주세요"
   ```

2. **페이지 캐시 동작**:
   ```
   "파일 읽기 시 페이지 캐시가 어떻게 동작하여 성능을 향상시키는지 
   단계별로 설명해주세요"
   ```

3. **파일 시스템 구현**:
   ```
   "간단한 read-only 파일 시스템을 구현하려면 어떤 함수들을 
   구현해야 하는지 예제와 함께 설명해주세요"
   ```

### 실습 과제

1. /proc 파일 시스템에 커스텀 엔트리 추가하는 커널 모듈 작성
2. 파일의 inode 정보를 출력하는 프로그램 작성
3. 디렉토리를 재귀적으로 탐색하는 프로그램 작성
4. 파일 디스크립터 리다이렉션 실험
5. ext4 파일 시스템의 슈퍼블록 정보 읽기

### 다음 장에서는...

다음 장에서는 장치 드라이버를 자세히 살펴보고, 하드웨어와 커널의 상호작용을 알아봅니다.

---

[← 이전: 메모리 관리](03-memory-management.md) | [다음: 장치 드라이버 →](05-device-drivers.md)
