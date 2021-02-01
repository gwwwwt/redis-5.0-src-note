# 从anet.c学习socket编程



> 在Redis epoll多路IO复用中, 为Server Socket以及Client Socket注册的事件函数中, 关于网络连接的操作全部在anet.c中定义; 在anet.c 源码中其实有很多值得借鉴的C 应用;



#### 1.  可变参数va_list及输出vsnprintf

```c
static void anetSetError(char *err, const char *fmt, ...)
{
    va_list ap;

    if (!err) return;
    va_start(ap, fmt);
    vsnprintf(err, ANET_ERR_LEN, fmt, ap); //ANET_ERR_LEN 常量值 256; 用来限制err字符串最大长度;
    va_end(ap);
}
```

> 1. printf 函数族
>
>    > ```c
>    > //下列内容来自 printf 的man page
>    > #include <stdio.h>
>    > 
>    > int printf(const char *format, ...);
>    > int fprintf(FILE *stream, const char *format, ...);
>    > int sprintf(char *str, const char *format, ...);
>    > int snprintf(char *str, size_t size, const char *format, ...);
>    > 
>    > #include <stdarg.h>
>    > 
>    > int vprintf(const char *format, va_list ap);
>    > int vfprintf(FILE *stream, const char *format, va_list ap);
>    > int vsprintf(char *str, const char *format, va_list ap);
>    > int vsnprintf(char *str, size_t size, const char *format, va_list ap);
>    > ```
>    > > 大概总结printf函数族中的函数名前缀: 
>    > >
>    > > f:  FILE指针;
>    > >
>    > > s: 输出到字符串;
>    > >
>    > > n: 包含长度参数
>    > >
>    > > v: va_list类型参数;
>    > >
>    > > 
>    > >
>    > > printf() 和 vprintf() 将内容写至 stdout;
>    > >
>    > > fprintf() 和 vfprintf() 将内容写至指定的输出流; 
>    > >
>    > > sprintf() 和 vsprintf() 将内容写至str 字符串; 
>    > >
>    > > snprinf() 和 vsnprintf() 同样将内容写至str 字符串, 但字符串长度最多size个字节, 包括最后的'\0';
>    > >
>    > > > **需要注意的是, 对于sprintf/snprintf以及对应的vs将输出写至str字符串的函数中,  char* str字符指针必须已经初始化, 如可将str定义为char[size]数组或者str=(char*)malloc(size)等方式;  不然的话, str为null时, 会报"segment fault"错误;**
>    > > >
>    > > > **此外, 对于snprintf/vsnprintf函数中, 若size比可能输出的str字符串长度小, 则str字符串结果会被截断; **
>
> 2. va_list 可变参数列表
>
>    ```c
>    // man stdarg 的示例;   
>    /*
>    va_start函数的大概逻辑是, 使用va_start(ap, fmt) 的第二个参数 来标识在栈桢中传入的参数地址;
>    当然若函数没有其它参数, 则可以调用va_start(ap) 来从第一个参数处就认为是可变参数列表开始处;
>    
>    之后调用va_arg(ap, char *) 或 va_arg(sp, int)等类似 va_arg(ap, type)形式移动对应大小的地址, 
>    并转换成对应的类型;
>    */
>    #include <stdio.h>
>    #include <stdarg.h>
>    
>    void foo(char *fmt, ...)
>    {
>      va_list ap;
>      int d;
>      char c, *s;
>    
>      va_start(ap, fmt);
>      while (*fmt)
>        switch (*fmt++) {
>          case 's':	/* string */
>            s = va_arg(ap, char *);
>            printf("string %s\n", s);
>            break;
>          case 'd':	/* int */
>            d = va_arg(ap, int);
>            printf("int %d\n", d);
>            break;
>          case 'c':	/* char */
>            /* need a cast here since va_arg only
>            	 takes fully promoted types */
>            c = (char) va_arg(ap, int);
>            printf("char %c\n", c);
>            break;
>        }
>     va_end(ap);
>    }
>    ```
>    
>    > ****



#### 2. 设置socket fd 阻塞或非阻塞(fcntl函数应用)

> 摘自《UNIX网络编程》第16章:
>
> 套接字默认状态是阻塞的, 可能阻塞的套接字调用分为四类: 
>
> > 1. 输入操作: read/readv/recv/recvfrom/recvmsg等; **若套接字的接收缓冲区中没有数据可读, 进程则被投入睡眠, 直接有一些数据到达; 这些数据既可能是单个字节, 也可以是一个完整的TCP分节中的数据, 如果想等到某个固定数目的数据可读为止, 那么可以调用readn函数或指定MSG_WAITALL标志;**
> > 2. 待补...

```c
int anetSetBlock(char *err, int fd, int non_block) {
    int flags;

    /*
    使用fcntl设置fd状态时, 需要先通过 F_GETFL 获取旧flags值后, 通过对flags设位(如flags |= O_NONBLOCK); 
    再fcntl F_SETFL的方式设置fd; 这样才不会影响fd的其它状态; 
    */
    if ((flags = fcntl(fd, F_GETFL)) == -1) {
        anetSetError(err, "fcntl(F_GETFL): %s", strerror(errno));
        return ANET_ERR;
    }

    if (non_block)
        flags |= O_NONBLOCK;
    else
        flags &= ~O_NONBLOCK;

    if (fcntl(fd, F_SETFL, flags) == -1) {
        anetSetError(err, "fcntl(F_SETFL,O_NONBLOCK): %s", strerror(errno));
        return ANET_ERR;
    }
    return ANET_OK;
}
```



#### 3. 设置套接字选项(setsockopt和getsockopt)

> 这里主要以anetSetReuseAddr、anetSetTcpNoDelay、anetV6Only 和 anetKeepAlive 为例;
>
> 说明设置的不同选项以及它们的作用;

```c
/* 摘自UNP 7.5.11节: 关于 SO_REUSEADDR 和 SO_REUSEPORT 套接字选项
SO_REUSEADDR能起到4个不同的作用: 
1.允许启动一个监听服务器并绑定到一个端口, 即使以前建立的将该端口用作它们的本地端口的连接仍存在; 这个条件通常是在: 
	1.a 启动一个监听服务器;
	1.b 连接请求到达, 派生一个子进程来处理这个客户; 
	1.c 监听服务器终止, 但子进程继续为现有连接上的客户提供服务; 
	1.d 重启监听服务器; 当监听服务器在通过socket/bind/listen想重新启动时, 由于它试图捆绑一个现有连接(即子进程)
			上的端口, 从而bind调用会失败; 但如果该套接字上设置了SO_REUSEADDR选项, bind将成功; 
2.允许在同一端口上启动同一服务器的多个实例, 只要每个实例捆绑不同的本地IP地址即可(如使用IP别名技术托管多个HTTP
	服务器网点); 但同样需要注意的是, 即使设置了本选项, 也不可能将相同的IP/端口号的socket再次绑定;
3.允许单个进程绑定同一端口到多个套接字上, 只要每次指定不同的本地IP地址;
4.(只对UDP套接字来说), 允许完全重复的绑定;
*/
static int anetSetReuseAddr(char *err, int fd) {
    int yes = 1;
    /* Make sure connection-intensive things like the redis benchmark
     * will be able to close/open sockets a zillion of times */
    if (setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &yes, sizeof(yes)) == -1) {
        anetSetError(err, "setsockopt SO_REUSEADDR: %s", strerror(errno));
        return ANET_ERR;
    }
    return ANET_OK;
}


//本函数和下面的anetDisableTcpNodelay函数都是通过调用anetSetTcpNoDelay工具函数
//来设置socket NODELAY选项的开启和关闭;
int anetEnableTcpNoDelay(char *err, int fd) 
{
    return anetSetTcpNoDelay(err, fd, 1);
}

int anetDisableTcpNoDelay(char *err, int fd)
{
    return anetSetTcpNoDelay(err, fd, 0);
}

/*
工具函数, 根据val参数设置或取消TCP_NODELAY选项;

关于TCP_NODELAY, 由于TCP/IP协议基于字节流传输, 默认情况下会开启Nagle; 
当应用层调用write函数发送数据时, TCP并不一定会立刻将数据发送出去, 根据Nagle算法, 还必须满足一定条件: 
1. 数据包长度大于一定门限, 立即发送;
2. 数据包中含有FIN字段, 立即发送;
3. 设置了TCP_NODELAY选项, 立即发送; 
如果以上条件都不满足, 默认需要等待200毫秒超时后才会发送; 
Redis为了能立即将回复发送给客户端, 因此需要设置 TCP_NODELAY;
*/
static int anetSetTcpNoDelay(char *err, int fd, int val)
{
    if (setsockopt(fd, IPPROTO_TCP, TCP_NODELAY, &val, sizeof(val)) == -1)
    {
        anetSetError(err, "setsockopt TCP_NODELAY: %s", strerror(errno));
        return ANET_ERR;
    }
    return ANET_OK;
}

/*
设置SO_KEEPALIVE选项后: 如果2小时内在该套接字的任一方向上都没有数据交换, TCP就自动给对端发送一个"保持存活探测分节(keep-alive probe)", 这是对端必须响应的TCP分节, 可能会导致如下三种情况之一: 
1. 对端以期望的ACK响应, 则应用程序得不到通知(因为底层套接字认为一切正常); 
2. 对端响应RST告知本端TCP: 对端已崩溃并已重新启动; errno值被设置为ECONNRESET, 套接字本身被关闭; 
3. 对端没有响应; 源自Berkeley的TCP将另外发送8个探测分节, 两两相隔75秒,试图得到一个响应; TCP在发出第一个探测分节后11分15秒内若没有得到任何响应则放弃;
*/
int anetTcpKeepAlive(char *err, int fd)
{
    int yes = 1;
    if (setsockopt(fd, SOL_SOCKET, SO_KEEPALIVE, &yes, sizeof(yes)) == -1) {
        anetSetError(err, "setsockopt SO_KEEPALIVE: %s", strerror(errno));
        return ANET_ERR;
    }
    return ANET_OK;
}

/*
本函数和上面的anetTcpKeepAlive函数的区别: 它们同样设置了SO_KEEPALIVE套接字选项;
但本函数还在测试如果在Linux环境下, 设置了:
TCP_KEEPIDLE: 在anetTcpKeepAlive函数注释上说明了, 默认2小时之后才发送keep-alive probe, 本选项好像想将它的
							发送间隔设置成参数 interval秒后; 
TCP_KEEPINTVL:设置keep-alive probe分节的发送间隔, 上面是说默认75秒, 这里改成interval/3秒后发送下一个分节; 
TCP_KEEPCNT: keep-alive probe分节发送数量; 上面是说默认8个, 本函数中设置了3个;

需要特别说明的是: 在UNP 7.2节中关于setsockopt/getsockopt选项图表中, 没有列出关于上面三个参数的说明, 所以关于它们的解释也是通过选项名推测出的, 后面可能需要验证;!!!
另外, 参考Redis中的默认参数"server.tcpkeepalive=300"; 所以根据上面的猜测, 在客户端连接服务端之后, 5分钟后发送keep-alive probe分节, 之后每隔100秒发送一次, 共发3次, 所以10分钟之后Redis会认为连接超时;
*/
int anetKeepAlive(char *err, int fd, int interval)
{
    int val = 1;

    if (setsockopt(fd, SOL_SOCKET, SO_KEEPALIVE, &val, sizeof(val)) == -1)
    {
        anetSetError(err, "setsockopt SO_KEEPALIVE: %s", strerror(errno));
        return ANET_ERR;
    }

#ifdef __linux__
    /* Default settings are more or less garbage, with the keepalive time
     * set to 7200 by default on Linux. Modify settings to make the feature
     * actually useful. */

    /* Send first probe after interval. */
    val = interval;
    if (setsockopt(fd, IPPROTO_TCP, TCP_KEEPIDLE, &val, sizeof(val)) < 0) {
        anetSetError(err, "setsockopt TCP_KEEPIDLE: %s\n", strerror(errno));
        return ANET_ERR;
    }

    /* Send next probes after the specified interval. Note that we set the
     * delay as interval / 3, as we send three probes before detecting
     * an error (see the next setsockopt call). */
    val = interval/3;
    if (val == 0) val = 1;
    if (setsockopt(fd, IPPROTO_TCP, TCP_KEEPINTVL, &val, sizeof(val)) < 0) {
        anetSetError(err, "setsockopt TCP_KEEPINTVL: %s\n", strerror(errno));
        return ANET_ERR;
    }
  
    /* Consider the socket in error state after three we send three ACK
     * probes without getting a reply. */
    val = 3;
    if (setsockopt(fd, IPPROTO_TCP, TCP_KEEPCNT, &val, sizeof(val)) < 0) {
        anetSetError(err, "setsockopt TCP_KEEPCNT: %s\n", strerror(errno));
        return ANET_ERR;
    }
#else
    ((void) interval); /* Avoid unused var warning for non Linux systems. */
#endif

    return ANET_OK;
}

//设置发送缓冲区大小; 
//另外还有一个参数: SO_RCVBUF; 用于设置接收缓冲区大小;
int anetSetSendBuffer(char *err, int fd, int buffsize)
{
    if (setsockopt(fd, SOL_SOCKET, SO_SNDBUF, &buffsize, sizeof(buffsize)) == -1)
    {
        anetSetError(err, "setsockopt SO_SNDBUF: %s", strerror(errno));
        return ANET_ERR;
    }
    return ANET_OK;
}

//设置发送超时, 需要注意的是 timeval中的两个字段: tv_sec单位为秒, 而tv_usec单位为微秒;
//另外还有一个参数: SO_RCVTIMEO; 用于设置接收超时时间;
int anetSendTimeout(char *err, int fd, long long ms) {
    struct timeval tv;

    tv.tv_sec = ms/1000;
    tv.tv_usec = (ms%1000)*1000;
    if (setsockopt(fd, SOL_SOCKET, SO_SNDTIMEO, &tv, sizeof(tv)) == -1) {
        anetSetError(err, "setsockopt SO_SNDTIMEO: %s", strerror(errno));
        return ANET_ERR;
    }
    return ANET_OK;
}

/*
设置socket禁止v4兼容; 在Redis中, 本选项也比较重要;

在Redis默认逻辑中, 会查找Server服务器的所有本机IP(既可以是IPv4也可是IPv6地址), 然后将端口
*/
static int anetV6Only(char *err, int s) {
    int yes = 1;
    if (setsockopt(s,IPPROTO_IPV6,IPV6_V6ONLY,&yes,sizeof(yes)) == -1) {
        anetSetError(err, "setsockopt: %s", strerror(errno));
        close(s);
        return ANET_ERR;
    }
    return ANET_OK;
}
```



#### 4. 获取IP地址信息

> 参考《UNIX网络编程》第11章; 本节最主要就是关于getaddrinfo API的使用;

##### 4.1 gethostbyname

> gethostbyname: 查询DNS, 获取主机名对应的IP地址;
>
> gethostbyaddr: 根据IP地址获取主机名, gethostbyname的逆向;
>
> > ```c
> > #include <netdb.h>
> > 
> > //对于gethostbyname的调用, 如果查询出错, 不会设置errno全局变量, 而是设置本h_errno变量;
> > //可调用下面定义的hstrerror获取它对应的字符串解释;
> > extern int h_errno; 
> > 
> > /*
> > struct hostent {
> > 	char  *h_name;	//DNS A类型记录中的official name           
> >   	char **h_aliases; //别名, 可能有多个, 所以为字符串数组;      
> >   	int    h_addrtype;	//地址类型;   
> >   	int    h_length;    //地址长度; IPv4地址的本字段值为4;
> >   	char **h_addr_list; //一个hostname可能对应多条IP地址, 所以同样是字符串数组;       
> > }
> > */
> > struct hostent *gethostbyname(const char *name);
> > 
> > #include <sys/socket.h>       /* for AF_INET */
> > struct hostent *gethostbyaddr(const void *addr, socklen_t len, int type);
> > const char *hstrerror(int err);
> > ```
> >
> > 
>
> ```c
> //gethostbyname示例;
> //命令行调用示例: hostent cnn.com
> 
> #include "unp.h"
> 
> int main(int argc, char* argv[]) {
>     char *ptr, **pptr;
>     char str[INET_ADDRSTRLEN];
> 
>     struct hostent *hptr;
> 
>     while(--argc > 0) {
>         ptr = *++argv;
>         if ((hptr = gethostbyname(ptr)) == NULL) {
>             err_msg("gethostbyname error for host: %s: %s", ptr, hstrerror(h_errno));
>             continue;
>         }
> 
>         printf("official hostname: %s\n", hptr->h_name);
> 
>         for(pptr = hptr->h_aliases; *pptr!=NULL; pptr++) {
>             printf("\talias: %s\n", *pptr);
>         }
> 
>         switch(hptr->h_addrtype) {
>             case AF_INET:
>                 pptr = hptr->h_addr_list;
>                 for( ; *pptr!=NULL; pptr++)
>                     printf("\taddress: %s\n", Inet_ntop(hptr->h_addrtype, *pptr, str, sizeof(str)));
>                 break;
>             default:
>                 err_ret("unknown address type");
>                 break;
>         }
>     }
>     exit(0);
> }
> ```
>
> 



##### 4.2 getservbyname和getservbyport

> **可以理解成解析/etc/services文件中的内容, 将每行记录转换成struct servent表示;**
>
> **servent中最主要的就是s_port端口号(结果已经是网络字节序); 可以修改/etc/services文件来设置每个服务的端口;**
>
> ```c
> struct servent {
>   char  *s_name;       /* official service name */
>   char **s_aliases;    /* alias list */
>   int    s_port;       /* port number */
>   char  *s_proto;      /* protocol to use */
> }
> ```



##### 4.3 getaddrinfo函数

> **getaddrinfo函数可以理解成综合了gethostbyname/getservbyname等函数的功能, 并添加了IPv6支持;**
>
> **能够处理名字到地址以及服务到端口这两种转换, 返回的是一个sockaddr结构, 而不是一个地址列表; 这些sockaddr结构随后可由套接字函数直接使用; 这样getaddrinfo函数把协议相关性完全隐藏在这个库函数内部;**
>
> > ```c
> > //man page
> > #include <sys/types.h>
> > #include <sys/socket.h>
> > #include <netdb.h>
> > 
> > int getaddrinfo(const char *node, const char *service,const struct addrinfo *hints,
> >                    struct addrinfo **res);
> > 
> > void freeaddrinfo(struct addrinfo *res);
> > 
> > const char *gai_strerror(int errcode);
> > 
> > #include <netdb.h>
> > struct addrinfo {
> > 	int              ai_flags;
> > 	int              ai_family;
> >   	int              ai_socktype;
> >   	int              ai_protocol;
> >   	socklen_t        ai_addrlen;
> >   	struct sockaddr *ai_addr;
> >   	char            *ai_canonname;
> >   	struct addrinfo *ai_next; //多个地址列表由ai_next字段进行链接;
> > };
> > ```



##### 4.4 getaddrinfo 示例(自已写的一个小Demo)

```c
//getaddrinfo_test.c
/*
本示例只调用了bind/listen步骤, 并没有真正调用Accept等待客户端连接, 但应该也够说明问题了; 
本示例参考的就是Redis源代码 anet.c 中的逻辑;

具体见下方代码说明; 
*/
#include <sys/types.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <netdb.h>
#include <stdio.h>
#include <stdlib.h>

#include <string.h>
#include <unistd.h>
#include <errno.h>

extern int errno;
char errMsg[128];

void static tcpBindAndListen(int af, short int port, char* destHost) {
    char cip[256];
    char cport[6];

    snprintf(cport, sizeof(cport) -1, "%d", port);

    int result;
    struct addrinfo hints, *serverinfo;

    if (destHost == NULL) {
        memset(&hints, 0, sizeof(hints));
        hints.ai_flags = AI_PASSIVE;
        hints.ai_family = af;
        hints.ai_socktype = SOCK_STREAM;

        result = getaddrinfo(NULL, cport, &hints, &serverinfo);

        if (result != 0) {
            printf("getaddrinfo fail! result count is %d\n", result);
            return;
        } else {
            printf("getaddrinfo success\n");
            
            struct addrinfo *p;
            for (p = serverinfo; p != NULL; p = p->ai_next) {
                int server_socket;
                server_socket = socket(p->ai_family, p->ai_socktype, p->ai_protocol);

                printf("socket fd is %d\n", server_socket);

                int yes = 1;
                if (setsockopt(server_socket, SOL_SOCKET, SO_REUSEADDR, &yes, sizeof(yes)) == -1) {
                    printf("Socket Reuse option set error!!\n");
                    exit(-1);
                }

                if (af == AF_INET6) {
                    if (setsockopt(server_socket,IPPROTO_IPV6,IPV6_V6ONLY,&yes,sizeof(yes)) == -1) {
                        printf("IPV6 Socket IPv6 Only set error!!\n");
                    }
                }

                bind(server_socket, p->ai_addr, p->ai_addrlen);

                listen(server_socket, 10);

                //打印结果网址对应的字符串表示;
                if (af == AF_INET) {
                    struct sockaddr_in *s = (struct sockaddr_in *)(p->ai_addr);
                    inet_ntop(AF_INET,(void*)&(s->sin_addr), cip, 255);
                } else if (af == AF_INET6) {
                    struct sockaddr_in6 *s = (struct sockaddr_in6 *)(p->ai_addr);
                    inet_ntop(AF_INET6,(void*)&(s->sin6_addr), cip, 255);
                } else {
                    printf("Error address family!\n");
                    exit(-1);
                }

                if (errno != 0) {
                    strerror_r(errno, errMsg, 127);
                    printf("Bind address error, %s\n", errMsg);
                    exit(-1);
                } else {
                    printf("Binded IP addr: %s\n", cip);
                }
            }
        }
    } else {
        memset(&hints, 0, sizeof(hints));
        hints.ai_flags = AI_PASSIVE;
        hints.ai_family = af;
        hints.ai_socktype = SOCK_STREAM;

        result = getaddrinfo(destHost, cport, &hints, &serverinfo);

        if (result != 0) {
            printf("Failed get address info for %s\n", destHost);
            exit(-1);
        }

        struct addrinfo *p;
        for (p = serverinfo; p != NULL; p = p->ai_next) {
            if (af == AF_INET) {
                struct sockaddr_in *s = (struct sockaddr_in *)(p->ai_addr);
                inet_ntop(AF_INET,(void*)&(s->sin_addr), cip, 255);
            } else if (af == AF_INET6) {
                struct sockaddr_in6 *s = (struct sockaddr_in6 *)(p->ai_addr);
                inet_ntop(AF_INET6,(void*)&(s->sin6_addr), cip, 255);
            } else {
                printf("Error address family!\n");
                exit(-1);
           }
        }

       printf("Remote Host address %s : %s\n", destHost, cip);

    }
}

int main(int argc, char* argv[]) {

    tcpBindAndListen(AF_INET, 8080, NULL);
    tcpBindAndListen(AF_INET6, 8080, NULL);
    tcpBindAndListen(AF_INET, 443, "www.baidu.com");

    while(1){
        sleep(3);
        printf("Loop ...\n");
    }
}
```

> 中间涉及到的API调用及其作用: 
> 1. getaddrinfo: 获取IP信息; 需要注意的是 **getaddrinfo的第一个参数 和 "hints.ai_flags = AI_PASSIVE" 一行;**
>
>     > getaddrinfo第一个参数为NULL的情况：
>     >
>     > a. 若设置了AI_PASSIVE这一标志位时， **表示获取本机通配地址；**
>     >
>     > > 返回的IP信息其实是可直接传入 bind 的通用IP信息(即INADDR_ANY或IN6ADDR_ANY_INIT);
>     > >
>     > > 举例为：当按如下示例代码逻辑获取通用IP信息并绑定后, 无论本机以及远程机器上通过调用telnet都可连接上本机；
>     > >
>     > > 原因是它是通配符绑定, 会监听所有连接;
>     >
>     > b. 若没有设置 AI_PASSIVE 标志位，会返回本机的 **环回地址；**
>     >
>     > > 返回结果实际为环回地址(127.0.0.1 或 IPv6版本的环回地址)
>     > >
>     > > 举例为: 当注释 AI_PASSIVE 一行且只监听IPv4时, 只能本机telnet连接, 远程机器telent无法连接; 
>     > >
>     > > 原因是它绑定的只是环回地址, 即只允许目标地址为127.0.0.1的连接; 而远程telnet肯定是外网IP连接, 所以无法匹配；
>
>     
>
>     > getaddrinfo第一个参数不为NULL的情况：
>     >
>     > 无论是否设置了 AI_PASSIVE 标志，方法都会获取 **第一个参数(即主机域名) + 第二个参数(即端口)**  ***所对应的IP地址；***
>     >
>     > **此时getaddrinfo的返回结果可被直接传入 connect/sendto/recvfrom 等函数中的远程主机参数；**
>
> 2. setsockopt(server_socket,IPPROTO_IPV6,IPV6_V6ONLY,&yes,sizeof(yes); 的作用
>
>     > 在上面的示例代码中，getaddrinfo的第一个参数为 NULL 且设置了 AI_PASSIVE 标志的情况下， 会获取本机的 **IPv4 和 IPv6 两个版本的通配符IP地址；**
>     >
>     > **代码的本意其实是想通过设置 SO_REUSEADDR 这个选项支持 IPv4 和 IPv6 两个通配符地址同时监听同一端口，但即使设置了这个选项， 后面绑定 IPv6通配符时还是会报 地址/端口 被占用的错误；**
>     >
>     > **此时就需要设置 IPV6_V6ONLY， 为IPv6通配符设置此选项后，即可以实现两个版本的通配符监听同一端口的功能；**
>     >
>     > **本选项也是 Redis 服务器启动后会同时在6379监听 IPv4和IPv6 连接的原因；**



##### 4.5 anet.c中的 getaddrinfo 第一个参数非NULL 的应用

> 参考 4.4 节中对 getaddrinfo 参数的说明，尤其是对于它第一个参数不为NULL 的说明，来加深理解 getaddrinfo

```C 
// 获取 host参数对应的 IP地址; 
// host 字符串 可以是 服务器域名, 也可以是 IP地址;
int anetResolve(char *err, char *host, char *ipbuf, size_t ipbuf_len) {
    return anetGenericResolve(err,host,ipbuf,ipbuf_len,ANET_NONE);
}

//同样获取 host参数对应的 IP地址; 但 host字符串 则必须是 IP地址而不能是域名
int anetResolveIP(char *err, char *host, char *ipbuf, size_t ipbuf_len) {
    return anetGenericResolve(err,host,ipbuf,ipbuf_len,ANET_IP_ONLY);
}

//上面两个函数实际调用的工具函数;
int anetGenericResolve(char *err, char *host, char *ipbuf, size_t ipbuf_len, int flags)
{
    struct addrinfo hints, *info;
    int rv;

    memset(&hints,0,sizeof(hints));
  
    //根据flags参数, 决定是否设置 AI_NUMERICHOST; 如果设置此标志, 则只处理IP host 而不处理域名host
    if (flags & ANET_IP_ONLY) hints.ai_flags = AI_NUMERICHOST;
    hints.ai_family = AF_UNSPEC;
    hints.ai_socktype = SOCK_STREAM;  /* specify socktype to avoid dups */

    if ((rv = getaddrinfo(host, NULL, &hints, &info)) != 0) {
        anetSetError(err, "%s", gai_strerror(rv));
        return ANET_ERR;
    }
    //getaddrinfo获取到了对应主机的IP信息, 然后将信息写入 ipbuf缓冲区中;
    if (info->ai_family == AF_INET) {
        struct sockaddr_in *sa = (struct sockaddr_in *)info->ai_addr;
        inet_ntop(AF_INET, &(sa->sin_addr), ipbuf, ipbuf_len);
    } else {
        struct sockaddr_in6 *sa = (struct sockaddr_in6 *)info->ai_addr;
        inet_ntop(AF_INET6, &(sa->sin6_addr), ipbuf, ipbuf_len);
    }

    freeaddrinfo(info); //最后如果长时间运行的程序的话, 不要忘记释放 info信息;
    return ANET_OK;
}
```



#### 5. accept

> 获取连接

```c
//anet.c
int anetTcpAccept(char *err, int s, char *ip, size_t ip_len, int *port) {
    int fd;
    struct sockaddr_storage sa;
    socklen_t salen = sizeof(sa);
    
    //调用accept获取用户端网络信息以及对应的client socket fd;
    if ((fd = anetGenericAccept(err,s,(struct sockaddr*)&sa,&salen)) == -1)
        return ANET_ERR;

    //将用户端ip地址以及端口port信息写到ip和port中返回;
    if (sa.ss_family == AF_INET) {
        struct sockaddr_in *s = (struct sockaddr_in *)&sa;
        if (ip) inet_ntop(AF_INET,(void*)&(s->sin_addr),ip,ip_len);
        if (port) *port = ntohs(s->sin_port);
    } else {
        struct sockaddr_in6 *s = (struct sockaddr_in6 *)&sa;
        if (ip) inet_ntop(AF_INET6,(void*)&(s->sin6_addr),ip,ip_len);
        if (port) *port = ntohs(s->sin6_port);
    }
    return fd;
}
```

