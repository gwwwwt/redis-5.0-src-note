# 信号

> **参考《UNIX环境高级编程》第10章结合Redis源码学习笔记。**

## 概念

**信号就软件中断，它提供了一种处理异步事件的方法，例如，终端用户键入中断键，会通过信号机制停止一个程序，或及早终止管道中的下一个程序。**

**每个信号都有一个名字，这些名字都以SIG开头。在头文件<signal.h>中，信号名都被定义为正整数常量（信号编号）。**

**不存在编号为`0`的信号，`kill`函数对信号编号`0`有特殊的应用。**

**很多条件可以产生信号。**

> - **当用户按某些终端键时，引发终端产生的信号。如(Ctrl+C)产生中断信号`SIGINT`。**
> - **硬件异常产生信号：除数为0、无效的内存引用等。这些条件通常由硬件检测到，并通知内核。然后内核为该我们的产生时正在运行的进程产生适当的信号。例如，对执行一个无效内存引用的进程产生`SIGSEGV`信号。**
> - **进程调用`kill(2)`函数可将任意信号发送给另一个进程或进程组。当然对此有所限制：接收信号进程和发送信号进行的所有者必须相同，或发送信号进程的所有者必须是超级用户。**
> - **用户可用`kill(1)`命令将信号发送给其它进程。**
> - **当检测到某种软件条件已经发生，并应将其通知有关进程时也产生信号。这里不是指硬件条件，而是软件条件。如`SIGURG(在网络连接上传来带外数据)`、`SIGPIPE(在管道的读进程已终止后，一个进程写此管道)`以及`SIGALRM(进程设置的定时器已经超时)`。**

**由于信号是异步产生的，进程不能简单的测试一个变量(如`errno`)来判断是否发生了一个信号，而是必须告诉内核"在此信号发生时，请执行下列操作"。**

**可以告诉内核按3种方式之一进行处理，即信号的处理或与信号相关的动作。**

> 1. **忽略此信号。`有两种信号决不能被忽略，SIGKILL和SIGSTOP`，原因是：它们向内核和超级用户提供了使进程终止或停止的可靠方法。另外，如果忽略某些由硬件异常产生的信号，则进程的运行行为是未定义的。**
> 2. **捕捉信号。通知内核在某种信号发生时，调用一个用户函数。**
> 3. **执行系统默认动作。**

## 函数signal

> **注意，对于重复信号：根据`signal man page`，如果`handler`传入一个非SIG_IGN或SIG_DFL的函数，内核在接收到信号，执行handler之前，可能会做两种操作：**
>
> **第一种方式：先将signum的信号处理函数重置为SIG_DFL，再执行handler; 这样如果在执行handler过程中再次收到signum信号，会导致调用SIG_DFL去执行了。**
>
> **第二种方式　阻塞signum信号，即如果在执行handler过程中，内核不会将发生的signum信号发送给进程，而是等到handler返回后再将信号递送给进程。但是问题就是，如果在handler执行过程中，有多个signum发送给进程，由于内核一般不会对信号进行排队，所以handler返回后，内核只会递送一个或小于几个signum到进程（总之递送给进程的signum数量会小于实际发送的个数。）**

```c
#include <signal.h>
typedef void (*sighandler_t)(int);
sighandler_t signal(int signum, sighandler_t handler);

/*
handler的值是常量SIG_ING、SIG_DFL或当接到此信号后调用的函数地址。
如果指定SIG_IGN，则向内核表示忽略此信号(SIGKILL和SIGSTOP不能忽略)
如果指定SIG_DFL，则表示接到此信号后的动作是系统默认动作。

signal函数返回值是一个函数地址，是指向在此之前的信号处理程序的指针。
*/
```

### 例子

> **下面的例子目的是想验证三点：**
>
> + **事件处理函数就是中断进程的当前执行流程，跳转到信号处理函数执行完成后再继续原来的流程，所以`process_signal`和`main()`中`getpid()`的返回值相同。另外通过分别在`main()#raise前后打印时间`和在`process_signal#处理信号流程前后打印时间`，可以得出`main()#raise导致跳转到process_signal执行，main()流程暂停，等到process_signal返回之后再继续执行。`**
>
> + **由于信号到达时会中断当前执行流程，如果当前进程正在进行一个阻塞操作，比如阻塞I/O，并等待其返回时，信号到达，阻塞操作返回值为负数(表错误)，并且全局errno会被设置为`EINTR`。示例代码中尝试在进程阻塞在"scanf"时在另一个终端窗口执行"kill -USR1 pid"的方式来中断scanf操作，以查看scanf结果，但是scanf跳转到process_signal返回后又继续等待输入了。...`之后再用socket的accept试试吧`;想要验证〈UNIX网络编程〉第5.10章关于`SIGCHLD`的问题，但是可能也得用Socket验证了， `...好烦`**
>
> + **特指本机系统（Ubuntu18.04），执行信号处理函数时，内核会阻塞相同信号的递送。进程阻塞在scanf时，在另一个终端窗口执行"kill -USR1 pid"多次，或用分号分隔命令在同一行执行，会发现`实际输出的信号处理执行次数是小于发送的信号数量的`。**
>
>     > **但是不会阻塞不同的信号，即如果执行"kill -USR1 pid ; kill -1 pid"，给进程发送信号后，根据打印结果：**
>     >
>     > 1. **"-1（即SIGHUP）"信号被递送给进程了; **
>     > 2. **在处理SIGUSR1时进程sleep了2秒，此时收到了SIGHUP信号，但没有执行，是等到处理SIGUSR1结束后，才执行的对SIGHUP的处理。**
>     > 3. **"kill -USR1 pid ; kill -1 pid; kill -1 pid"的命令，根据前面的阻塞信号说明以及输出结果，进程只收到了一个SIGHUP信号。**
>     > 4. **丧心病狂的跑了一个命令："kill -USR1 3157; kill -1 3157 ; kill -USR1 3157; kill -1 3157; kill -USR1 3157; kill -1 3157"，也是只收到一个USR1和一个SIGHUP。**

```c
#include <signal.h>
#include <sys/types.h>
#include <unistd.h>
#include <stdio.h>

static int count = 1;
static int chld_count = 1;
void process_signal(int signo) {
    printf("[process_signal] pid = %d\n", getpid());

    if (signo == SIGUSR1){
        printf("[process_signal] time %d\n", time(NULL));
        printf("[process_signal] received SIGUSR1 %d\n", count++);
        sleep(2);
        printf("[process_signal] time %d\n", time(NULL));
    } 
    else if (signo == SIGCHLD) {
        printf("[process_signal] received SIGCHLD %d\n", chld_count++);
        sleep(2);
    }
    else if(signo == SIGHUP) {
        printf("[process_signal] HUP SIGNAL\n");
    }
}

int main() {
    signal(SIGUSR1, process_signal);
    signal(SIGCHLD, process_signal);
    signal(SIGHUP, process_signal);
    printf("[Main] start raising signals\n");

    int a = 0;
    printf("Main] pid = %d\n", getpid());

    for (int i = 0; i < 5; i++)
    {
        printf("[Main] time %d\n", time(NULL));
        raise(SIGUSR1);
    }
        
    int result = scanf("%d", &a);
    printf("[Main] scanf result: %d and a = %d\n", result, a);
    printf("[Main] errno is %d\n", errno);
    for (int i = 0; i < 5; i++)
    {
        raise(SIGCHLD);
    }

    sleep(50);
    printf("[main] exit\n");

    return 0;
}
```



 ## 函数 sigaction

> **`sigaction`函数可取代`signal`函数**

```c
#include <signal.h>
/*
参数signum: 要检测或修改其具体动作的信号编号;
action参数若非空: 要将signum修改成本参数指定的动作;
oldact参数若非空: 可用来返回signum信号的上一个动作
*/
int sigaction(int signum, const struct sigaction *act,
                     struct sigaction *oldact);
struct sigaction {
    void     (*sa_handler)(int); //信号处理函数,可以是SIG_IGN/SIG_DFL
    void     (*sa_sigaction)(int, siginfo_t *, void *);
    sigset_t   sa_mask; /*调用信号处理函数时阻塞本信号集中包含的信号
    					本信号集中默认包含signum*/
    int        sa_flags; //指定执行处理函数时的标志位
    void     (*sa_restorer)(void); //未指定, 设空值
};

siginfo_t { //包含信号产生原因的有关信息
    int      si_signo;     /* Signal number */
    int      si_errno;     /* An errno value */
    int      si_code;      /* Signal code */
    int      si_trapno;    /* Trap number that caused
                              hardware-generated signal
                              (unused on most architectures) */
    pid_t    si_pid;       /* Sending process ID */
    uid_t    si_uid;       /* Real user ID of sending process */
    int      si_status;    /* Exit value or signal */
    clock_t  si_utime;     /* User time consumed */
    clock_t  si_stime;     /* System time consumed */
    sigval_t si_value;     /* Signal value */
    int      si_int;       /* POSIX.1b signal */
    void    *si_ptr;       /* POSIX.1b signal */
    int      si_overrun;   /* Timer overrun count;
                              POSIX.1b timers */
    int      si_timerid;   /* Timer ID; POSIX.1b timers */
    void    *si_addr;      /* Memory location which caused fault */
    long     si_band;      /* Band event (was int in
                              glibc 2.3.2 and earlier) */
    int      si_fd;        /* File descriptor */
    short    si_addr_lsb;  /* Least significant bit of address
                              (since Linux 2.6.32) */
    void    *si_lower;     /* Lower bound when address violation
                              occurred (since Linux 3.19) */
    void    *si_upper;     /* Upper bound when address violation
                              occurred (since Linux 3.19) */
    int      si_pkey;      /* Protection key on PTE that caused
                              fault (since Linux 4.6) */
    void    *si_call_addr; /* Address of system call instruction
                              (since Linux 3.5) */
    int      si_syscall;   /* Number of attempted system call
                              (since Linux 3.5) */
    unsigned int si_arch;  /* Architecture of attempted system call
                              (since Linux 3.5) */
}
```



# 待补