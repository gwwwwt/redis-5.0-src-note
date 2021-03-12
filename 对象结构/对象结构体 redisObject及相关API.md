# redisObject

> **参考：《Redis 5设计与源码分析》第9章 "命令处理生命周期"**

## 概念

> **Redis是一个 key-value 数据库，key只能是字符串，而 value 可以是字符串、列表、集合、有序集合、散列表这几种类型；**
>
> **为了能够用统一的类型包装以上所有value类型； 所以设计了`struct redisObject`结构体;**

```C
// server.h
#define LRU_BITS 24
#define LRU_CLOCK_MAX ((1<<LRU_BITS)-1) /* Max value of obj->lru */
#define LRU_CLOCK_RESOLUTION 1000 /* LRU clock resolution in ms */

#define OBJ_SHARED_REFCOUNT INT_MAX

typedef struct redisObject {
    unsigned type:4; //类型
    unsigned encoding:4; //编码
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount; //引用计数
    void *ptr; //指向实际对象的指针
} robj;
```



## 字段说明

### 1. type字段

> **内部ptr指向的对象类型**

```C
//server.h
#define OBJ_STRING 0    /* String object. */
#define OBJ_LIST 1      /* List object. */
#define OBJ_SET 2       /* Set object. */
#define OBJ_ZSET 3      /* Sorted set object. */
#define OBJ_HASH 4      /* Hash object. */
```

### 2. encoding字段

> **内部ptr指向的对象底层存储数据结构**

```C
//server.h
#define OBJ_ENCODING_RAW 0     /* Raw representation */
#define OBJ_ENCODING_INT 1     /* Encoded as integer */
#define OBJ_ENCODING_HT 2      /* Encoded as hash table */
#define OBJ_ENCODING_ZIPMAP 3  /* Encoded as zipmap */
#define OBJ_ENCODING_LINKEDLIST 4 /* No longer used: old list encoding. */
#define OBJ_ENCODING_ZIPLIST 5 /* Encoded as ziplist */
#define OBJ_ENCODING_INTSET 6  /* Encoded as intset */
#define OBJ_ENCODING_SKIPLIST 7  /* Encoded as skiplist */
#define OBJ_ENCODING_EMBSTR 8  /* Embedded sds string encoding */
#define OBJ_ENCODING_QUICKLIST 9 /* Encoded as linked list of ziplists */
#define OBJ_ENCODING_STREAM 10 /* Encoded as a radix tree of listpacks */
```

+ **在对象的生命周期中，编码（底层结构）并非一成不变的；如集合对象，当集合中所有元素都可以用整数表示时，底层数据结构采用整数集合；而当执行 sadd 命令向集合中添加元素时，Redis总会检查待添加的元素是否可以解析成整数，如果解析失败，则会将集合存储结构转换为字典；**
+ **摘自《Redis深度历险》5.1节:  关于SDS类型的两种存储方式 embstr 和 raw；当字符串长度特别短时，使用embstr形式存储(embeded)，而长度超过44字节时，使用raw形式存储；那它们的区别是什么呢？为什么分界线是什么44字节呢？ 这个和内存分配器 jemalloc、tcmalloc等的实现有关，它们分配内存的大小单位都是2/4/8/16/32/64字节等，为了能容纳一个完整的 emdstr对象， jemalloc最少会分配32字节的空间，如果字符串再稍微长一点，那就是64字节的空间。如果字符串整体超过了64字节，Redis会认为它是一个大字符串，不再适合使用emdstr形式存人言土禾，而该用raw形式。那么如果内存分配器分配了64字节，那么这个字符串的长度最大可以是多少呢？ 44字节。原因在于对于超过32字节长度的字符串，Redis会使用 sdshdr8 结构体来存储，而在结构体中，len/alloc/flags 三个字段分别占1字节，再加上RedisObject本身占16字节，以及SDS结尾会添加'\0'结束符，所以 64 -(3 + 16 + 1) = 44字节;**

### 3. ptr字段

> **指向 robj 包装的实际数据; 有一些特殊情况，当ptr中存储的数据可以用long类型表示时，数据会直接存储在ptr字段中**

### 4. refcount字段

> **当前对象的引用次数，用于实现对象共享；当refcount值为0时释放对象空间；**

> **释放对象空间函数为： decrRefCount**
>
> ```c
>//object.c
> void decrRefCount(robj *o) {
>  if (o->refcount == 1) {
>      switch(o->type) {
>         case OBJ_STRING: freeStringObject(o); break;
>         case OBJ_LIST: freeListObject(o); break;
>         case OBJ_SET: freeSetObject(o); break;
>         case OBJ_ZSET: freeZsetObject(o); break;
>         case OBJ_HASH: freeHashObject(o); break;
>         case OBJ_MODULE: freeModuleObject(o); break;
>         case OBJ_STREAM: freeStreamObject(o); break;
>         default: serverPanic("Unknown object type"); break;
>         }
>         zfree(o);
>     } else {
>         if (o->refcount <= 0) 
>             serverPanic("decrRefCount against refcount <= 0");
>         if (o->refcount != OBJ_SHARED_REFCOUNT) o->refcount--;
>     }
>    }
> ```

### 5. lru字段

> **24位值，用于缓存淘汰策略；可以在配置文件中使用 `maxmemory-policy` 进行配置；**

#### 5.1 LRU策略

>  **对于LRU策略：lru字段存储上次访问时间（单位：秒）；**

> **在调用"GET"命令对应的函数时, 中间会调用如下代码:**
>
> ```c
> //db.c
> robj *lookupKey(redisDb *db, robj *key, int flags) {
>  dictEntry *de = dictFind(db->dict,key->ptr);
>  if (de) {
>      robj *val = dictGetVal(de);
> 
>      /* Update the access time for the ageing algorithm.
>      * Don't do it if we have a saving child, as this will trigger
>      * a copy on write madness. */
>      if (server.rdb_child_pid == -1 &&server.aof_child_pid == -1 
>             &&!(flags & LOOKUP_NOTOUCH))
>      {
>          if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
>              updateLFU(val);
>          } else {
>              //调用LRU_CLOCK()更新访问时间;
>              //逻辑: time(NULL)获取当前时间(单位:秒), 再取低24位即可
>              val->lru = LRU_CLOCK(); 
>          }
>      }
>      return val;
>  } else {
>      return NULL;
>  }
> }
> ```
>
> 

#### 5.2 LFU策略

> **对于LFU策略：lru字段存储上次访问时间以及访问次数； 参考 updateLFU 函数，低8位存储访问次数，高16位存储对象上次访问时间（单位：分钟）；**

> **updateLFU：**
>
> ```c
> //db.c
> void updateLFU(robj *val) {
>  //关于更新对象上次访问时间与访问次数, 使用LFUDecrAndReturn来计算该值, 而不是直接在lru字段上累加;
>  //如果简单累加的话, 会造成越老的数据访问次数越大, 即使它不是最近常访问的数据;
>  //所以访问次数应该有一个随时间衰减的过程, 所以有了LFUDecrAndReturn函数;
>  unsigned long counter = LFUDecrAndReturn(val);
>  counter = LFULogIncr(counter);
>  val->lru = (LFUGetTimeInMinutes()<<8) | counter;
> }
> 
> //evict.c
> unsigned long LFUDecrAndReturn(robj *o) {
>  unsigned long ldt = o->lru >> 8;
>  unsigned long counter = o->lru & 255;
>  
>  //server.lfu_decay_time, 在initServerConfig()中默认初始为1;
>  //表示开启lfu按时间衰减的功能
>  unsigned long num_periods = server.lfu_decay_time ? LFUTimeElapsed(ldt) / server.lfu_decay_time : 0;
>  if (num_periods)
>      counter = (num_periods > counter) ? 0 : counter - num_periods;
>  return counter;
> }
> 
> //返回自ldt至今为止所经过时间,单位:分钟
> unsigned long LFUTimeElapsed(unsigned long ldt) {
>     //LFUGetTimeInMinutes取当前时间, 单位:分钟, 再取低16位的结果
>     unsigned long now = LFUGetTimeInMinutes();
>     //由于只有16位存储时间, 是有可能出现新时间按位截断后的结果
>     //比原ldt存储的值小的情况, 如果出现, 
>     if (now >= ldt) return now-ldt;
>     return 65535-ldt+now;
> }
> ```

# API

> **object.c**

## 创建

```c
//--- 内部创建robj的接口, 除创建EMBSTR robj时是直接分配内存并赋值外, 
//--- 其它类型robj都最终会通过本接口创建对象
//--- 当然由于本函数中encoding默认设置成了RAW字符串类型, 
//--- 如果是是非RAW或非sds类型, 可以创建后修改encoding
robj *createObject(int type, void *ptr) {
    robj *o = zmalloc(sizeof(*o));
    o->type = type;
    o->encoding = OBJ_ENCODING_RAW; //默认robject编码方式
    o->ptr = ptr;
    o->refcount = 1; //创建时refcount初始化为1

    /* Set the LRU to the current lruclock (minutes resolution), or
     * alternatively the LFU counter. */
    if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
        o->lru = (LFUGetTimeInMinutes()<<8) | LFU_INIT_VAL;
    } else {
        o->lru = LRU_CLOCK();
    }
    return o;
}
```



### 1. 创建sds robj

```c
#define OBJ_ENCODING_EMBSTR_SIZE_LIMIT 44 //EMBSTR sds的最大长度限制
//--- 用字符串创建sds robj的统一对外接口
robj *createStringObject(const char *ptr, size_t len) {
    if (len <= OBJ_ENCODING_EMBSTR_SIZE_LIMIT)
        return createEmbeddedStringObject(ptr,len);
    else
        return createRawStringObject(ptr,len);
}
//--- RAW类型字符串 robj
robj *createRawStringObject(const char *ptr, size_t len) {
    return createObject(OBJ_STRING, sdsnewlen(ptr,len));
}
//--- EMBSTR类型字符串 robj
robj *createEmbeddedStringObject(const char *ptr, size_t len) {
    robj *o = zmalloc(sizeof(robj)+sizeof(struct sdshdr8)+len+1);
    struct sdshdr8 *sh = (void*)(o+1);

    o->type = OBJ_STRING;
    o->encoding = OBJ_ENCODING_EMBSTR;
    o->ptr = sh+1;
    o->refcount = 1;
    if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
        o->lru = (LFUGetTimeInMinutes()<<8) | LFU_INIT_VAL;
    } else {
        o->lru = LRU_CLOCK();
    }

    sh->len = len;
    sh->alloc = len;
    sh->flags = SDS_TYPE_8;
    if (ptr == SDS_NOINIT)
        sh->buf[len] = '\0';
    else if (ptr) {
        memcpy(sh->buf,ptr,len);
        sh->buf[len] = '\0';
    } else {
        memset(sh->buf,0,len+1);
    }
    return o;
}

//--- 通过数值创建sds robj
robj *createStringObjectFromLongLong(long long value) {
    return createStringObjectFromLongLongWithOptions(value,0);
}

robj *createStringObjectFromLongLongForValue(long long value) {
    return createStringObjectFromLongLongWithOptions(value,1);
}

/*
参数value表示数值; 参数valueobj主要用来标识想获取一个共享对象还是新建一个对象;
因为共享对象是不会过期的, 如果调用本函数是想创建key/value键值对的话, 其实是希望
对象能够过期的, 所以此时valueobj可以传入一个非0值来标识想创建一个新对象
*/
robj *createStringObjectFromLongLongWithOptions(long long value, int valueobj) {
    robj *o;

    //在内存或配置允许的情况下, 尽量返回缓存的内容
    if (server.maxmemory == 0 ||
        !(server.maxmemory_policy & MAXMEMORY_FLAG_NO_SHARED_INTEGERS))
    {
        valueobj = 0;
    }
    
    //缓存范围 OBJ_SHARED_INTEGERS 10000
    if (value >= 0 && value < OBJ_SHARED_INTEGERS && valueobj == 0) {
        incrRefCount(shared.integers[value]);
        o = shared.integers[value];
    } else {
        //由于robj ptr只有8字节长度, 且对于整数类型是直接存储在ptr值中的,
        //所以ptr只能存储long型数值, 超出Long范围的值则会使用字符串表示;
        if (value >= LONG_MIN && value <= LONG_MAX) {
            o = createObject(OBJ_STRING, NULL);
            o->encoding = OBJ_ENCODING_INT;
            o->ptr = (void*)((long)value);
        } else {
            o = createObject(OBJ_STRING, sdsfromlonglong(value));
        }
    }
    return o;
}

//--- 复制一个现有sds robj
robj *dupStringObject(const robj *o) {
    robj *d;

    serverAssert(o->type == OBJ_STRING);

    switch(o->encoding) {
    case OBJ_ENCODING_RAW:
        return createRawStringObject(o->ptr,sdslen(o->ptr));
    case OBJ_ENCODING_EMBSTR:
        return createEmbeddedStringObject(o->ptr,sdslen(o->ptr));
    case OBJ_ENCODING_INT:
        d = createObject(OBJ_STRING, NULL);
        //数值型的值存储在ptr中,所以拷贝ptr值以及设置encoding    
        d->encoding = OBJ_ENCODING_INT;
        d->ptr = o->ptr;
        return d;
    default:
        serverPanic("Wrong encoding.");
        break;
    }
}
```

### 2. 创建其它类型

```c
robj *createQuicklistObject(void) {
    quicklist *l = quicklistCreate();
    robj *o = createObject(OBJ_LIST,l);
    o->encoding = OBJ_ENCODING_QUICKLIST;
    return o;
}

robj *createSetObject(void) {
    dict *d = dictCreate(&setDictType,NULL);
    robj *o = createObject(OBJ_SET,d);
    o->encoding = OBJ_ENCODING_HT;
    return o;
}

robj *createHashObject(void) {
    unsigned char *zl = ziplistNew();
    robj *o = createObject(OBJ_HASH, zl);
    o->encoding = OBJ_ENCODING_ZIPLIST;
    return o;
}

robj *createZsetObject(void) {
    zset *zs = zmalloc(sizeof(*zs));
    robj *o;

    zs->dict = dictCreate(&zsetDictType,NULL);
    zs->zsl = zslCreate();
    o = createObject(OBJ_ZSET,zs);
    o->encoding = OBJ_ENCODING_SKIPLIST;
    return o;
}
//.... 其它略
```



## 其它操作待补