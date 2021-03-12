# serverLog函数

```c
// server.h 中定义的日志级别
#define LL_DEBUG 0
#define LL_VERBOSE 1
#define LL_NOTICE 2
#define LL_WARNING 3
#define LL_RAW (1<<10) /* Modifier to log without timestamp */
#define CONFIG_DEFAULT_VERBOSITY LL_NOTICE //默认的日志级别为LL_NOTICE

//在 server.c#main 方法中记录日志应用, 以下一行输出的是redis启动时的第二行信息;
serverLog(LL_WARNING,
   "Redis version=%s, bits=%d, commit=%s, modified=%d, pid=%d, just started",
   REDIS_VERSION, (sizeof(long) == 8) ? 64 : 32, redisGitSHA1(),
   strtol(redisGitDirty(),NULL,10) > 0, (int)getpid());

//server.c#serverLog函数实现; level指示要输出信息的级别, fmt以及后面的变长参数用于格式化输出;
void serverLog(int level, const char *fmt, ...) {
    va_list ap;
    char msg[LOG_MAX_LEN];
    
    //server.verbosity字段存储的是指定的server日志级别; 如果level参数值小于server.verbosity, 直接跳过;
    if ((level&0xff) < server.verbosity) return; 

    va_start(ap, fmt);
    vsnprintf(msg, sizeof(msg), fmt, ap); //将格式化字符串写入msg字符数组中;
    va_end(ap);

    serverLogRaw(level,msg); //实际的输出函数;
}

//server.c#serverLogRaw 函数; 
/*
根据函数逻辑, level可以使用 "LL_WARNING" 来指定, 也可以使用 "LL_WARNING & LL_RAW" 来指定; 
如果带 LL_RAW 标志, 会直接输出 msg 参数; 
但如果不带 LL_RAW 标志, 则会先输出 pid/启动模式/时间 (具体参考下面的实现) 等信息之后, 再输出 msg 字符串; 
*/
void serverLogRaw(int level, const char *msg) {
    const int syslogLevelMap[] = { LOG_DEBUG, LOG_INFO, LOG_NOTICE, LOG_WARNING };
    const char *c = ".-*#";
    FILE *fp;
    char buf[64];
    int rawmode = (level & LL_RAW); 
    int log_to_stdout = server.logfile[0] == '\0';

    level &= 0xff; /* clear flags */
    if (level < server.verbosity) return;

    fp = log_to_stdout ? stdout : fopen(server.logfile,"a");
    if (!fp) return;

    if (rawmode) { //若level中指定了 LL_RAW标志, 则直接输出msg字符串
        fprintf(fp,"%s",msg);
    } else {
        int off;
        struct timeval tv;
        int role_char;
        pid_t pid = getpid();

        gettimeofday(&tv,NULL);
        struct tm tm;
        nolocks_localtime(&tm,tv.tv_sec,server.timezone,server.daylight_active);
        off = strftime(buf,sizeof(buf),"%d %b %Y %H:%M:%S.",&tm);
        snprintf(buf+off,sizeof(buf)-off,"%03d",(int)tv.tv_usec/1000);
        if (server.sentinel_mode) {
            role_char = 'X'; /* Sentinel. */
        } else if (pid != server.pid) {
            role_char = 'C'; /* RDB / AOF writing child. */
        } else {
            role_char = (server.masterhost ? 'S':'M'); /* Slave or Master. */
        }
        //c[level]用于根据level参数确定显示提示符信息, 即".-*#"中的其中一个;
        fprintf(fp,"%d:%c %s %c %s\n", (int)getpid(),role_char, buf,c[level],msg);
    }
    }
    fflush(fp);

    if (!log_to_stdout) fclose(fp);
    if (server.syslog_enabled) syslog(syslogLevelMap[level], "%s", msg);
}
```

