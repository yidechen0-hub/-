# c++基于hiredis实现分布式锁

```c++
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <time.h>
 
#include <hiredis.h>
/*
#define REDIS_REPLY_STRING 1
#define REDIS_REPLY_ARRAY 2
#define REDIS_REPLY_INTEGER 3
#define REDIS_REPLY_NIL 4
#define REDIS_REPLY_STATUS 5
#define REDIS_REPLY_ERROR 6
*/
 
int main(int argc, char **argv) {
        unsigned int j;
        redisContext *c;
        redisReply *reply;
        const char *hostname = (argc > 1) ? argv[1] : "127.0.0.1";
        int port = (argc > 2) ? atoi(argv[2]) : 6379;
 
        struct timeval timeout = { 1, 500000 }; // 1.5 seconds
        c = redisConnectWithTimeout(hostname, port, timeout);
        if (c == NULL || c->err) {
                if (c) {
                        printf("Connection error: %s\n", c->errstr);
                        redisFree(c);
                } else {
                        printf("Connection error: can't allocate redis context\n");
                }
                exit(1);
        }
 
        //********************** set lock **********************
        srand((unsigned)time(NULL));
        int randomvalue = rand()%1000000;
        char s[10];
        sprintf(s, "%d", randomvalue);
        while(1)
        {
                reply = redisCommand(c,"set lock %d ex %d nx", randomvalue, 20);    // 20s
                printf("SET lock: type=%d, integer=%lld, str=%s\n", reply->type, reply->integer, reply->str);
                if(reply->type != REDIS_REPLY_NIL && strcmp(reply->str, "OK") == 0){
                        printf("SET lock ok\n");
                        freeReplyObject(reply);
                        break;
                }
                else{
                        printf("set lock failed\n");
                        freeReplyObject(reply);
                        sleep(1);
                        continue;
                }
        }
        //*****************************************************
        //********************** Process **********************
        // get lock
        reply = redisCommand(c,"GET lock");
        printf("GET lock: integer=%lld, str=%s\n", reply->integer, reply->str);
        freeReplyObject(reply);
 
        // ttl lock
        reply = redisCommand(c,"TTL lock");
        printf("TTL lock: integer=%lld, str=%s\n", reply->integer, reply->str);
        freeReplyObject(reply);
 
        sleep(5);
        //*****************************************************
        //********************** del lock *********************
    	// redis使用eval命令运行lua脚本
    	
        char script[100] = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
    	// 参数1：脚本	参数2：KEYS数组中的元素数量	参数3：KEYS数组的第一个元素，即要检查和可能删除的键的名称
    	// 参数4：ARGV数组的第一个元素，即要与键"lock"的值比较的值
        reply = redisCommand(c,"eval %s %d %s %s", script, 1, "lock", s);
        printf("DEL lock: integer=%lld, str=%s\n", reply->integer, reply->str);
        freeReplyObject(reply);
        //*****************************************************
 
        // Disconnects and frees the context
        redisFree(c);
 
        return 0;
 
}
```

## 为什么需要使用lua脚本

因为解锁的过程就是将 lock_key 键删除，但不能乱删，要保证执行操作的客户端就是加锁的客户端。所以，解锁的时候，我们要先判断锁的 unique_value 是否为加锁客户端，是的话，才将 lock_key 键删除。

可以看到，解锁是有两个操作，这时就需要 Lua 脚本来保证解锁的原子性，因为 Redis 在执行 Lua 脚本时，可以以原子性的方式执行，保证了锁释放操作的原子性。

# 测试结果

## client 1

```c++
SET lock: type=5, integer=0, str=OK
SET lock ok
GET lock: integer=0, str=397618
TTL lock: integer=20, str=(null)
DEL lock: integer=1, str=(null)
```



## client 2

~~~
SET lock: type=4, integer=0, str=(null)
set lock failed
SET lock: type=4, integer=0, str=(null)
set lock failed
SET lock: type=4, integer=0, str=(null)
set lock failed
SET lock: type=4, integer=0, str=(null)
set lock failed
SET lock: type=5, integer=0, str=OK
SET lock ok
GET lock: integer=0, str=763395
TTL lock: integer=20, str=(null)
DEL lock: integer=1, str=(null)
~~~

