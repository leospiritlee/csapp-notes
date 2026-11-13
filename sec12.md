# 网络编程（L21—L22）

> 从 socket API 到客户端-服务器模型，写一个最小的 Web 服务器需要哪些系统调用。

### Lecture 21 Network Programming Part 1（77.0 分钟）
- **主旨**：自底向上建立互联网的物理与逻辑模型（以太网段、路由器互联、TCP/IP 协议族、IP 地址与 DNS、端口），并给出 socket（套接字）接口的调用序列与 getaddrinfo 的用法，为写客户端-服务器程序打基础。
- **关键点**：
  1. 绝大多数网络应用基于 client-server（客户端-服务器）模型：server 进程管理某个 resource，通过替 client 操作该资源来提供服务，并由 client 的请求激活（自动售货机类比）。
  2. 最底层 Ethernet segment（以太网段）由双绞线把 hosts 连到 hub 构成，覆盖一个房间或楼层；每个以太网适配器有唯一的 48 位 MAC 地址（如 00:16:ea:e3:54:e6），host 以 frame（帧）为单位发送比特，hub 把每个端口收到的每个比特原样复制到所有其他端口，故每台主机都看到全部比特。
  3. internet 是众多网络的 ad hoc 互联，没有特定拓扑且各路由器与链路容量差异极大；数据靠 encapsulation（封装）逐层加头——data → internet packet（data + PH）→ LAN frame（data + PH + FH），路由器拆掉旧帧头换上新帧头继续转发，不同网络使用不同格式的帧。
  4. TCP/IP 协议族分工：IP 提供基本的命名机制与 host-to-host 的不可靠数据报（datagram）投递；UDP 借助 IP 提供 process-to-process 的不可靠数据报；TCP 借助 IP 在连接上提供 process-to-process 的可靠字节流；三者统一通过 Unix 文件 I/O 与 sockets 接口访问。
  5. IPv4 用 32 位地址，按 dotted decimal notation（点分十进制）书写：0x8002C2F2 = 128.2.194.242；`inet_pton` 把点分十进制串转成 network byte order 的 IP 地址，`inet_ntop` 反向转换，其中 n 表示 network、p 表示 presentation；IPv6 为 128 位，1996 年由 IETF 引入，但截至 2015 年绝大多数流量仍走 IPv4。
  6. DNS 是维护域名与 IP 映射的全球分布式数据库，概念上可视为数百万条 host entry 的集合；映射是多对多的（`nslookup www.twitter.com` 返回 199.16.156.6/.70/.102/.230，`nslookup twitter.com` 返回同样四个地址但顺序不同），也有合法域名查不到地址（`nslookup ics.cs.cmu.edu` 返回 No answer）。
  7. well-known ports 与服务名固定绑定：7/echo、22/ssh、25/smtp、80/http，映射表存放在每台 Linux 机器的 /etc/services；socket 地址用通用结构 `struct sockaddr { uint16_t sa_family; char sa_data[14]; }` 加 `typedef struct sockaddr SA` 以便强转。
  8. sockets 接口序列：`int socket(int domain, int type, int protocol)` 创建端点描述符（如 `Socket(AF_INET, SOCK_STREAM, 0)`，AF_INET 表示 32 位 IPv4 地址、SOCK_STREAM 表示该 socket 是连接的端点）；`int listen(int sockfd, int backlog)` 把 active socket 转为 listening socket，backlog 是内核在开始拒绝请求前应排队的未完成连接请求数提示。
  9. getaddrinfo 把主机名、地址、端口与服务名的字符串表示转成 socket 地址结构（取代过时的 gethostbyname/getservbyname），可重入、协议无关、同时支持 IPv4 与 IPv6，缺点是偏复杂；它返回 addrinfo 链表，client 逐个尝试 socket+connect 直到成功，server 逐个尝试 socket+bind 直到成功。
- **易错/考点**：socket 的三个参数是协议相关的，最佳实践是用 getaddrinfo 自动生成以保证协议无关；`./hostinfo localhost` 输出的 127.0.0.1 是靠 NI_NUMERICHOST 打印数字地址而非域名。
### Lecture 22 Network Programming Part II（78.0 分钟）
- **主旨**：把上一讲的 socket 调用序列封装成 open_clientfd/open_listenfd 两个辅助函数，实现迭代式 echo 服务器与客户端，并在此基础上构建一个支持静态文件与 CGI 动态内容的 Tiny Web Server。
- **关键点**：
  1. open_clientfd 遍历 getaddrinfo 返回的链表：对每项先 `socket(p->ai_family, p->ai_socktype, p->ai_protocol)`，失败则 continue；再 `connect(clientfd, p->ai_addr, p->ai_addrlen)`，返回 -1 则 Close 后换下一项，成功则 break；最后 Freeaddrinfo，p 为空表示全部失败返回 -1，否则返回该 clientfd。
  2. open_listenfd 以 `hints.ai_socktype = SOCK_STREAM`、`hints.ai_flags = AI_PASSIVE | AI_ADDRCONFIG` 再加 `AI_NUMERICSERV` 调 Getaddrinfo(NULL, port, ...)，然后遍历链表 socket+bind；bind 前先 `Setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &optval, sizeof(int))` 以消除 "Address already in use" 错误。
  3. getnameinfo 是 getaddrinfo 的逆操作，把 socket 地址转回 host 与 service 字符串，取代过时的 gethostbyaddr/getservbyport，可重入且协议无关；示例用 `flags = NI_NUMERICHOST` 遍历链表打印每个 IP 地址，故 `./hostinfo localhost` 只输出 127.0.0.1。
  4. iterative echo server（迭代式回显服务器）在 `Open_listenfd` 后进入 `while (1)`：`clientlen = sizeof(struct sockaddr_storage)`（注释强调 Important!）、`Accept`、`Getnameinfo` 打印 Connected to (host, port)、`echo(connfd)`、`Close(connfd)`，一次只服务一个客户端。
  5. echo client 用 `Open_clientfd(host, port)` 与 `Rio_readinitb` 初始化缓冲区，循环 `Fgets` 读 stdin、`Rio_writen` 发出、`Rio_readlineb` 收回、`Fputs` 打印；`telnet <host> <portnumber>` 可手工连接服务器测试传输 ASCII 字符串的服务（echo、web、mail）。
  6. HTTP response 由 response line（`<version> <status code> <status msg>`，如 200 OK 表示无错处理、301 Moved 提供替代 URL、404 Not found 服务器找不到文件）、零或多个 response header（`<header name>: <header data>`）以及空行（`\r\n`）之后的 content 组成；content 是带 MIME 类型的字节序列（text/html、text/plain、image/gif、image/png、image/jpeg）。
  7. serve_static 先用 sprintf 依次拼出 `HTTP/1.0 200 OK`、`Server: Tiny Web Server`、`Connection: close`、`Content-length: %d`、`Content-type: %s` 加空行，再 Rio_writen 发出头，随后 `Open` 文件、`Mmap(0, filesize, PROT_READ, MAP_PRIVATE, srcfd, 0)` 映射后按 filesize 发送并 Munmap。
  8. 动态内容 GET 请求的参数放在环境变量 QUERY_STRING 中（即 "?" 之后的整个字符串，如 `"15213&18213"`），子进程用 `strchr(buf, '&')` 切开再 atoi 得到 n1、n2；因为只有 CGI 子进程知道内容的类型与长度，必须由它自己 printf 出 Content-length 与 Content-type 头后再输出正文并 fflush。
- **易错/考点**：Accept 前必须把 clientlen 重置为 sizeof(struct sockaddr_storage) 否则地址被截断；HTTP 头与正文之间必须有空行，且 Content-length 是字节数、须在正文之前给出。
---
