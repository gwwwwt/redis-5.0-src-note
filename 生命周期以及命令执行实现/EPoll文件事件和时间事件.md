# Redis中EPoll机制应用



### 1. 简介

> Redis为了支持不同的操作系统（而有些操作系统只实现了Select，并没有Epoll）的情况，
>
> 所以Redis定义了 **struct aeEventLoop**结构体包装了**不同操作系统的IO多路复用实现；**

### 2.  aeEventLoop封装

```C
// ae.h
typedef struct aeEventLoop {
    int maxfd;   /* highest file descriptor currently registered */
    int setsize; /* max number of file descriptors tracked */
    long long timeEventNextId;
    time_t lastTime;     /* Used to detect system clock skew */
    aeFileEvent *events; /* Registered events */
    aeFiredEvent *fired; /* Fired events */
    aeTimeEvent *timeEventHead;
    int stop;
    void *apidata; /* This is used for polling API specific data */
    aeBeforeSleepProc *beforesleep;
    aeBeforeSleepProc *aftersleep;
} aeEventLoop;
```

#### 2.1 相关字段

1. maxfd: 当前监听的最大客户端socket fd值;

2. setsize: 初始化aeEventLoop可监听的最大fd数量； 结构体中的其它字段包括 events数组以及apidata中的数组长度都会根据这个值进行初始化；  

   而整个初始化aeEventLoop变量时也就是需要往函数中传入 **setsize** 这个参数值，其它参数都不需要；

3. events: 文件事件数组，存储已注册的文件事件；这里events中的数据为 aeFileEvent 类型；

   ```C
   // ae.h
   /*
   在aeEventLoop中, events存储已注册的文件事件; 而文件事件就存放在events数组中以文件fd为索引的位置处
   */
   typedef struct aeFileEvent {
       int mask; /* one of AE_(READABLE|WRITABLE|BARRIER) */
     
       //rfileProc和wfileProc分别对应文件fd可读或可写时对应的操作函数指针
       aeFileProc *rfileProc; 
       aeFileProc *wfileProc;
     
       void *clientData; //在监听到读/写事件后,调用处理函数时, clientData会作为处理函数的参数传入;
   } aeFileEvent;
   ```

4. timeEventHead: **时间事件链表头节点； 链表节点分别对应不同的定时任务;**

   > 关于 **文件事件**以及**时间事件**，会在下面进行说明；

   ```C
   /* Time event structure */
   typedef struct aeTimeEvent {
       long long id; /* time event identifier. */
       long when_sec; /* seconds */
       long when_ms; /* milliseconds */
       aeTimeProc *timeProc;
       aeEventFinalizerProc *finalizerProc;
       void *clientData;
       struct aeTimeEvent *prev;
       struct aeTimeEvent *next;
   } aeTimeEvent;
   ```

   

5. fired: 每次epoll_wait获取到的已触发的文件事件结果； 类型为 aeFiredEvent数组； 但epoll_wait API函数的返回结果是 epoll_event类型数组；Redis会在每次得到结果之后遍历结果的 epoll_wait数组，读取相关信息后再转成对应的 aeFiredEvent数组结果；

   ```C
   // ae.h
   /* A fired event */
   typedef struct aeFiredEvent {
       int fd;
       int mask;
   } aeFiredEvent;
   ```

6. apidata: 在epoll中，apidata实际类型为 **struct aeApiState*** 指针类型；而apidata其实是真正封装了epoll api操作的结构体数据类型；

   ```C
   /*
   在 ae.c中, 会依次检查当前操作系统是否支持 evport\epoll\kqueue IO多路复用实现, 如果支持, 则会将对应的 ae_***.c include到 ae.c中; 在这些 ae_***.c中, 都将对应的IO操作包装成了统一的函数名;
   
   如: aeApiCreate\aeApiAddEvent\aeApiDelEvent\aeApiPoll;
   
   通过这种方式, ae.c实现了统一的IO多路复用操作;
   */
   // ae_epoll.c
   /*
   参考上面关于evport\epoll\kqueue的说明, 其实在aeEventLoop结构体中,  apidata就是指向实际进行多路复用操作的数据结构, 所以apidata被定义为 void* 指针类型;
   */
   typedef struct aeApiState {
       int epfd;
       struct epoll_event *events;
   } aeApiState;
   ```

7. beforesleep 和 aftersleep: 在进入IO wait之前以及从wait状态返回之后的相关操作函数指针； 在创建aeEventLoop结构体时，它们初始化为NULL; 而在server.c main()函数启动时，将它们分别指向了 main.c 中定义的 **beforeSleep函数 和 afterSleep函数**，关于它们的说明， 见下面；

### 3. epoll文件事件处理实现

#### 3.1 创建aeEventLoop结构体 aeCreateEventLoop

> 在server.c initServer()函数中进行初始化：
>
> **void initServer(void) {**
>
> **//...**
>
> ​    **server.el = aeCreateEventLoop(server.maxclients+CONFIG_FDSET_INCR);**
>
> **//...**
>
> **}**

```C
//ae.c
// 参数setsize: epoll同时监听的最大fd数量
aeEventLoop *aeCreateEventLoop(int setsize) {
    aeEventLoop *eventLoop;
    int i;

    //1. 分配aeEventLoop内存
    if ((eventLoop = zmalloc(sizeof(*eventLoop))) == NULL) goto err;
  
    //2. 为events和fired字段指向新分配的内存区域;
    //由于events和fired分别为每个fd注册的事件以及为每个fd触发事件; 所以它们的长度也得是setsize;
    eventLoop->events = zmalloc(sizeof(aeFileEvent)*setsize);
    eventLoop->fired = zmalloc(sizeof(aeFiredEvent)*setsize);
    //... 省略错误检查
  
    //3. 将内部其它字段设置为默认值;
    eventLoop->setsize = setsize;
    eventLoop->lastTime = time(NULL);
    eventLoop->timeEventHead = NULL;
    eventLoop->timeEventNextId = 0;
    eventLoop->stop = 0;
    eventLoop->maxfd = -1;
    eventLoop->beforesleep = NULL;
    eventLoop->aftersleep = NULL;
  
    //4. 初始化epoll相关数据
    if (aeApiCreate(eventLoop) == -1) goto err;
  
    //5. 因为服务器在启动中, 现在没有客户端连接, 所以将events数组中的所有注册事件清空;
    /* Events with mask == AE_NONE are not set. So let's initialize the
     * vector with it. */
    for (i = 0; i < setsize; i++)
        eventLoop->events[i].mask = AE_NONE;
    return eventLoop;

err: //... 错误处理, 略
}
```

##### 3.1.1 初始化epoll相关数据

```C
//ae_epoll.c
//结合前面的说明, aeApiState这种用于epoll操作的数据结构肯定是定义在 ae_epoll.c中
typedef struct aeApiState {
    int epfd; //用来保存创建的epoll监听套接字
    /*
    根据上面关于 aeEventLoop.fired字段的说明, fired用于保存epoll_wait之后经过转换后的结果;
    
    而直接的epoll_wait的直接的epoll_event[]数组结果则会返回到 aeApiState.events中
    */
    struct epoll_event *events; //用来储存epoll_wait的返回结果
} aeApiState;

//创建epoll相关数据
/*
aeApiState.epfd 存储的是调用epoll_creaete生成的文件描述符

aeApiState.events 初始化之后会被用来做为传入epoll_wait的参数以及epoll_wait返回时存储触发的事件信息;
*/
static int aeApiCreate(aeEventLoop *eventLoop) {
    //1. 分配内存
    aeApiState *state = zmalloc(sizeof(aeApiState));
    if (!state) return -1;
  
    //2. 因为eventLoop最多监听setsize数量的fd, 所以这里events做为存储epoll_wait函数返加结果的数组，它的长度也必须是setsize个；
    state->events = zmalloc(sizeof(struct epoll_event)*eventLoop->setsize);
    if (!state->events) {
        zfree(state);
        return -1;
    }
  
    //3. epoll_create api调用, 创建
    state->epfd = epoll_create(1024); /* 1024 is just a hint for the kernel */
    if (state->epfd == -1) {
        zfree(state->events);
        zfree(state);
        return -1;
    }
    eventLoop->apidata = state;
    return 0;
}
```



#### 3.2 注册需要监听的文件事件 aeCreateFileEvent

> 在server.c initServer()函数体中:
>
> void initServer(void) {
>
> ...
>
> for (j = 0; j < server.ipfd_count; j++) {      **//注册server socket监听文件事件以及其 handler 函数**
>         if (aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,acceptTcpHandler,NULL) == AE_ERR) {
>                 serverPanic( "Unrecoverable error creating server.ipfd file event.");
>             }
>    }
>    
> ...
>
> }

```C
// ae.c
// 注册文件事件
/*
fd参数: 需要监听的文件描述符
mask参数: 标志, 标识需要监听fd参数的哪些事件
proc: 当fd上发生mask标识的事件时, 需要调用的函数指针
clientData: 在调用proc时可能会用到的参数
*/
int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask, aeFileProc *proc, void *clientData)
{
    //1. 检查fd是否超过了最大监听描术符限制, 若有, 返回错误
    if (fd >= eventLoop->setsize) {
        errno = ERANGE;
        return AE_ERR;
    }
    //2.获取fd在aeEventLoop.events数组中对应的aeFileEvent结构体;用于更新它的mask标志以及存储Handler函数
    aeFileEvent *fe = &eventLoop->events[fd];

    //3.调用epoll api函数, 添加或更新fd描述符对应的epoll_event
    if (aeApiAddEvent(eventLoop, fd, mask) == -1)
        return AE_ERR;
    
    //4. 更新fe中的相关信息;
    fe->mask |= mask;
    if (mask & AE_READABLE) fe->rfileProc = proc;
    if (mask & AE_WRITABLE) fe->wfileProc = proc;
    fe->clientData = clientData;
  
    //5. 更新 aeEventLoop.maxfd值
    if (fd > eventLoop->maxfd)
        eventLoop->maxfd = fd;
    return AE_OK;
}
```

##### 3.2.1  关于 mask参数

> aeCreateFileEvent函数接受的mask参数定义在 ae.h 头文件中；
>
> > **需要注意AE_BARRIER标志的作用；**

```C 
// ae.h

#define AE_NONE 0       /* No events registered. */
#define AE_READABLE 1   /* Fire when descriptor is readable. */
#define AE_WRITABLE 2   /* Fire when descriptor is writable. */
#define AE_BARRIER 4    /* With WRITABLE, never fire the event if the
                           READABLE event already fired in the same event
                           loop iteration. Useful when you want to persist
                           things to disk before sending replies, and want
                           to do that in a group fashion. */
```

##### 3.2.2 添加或更新fd在epoll中对应的监听参数 aeApiAddEvent

```C
// ae_epoll.c
static int aeApiAddEvent(aeEventLoop *eventLoop, int fd, int mask) {
  	// 1. 获取aeEventLoop.state字段, 因为关于epoll相关的操作都由它来完成
    aeApiState *state = eventLoop->apidata;
  
    // 2. 在调用epoll api更新监听信息时需要传入struct epoll_event类型参数, 所以这里创建一个新局部变量
    struct epoll_event ee = {0}; /* avoid valgrind warning */
    
    // 3. 根据aeEventLoop.events[fd].mask值是否是AE_NONE, 来确定是采用添加操作 还是采用更新操作
    int op = eventLoop->events[fd].mask == AE_NONE ? EPOLL_CTL_ADD : EPOLL_CTL_MOD;

    ee.events = 0;
    mask |= eventLoop->events[fd].mask; /* Merge old events */
    
    // 4. 根据mask, 转为在epoll_event中需要的标志; 即 AE_READABLE=>EPOLLIN； AE_WRITABLE=>EPOLLOUT
    if (mask & AE_READABLE) ee.events |= EPOLLIN; 
    if (mask & AE_WRITABLE) ee.events |= EPOLLOUT;
    ee.data.fd = fd;
    
    // 5. epoll_ctl api调用; op为第3步中的结果;
    if (epoll_ctl(state->epfd,op,fd,&ee) == -1) return -1;
    return 0;
}
```

#### 3.3 开启事件监听 aeMain

> **在 server.c main()函数体最后包含如下三行：**
>
> **aeSetBeforeSleepProc(server.el,beforeSleep); //设置beforeSleep函数**
>
> **aeSetAfterSleepProc(server.el,afterSleep); //设置afterSleep函数**
>
> **aeMain(server.el); //server.el的初始化在initServer()函数中**

```C
// ae.c
/*
事件循环一般都存在while/for循环中，等待事件发生并进行处理；
*/
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);
        aeProcessEvents(eventLoop, AE_ALL_EVENTS|AE_CALL_AFTER_SLEEP);
    }
}
```

##### 3.3.1 关于传入到aeProcessEvents函数的第2个参数，标志位

```C
// ae.h

#define AE_FILE_EVENTS 1 //只处理文件事件
#define AE_TIME_EVENTS 2 //只处理时间事件
#define AE_ALL_EVENTS (AE_FILE_EVENTS|AE_TIME_EVENTS) //处理文件事件和时间事件
#define AE_DONT_WAIT 4   //...不等待文件事件, 将timeout设置为0

//上面aeMain中未显式调用eventLoop->aftersleep函数；需要指定这个标志，才会在epoll_wait返回时调用该函数
#define AE_CALL_AFTER_SLEEP 8 
```



##### 3.3.2 aeProcessEvents函数

> 本函数为具体的事件循环实现

```C
// ae.c

/* Process every pending time event, then every pending file event
 * (that may be registered by time event callbacks just processed).
 * Without special flags the function sleeps until some file event
 * fires, or when the next time event occurs (if any).
 *
 * If flags is 0, the function does nothing and returns.
 * if flags has AE_ALL_EVENTS set, all the kind of events are processed.
 * if flags has AE_FILE_EVENTS set, file events are processed.
 * if flags has AE_TIME_EVENTS set, time events are processed.
 * if flags has AE_DONT_WAIT set the function returns ASAP until all
 * if flags has AE_CALL_AFTER_SLEEP set, the aftersleep callback is called.
 * the events that's possible to process without to wait are processed.
 *
 * The function returns the number of events processed. */
int aeProcessEvents(aeEventLoop *eventLoop, int flags)
{
    int processed = 0, numevents;

    // 1. 标志既不处理文件事件，也不处理时间事件，直接返回
    if (!(flags & AE_TIME_EVENTS) && !(flags & AE_FILE_EVENTS)) return 0;

    /* 2. maxfd不为-1时表示有需要监听的文件事件; 或flags标志设置了处理时间事件(且未设置AE_DONT_WAIT)时,
    进行事件处理逻辑; 也就是说即使没有注册监听的文件fd, 也可以检查时间事件进行处理;
    */
    if (eventLoop->maxfd != -1 || ((flags & AE_TIME_EVENTS) && !(flags & AE_DONT_WAIT))) {
        int j;
        aeTimeEvent *shortest = NULL;
        struct timeval tv, *tvp;

        // 3. 查找最近的时间事件； 并以最近的该事件指定的时间为差值来设置epoll_wait的timeout参数
        //根据上面对aeEventLoop.timeEventHead字段的说明，它通内部字段prev/next组成了双向链表，可供遍历；
      	// 通过when_sec/when_ms指定每个时间事件的下次发生时间；
   // aeSearchNearestTimer函数会遍历timeEventHead指定的链表，查找最近的时间事件（因为链表未排序）,见4.2节
        if (flags & AE_TIME_EVENTS && !(flags & AE_DONT_WAIT))
            shortest = aeSearchNearestTimer(eventLoop);

        if (shortest) {
           // 3.1 找到最近的时间事件后, shortest值应该为非NULL值; 计算它与当前时间的差值来设置timeout参数
            long now_sec, now_ms;
            aeGetTime(&now_sec, &now_ms); //获取当前时间, 秒数和毫秒数
            tvp = &tv;
          
            //计算从当前时间到最近的时间事件的差值，以毫秒为单位;
            long long ms = (shortest->when_sec - now_sec)*1000 + shortest->when_ms - now_ms;

            //根据ms差值设置tvp结构体字段对应的值, 在tvp结构体中, tv_sec单位为秒, tv_usec单位为微秒;
            //所以需要将 ms%1000 的结果再乘以1000;
            if (ms > 0) {
                tvp->tv_sec = ms/1000;
                tvp->tv_usec = (ms % 1000)*1000;
            } else {
                tvp->tv_sec = 0;
                tvp->tv_usec = 0;
            }
        } else {
            //3.2 没有时间事件, 根据flags标志, 若设置了AE_DONT_WAIT标志, 则将timeout设置为0, 则epoll_wait会直接返回; 而若未设置AE_DONT_WAIT时, 则将timeout设置为NULL, epoll_wait会一直等待文件事件触发;
            if (flags & AE_DONT_WAIT) {
                tv.tv_sec = tv.tv_usec = 0;
                tvp = &tv;
            } else {
                /* Otherwise we can block */
                tvp = NULL; /* wait forever */
            }
        }
        /*
        4. aeApiPoll调用 epoll_wait函数; 它会在文件事件触发或到达timeout后返回; 
        而numevents为触发的文件事件数量( timeout情况下可能为0 );
        
        tvp中有值, 表示epoll_wait的超时时间; tvp中的值均为0, 表示不等待; tvp为NULL, 表示无超时时间, 一直等待;
        */
        numevents = aeApiPoll(eventLoop, tvp);

        //5. 根据flags是否设置了AE_CALL_AFTER_SLEEP标志, 若有, 则调用aftersleep函数;
        if (eventLoop->aftersleep != NULL && flags & AE_CALL_AFTER_SLEEP)
            eventLoop->aftersleep(eventLoop);

        /*
        6. aeEventLoop.events中为注册的fd关心的事件; 比较fired中触发的每个fd事件和注册的事件比较; 
        如果触发的事件和关心的事件相同, 则调用对应的函数;(可读时调用rFileProc, 可写时调用wfileProc)
        */
        for (j = 0; j < numevents; j++) {
            aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];
            int mask = eventLoop->fired[j].mask;
            int fd = eventLoop->fired[j].fd;
            int fired = 0; /* Number of events fired for current fd. */
            
            //关于mask检查, 这里省略了一些比较长的注释, 可参考源码中的注释;
            int invert = fe->mask & AE_BARRIER;

            if (!invert && fe->mask & mask & AE_READABLE) {
                fe->rfileProc(eventLoop,fd,fe->clientData,mask);
                fired++;
            }
          
            if (fe->mask & mask & AE_WRITABLE) {
                if (!fired || fe->wfileProc != fe->rfileProc) {
                    fe->wfileProc(eventLoop,fd,fe->clientData,mask);
                    fired++;
                }
            }

            if (invert && fe->mask & mask & AE_READABLE) {
                if (!fired || fe->wfileProc != fe->rfileProc) {
                    fe->rfileProc(eventLoop,fd,fe->clientData,mask);
                    fired++;
                }
            }
            processed++;
        }
    }
    // 7. 处理时间事件; 关于processTimeEvents, 见4.3节
    if (flags & AE_TIME_EVENTS)
        processed += processTimeEvents(eventLoop);

    return processed; /* return the number of processed file/time events */
}
```



##### 3.3.3 aeApiPoll函数

> 实际调用epoll_wait的函数

```C
// ae_epoll.c
/*
可以看到在这个函数中实际调用了 epoll_wait函数;

关于tvp参数(即timeout时间), 它实际是根据aeProcessEvents函数的参数flags以及是否有时间事件来确定的;
1. 如果flags中包含了AE_DONT_WAIT标志, tvp中的值被初始化为0, 表示epoll_wait不会等待, 立即返回;
2. 如果falgs中设置了AE_TIME_EVENTS标志且eventLoop->timeEventHead非NULL, 则将最近的时间事件以及当前时间的差值设置为tvp时间中
3. 如果不包含 AE_TIME_EVENTS标志 或eventLoop->timeEventHead为NULL,则tvp设置为NULL, 表等待到文件事件才返回;

根据代码中的epoll_wait参数, eventLoop->apidata->efpd为通过epoll_create创建的文件标识符, 而eventLoop->apidata->events为用于接收触发的文件事件的数组; eventLoop->setsize为epoll_wait的最大数量值;

此外, aeApiPoll在epoll_wait返回后, 会根据retval值遍历返回结果, 
其目的是将触发的EPOLL事件转为对应的 AE_READABLE/AE_WRITABLE, 即redis只关心两种事件;
再将结果写入到eventLoop->fired数组中;
*/
static int aeApiPoll(aeEventLoop *eventLoop, struct timeval *tvp) {
    aeApiState *state = eventLoop->apidata;
    int retval, numevents = 0;

    retval = epoll_wait(state->epfd,state->events,eventLoop->setsize,
            tvp ? (tvp->tv_sec*1000 + tvp->tv_usec/1000) : -1);
    if (retval > 0) {
        int j;

        numevents = retval;
        for (j = 0; j < numevents; j++) {
            int mask = 0;
            struct epoll_event *e = state->events+j;

            if (e->events & EPOLLIN) mask |= AE_READABLE;
            if (e->events & EPOLLOUT) mask |= AE_WRITABLE;
            if (e->events & EPOLLERR) mask |= AE_WRITABLE;
            if (e->events & EPOLLHUP) mask |= AE_WRITABLE;
            eventLoop->fired[j].fd = e->data.fd;
            eventLoop->fired[j].mask = mask;
        }
    }
    return numevents;
}
```

#### 3.4 其它操作说明暂略

> **关于删除监听文件事件以及其它操作基本与上面的流程相似;**

### 4.  时间事件处理

> 关于时间事件注册，在 server.c initServer函数中， 且就在 3.2节的注册文件事件之前：
>
> ```c
> void initServer(void) {
> //...
> 
> //需要注意的是, 根据下面对serverCron函数的说明, serverCron只是注册之后是1毫秒之后触发; 
> //而之后的默认情况下, serverCron返回值为100, 即之后的serverCron的触发间隔是100毫秒;  
> if (aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL) == AE_ERR) {
>  serverPanic("Can't create event loop timers.");
>  exit(1);
> }
> //... 下面就是注册文件事件等
> //...
> }
> ```
>

#### 4.1 创建 aeCreateTimeEvent

```C
//ae.c
long long aeCreateTimeEvent(aeEventLoop *eventLoop, long long milliseconds,
        aeTimeProc *proc, void *clientData,
        aeEventFinalizerProc *finalizerProc)
{
    //1. 每次id从 aeEventLoop.timeEventNextId获取, 在3.1中关于aeEventLoop初始化时, 它的值从0开始自增
    long long id = eventLoop->timeEventNextId++;
    aeTimeEvent *te;

    te = zmalloc(sizeof(*te));
    if (te == NULL) return AE_ERR;
  
    //2. 指定id
    te->id = id; 
  
    //3. milliseconds参数指定了从当前时间开始之后多少毫秒之后执行, 结果写入 ta->when_sec和when_ms中;
    aeAddMillisecondsToNow(milliseconds,&te->when_sec,&te->when_ms);
    te->timeProc = proc;
    te->finalizerProc = finalizerProc;
    te->clientData = clientData;
    te->prev = NULL;
  
    //将te设置为新timeEventHead头部; 通过这里可以看到eventLoop->timeEventHead双向链表并非为排序后的结果;
    te->next = eventLoop->timeEventHead; 
    if (te->next)
        te->next->prev = te;
    eventLoop->timeEventHead = te;
    return id;
}
```

#### 4.2 查找最近时间事件

> **根据aeCreateTimeEvent函数可以看到 aeEventLoop.timeEventHead 并非为排序双向链表, 所以需要遍历链表找到最近的时间事件;**

```C
// ae.c
/* Search the first timer to fire.
 * This operation is useful to know how many time the select can be
 * put in sleep without to delay any event.
 * If there are no timers NULL is returned.
 *
 * Note that's O(N) since time events are unsorted.
 * Possible optimizations (not needed by Redis so far, but...):
 * 1) Insert the event in order, so that the nearest is just the head.
 *    Much better but still insertion or deletion of timers is O(N).
 * 2) Use a skiplist to have this operation as O(1) and insertion as O(log(N)).
 */

// 根据函数体, 函数可能会返回NULL值;
static aeTimeEvent *aeSearchNearestTimer(aeEventLoop *eventLoop)
{
    aeTimeEvent *te = eventLoop->timeEventHead;
    aeTimeEvent *nearest = NULL;
    //遍历链表
    while(te) {
        if (!nearest || te->when_sec < nearest->when_sec ||
                (te->when_sec == nearest->when_sec &&
                 te->when_ms < nearest->when_ms))
            nearest = te;
        te = te->next;
    }
    return nearest;
}
```

#### 4.3 处理时间事件 processTimeEvents

> 参考 3.3.2, aeProcessEvents函数等待文件事件timeout之后, 继续处理时间事件;

```C
// ae.c
/* Process time events */
static int processTimeEvents(aeEventLoop *eventLoop) {
    int processed = 0;
    aeTimeEvent *te;
    long long maxId;
    time_t now = time(NULL);

    /* If the system clock is moved to the future, and then set back to the
     * right value, time events may be delayed in a random way. Often this
     * means that scheduled operations will not be performed soon enough.
     *
     * Here we try to detect system clock skews, and force all the time
     * events to be processed ASAP when this happens: the idea is that
     * processing events earlier is less dangerous than delaying them
     * indefinitely, and practice suggests it is. */
  
    //???, eventLoop->lastTime初始化值为创建aeEventLoop结构体的时间, 且每次更新,
    //一般now应该大于lastTime值;  所以now小于lastTime值是否意味着 系统的时间发生了变化???
    if (now < eventLoop->lastTime) { 
        te = eventLoop->timeEventHead;
        while(te) {
            te->when_sec = 0;
            te = te->next;
        }
    }
    eventLoop->lastTime = now; //更新lastTime字段

    te = eventLoop->timeEventHead;
    maxId = eventLoop->timeEventNextId-1;
    while(te) {
        long now_sec, now_ms;
        long long id;
      
        /* Remove events scheduled for deletion. */
        // 关于AE_DELETED_EVENT_ID值, 是由于在删除时间事件时添加的标志值, 会在下面进行说明;
        // 这里实际处理被删除的时间事件;
        if (te->id == AE_DELETED_EVENT_ID) {
            aeTimeEvent *next = te->next;
            if (te->prev)
                te->prev->next = te->next;
            else
                eventLoop->timeEventHead = te->next;
            if (te->next)
                te->next->prev = te->prev;
            if (te->finalizerProc)
                te->finalizerProc(eventLoop, te->clientData);
            zfree(te);
            te = next;
            continue;
        }

        /* Make sure we don't process time events created by time events in
         * this iteration. Note that this check is currently useless: we always
         * add new timers on the head, however if we change the implementation
         * detail, this check may be useful again: we keep it here for future
         * defense. */
        if (te->id > maxId) {
            te = te->next;
            continue;
        }
        aeGetTime(&now_sec, &now_ms);
        if (now_sec > te->when_sec ||
            (now_sec == te->when_sec && now_ms >= te->when_ms))
        {
            int retval;

            id = te->id;
            retval = te->timeProc(eventLoop, id, te->clientData);
            processed++;
          
            /*
            重要: 这里注意几件事:
            1. timeProc返回-1时, 表示删除本时间事件;
            2. 而对于timeProc返回的其它值, 会被用来做为下次执行本时间任务的时间间隔;
            */
            if (retval != AE_NOMORE) {
                aeAddMillisecondsToNow(retval,&te->when_sec,&te->when_ms);
            } else {
                te->id = AE_DELETED_EVENT_ID;
            }
        }
        te = te->next;
    }
    return processed;
}               
```



##### 4.3.1 删除时间事件以及AE_DELETED_EVENT_ID标志

> **processTimeEvents函数中, 有对 AE_DELETED_EVENT_ID 标志的检查以及处理, 原因就在于删除时间事件逻辑.**

```C
// ae.c
int aeDeleteTimeEvent(aeEventLoop *eventLoop, long long id)
{
    aeTimeEvent *te = eventLoop->timeEventHead;
    while(te) {
        if (te->id == id) {
            /*
            从这儿可以看到, 在删除时间事件时, eventLoop并没有直接将它从eventLoop->timeEventHead链表中删除出去, 而是将对应的时间事件的 id字段设置为 AE_DELETED_EVENT_ID （即-1）; 等到具体调用 processTimeEvents时才实际的删除; 
            */
            te->id = AE_DELETED_EVENT_ID;
            return AE_OK;
        }
        te = te->next;
    }
    return AE_ERR; /* NO event with the specified ID found */
}
```



#### 4.4 Redis唯一注册的时间事件

> 搜索Redis源码, 它只注册了一个 serverCron 时间处理函数; 参考第4节开头;
>
> ```c
> //server.c
> //这个函数比较长, 这里大概介绍其中的几个典型的函数调用
> int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData) {
> 	//...
>   run_with_period(100){
>     //100毫秒周期执行
>   }
>   
>   run_with_period(5000){
>     //5000毫秒周期执行
>   }
>   
>   clientsCron(); //清除超时客户端连接
>   
>   databasesCron(); //处理数据库, 包括清除过期键等
>   
>   server.cronloops++; //server.cronloops用于记录serverCron函数的执行次数
>   
>   //server.hz表示serverCron函数的执行频率, 用户可配置, 最小为1最大为500, 默认为10;
>   //假设server.hz取默认值10, 函数返回 1000/server.hz, 会更新serverCron的执行周期为100毫秒;
>   return 1000/server.hz;
> }
> ```



##### 4.4.1 示例

> serverCron函数中调用的函数执行时间不能过长 , 否则会导致服务器不能及时响应客户端的请求; 以databasesCron为例 :
>
> ```c
> //server.c
> void databasesCron(void) {
>     if (server.active_expire_enabled && server.masterhost == NULL) {
>         /*
>         在activeExpireCycle最多遍历"dbs_per_call"个数据库, 并记录每个数据库删除的过期键数目;
>         当删除过期键数目大于门限时, 认为此数据库过期键较多, 需要再次处理; 
>         并且还添加了timelimit时间限制, 每执行16次do-while循环就检查是否超过了timelimit, 超过则结束; 
>         */
>         activeExpireCycle(ACTIVE_EXPIRE_CYCLE_SLOW);
>     } else if (server.masterhost != NULL) {
>         expireSlaveKeys();
>     }
>     //...
> }
> ```
>
> 

### 5. 关于 beforeSleep 和 afterSleep 函数

> **见3.3节 ,  server.c main()函数中设置 aeEventLoop.{beforesleep和 aftersleep}字段; **



#### 5.1 beforeSleep

```C
//server.c
/* This function gets called every time Redis is entering the
 * main loop of the event driven library, that is, before to sleep
 * for ready file descriptors. */
void beforeSleep(struct aeEventLoop *eventLoop) {
    UNUSED(eventLoop);

    /* Call the Redis Cluster before sleep function. Note that this function
     * may change the state of Redis Cluster (from ok to fail or vice versa),
     * so it's a good idea to call it before serving the unblocked clients
     * later in this function. */
    if (server.cluster_enabled) clusterBeforeSleep();

    /* Run a fast expire cycle (the called function will return
     * ASAP if a fast cycle is not needed). */
    if (server.active_expire_enabled && server.masterhost == NULL)
        activeExpireCycle(ACTIVE_EXPIRE_CYCLE_FAST);

    /* Send all the slaves an ACK request if at least one client blocked
     * during the previous event loop iteration. */
    if (server.get_ack_from_slaves) {
        robj *argv[3];

        argv[0] = createStringObject("REPLCONF",8);
        argv[1] = createStringObject("GETACK",6);
        argv[2] = createStringObject("*",1); /* Not used argument. */
        replicationFeedSlaves(server.slaves, server.slaveseldb, argv, 3);
        decrRefCount(argv[0]);
        decrRefCount(argv[1]);
        decrRefCount(argv[2]);
        server.get_ack_from_slaves = 0;
    }
    /* Unblock all the clients blocked for synchronous replication
     * in WAIT. */
    if (listLength(server.clients_waiting_acks))
        processClientsWaitingReplicas();

    /* Check if there are clients unblocked by modules that implement
     * blocking commands. */
    moduleHandleBlockedClients();

    /* Try to process pending commands for clients that were just unblocked. */
    if (listLength(server.unblocked_clients))
        processUnblockedClients();

    /* Write the AOF buffer on disk */
    flushAppendOnlyFile(0);

    /* Handle writes with pending output buffers. */
    handleClientsWithPendingWrites();

    /* Before we are going to sleep, let the threads access the dataset by
     * releasing the GIL. Redis main thread will not touch anything at this
     * time. */
    if (moduleCount()) moduleReleaseGIL();
}
```



#### 5.2 afterSleep

```C
// server.c

/* This function is called immadiately after the event loop multiplexing
 * API returned, and the control is going to soon return to Redis by invoking
 * the different events callbacks. */
void afterSleep(struct aeEventLoop *eventLoop) {
    UNUSED(eventLoop);
    if (moduleCount()) moduleAcquireGIL();
}
```

