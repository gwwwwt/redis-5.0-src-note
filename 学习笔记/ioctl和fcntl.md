# ioctl系统调用

> Man Page:
>
> ```shell
> NAME
>        ioctl - control device
> 
> SYNOPSIS
>        #include <sys/ioctl.h>
> 
>        int ioctl(int fd, unsigned long request, ...);
> 
> DESCRIPTION
>        The  ioctl()  system call manipulates the underlying device parameters of special files.  In particular, many
>        operating characteristics of character special  files  (e.g.,  terminals)  may  be  controlled  with  ioctl()
>        requests.  The argument fd must be an open file descriptor.
> 
>        The  second argument is a device-dependent request code.  The third argument is an untyped pointer to memory.
>        It is traditionally char *argp (from the days before void * was valid C), and will be so named for  this  dis‐cussion.
> 
>        An  ioctl()  request has encoded in it whether the argument is an in parameter or out parameter, and the size
>        of the argument argp in bytes.  Macros and defines used in specifying an ioctl() request are located  in  the
>        file <sys/ioctl.h>.
> 
> ```

## 1. ioctl request参数

> **rquest参数为ioctl提供了2个信息：1. 获取信息(in paramer)还是设置信息(out paramer)；2. 可能存在的第三个参数的类型；**
>
> **在"sys/ioctl.h"头文件中为不同的设备类型定义了不同的request参数，由于有大量的request参数，ioctl将这些参数按不同的设备类型分类，可以通过不同的man page查到，这些man page记录在ioctl的man page，如下：**
>
> ```shell
> ioctl_console(2),   ioctl_fat(2),   ioctl_ficlonerange(2),   ioctl_fideduperange(2), ioctl_getfsmap(2),  ioctl_iflags(2), ioctl_list(2), ioctl_ns(2), ioctl_tty(2), ioctl_userfaultfd(2)
> ```

## 2. ioctl的应用示例

> **在《4. 解析配置文件和命令行参数.md》中，如果为redis传入"--test-memory"参数，会执行内存测试函数，其中在`memtest.c#memtest`函数中：**

```c
void memtest(size_t megabytes, int passes) {
    //获取当前window size
    if (ioctl(1, TIOCGWINSZ, &ws) == -1) {
        ws.ws_col = 80;
        ws.ws_row = 20;
    }
    //... 后续略
}

/*
可以在ioctl_tty man page中查到 struct winsize ws 和 TIOCGWINSZ 的说明
	TIOCGWINSZ: 获取window size
	struct winsize {
        unsigned short ws_row;
        unsigned short ws_col;
        unsigned short ws_xpixel;   //unused
        unsigned short ws_ypixel;   //unused
    };

*/
```



# fcntl系统调用

> **fcntl和ioctl的区别在于：`fcntl`主要用于对fd的属性操作，而`ioctl`主要是通过fd获取/设置与其相关联的底层硬件设备的信息**