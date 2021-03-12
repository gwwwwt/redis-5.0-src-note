# redisCommand & redisCommandTable



### 1. 作用

> Redis支持的每个命令都对应了一个redisCommand；

### 2. 定义

```C
//server.h
typedef void redisCommandProc(client *c);

typedef int *redisGetKeysProc(struct redisCommand *cmd, robj **argv, int argc, int *numkeys);

struct redisCommand {
    char *name;
    redisCommandProc *proc;
    int arity;
    char *sflags; /* Flags as string representation, one char per flag. */
    int flags;    /* The actual flags, obtained from the 'sflags' field. */
    /* Use a function to determine keys arguments in a command line.
     * Used for Redis Cluster redirect. */
    redisGetKeysProc *getkeys_proc;
    /* What keys should be loaded in background when calling this command? */
    int firstkey; /* The first argument that's a key (0 = no keys) */
    int lastkey;  /* The last argument that's a key */
    int keystep;  /* The step between first and last key */
    long long microseconds, calls;
};
```

#### 2.1 name

> 命令名称

#### 2.2 proc

> 命令处理函数

#### 2.3 arity

> **命令参数数量，用于校验命令请求格式是否正确；当arity值小于0时，表示命令可接受变长参数，但参数数量必须大于或等于arity的绝对值； 而当arity值大于0时，表示命令接受定长参数，参数数量必须等于arity值；**
>
> **这里需要注意的是命令的名称本身也是一个参数，如 get命令的参数数量为2，请求格式为 get key；**

#### 2.4 sflags

> 命令标志，标识命令是读还是写命令；字符串类型，主要是为了方便用阅读，它不用于实际的标志检查；

#### 2.5 flags

> 实际的二进制标志，服务器启动时解析sflags字段生成；

#### 2.6 calls

> 从服务器启动至今命令执行的次数，用于统计；

#### 2.7 microseconds

> 服务器启动至今命令总的执行时间，microseconds/calls 即可计算出该命令的平均处理时间，用于统计；



### 3. 所有命令初始列表

```C
// server.c
/*
服务器启动时会调用 populateCommandTable 函数将redisCommandTable数组中的内容写入到 server.commands 字段

server为 redisServer结构体类型; server.commands字段为字典类型;
*/
struct redisCommand redisCommandTable[] = {
    {"module",moduleCommand,-2,"as",0,NULL,0,0,0,0,0},
    {"get",getCommand,2,"rF",0,NULL,1,1,1,0,0},
    {"set",setCommand,-3,"wm",0,NULL,1,1,1,0,0},
    {"setnx",setnxCommand,3,"wmF",0,NULL,1,1,1,0,0},
  ...
};

void populateCommandTable(void) {
    int j;
    int numcommands = sizeof(redisCommandTable)/sizeof(struct redisCommand);

    for (j = 0; j < numcommands; j++) {
        struct redisCommand *c = redisCommandTable+j;
        char *f = c->sflags;
        int retval1, retval2;

        while(*f != '\0') {
            switch(*f) {
            case 'w': c->flags |= CMD_WRITE; break;
            case 'r': c->flags |= CMD_READONLY; break;
            case 'm': c->flags |= CMD_DENYOOM; break;
            case 'a': c->flags |= CMD_ADMIN; break;
            case 'p': c->flags |= CMD_PUBSUB; break;
            case 's': c->flags |= CMD_NOSCRIPT; break;
            case 'R': c->flags |= CMD_RANDOM; break;
            case 'S': c->flags |= CMD_SORT_FOR_SCRIPT; break;
            case 'l': c->flags |= CMD_LOADING; break;
            case 't': c->flags |= CMD_STALE; break;
            case 'M': c->flags |= CMD_SKIP_MONITOR; break;
            case 'k': c->flags |= CMD_ASKING; break;
            case 'F': c->flags |= CMD_FAST; break;
            default: serverPanic("Unsupported command flag"); break;
            }
            f++;
        }

        retval1 = dictAdd(server.commands, sdsnew(c->name), c);
        /* Populate an additional dictionary that will be unaffected
         * by rename-command statements in redis.conf. */
				/*
        这里通过调用 dictAdd 也将命令加入到 orig_commands字典中;
        它的作用是 客户端可以通过在 redis.conf 配置文件中重命名命令, 该命令将写到 server.commands 字典中;
        但原始命令还保存在 orig_commands 中;
        */
        retval2 = dictAdd(server.orig_commands, sdsnew(c->name), c);
      
      	serverAssert(retval1 == DICT_OK && retval2 == DICT_OK);
    }
}
```



### 4. redisServer 中的命令缓存

> **上面将所有命令加入到 server.commands 字典中；**
>
> **为了加快命令搜索过程，redisServer结构体中定义了几个缓存字段；**

```c
struct redisServer {
  //...
  struct redisCommand *delCommand, *multiCommand, *lpushCommand,
                        *lpopCommand, *rpopCommand, *zpopminCommand,
                        *zpopmaxCommand, *sremCommand, *execCommand,
                        *expireCommand, *pexpireCommand, *xclaimCommand,
                        *xgroupCommand;
  //...
}
```

