# 제 6장: 네트워킹

## Networking

### 네트워킹 스택 개요

리눅스 네트워킹 스택은 OSI 7계층 모델을 구현합니다.

```
┌──────────────────────────────┐
│    Application Layer         │ (응용 계층)
│    (User Space)              │
└──────────────────────────────┘
            ↕ Socket API
┌──────────────────────────────┐
│    Transport Layer           │ (전송 계층)
│    TCP / UDP                 │
├──────────────────────────────┤
│    Network Layer             │ (네트워크 계층)
│    IP (IPv4 / IPv6)          │
├──────────────────────────────┤
│    Link Layer                │ (링크 계층)
│    Ethernet / WiFi           │
├──────────────────────────────┤
│    Device Driver             │ (장치 드라이버)
└──────────────────────────────┘
            ↕
┌──────────────────────────────┐
│    Physical Layer            │ (물리 계층)
│    Network Hardware          │
└──────────────────────────────┘
```

### Socket Buffer (sk_buff)

네트워크 패킷을 나타내는 핵심 구조체입니다.

```c
// include/linux/skbuff.h

struct sk_buff {
    union {
        struct {
            struct sk_buff *next;      // 다음 버퍼
            struct sk_buff *prev;      // 이전 버퍼
            union {
                struct net_device *dev;  // 네트워크 장치
                unsigned long dev_scratch;
            };
        };
        struct rb_node rbnode;
        struct list_head list;
    };
    
    struct sock *sk;                   // 소켓
    
    union {
        ktime_t tstamp;               // 타임스탬프
        u64 skb_mstamp_ns;
    };
    
    char cb[48] __aligned(8);         // 제어 버퍼
    
    unsigned int len,                 // 데이터 길이
                 data_len;            // 비선형 데이터 길이
    __u16 mac_len,                    // MAC 헤더 길이
          hdr_len;                    // 복제 가능한 헤더 길이
    
    __be16 protocol;                  // 프로토콜
    
    __u8 pkt_type:3;                 // 패킷 타입
    __u8 ignore_df:1;
    __u8 nf_trace:1;
    __u8 ip_summed:2;
    __u8 ooo_okay:1;
    
    // 데이터 포인터
    sk_buff_data_t transport_header;  // 전송 계층 헤더
    sk_buff_data_t network_header;    // 네트워크 계층 헤더
    sk_buff_data_t mac_header;        // MAC 계층 헤더
    
    sk_buff_data_t tail;              // 데이터 끝
    sk_buff_data_t end;               // 버퍼 끝
    
    unsigned char *head,              // 버퍼 시작
                  *data;              // 데이터 시작
    
    unsigned int truesize;            // 실제 크기
    refcount_t users;                 // 참조 카운트
    
    // ...
};
```

**sk_buff 관리 함수**:
```c
// 새 sk_buff 할당
struct sk_buff *alloc_skb(unsigned int size, gfp_t priority);

// sk_buff 해제
void kfree_skb(struct sk_buff *skb);

// sk_buff 복제
struct sk_buff *skb_clone(struct sk_buff *skb, gfp_t priority);

// 데이터 영역 조작
unsigned char *skb_put(struct sk_buff *skb, unsigned int len);
unsigned char *skb_push(struct sk_buff *skb, unsigned int len);
unsigned char *skb_pull(struct sk_buff *skb, unsigned int len);
void skb_reserve(struct sk_buff *skb, int len);
```

### 소켓 (Socket)

소켓은 네트워크 통신의 엔드포인트입니다.

```c
// include/net/sock.h

struct sock {
    struct sock_common __sk_common;
    
    socket_lock_t sk_lock;
    atomic_t sk_drops;
    
    int sk_rcvbuf;                    // 수신 버퍼 크기
    int sk_sndbuf;                    // 송신 버퍼 크기
    
    struct sk_buff_head sk_receive_queue;  // 수신 큐
    struct sk_buff_head sk_write_queue;    // 송신 큐
    
    unsigned long sk_flags;
    unsigned long sk_lingertime;
    struct proto *sk_prot_creator;
    
    rwlock_t sk_callback_lock;
    int sk_err,
        sk_err_soft;
    
    u32 sk_ack_backlog;              // ACK 백로그
    u32 sk_max_ack_backlog;
    
    kuid_t sk_uid;
    struct pid *sk_peer_pid;
    
    long sk_rcvtimeo;                // 수신 타임아웃
    long sk_sndtimeo;                // 송신 타임아웃
    
    struct timer_list sk_timer;      // 타이머
    
    // ...
};

struct socket {
    socket_state state;              // 소켓 상태
    short type;                      // 소켓 타입
    unsigned long flags;
    
    struct file *file;               // 연결된 파일
    struct sock *sk;                 // 소켓 구조체
    const struct proto_ops *ops;     // 프로토콜 연산
    
    struct socket_wq wq;             // 대기 큐
};
```

### 소켓 프로그래밍

#### TCP 서버

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

#define PORT 8080
#define BUFFER_SIZE 1024

int main() {
    int server_fd, client_fd;
    struct sockaddr_in address;
    int addrlen = sizeof(address);
    char buffer[BUFFER_SIZE] = {0};
    
    // 1. 소켓 생성
    server_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (server_fd == 0) {
        perror("socket failed");
        exit(EXIT_FAILURE);
    }
    
    // 2. 주소 설정
    address.sin_family = AF_INET;
    address.sin_addr.s_addr = INADDR_ANY;
    address.sin_port = htons(PORT);
    
    // 3. 바인드
    if (bind(server_fd, (struct sockaddr *)&address,
             sizeof(address)) < 0) {
        perror("bind failed");
        exit(EXIT_FAILURE);
    }
    
    // 4. 리슨
    if (listen(server_fd, 3) < 0) {
        perror("listen failed");
        exit(EXIT_FAILURE);
    }
    
    printf("Server listening on port %d\n", PORT);
    
    // 5. 수락
    client_fd = accept(server_fd, (struct sockaddr *)&address,
                      (socklen_t*)&addrlen);
    if (client_fd < 0) {
        perror("accept failed");
        exit(EXIT_FAILURE);
    }
    
    // 6. 데이터 수신
    read(client_fd, buffer, BUFFER_SIZE);
    printf("Message from client: %s\n", buffer);
    
    // 7. 데이터 송신
    const char *message = "Hello from server";
    send(client_fd, message, strlen(message), 0);
    
    // 8. 종료
    close(client_fd);
    close(server_fd);
    
    return 0;
}
```

#### TCP 클라이언트

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

#define PORT 8080
#define BUFFER_SIZE 1024

int main() {
    int sock = 0;
    struct sockaddr_in serv_addr;
    char buffer[BUFFER_SIZE] = {0};
    
    // 1. 소켓 생성
    sock = socket(AF_INET, SOCK_STREAM, 0);
    if (sock < 0) {
        perror("Socket creation error");
        return -1;
    }
    
    // 2. 서버 주소 설정
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(PORT);
    
    if (inet_pton(AF_INET, "127.0.0.1", &serv_addr.sin_addr) <= 0) {
        perror("Invalid address");
        return -1;
    }
    
    // 3. 연결
    if (connect(sock, (struct sockaddr *)&serv_addr,
                sizeof(serv_addr)) < 0) {
        perror("Connection failed");
        return -1;
    }
    
    // 4. 데이터 송신
    const char *message = "Hello from client";
    send(sock, message, strlen(message), 0);
    printf("Message sent\n");
    
    // 5. 데이터 수신
    read(sock, buffer, BUFFER_SIZE);
    printf("Message from server: %s\n", buffer);
    
    // 6. 종료
    close(sock);
    
    return 0;
}
```

#### UDP 통신

```c
// UDP Server
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

#define PORT 8080
#define BUFFER_SIZE 1024

int main() {
    int sockfd;
    char buffer[BUFFER_SIZE];
    struct sockaddr_in servaddr, cliaddr;
    
    // 소켓 생성
    sockfd = socket(AF_INET, SOCK_DGRAM, 0);
    
    memset(&servaddr, 0, sizeof(servaddr));
    memset(&cliaddr, 0, sizeof(cliaddr));
    
    // 서버 정보 설정
    servaddr.sin_family = AF_INET;
    servaddr.sin_addr.s_addr = INADDR_ANY;
    servaddr.sin_port = htons(PORT);
    
    // 바인드
    bind(sockfd, (const struct sockaddr *)&servaddr,
         sizeof(servaddr));
    
    int len, n;
    len = sizeof(cliaddr);
    
    // 수신
    n = recvfrom(sockfd, (char *)buffer, BUFFER_SIZE,
                MSG_WAITALL, (struct sockaddr *)&cliaddr,
                &len);
    buffer[n] = '\0';
    printf("Client : %s\n", buffer);
    
    // 송신
    const char *message = "Hello from server";
    sendto(sockfd, (const char *)message, strlen(message),
          MSG_CONFIRM, (const struct sockaddr *)&cliaddr, len);
    
    close(sockfd);
    return 0;
}
```

### 네트워크 장치

```c
// include/linux/netdevice.h

struct net_device {
    char name[IFNAMSIZ];              // 장치 이름
    
    unsigned long state;              // 장치 상태
    
    struct net_device_stats stats;    // 통계
    
    unsigned int flags;               // 플래그
    unsigned int priv_flags;
    
    unsigned short type;              // 하드웨어 타입
    unsigned short hard_header_len;   // 하드웨어 헤더 길이
    
    unsigned char *dev_addr;          // 하드웨어 주소
    unsigned char broadcast[MAX_ADDR_LEN];  // 브로드캐스트 주소
    
    unsigned int mtu;                 // MTU
    unsigned int min_mtu;
    unsigned int max_mtu;
    
    const struct net_device_ops *netdev_ops;  // 장치 연산
    const struct ethtool_ops *ethtool_ops;
    
    struct netdev_queue *_tx;         // 송신 큐
    unsigned int num_tx_queues;
    
    unsigned int real_num_tx_queues;
    unsigned int real_num_rx_queues;
    
    // ...
};

struct net_device_ops {
    int (*ndo_init)(struct net_device *dev);
    void (*ndo_uninit)(struct net_device *dev);
    int (*ndo_open)(struct net_device *dev);
    int (*ndo_stop)(struct net_device *dev);
    netdev_tx_t (*ndo_start_xmit)(struct sk_buff *skb,
                                  struct net_device *dev);
    void (*ndo_set_rx_mode)(struct net_device *dev);
    int (*ndo_set_mac_address)(struct net_device *dev, void *addr);
    int (*ndo_validate_addr)(struct net_device *dev);
    int (*ndo_do_ioctl)(struct net_device *dev,
                       struct ifreq *ifr, int cmd);
    int (*ndo_change_mtu)(struct net_device *dev, int new_mtu);
    
    // ...
};
```

### Netfilter / iptables

패킷 필터링과 NAT를 담당합니다.

```c
// include/linux/netfilter.h

struct nf_hook_ops {
    struct list_head list;
    
    nf_hookfn *hook;                 // 훅 함수
    struct net_device *dev;
    void *priv;
    u_int8_t pf;                     // 프로토콜 패밀리
    unsigned int hooknum;            // 훅 번호
    int priority;                    // 우선순위
};

// 훅 포인트
enum nf_inet_hooks {
    NF_INET_PRE_ROUTING,     // 라우팅 전
    NF_INET_LOCAL_IN,        // 로컬 입력
    NF_INET_FORWARD,         // 포워딩
    NF_INET_LOCAL_OUT,       // 로컬 출력
    NF_INET_POST_ROUTING,    // 라우팅 후
    NF_INET_NUMHOOKS
};
```

**간단한 Netfilter 모듈**:
```c
#include <linux/module.h>
#include <linux/netfilter.h>
#include <linux/netfilter_ipv4.h>
#include <linux/ip.h>
#include <linux/tcp.h>

static struct nf_hook_ops nfho;

static unsigned int hook_func(void *priv,
                              struct sk_buff *skb,
                              const struct nf_hook_state *state)
{
    struct iphdr *iph;
    struct tcphdr *tcph;
    
    if (!skb)
        return NF_ACCEPT;
    
    iph = ip_hdr(skb);
    
    if (iph->protocol == IPPROTO_TCP) {
        tcph = tcp_hdr(skb);
        
        printk(KERN_INFO "TCP packet: %pI4:%d -> %pI4:%d\n",
               &iph->saddr, ntohs(tcph->source),
               &iph->daddr, ntohs(tcph->dest));
        
        // 특정 포트 차단 예제
        if (ntohs(tcph->dest) == 80) {
            printk(KERN_INFO "Blocking HTTP traffic\n");
            return NF_DROP;
        }
    }
    
    return NF_ACCEPT;
}

static int __init nf_init(void)
{
    nfho.hook = hook_func;
    nfho.hooknum = NF_INET_PRE_ROUTING;
    nfho.pf = PF_INET;
    nfho.priority = NF_IP_PRI_FIRST;
    
    nf_register_net_hook(&init_net, &nfho);
    
    printk(KERN_INFO "Netfilter module loaded\n");
    return 0;
}

static void __exit nf_exit(void)
{
    nf_unregister_net_hook(&init_net, &nfho);
    printk(KERN_INFO "Netfilter module unloaded\n");
}

module_init(nf_init);
module_exit(nf_exit);

MODULE_LICENSE("GPL");
```

### 네트워크 도구

#### 1. ifconfig / ip

```bash
# 네트워크 인터페이스 확인
ifconfig
ip addr show

# IP 주소 설정
sudo ifconfig eth0 192.168.1.100 netmask 255.255.255.0
sudo ip addr add 192.168.1.100/24 dev eth0

# 인터페이스 활성화/비활성화
sudo ifconfig eth0 up
sudo ifconfig eth0 down
sudo ip link set eth0 up
sudo ip link set eth0 down
```

#### 2. netstat / ss

```bash
# 열린 포트 확인
netstat -tuln
ss -tuln

# 소켓 통계
ss -s

# TCP 연결 확인
netstat -tan
ss -tan
```

#### 3. tcpdump

```bash
# 모든 패킷 캡처
sudo tcpdump -i eth0

# 특정 포트
sudo tcpdump -i eth0 port 80

# 특정 호스트
sudo tcpdump -i eth0 host 192.168.1.1

# 패킷을 파일로 저장
sudo tcpdump -i eth0 -w capture.pcap

# 파일 읽기
tcpdump -r capture.pcap
```

#### 4. iptables

```bash
# 규칙 목록
sudo iptables -L -v -n

# 포트 차단
sudo iptables -A INPUT -p tcp --dport 22 -j DROP

# 특정 IP 차단
sudo iptables -A INPUT -s 192.168.1.100 -j DROP

# 규칙 삭제
sudo iptables -D INPUT -p tcp --dport 22 -j DROP

# 모든 규칙 삭제
sudo iptables -F
```

### 네트워크 프로그래밍 고급

#### epoll을 사용한 비동기 I/O

```c
#include <sys/epoll.h>

#define MAX_EVENTS 10

int main() {
    int epoll_fd, nfds;
    struct epoll_event ev, events[MAX_EVENTS];
    
    // epoll 인스턴스 생성
    epoll_fd = epoll_create1(0);
    if (epoll_fd == -1) {
        perror("epoll_create1");
        exit(EXIT_FAILURE);
    }
    
    // 소켓을 epoll에 등록
    ev.events = EPOLLIN;
    ev.data.fd = server_fd;
    if (epoll_ctl(epoll_fd, EPOLL_CTL_ADD, server_fd, &ev) == -1) {
        perror("epoll_ctl");
        exit(EXIT_FAILURE);
    }
    
    // 이벤트 루프
    for (;;) {
        nfds = epoll_wait(epoll_fd, events, MAX_EVENTS, -1);
        if (nfds == -1) {
            perror("epoll_wait");
            exit(EXIT_FAILURE);
        }
        
        for (int n = 0; n < nfds; n++) {
            if (events[n].data.fd == server_fd) {
                // 새 연결 수락
                int client_fd = accept(server_fd, NULL, NULL);
                
                // 클라이언트 소켓을 epoll에 추가
                ev.events = EPOLLIN | EPOLLET;
                ev.data.fd = client_fd;
                epoll_ctl(epoll_fd, EPOLL_CTL_ADD, client_fd, &ev);
            } else {
                // 데이터 처리
                char buf[512];
                int count = read(events[n].data.fd, buf, sizeof(buf));
                if (count == 0) {
                    // 연결 종료
                    close(events[n].data.fd);
                } else {
                    // 데이터 처리
                    // ...
                }
            }
        }
    }
    
    close(epoll_fd);
    return 0;
}
```

### GPT 활용 예제

1. **TCP/IP 스택 이해**:
   ```
   "리눅스에서 TCP 패킷이 송신될 때 각 계층을 거치면서 어떤 
   처리가 일어나는지 단계별로 설명해주세요"
   ```

2. **sk_buff 관리**:
   ```
   "sk_buff 구조체의 head, data, tail, end 포인터들이 
   어떻게 사용되는지 그림과 함께 설명해주세요"
   ```

3. **Netfilter 훅**:
   ```
   "Netfilter의 5개 훅 포인트가 패킷 처리 경로에서 
   어느 위치에 있는지 설명해주세요"
   ```

### 실습 과제

1. 간단한 에코 서버와 클라이언트 구현
2. 멀티스레드 TCP 서버 구현
3. UDP 브로드캐스트 프로그램 작성
4. epoll을 사용한 비동기 서버 구현
5. 간단한 Netfilter 모듈 작성

### 다음 장에서는...

다음 장에서는 이 책의 내용을 정리하고, 더 깊이 있는 학습을 위한 참고 자료를 소개합니다.

---

[← 이전: 장치 드라이버](05-device-drivers.md) | [다음: 결론 및 참고 자료 →](07-conclusion.md)
