+++
title = '【Redis源码分析】Redis真的不是单线程，后台IO服务 BIO'
date = 2020-07-24T10:18:33+08:00
draft = false
tags = [
    "redis",
    "Linux内核",
    "Linux编程"
]
categories = [
    "C/C++"
]
+++

面试总喜欢被问Redis是单线程还是多线程，千篇一律的回答单线程却不知所以然，严格来说Redis是多线程多进程、单线程处理请求，本文说的就是多线程下的BIO(Background I/O service)。

<!--more-->

由于redis单线程处理请求，所以某些耗时的操作被作为异步任务，有三种任务分别是：关闭文件、AOF同步磁盘、释放空间。

``` cpp
/* Background job opcodes */
#define BIO_CLOSE_FILE    0 // close()，重写aof，会关闭旧文件fd，具体实现可参考aof.c的backgroundRewriteDoneHandler()
#define BIO_AOF_FSYNC     1 // fsync()
#define BIO_LAZY_FREE     2 // 延迟释放
#define BIO_NUM_OPS       3
```

每种任务类型都有独立的队列、执行线程、互斥锁，每个任务执行完成并不会callback通知调用方。
``` cpp
static pthread_t bio_threads[BIO_NUM_OPS];
static pthread_mutex_t bio_mutex[BIO_NUM_OPS];
static pthread_cond_t bio_newjob_cond[BIO_NUM_OPS];
static pthread_cond_t bio_step_cond[BIO_NUM_OPS];
static list *bio_jobs[BIO_NUM_OPS];
// 用于记录每个任务类型的任务数，假如主线程有与BIO共享的数据时，在主线程操作前，会等待直到队列不会有新的任务
// 通常在aof同步以及延迟释放会用到
static unsigned long long bio_pending[BIO_NUM_OPS];

/* This structure represents a background Job. It is only used locally to this
 * file as the API does not expose the internals at all. */
struct bio_job {
    time_t time; /* Time at which the job was created. */
    // 多于三个参数可以传递指针or结构体
    void *arg1, *arg2, *arg3;
};

void *bioProcessBackgroundJobs(void *arg);
// 下面三个方法实现均在lazyfree.c
void lazyfreeFreeObjectFromBioThread(robj *o);
void lazyfreeFreeDatabaseFromBioThread(dict *ht1, dict *ht2);
void lazyfreeFreeSlotsMapFromBioThread(zskiplist *sl);

/* Make sure we have enough stack to perform all the things we do in the
 * main thread. */
#define REDIS_THREAD_STACK_SIZE (1024*1024*4)

// 在server.c中InitServerLast()调用
void bioInit(void) {
    pthread_attr_t attr;
    pthread_t thread;
    size_t stacksize;
    int j;

    /* Initialization of state vars and objects */
    for (j = 0; j < BIO_NUM_OPS; j++) {
        pthread_mutex_init(&bio_mutex[j],NULL);
        pthread_cond_init(&bio_newjob_cond[j],NULL);
        pthread_cond_init(&bio_step_cond[j],NULL);
        bio_jobs[j] = listCreate();
        bio_pending[j] = 0;
    }

    pthread_attr_init(&attr);
    pthread_attr_getstacksize(&attr,&stacksize);
    if (!stacksize) stacksize = 1; // 不废话，至少4MB
    while (stacksize < REDIS_THREAD_STACK_SIZE) stacksize *= 2;
    pthread_attr_setstacksize(&attr, stacksize);

    // just do it
    for (j = 0; j < BIO_NUM_OPS; j++) {
        void *arg = (void*)(unsigned long) j;
        if (pthread_create(&thread,&attr,bioProcessBackgroundJobs,arg) != 0) {
            serverLog(LL_WARNING,"Fatal: Can't initialize Background Jobs.");
            exit(1);
        }
        bio_threads[j] = thread;
    }
}

void bioCreateBackgroundJob(int type, void *arg1, void *arg2, void *arg3) {
    struct bio_job *job = zmalloc(sizeof(*job));
    // 省略
}

// 任务线程主体
void *bioProcessBackgroundJobs(void *arg) {
    struct bio_job *job;
    unsigned long type = (unsigned long) arg;
    sigset_t sigset;

    /* Check that the type is within the right interval. */
    if (type >= BIO_NUM_OPS) {
        serverLog(LL_WARNING,
            "Warning: bio thread started with wrong type %lu",type);
        return NULL;
    }

    // 响应bioKillThreads的调用
    pthread_setcancelstate(PTHREAD_CANCEL_ENABLE, NULL);
    pthread_setcanceltype(PTHREAD_CANCEL_ASYNCHRONOUS, NULL); // 意味着线程收到终止信号时，会立即取消

    pthread_mutex_lock(&bio_mutex[type]);
    // 屏蔽SIGALRM定时器信号
    sigemptyset(&sigset);
    sigaddset(&sigset, SIGALRM);
    if (pthread_sigmask(SIG_BLOCK, &sigset, NULL))
        serverLog(LL_WARNING,
            "Warning: can't mask SIGALRM in bio.c thread: %s", strerror(errno));

    while(1) {
        listNode *ln;

        // 队列没有任务会一直hold着
        if (listLength(bio_jobs[type]) == 0) {
            pthread_cond_wait(&bio_newjob_cond[type],&bio_mutex[type]);
            continue;
        }
        /* Pop the job from the queue. */
        ln = listFirst(bio_jobs[type]);
        job = ln->value;
        // 成功取出任务，则可以释放锁
        pthread_mutex_unlock(&bio_mutex[type]);

        /* Process the job accordingly to its type. */
        if (type == BIO_CLOSE_FILE) {
            close((long)job->arg1);
        } else if (type == BIO_AOF_FSYNC) {
            redis_fsync((long)job->arg1); // syscall fsync
        } else if (type == BIO_LAZY_FREE) {
            // 原注解意思，
            // arg1为要释放的对象指针
            // arg2、arg3为要释放的redis db指针
            // arg3则是个跳表
            if (job->arg1)
                lazyfreeFreeObjectFromBioThread(job->arg1); // 具体参数见：freeObjAsync
            else if (job->arg2 && job->arg3)
                lazyfreeFreeDatabaseFromBioThread(job->arg2,job->arg3); // 具体参数见：emptyDbAsync
            else if (job->arg3)
                lazyfreeFreeSlotsMapFromBioThread(job->arg3); // 具体参数见：slotToKeyFlushAsync
        } else {
            serverPanic("Wrong job type in bioProcessBackgroundJobs().");
        }
        zfree(job); //执行完释放任务

        /* Lock again before reiterating the loop, if there are no longer
         * jobs to process we'll block again in pthread_cond_wait(). */
        pthread_mutex_lock(&bio_mutex[type]);
        listDelNode(bio_jobs[type],ln);
        bio_pending[type]--;

        /* Unblock threads blocked on bioWaitStepOfType() if any. */
        pthread_cond_broadcast(&bio_step_cond[type]);
    }
}

/* Return the number of pending jobs of the specified type. */
unsigned long long bioPendingJobsOfType(int type) {
    unsigned long long val;
    pthread_mutex_lock(&bio_mutex[type]);
    val = bio_pending[type];
    pthread_mutex_unlock(&bio_mutex[type]);
    return val;
}

// 目前看来redis 5.0也没有用到该方法
unsigned long long bioWaitStepOfType(int type) {
    unsigned long long val;
    pthread_mutex_lock(&bio_mutex[type]);
    val = bio_pending[type];
    if (val != 0) {
        pthread_cond_wait(&bio_step_cond[type],&bio_mutex[type]);
        val = bio_pending[type];
    }
    pthread_mutex_unlock(&bio_mutex[type]);
    return val;
}

// 只有在进程奔溃收到SIGSEGV信号，才会执行该方法
void bioKillThreads(void) {
    int err, j;

    for (j = 0; j < BIO_NUM_OPS; j++) {
        if (pthread_cancel(bio_threads[j]) == 0) {
            if ((err = pthread_join(bio_threads[j],NULL)) != 0) {
                serverLog(LL_WARNING,
                    "Bio thread for job type #%d can be joined: %s",
                        j, strerror(err));
            } else {
                serverLog(LL_WARNING,
                    "Bio thread for job type #%d terminated",j);
            }
        }
    }
}

```