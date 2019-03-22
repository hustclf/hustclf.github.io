---
title: redis内存使用统计及碎片率分析
tags: redis
abbrlink: a39640ef
date: 2017-04-11 22:38:36
---
# 如何获取redis内存使用情况
redis提供了info命令，可以获取redis内存使用情况。
其中关于redis内存的部分有七个参数，used_memory、used_memory_human、used_memory_rss、used_memory_peak、used_memory_peak_human、mem_fragmentation_ratio、used_memory_lua和mem_allocator。

各参数的含义如下：

+ used_memory : 由 Redis 分配器分配的内存总量，以字节（byte）为单位
+ used_memory_human : 以人类可读的格式返回 Redis 分配的内存总量
+ used_memory_rss : 从操作系统的角度，返回Redis已分配的内存总量（俗称常驻集大小）。这个值和 top 、 ps 等命令的输出一致。
+ used_memory_peak : Redis的内存消耗峰值（以字节为单位）
+ used_memory_peak_human : 以人类可读的格式返回 Redis 的内存消耗峰值
+ mem_fragmentation_ratio : used_memory_rss 和 used_memory 之间的比率
+ used_memory_lua : Lua 引擎所使用的内存大小（以字节为单位）
+ mem_allocator : 在编译时指定的，Redis所使用的内存分配器。可以是 libc 、 jemalloc 或者 tcmalloc 。

<!-- more -->
```
redis 127.0.0.1:6389> info
    ...
    used_memory:832323728
    used_memory_human:793.77M
    used_memory_rss:874582016
    used_memory_peak:1067840504
    used_memory_peak_human:1018.37M
    mem_fragmentation_ratio:1.05
    mem_allocator:jemalloc-2.2.5
    ...
```
# 分析
  从以上参数的官方介绍来看，统计redis内存使用量主要是used_memory used_memory_rss used_memory_lua,这三部分。其他参数（除mem_allocator）都是由它们产生的统计值。

其中

+ used_memory_peak统计的是used_memory的最大值。

+ mem_fragmentation_ratio统计的是used_memory_rss和used_memory的比值，其值与1的大小关系，代表redis的内存碎片率情况。正常情况下，从操作系统统计的内存使用量（used_memory_rss）应该稍大于redis内存分配器统计的内存使用（used_momory）,即mem_fragmentation_ratio应稍大于1。若mem_fragmentation远大于1，代表操作系统实际分配内存要远大于redis自身统计值，代表redis内部可能有内存碎片，没有被redis统计到；若mem_fragmentation小于1，代表操作系统实际分配内存小于redis自身统计值，此时表示Redis的部分内存被操作系统换出交换空间了，此时redis会产生明显的延迟。

# 从源码跟踪这些参数
获取INFO返回信息的入口定义在redis.c的genRedisInfoString中

```
//redis.c

    sds genRedisInfoString(char *section) {
    ...
    /* Memory */
        info = sdscatprintf(info,
            "# Memory\r\n"
            "used_memory:%zu\r\n"
            "used_memory_human:%s\r\n"
            "used_memory_rss:%zu\r\n"
            "used_memory_peak:%zu\r\n"
            "used_memory_peak_human:%s\r\n"
            "used_memory_lua:%lld\r\n"
            "mem_fragmentation_ratio:%.2f\r\n"
            "mem_allocator:%s\r\n",
            zmalloc_used,
            hmem,
            server.resident_set_size,
            server.stat_peak_memory,
            peak_hmem,
            ((long long)lua_gc(server.lua,LUA_GCCOUNT,0))*1024LL,
            zmalloc_get_fragmentation_ratio(server.resident_set_size),
            ZMALLOC_LIB
            );
    ...
    }
```


    used_memory       => zmalloc_used
    used_memory_rss   => server.resident_set_size
    used_memory_peak  => server.stat_pear_memory
    used_memory_lua   => lua_gc(server.lua,LUA_GCCOUNT,0))*1024LL
## 跟踪used_memory
used_memory对应的变量zmalloc_used，是通过zmalloc_used_memory()返回的，该函数定义在zmalloc.c中

```
// zmalloc.c

size_t zmalloc_used_memory(void) {
    size_t um;

    if (zmalloc_thread_safe) {
#ifdef HAVE_ATOMIC
        um = __sync_add_and_fetch(&used_memory, 0);
#else
        pthread_mutex_lock(&used_memory_mutex);
        um = used_memory;
        pthread_mutex_unlock(&used_memory_mutex);
#endif
    }
    else {
        um = used_memory;
    }

    return um;
}
```

从该函数可以看出，zmalloc_used的值取自used_memory变量。当然针对是否线程安全以及是否是有原子性操作，对used_memory的读取策略不同，但都是读取的used_memory的值。

used_memory的初始化以及修改操作都在zmalloc.c中完成。

```
//zmalloc.c

// 根据HAVE_ATOMIC来定义used_memory的加和减的方式
#ifdef HAVE_ATOMIC
#define update_zmalloc_stat_add(__n) __sync_add_and_fetch(&used_memory, (__n))
#define update_zmalloc_stat_sub(__n) __sync_sub_and_fetch(&used_memory, (__n))
#else
#define update_zmalloc_stat_add(__n) do { \
    pthread_mutex_lock(&used_memory_mutex); \
    used_memory += (__n); \
    pthread_mutex_unlock(&used_memory_mutex); \
} while(0)

#define update_zmalloc_stat_sub(__n) do { \
    pthread_mutex_lock(&used_memory_mutex); \
    used_memory -= (__n); \
    pthread_mutex_unlock(&used_memory_mutex); \
} while(0)

#endif

#define update_zmalloc_stat_alloc(__n) do { \
    size_t _n = (__n); \
    if (_n&(sizeof(long)-1)) _n += sizeof(long)-(_n&(sizeof(long)-1)); \
    if (zmalloc_thread_safe) { \
        update_zmalloc_stat_add(_n); \
    } else { \
        used_memory += _n; \
    } \
} while(0)

#define update_zmalloc_stat_free(__n) do { \
    size_t _n = (__n); \
    if (_n&(sizeof(long)-1)) _n += sizeof(long)-(_n&(sizeof(long)-1)); \
    if (zmalloc_thread_safe) { \
        update_zmalloc_stat_sub(_n); \
    } else { \
        used_memory -= _n; \
    } \
} while(0)

// used_memory 初始化
static size_t used_memory = 0;
static int zmalloc_thread_safe = 0;
```
## 跟踪used_memory_rss
used_memory_rss对应的变量server.resident_set_size，该值来自于redis.c中的zmalloc_get_rss函数，
```
//redis.c

    /* Sample the RSS here since this is a relatively slow call. */
    server.resident_set_size = zmalloc_get_rss();
```

这个函数定义在zmalloc.c中，会根据不同的操作系统，读取对应的操作系统分配的内存
```
// zmalloc.c

// 如果是linux环境，则读取proc信息
#if defined(HAVE_PROC_STAT)
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>

size_t zmalloc_get_rss(void) {
    int page = sysconf(_SC_PAGESIZE);
    size_t rss;
    char buf[4096];
    char filename[256];
    int fd, count;
    char *p, *x;

    snprintf(filename,256,"/proc/%d/stat",getpid());
    if ((fd = open(filename,O_RDONLY)) == -1) return 0;
    if (read(fd,buf,4096) <= 0) {
        close(fd);
        return 0;
    }
    close(fd);

    p = buf;
    count = 23; /* RSS is the 24th field in /proc/<pid>/stat */
    while(p && count--) {
        p = strchr(p,' ');
        if (p) p++;
    }
    if (!p) return 0;
    x = strchr(p,' ');
    if (!x) return 0;
    *x = '\0';

    rss = strtoll(p,NULL,10);
    rss *= page;
    return rss;
}

// 如果是apple, 则读取task info
#elif defined(HAVE_TASKINFO)
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/sysctl.h>
#include <mach/task.h>
#include <mach/mach_init.h>

size_t zmalloc_get_rss(void) {
    task_t task = MACH_PORT_NULL;
    struct task_basic_info t_info;
    mach_msg_type_number_t t_info_count = TASK_BASIC_INFO_COUNT;

    if (task_for_pid(current_task(), getpid(), &task) != KERN_SUCCESS)
        return 0;
    task_info(task, TASK_BASIC_INFO, (task_info_t)&t_info, &t_info_count);

    return t_info.resident_size;
}
// 如果都不是，则读取used_memory作为used_memory_rss
#else
size_t zmalloc_get_rss(void) {
    /* If we can't get the RSS in an OS-specific way for this system just
     * return the memory usage we estimated in zmalloc()..
     *
     * Fragmentation will appear to be always 1 (no fragmentation)
     * of course... */
    return zmalloc_used_memory();
}
#endif
```


## 跟踪used_memory_lua
used_memory_lua从lua_gc(server.lua,LUA_GCCOUNT,0))*1024LL获取，lua_gc定义在redis\deps\lua\src\lapi.c，通过该函数可以获取LUA引擎占用的内存。（具体lua内部是如何获取的就不清楚了。。）
```
// deps\lua\src\lapi.c

/*
** Garbage-collection function
*/

LUA_API int lua_gc (lua_State *L, int what, int data) {
  int res = 0;
  global_State *g;
  lua_lock(L);
  g = G(L);
  switch (what) {
    case LUA_GCSTOP: {
      g->GCthreshold = MAX_LUMEM;
      break;
    }
    case LUA_GCRESTART: {
      g->GCthreshold = g->totalbytes;
      break;
    }
    case LUA_GCCOLLECT: {
      luaC_fullgc(L);
      break;
    }
    case LUA_GCCOUNT: {
      /* GC values are expressed in Kbytes: #bytes/2^10 */
      res = cast_int(g->totalbytes >> 10);
      break;
    }
    case LUA_GCCOUNTB: {
      res = cast_int(g->totalbytes & 0x3ff);
      break;
    }
    case LUA_GCSTEP: {
      lu_mem a = (cast(lu_mem, data) << 10);
      if (a <= g->totalbytes)
        g->GCthreshold = g->totalbytes - a;
      else
        g->GCthreshold = 0;
      while (g->GCthreshold <= g->totalbytes) {
        luaC_step(L);
        if (g->gcstate == GCSpause) {  /* end of cycle? */
          res = 1;  /* signal it */
          break;
        }
      }
      break;
    }
    case LUA_GCSETPAUSE: {
      res = g->gcpause;
      g->gcpause = data;
      break;
    }
    case LUA_GCSETSTEPMUL: {
      res = g->gcstepmul;
      g->gcstepmul = data;
      break;
    }
    default: res = -1;  /* invalid option */
  }
  lua_unlock(L);
  return res;
}
```

## 跟踪used_memory_peak
used_memory_peak对应server.stat_peak_memory。该值每次读取时，都会通过zmalloc_used_memory()获取最新的used_memory的值，并记录used_memory的最大值。
```
//redis.c
        // used_memory是通过时间事件更新的，为防止出现峰值内存小于当前redis使用内存的情况，此时会重新读取一次最新的内存值，并更新峰值内存
        size_t zmalloc_used = zmalloc_used_memory();
        if (zmalloc_used > server.stat_peak_memory)
            server.stat_peak_memory = zmalloc_used;
```


# 内存碎片产生原因
+ 操作系统使用不连续的内存碎片来提供给内存分配器。
当操作系统没有足够的内存时，无法提供整块的内存供redis内存分配器，内存分配器对内存碎片的利用率较低。

+ 内存分配器为加快程序运行，额外使用了内存

+ 内存分配器释放了内存块，但没有将内存返还给系统

+ 分配内存块大小与内存分配器的类型（libc    jemalloc tcmalloc)不同, 产生的内存碎片量也不同。
+ 如果used_memory_peak和used_memory_rss大致相等，且远大于used_memory,则说明额外的内存碎片正在产生。
