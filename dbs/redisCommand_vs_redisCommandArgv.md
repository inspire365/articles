# redisCommand VS redisCommandArgv

####   经过对比发现 Redis 官方C API中 redisCommandArgv 比 redisCommand 更高效！
####   redisCommandArgv is more effective than redisCommand

   redisCommand & redisCommandArgv are two interfaces of redis official C client [hiredis](https://github.com/redis/hiredis). At the early begining, I used redisCommand to access Redis, it is intuitive and easy to understand. This lasted a long time. 
   
   Recently, I find that redisCommandArgv is more effective than redisCommand. Because redisCommand first parse the cmd and then reassemble to redis protocol, while redisCommandArgv has not parse phase. Show the code:

```C++
void *redisCommand(redisContext *c, const char *format, ...) {
    va_list ap;
    void *reply = NULL;
    va_start(ap,format);
    reply = redisvCommand(c,format,ap);
    va_end(ap);
    return reply;
}

void *redisvCommand(redisContext *c, const char *format, va_list ap) {
    if (redisvAppendCommand(c,format,ap) != REDIS_OK)
        return NULL;
    return __redisBlockForReply(c);
}

int redisvAppendCommand(redisContext *c, const char *format, va_list ap) {
    char *cmd;
    int len;

    len = redisvFormatCommand(&cmd,format,ap);   // here parse and format redis cmd
    if (len == -1) {
        __redisSetError(c,REDIS_ERR_OOM,"Out of memory");
        return REDIS_ERR;
    }

    if (__redisAppendCommand(c,cmd,len) != REDIS_OK) {
        free(cmd);
        return REDIS_ERR;
    }

    free(cmd);
    return REDIS_OK;
}
```
  
```C++
void *redisCommandArgv(redisContext *c, int argc, const char **argv, const size_t *argvlen) {
    if (redisAppendCommandArgv(c,argc,argv,argvlen) != REDIS_OK)    // no parse phase
        return NULL;
    return __redisBlockForReply(c);
}
```




