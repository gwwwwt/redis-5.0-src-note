# 客户端结构体 client



> Redis是典型的客户端服务器结构，客户端通过socket与服务端建立网络连接并发送命令请求，服务端处理命令请求并加回复；Redis 使用结构体client存储客户端连接的所有信息，包括但不限于客户端的名称、客户端连接的套接字描述符、客户端当前选择的数据库ID、客户端的输入缓冲区与输出缓冲区等；

```c
//server.h
typedef struct client {
    uint64_t id;            /* Client incremental unique ID. */
    int fd;                 /* Client socket. */
    redisDb *db;            /* Pointer to currently SELECTed DB. */
    robj *name;             /* As set by CLIENT SETNAME. */
    sds querybuf;           /* Buffer we use to accumulate client queries. */
    size_t qb_pos;          /* The position we have read in querybuf. */
    sds pending_querybuf;   /* If this client is flagged as master, this buffer
                               represents the yet not applied portion of the
                               replication stream that we are receiving from
                               the master. */
    size_t querybuf_peak;   /* Recent (100ms or more) peak of querybuf size. */
    int argc;               /* Num of arguments of current command. */
    robj **argv;            /* Arguments of current command. */
    struct redisCommand *cmd, *lastcmd;  /* Last command executed. */
    int reqtype;            /* Request protocol type: PROTO_REQ_* */
    int multibulklen;       /* Number of multi bulk arguments left to read. */
    long bulklen;           /* Length of bulk argument in multi bulk request. */
    list *reply;            /* List of reply objects to send to the client. */
    unsigned long long reply_bytes; /* Tot bytes of objects in reply list. */
    size_t sentlen;         /* Amount of bytes already sent in the current
                               buffer or object being sent. */
    time_t ctime;           /* Client creation time. */
    time_t lastinteraction; /* Time of the last interaction, used for timeout */
    time_t obuf_soft_limit_reached_time;
    int flags;              /* Client flags: CLIENT_* macros. */
    int authenticated;      /* When requirepass is non-NULL. */
    int replstate;          /* Replication state if this is a slave. */
    int repl_put_online_on_ack; /* Install slave write handler on ACK. */
    int repldbfd;           /* Replication DB file descriptor. */
    off_t repldboff;        /* Replication DB file offset. */
    off_t repldbsize;       /* Replication DB file size. */
    sds replpreamble;       /* Replication DB preamble. */
    long long read_reploff; /* Read replication offset if this is a master. */
    long long reploff;      /* Applied replication offset if this is a master. */
    long long repl_ack_off; /* Replication ack offset, if this is a slave. */
    long long repl_ack_time;/* Replication ack time, if this is a slave. */
    long long psync_initial_offset; /* FULLRESYNC reply offset other slaves
                                       copying this slave output buffer
                                       should use. */
    char replid[CONFIG_RUN_ID_SIZE+1]; /* Master replication ID (if master). */
    int slave_listening_port; /* As configured with: SLAVECONF listening-port */
    char slave_ip[NET_IP_STR_LEN]; /* Optionally given by REPLCONF ip-address */
    int slave_capa;         /* Slave capabilities: SLAVE_CAPA_* bitwise OR. */
    multiState mstate;      /* MULTI/EXEC state */
    int btype;              /* Type of blocking op if CLIENT_BLOCKED. */
    blockingState bpop;     /* blocking state */
    long long woff;         /* Last write global replication offset. */
    list *watched_keys;     /* Keys WATCHED for MULTI/EXEC CAS */
    dict *pubsub_channels;  /* channels a client is interested in (SUBSCRIBE) */
    list *pubsub_patterns;  /* patterns a client is interested in (SUBSCRIBE) */
    sds peerid;             /* Cached peer ID. */
    listNode *client_list_node; /* list node in client list */

    /* Response buffer */
    int bufpos;
    char buf[PROTO_REPLY_CHUNK_BYTES];
} client;
```



> id: 客户端唯一ID, 通过全局变量 server.next_client_id 实现;
>
> fd: 客户端socket的文件描述符; 
>
> db: 客户端使用 select 命令选择的数据库对象;
>
> > redisDb结构体定义:
> >
> > ```c
> > //server.h
> > typedef struct redisDb {
> >     dict *dict;                 /* The keyspace for this DB */
> >     dict *expires;              /* Timeout of keys with a timeout set */
> >     dict *blocking_keys;        /* Keys with clients waiting for data (BLPOP)*/
> >     dict *ready_keys;           /* Blocked keys that received a PUSH */
> >     dict *watched_keys;         /* WATCHED keys for MULTI/EXEC CAS */
> >     int id;                     /* Database ID */
> >     long long avg_ttl;          /* Average TTL, just for stats */
> >     
> >     /* List of key names to attempt to defrag one by one, gradually. */
> >     list *defrag_later;         
> > } redisDb;
> > ```
> >
> > > id: 数据库序号, 默认情况下Redis有16个数据库, id序号为 0～15;
> > >
> > > dict: 存储数据库所有键值对;
> > >
> > > expires: 存储键的过期时间;
> > >
> > > avg_ttl: 数据库对象的平均TTL, 用于统计
> > >
> > > blocking_keys 和 ready_keys: 当使用命令"BLPOP"阻塞获取列表元素时, 如果列表为空, 会阻塞客户端, 同时将此列表键记录在blocking_keys;  当使用命令"PUSH"向列表添加元素时, 会从字典 blocking_keys中查找该列表键, 如果找到说明有客户端正阻塞等待获取此列表键, 于是将此列表键记录到字典 ready_keys, 以便后续响应正在阻塞的客户端;
> > >
> > > watch_keys: 在"multi/exec"事务处理期间, 使用"乐观锁"保证关心的数据不会被修改; 开启事务的同时可以使用"watch key" 命令监控关心的数据键, **而watched_keys字典存储的就是被watch命令监控的所有数据键, 其中key-value分别为数据键与客户端对象. 当Redis服务器接收到写命令时, 会从字典 watched_keys中查找该数据键, 如果找到说明有客户端正在监控此数据键, 于是标记客户端对象为dirty; 待Redis服务器收到客户端exec命令时, 如果客户端带有dirty标记, 则会拒绝执行事务;**
>
> name: 客户端名称, 可以使用命令 CLIENT SETNAME 设置;
>
> lastinteraction: 客户端上次与服务器交互的时间, 以此实现客户端的超时处理;
>
> querybuf: 输入缓冲区, recv 函数接收的客户端命令请求会暂时缓存在此缓冲区;
>
> argc: 输入缓冲区的命令请求时按照Redis 协议格式编码字符串, 需要解析出命令请求的所有参数, 参数个数存储在argc字段, 参数内容被解析成robj对象, 存储在argv数组;
>
> cmd: 待执行的客户端命令; 解析命令请求后, 会根据命令名称查找该命令对应的命令对象, 存储在客户端cmd字段;
>
> reply: 输出链表, 存储待返回给客户端的命令回复数据; 链表中节点存储的值类型为 clientReplyBlock:
>
> > ```c
> > //server.h
> > typedef struct clientReplyBlock {
> >     size_t size, used;
> >     char buf[];
> > } clientReplyBlock;
> > ```
>
> reply_bytes: 输出链表中所有节点的存储空间总和;
>
> sentlen: 表示已返回给客户端的字节数; 
>
> buf: 输出缓冲区; 存储待返回给客户端的命令回复数据, bufpos表示输出缓冲区中数据的最大字节位置, 显然sentlen~bufpos 之间的数据都是要返回给客户端;  可以看到reply 和buf 都是用于缓存待返回给客户端的命令回复数据, 为什么同时需要reply 和 buf 的存在呢? 其实主要是二者用于返回不同的数据类型; 



# 服务端结构体 redisServer

> 结构体 redisServer存储Redis服务器的所有信息, 包含但不限于 数据库、配置参数、命令表、监听端口与地址、客户端列表、若干统计信息、RDB与AOF持久化相关信息、主从复制相关信息、集群相关信息等; 
>
> redisServer中的字段非常多, 这里主要简单说明其中的部分字段

```c
//server.h

struct redisServer{
  char *configfile;
  int dbnum;
  redisDb *db;
  dict *commands;
  
  aeEventLoop *el;
  
  int port;
  char *bindaddr[CONFIG_BINDADDR_MAX];
  int bindaddr_count;
  int ipfd[CONFIG_BINDADDR_MAX];
  int ipfd_count;
  
  list *clients;
  int maxidletime;
  
  //...其它字段在以后的说明中陆续补充
}
```

> configfile: 配置文件绝对路径;
>
> dbnum: 数据库数目, 可通过参数databases配置, 默认16;
>
> db: 数据库数组, 每个元素都是redisDb类型;
>
> commands: 命令字典, Redis支持的所有命令, key为命令名称, value为struct redisCommand对象;
>
> el: 事件循环, 类型为aeEventLoop; 
>
> port: 监听端口, 默认6379;
>
> bindaddr: 绑定的所有IP地址, 可通过参数bind配置多个, 如"bind 192.168.1.100 10.0.0.1"; 
>
> bindaddr_count: 用户配置的IP地址数目;  CONFIG_BINDADDR_MAX 常量值为 16;
>
> ipfd: 针对bindaddr字段的所有IP地址创建的socket文件描述符, ipfd_count为创建的socket文件描述符数目;
>
> clients: 当前连接到Redis服务器的所有客户端;
>
> maxidletime: 最大空闲时间, 可通过参数timeout配置; 结合client对象的lastineration字段, 当客户端没有与服务器交互的时间超过 maxidletime时, 会认为客户端超时并释放该客户端连接;