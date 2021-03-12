# sds.c/util.c学习字符串API

> 在学习Redis源码的过程中会接触到对sds相关工具函数的调用，这样通过研究相关函数实现来学习各种`standard C API`接口。
>
> 相关源码：sds.c

## sdstrim => strchr

> sdstrim的应用：sdstrim(relpath," \r\n\t"); 
>
> 清除sds两端的空白字符，同样会修改relpath的len字段.

> **strchr 系列api**
>
> > 在字符串中查找指定字符
>
> ```c
> #include <string.h>
> 
> //参数s为被搜索的字符串
> //参数c虽然是int类型,但实际应该是要查找的char值
> //返回第一个与c相等的字符指针
> char *strchr(const char *s, int c); //从左侧开始查找
> char *strrchr(const char *s, int c); //从右侧开始查找
> ```
>
> 

```c
/*
以sdstrim(relpath," \r\n\t");函数调用为例;
参数cset为包含所有认为是空白字符的字符串;
*/
sds sdstrim(sds s, const char *cset) {
    char *start, *end, *sp, *ep;
    size_t len;
    
    sp = start = s; //左端开始char指针
    ep = end = s+sdslen(s)-1; //右端开始char指针

    //从s左侧开始, 如果*sp字符是cset字符串中的某一字符, 则递增sp值
    while(sp <= end && strchr(cset, *sp)) sp++;
    //从s右侧开始查找, 若是空白字符, 则递减ep值
    while(ep > sp && strchr(cset, *ep)) ep--;
    
    //之后 sp~ep之间即为trim之后的结果
    len = (sp > ep) ? 0 : ((ep-sp)+1);
    if (s != sp) memmove(s, sp, len);
    s[len] = '\0';
    sdssetlen(s,len); //更新len
    return s;
}
```

### sdssetlen函数

> 上面sdstrim以及下面的工具函数在对字符串进行操作后都会调用sdssetlen更新len字段，这里就出现了一个需要注意的点：
>
> **虽然redis在创建sds时会根据字符串长度来创建对应的`sdshdrT结构`，但是在调用`sdstrim等函数`对sds进行修改后，会影响sds len，但此时也只是更新len字段，不会再根据新的sds len来确定是否需要分配不同的sdshdrT来存储。**
>
> **举例来说，比如使用sds存储40字节长度的字符串，sdsnew()函数会分配一个sdshdr8结构体来存储这些信息；但假如字符串左右都包含许多空白字符，调用sdstrim()后，sds长度可能只有5字节，也只会将sdshdr8.len字段更新为5。即是可能出现某种结构体中持有与本类型初始化时不一致长度的字符串。**
>
> ```c
> #define SDS_TYPE_5  0
> #define SDS_TYPE_8  1
> #define SDS_TYPE_16 2
> #define SDS_TYPE_32 3
> #define SDS_TYPE_64 4
> 
> struct __attribute__ ((__packed__)) sdshdr5;
> struct __attribute__ ((__packed__)) sdshdr8;
> struct __attribute__ ((__packed__)) sdshdr16;
> struct __attribute__ ((__packed__)) sdshdr32;
> struct __attribute__ ((__packed__)) sdshdr64;
> ```

```c
//只更新len字段, 不会判断是否需要用不同的结构类型来存储
static inline void sdssetlen(sds s, size_t newlen) {
    unsigned char flags = s[-1];
    switch(flags&SDS_TYPE_MASK) {
        case SDS_TYPE_5:
            {
                unsigned char *fp = ((unsigned char*)s)-1;
                *fp = SDS_TYPE_5 | (newlen << SDS_TYPE_BITS);
            }
            break; 
        case SDS_TYPE_8:
            SDS_HDR(8,s)->len = newlen;
            break;
        case SDS_TYPE_16:
            SDS_HDR(16,s)->len = newlen;
            break;
        case SDS_TYPE_32:
            SDS_HDR(32,s)->len = newlen;
            break;
        case SDS_TYPE_64:
            SDS_HDR(64,s)->len = newlen;
            break;
    }           
}   
```



## sdsrange

> 将sds内容更新为子字符串，子字符串范围由参数指定。
>
> 需要注意的是sdsrange函数对负数参数的处理。

```c
void sdsrange(sds s, ssize_t start, ssize_t end) {
    size_t newlen, len = sdslen(s);
	if (len == 0) return;
    /*
    对start/end参数为负数的处理:
    1. start=len+start; 如果此时start结果为非负数, 则取该值
    2. 如果len+start的结果仍然是负数, 则直接取0值
    */
    if (start < 0) {
        start = len+start;
        if (start < 0) start = 0;
    }
    if (end < 0) { 
        end = len+end;
        if (end < 0) end = 0;
    }
    newlen = (start > end) ? 0 : (end-start)+1;
    if (newlen != 0) {
        if (start >= (ssize_t)len) {
            newlen = 0;
        } else if (end >= (ssize_t)len) {
            end = len-1;
            newlen = (start > end) ? 0 : (end-start)+1;
        }
    } else {
        start = 0;
    }
    if (start && newlen) memmove(s, s+start, newlen);
    s[newlen] = 0;
    sdssetlen(s,newlen);
}

```



## getAbsolutePath函数

> 应用：获取redis-server的绝对路径；
>
> 启动redis-server两种方式：
>
> + 绝对路径， 直接返回即可
> + 相对路径，需要调用`getcwd()`获取当前目录完整路径，之后再根据redis-server的相对路径进行处理和拼接

```c
sds getAbsolutePath(char *filename) {
    char cwd[1024];
    sds abspath;
    sds relpath = sdsnew(filename);

    relpath = sdstrim(relpath," \r\n\t");
    
    //如果是绝对路径, 直接返回
    if (relpath[0] == '/') return relpath; /* Path is already absolute. */

    //getcwd()获取当前目录路径
    if (getcwd(cwd,sizeof(cwd)) == NULL) {
        sdsfree(relpath);
        return NULL; 
    }
    abspath = sdsnew(cwd);
    
    //如果获取的当前目录路径不以'/'结尾, 手动在后面添加一个'/'
    if (sdslen(abspath) && abspath[sdslen(abspath)-1] != '/')
        abspath = sdscat(abspath,"/");

    //如果redis-server是以 "../redis-server"方式启动
    //需要从当前目录路径从后往前查找上一个'/'字符, 再取子字符串
    //之后再和redis-server拼接得到结果
    while (sdslen(relpath) >= 3 &&
           relpath[0] == '.' && relpath[1] == '.' && relpath[2] == '/')
    {
        sdsrange(relpath,3,-1);
        if (sdslen(abspath) > 1) {
            char *p = abspath + sdslen(abspath)-2;
            int trimlen = 1;

            while(*p != '/') {
                p--;
                trimlen++;
            }
            sdsrange(abspath,0,-(trimlen+1));
        }
    }

    //对于以'./redis-server'启动的方式, 好像直接就把'.'也拼接了
    //即可能出现 '/usr/local/bin/./redis-server'字符串??
    abspath = sdscatsds(abspath,relpath);
    sdsfree(relpath);
    return abspath;
}
```

