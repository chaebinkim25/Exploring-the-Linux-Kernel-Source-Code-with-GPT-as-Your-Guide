# 제 5장: 장치 드라이버

## Device Drivers

### 장치 드라이버 개요

장치 드라이버는 하드웨어와 커널 사이의 인터페이스 역할을 합니다. 리눅스는 세 가지 주요 장치 타입을 지원합니다:

1. **문자 장치 (Character Device)**: 순차적 접근 (키보드, 시리얼 포트)
2. **블록 장치 (Block Device)**: 랜덤 접근 (하드디스크, SSD)
3. **네트워크 장치 (Network Device)**: 네트워크 인터페이스

### 문자 장치 드라이버

#### 기본 구조

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/uaccess.h>

#define DEVICE_NAME "mychardev"
#define CLASS_NAME  "mychar"

static int major_number;
static struct class* mychar_class = NULL;
static struct device* mychar_device = NULL;
static char message[256] = {0};
static short size_of_message;

// 장치 열기
static int dev_open(struct inode *inodep, struct file *filep)
{
    printk(KERN_INFO "MyCharDev: Device opened\n");
    return 0;
}

// 장치에서 읽기
static ssize_t dev_read(struct file *filep, char *buffer,
                       size_t len, loff_t *offset)
{
    int error_count = 0;
    
    // 커널 버퍼를 사용자 공간으로 복사
    error_count = copy_to_user(buffer, message, size_of_message);
    
    if (error_count == 0) {
        printk(KERN_INFO "MyCharDev: Sent %d characters\n",
               size_of_message);
        return (size_of_message = 0);
    } else {
        printk(KERN_INFO "MyCharDev: Failed to send %d characters\n",
               error_count);
        return -EFAULT;
    }
}

// 장치에 쓰기
static ssize_t dev_write(struct file *filep, const char *buffer,
                        size_t len, loff_t *offset)
{
    // 사용자 공간 버퍼를 커널로 복사
    sprintf(message, "%s(%zu letters)", buffer, len);
    size_of_message = strlen(message);
    
    printk(KERN_INFO "MyCharDev: Received %zu characters\n", len);
    return len;
}

// 장치 닫기
static int dev_release(struct inode *inodep, struct file *filep)
{
    printk(KERN_INFO "MyCharDev: Device closed\n");
    return 0;
}

// 파일 연산 구조체
static struct file_operations fops = {
    .open = dev_open,
    .read = dev_read,
    .write = dev_write,
    .release = dev_release,
};

// 모듈 초기화
static int __init mychardev_init(void)
{
    printk(KERN_INFO "MyCharDev: Initializing\n");
    
    // 주 번호 동적 할당
    major_number = register_chrdev(0, DEVICE_NAME, &fops);
    if (major_number < 0) {
        printk(KERN_ALERT "MyCharDev: Failed to register\n");
        return major_number;
    }
    
    printk(KERN_INFO "MyCharDev: Registered with major %d\n",
           major_number);
    
    // 장치 클래스 등록
    mychar_class = class_create(THIS_MODULE, CLASS_NAME);
    if (IS_ERR(mychar_class)) {
        unregister_chrdev(major_number, DEVICE_NAME);
        printk(KERN_ALERT "MyCharDev: Failed to create class\n");
        return PTR_ERR(mychar_class);
    }
    
    // 장치 등록
    mychar_device = device_create(mychar_class, NULL,
                                  MKDEV(major_number, 0),
                                  NULL, DEVICE_NAME);
    if (IS_ERR(mychar_device)) {
        class_destroy(mychar_class);
        unregister_chrdev(major_number, DEVICE_NAME);
        printk(KERN_ALERT "MyCharDev: Failed to create device\n");
        return PTR_ERR(mychar_device);
    }
    
    printk(KERN_INFO "MyCharDev: Device created successfully\n");
    return 0;
}

// 모듈 종료
static void __exit mychardev_exit(void)
{
    device_destroy(mychar_class, MKDEV(major_number, 0));
    class_unregister(mychar_class);
    class_destroy(mychar_class);
    unregister_chrdev(major_number, DEVICE_NAME);
    printk(KERN_INFO "MyCharDev: Goodbye!\n");
}

module_init(mychardev_init);
module_exit(mychardev_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Your Name");
MODULE_DESCRIPTION("A simple character device driver");
MODULE_VERSION("1.0");
```

#### 사용자 공간에서 테스트

```c
// test_chardev.c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>

#define DEVICE "/dev/mychardev"

int main()
{
    int fd;
    char write_buf[] = "Hello, Device!";
    char read_buf[256] = {0};
    
    // 장치 열기
    fd = open(DEVICE, O_RDWR);
    if (fd < 0) {
        perror("Failed to open device");
        return -1;
    }
    
    // 장치에 쓰기
    printf("Writing to device: %s\n", write_buf);
    write(fd, write_buf, strlen(write_buf));
    
    // 장치에서 읽기
    read(fd, read_buf, sizeof(read_buf));
    printf("Read from device: %s\n", read_buf);
    
    // 장치 닫기
    close(fd);
    
    return 0;
}
```

### cdev 구조체 사용

더 현대적인 방법은 `cdev` 구조체를 사용하는 것입니다:

```c
#include <linux/cdev.h>

static dev_t dev_num;
static struct cdev my_cdev;

static int __init modern_chardev_init(void)
{
    int ret;
    
    // 장치 번호 할당
    ret = alloc_chrdev_region(&dev_num, 0, 1, "modern_dev");
    if (ret < 0) {
        printk(KERN_ALERT "Failed to allocate device number\n");
        return ret;
    }
    
    // cdev 초기화
    cdev_init(&my_cdev, &fops);
    my_cdev.owner = THIS_MODULE;
    
    // cdev 등록
    ret = cdev_add(&my_cdev, dev_num, 1);
    if (ret < 0) {
        unregister_chrdev_region(dev_num, 1);
        printk(KERN_ALERT "Failed to add cdev\n");
        return ret;
    }
    
    printk(KERN_INFO "Modern CharDev: Major=%d, Minor=%d\n",
           MAJOR(dev_num), MINOR(dev_num));
    
    return 0;
}

static void __exit modern_chardev_exit(void)
{
    cdev_del(&my_cdev);
    unregister_chrdev_region(dev_num, 1);
    printk(KERN_INFO "Modern CharDev: Removed\n");
}
```

### ioctl 인터페이스

장치 특정 제어 명령을 위한 인터페이스입니다:

```c
#include <linux/ioctl.h>

#define MY_IOC_MAGIC  'k'
#define MY_IOC_RESET  _IO(MY_IOC_MAGIC, 0)
#define MY_IOC_GETVAL _IOR(MY_IOC_MAGIC, 1, int)
#define MY_IOC_SETVAL _IOW(MY_IOC_MAGIC, 2, int)

static long dev_ioctl(struct file *file, unsigned int cmd,
                     unsigned long arg)
{
    int value;
    
    switch (cmd) {
    case MY_IOC_RESET:
        // 장치 리셋
        printk(KERN_INFO "Device reset\n");
        break;
        
    case MY_IOC_GETVAL:
        // 값 읽기
        value = 42;  // 예제 값
        if (copy_to_user((int __user *)arg, &value, sizeof(value)))
            return -EFAULT;
        break;
        
    case MY_IOC_SETVAL:
        // 값 쓰기
        if (copy_from_user(&value, (int __user *)arg, sizeof(value)))
            return -EFAULT;
        printk(KERN_INFO "Value set to %d\n", value);
        break;
        
    default:
        return -EINVAL;
    }
    
    return 0;
}

static struct file_operations fops = {
    .owner = THIS_MODULE,
    .open = dev_open,
    .read = dev_read,
    .write = dev_write,
    .release = dev_release,
    .unlocked_ioctl = dev_ioctl,
};
```

**사용자 공간에서 ioctl 사용**:
```c
#include <sys/ioctl.h>

int value;

// GETVAL
ioctl(fd, MY_IOC_GETVAL, &value);
printf("Got value: %d\n", value);

// SETVAL
value = 100;
ioctl(fd, MY_IOC_SETVAL, &value);

// RESET
ioctl(fd, MY_IOC_RESET);
```

### 블록 장치 드라이버

블록 장치는 더 복잡한 구조를 가집니다:

```c
#include <linux/blkdev.h>
#include <linux/bio.h>

#define KERNEL_SECTOR_SIZE 512
#define DEVICE_SIZE (1024 * 1024 * 256)  // 256MB

static struct gendisk *my_disk;
static struct request_queue *my_queue;
static u8 *device_data;

// 요청 처리
static void my_request(struct request_queue *q)
{
    struct request *req;
    
    req = blk_fetch_request(q);
    while (req != NULL) {
        struct bio_vec bvec;
        struct req_iterator iter;
        sector_t sector = blk_rq_pos(req);
        
        // 각 bio 세그먼트 처리
        rq_for_each_segment(bvec, req, iter) {
            unsigned long offset = sector * KERNEL_SECTOR_SIZE;
            void *buffer = page_address(bvec.bv_page) + bvec.bv_offset;
            unsigned int len = bvec.bv_len;
            
            if (rq_data_dir(req) == WRITE) {
                // 쓰기
                memcpy(device_data + offset, buffer, len);
            } else {
                // 읽기
                memcpy(buffer, device_data + offset, len);
            }
            
            sector += len / KERNEL_SECTOR_SIZE;
        }
        
        __blk_end_request_all(req, 0);
        req = blk_fetch_request(q);
    }
}

static int __init my_block_init(void)
{
    // 디바이스 데이터 할당
    device_data = vmalloc(DEVICE_SIZE);
    if (!device_data)
        return -ENOMEM;
    
    // 요청 큐 생성
    my_queue = blk_init_queue(my_request, NULL);
    if (!my_queue) {
        vfree(device_data);
        return -ENOMEM;
    }
    
    // gendisk 할당
    my_disk = alloc_disk(1);
    if (!my_disk) {
        blk_cleanup_queue(my_queue);
        vfree(device_data);
        return -ENOMEM;
    }
    
    // gendisk 설정
    strcpy(my_disk->disk_name, "myblock");
    my_disk->major = register_blkdev(0, "myblock");
    my_disk->first_minor = 0;
    my_disk->fops = &my_block_fops;
    my_disk->queue = my_queue;
    set_capacity(my_disk, DEVICE_SIZE / KERNEL_SECTOR_SIZE);
    
    // gendisk 등록
    add_disk(my_disk);
    
    printk(KERN_INFO "MyBlock: Device registered\n");
    return 0;
}
```

### 인터럽트 처리

장치 드라이버는 하드웨어 인터럽트를 처리해야 합니다:

```c
#include <linux/interrupt.h>

static int irq_number;

// 인터럽트 핸들러
static irqreturn_t my_irq_handler(int irq, void *dev_id)
{
    printk(KERN_INFO "Interrupt received: IRQ %d\n", irq);
    
    // 인터럽트 처리 로직
    // ...
    
    return IRQ_HANDLED;
}

// 인터럽트 등록
static int __init my_driver_init(void)
{
    int ret;
    
    irq_number = 17;  // 예제 IRQ 번호
    
    ret = request_irq(irq_number,
                     my_irq_handler,
                     IRQF_SHARED,
                     "my_device",
                     (void *)my_irq_handler);
    
    if (ret) {
        printk(KERN_ERR "Failed to request IRQ %d\n", irq_number);
        return ret;
    }
    
    printk(KERN_INFO "IRQ %d registered\n", irq_number);
    return 0;
}

// 인터럽트 해제
static void __exit my_driver_exit(void)
{
    free_irq(irq_number, (void *)my_irq_handler);
    printk(KERN_INFO "IRQ %d freed\n", irq_number);
}
```

### DMA (Direct Memory Access)

DMA를 사용하면 CPU 개입 없이 데이터를 전송할 수 있습니다:

```c
#include <linux/dma-mapping.h>

static dma_addr_t dma_handle;
static void *dma_buffer;

static int setup_dma(struct device *dev)
{
    // DMA 버퍼 할당
    dma_buffer = dma_alloc_coherent(dev, PAGE_SIZE,
                                    &dma_handle, GFP_KERNEL);
    if (!dma_buffer) {
        printk(KERN_ERR "Failed to allocate DMA buffer\n");
        return -ENOMEM;
    }
    
    printk(KERN_INFO "DMA buffer: virt=%p, phys=%pad\n",
           dma_buffer, &dma_handle);
    
    // DMA 전송 시작
    // ...
    
    return 0;
}

static void cleanup_dma(struct device *dev)
{
    if (dma_buffer) {
        dma_free_coherent(dev, PAGE_SIZE, dma_buffer, dma_handle);
        dma_buffer = NULL;
    }
}
```

### Platform 드라이버

플랫폼 장치를 위한 드라이버입니다:

```c
#include <linux/platform_device.h>

static int my_platform_probe(struct platform_device *pdev)
{
    struct resource *res;
    void __iomem *base;
    
    printk(KERN_INFO "Platform device probed\n");
    
    // 리소스 가져오기
    res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    if (!res) {
        dev_err(&pdev->dev, "No memory resource\n");
        return -ENODEV;
    }
    
    // 메모리 매핑
    base = devm_ioremap_resource(&pdev->dev, res);
    if (IS_ERR(base))
        return PTR_ERR(base);
    
    // 장치 초기화
    // ...
    
    return 0;
}

static int my_platform_remove(struct platform_device *pdev)
{
    printk(KERN_INFO "Platform device removed\n");
    return 0;
}

static const struct of_device_id my_of_match[] = {
    { .compatible = "vendor,my-device", },
    { },
};
MODULE_DEVICE_TABLE(of, my_of_match);

static struct platform_driver my_platform_driver = {
    .probe = my_platform_probe,
    .remove = my_platform_remove,
    .driver = {
        .name = "my-platform-device",
        .of_match_table = my_of_match,
    },
};

module_platform_driver(my_platform_driver);
```

### 디버깅 기법

#### 1. printk와 로그 레벨

```c
printk(KERN_DEBUG "Debug message\n");
printk(KERN_INFO "Info message\n");
printk(KERN_WARNING "Warning message\n");
printk(KERN_ERR "Error message\n");

// pr_* 매크로 사용
pr_debug("Debug: value = %d\n", value);
pr_info("Info: initialized\n");
pr_warn("Warning: unusual condition\n");
pr_err("Error: operation failed\n");
```

#### 2. Dynamic Debug

```bash
# 동적 디버깅 활성화
echo 'file mydriver.c +p' > /sys/kernel/debug/dynamic_debug/control

# 함수별 활성화
echo 'func my_function +p' > /sys/kernel/debug/dynamic_debug/control

# 모듈별 활성화
echo 'module mymodule +p' > /sys/kernel/debug/dynamic_debug/control
```

#### 3. ftrace

```bash
# ftrace 활성화
cd /sys/kernel/debug/tracing
echo function > current_tracer
echo 1 > tracing_on

# 특정 함수 추적
echo my_driver_function > set_ftrace_filter

# 추적 결과 확인
cat trace
```

### GPT 활용 예제

1. **드라이버 구조 이해**:
   ```
   "리눅스 문자 장치 드라이버의 file_operations 구조체에서 
   각 콜백 함수의 역할과 호출 시점을 설명해주세요"
   ```

2. **인터럽트 처리**:
   ```
   "하드웨어 인터럽트가 발생했을 때 리눅스 커널에서 인터럽트 
   핸들러가 호출되는 과정을 단계별로 설명해주세요"
   ```

3. **DMA 전송**:
   ```
   "DMA를 사용한 데이터 전송과 일반 메모리 복사의 차이점과 
   DMA 사용 시 주의할 점을 설명해주세요"
   ```

### 실습 과제

1. 간단한 문자 장치 드라이버 작성 및 테스트
2. ioctl 명령을 추가하여 장치 제어
3. /sys에 속성 파일 추가
4. 타이머를 사용하는 드라이버 작성
5. 인터럽트 핸들러 구현 (시뮬레이션)

### 다음 장에서는...

다음 장에서는 네트워킹 서브시스템을 자세히 살펴보고, 네트워크 스택과 소켓 프로그래밍을 알아봅니다.

---

[← 이전: 파일 시스템](04-file-systems.md) | [다음: 네트워킹 →](06-networking.md)
