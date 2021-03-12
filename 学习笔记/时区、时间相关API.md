# tzset()

> 初始化时区相关信息：
>
> ```c
> extern char *tzname[2];
> extern long timezone;
> extern int daylight;
> ```

## 示例1

```c
#include <stdio.h>
#include <string.h>

//在Redis里对`tzset()`的使用方式有点晦涩，原因是在访问tzname，timezone等变量时，
//它们由链接器自动初始化的变量，所以不需要声明，直接就可以在C代码中访问。
extern char *tzname[2];
extern long timezone;
extern int daylight;
int main()
{
    printf("Before tzset...\n");
    printf("tzname: %s %s\n", *tzname, *(tzname+1));
    printf("timezone: %ld\n", timezone);
    printf("daylight: %d\n", daylight);
    tzset();

    printf("After tzset...\n");
    printf("tzname: %s %s\n", *tzname, *(tzname+1));
    printf("timezone: %ld\n", timezone);
    printf("daylight: %d\n", daylight);
}

```

## Redis中的使用方式

> 在调用tzset()之后，后面出现了如下赋值语句，其实就是没有提前声明timezone变量，但是可以直接访问。

```c
server.timezone = timezone; /* Initialized by tzset(). */
```



# time() & gettimeofday() 

> time() 和 gettimeofday() 两个函数都是返回从`Epoch time, 1970-01-01 00:00:00`至今的时间，只是 time()返回的是秒数，而gettimeofday()比time()更精确点，能返回秒数以及微秒数。

## 示例代码

```c
#include <stdio.h>
#include <time.h>
#include <sys/time.h>

int main()
{
    time_t result = time(NULL); //time.h
    printf("Current time %ld\n", result);

    /*
    struct timeval {
    	time_t      tv_sec;     //seconds
    	suseconds_t tv_usec;    //microseconds, 微秒
    };
    */
    struct timeval tv; //<sys/time.h>
    gettimeofday(&tv, NULL);

    printf("Current time %ld:%ld\n", tv.tv_sec, tv.tv_usec);
}
```



# localtime()：time()结果转换表示

> 上面`time()`接口返回自`Epoch Time`到当前时间的秒数，之后可以调用`localtime()/localtime_r()`函数将`time()`结果转换成`struct tm`结构体信息。

## struct tm结构体

```c
//由于time()结果精度为秒, 所以转换成tm结构体后的最小精度也是秒
struct tm {
    int tm_sec;    /* Seconds (0-60) */
    int tm_min;    /* Minutes (0-59) */
    int tm_hour;   /* Hours (0-23) */
    int tm_mday;   /* Day of the month (1-31) */
    int tm_mon;    /* Month (0-11) */
    int tm_year;   /* Year - 1900 */
    int tm_wday;   /* Day of the week (0-6, Sunday = 0) */
    int tm_yday;   /* Day in the year (0-365, 1 Jan = 0) */
    int tm_isdst;  /* Daylight saving time */
};
```

## 示例

```c
#include <stdio.h>
#include <time.h>

int main() 
{
    time_t curr = time(NULL);
    printf("Current time in seconds is %ld\n", curr);

    struct tm info;
    localtime_r(&curr, &info);

    //ctime接受time_t指针参数, asctime接受struct tm指针参数
    //打印结果格式是相同的
    printf("Current time in detail %s\n", ctime(&curr));
    printf("Current time in detail %s\n", asctime(&info));
}

```

