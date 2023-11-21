# SRS源码解析1

## 环境搭建

```
git clone git@github.com:ossrs/srs.git
git checkout 5.0release
```

可见本文是基于srs-5.0分支写的

首先，阅读本文前，需要有一定的协程概念。srs使用了state-threads协程库，是单线程多协程模型。 这个协程的概念类似于lua的协程，都是单线程中可以创建多个协程。而golang中的goroutine协程是多线程并发的，goroutine有可能运行在同一个线程也可能在不同线程，这样就有了线程安全问题，所以需要chan通信或者mutex加锁共享资源。 而srs因为是单线程多协程所以不用考虑线程安全，数据不用加锁。

## SRS初始化和配置

```
int main(int argc, char** argv, char** envp)
{
    srs_error_t err = do_main(argc, argv, envp); // 实际启动代码

    if (err != srs_success) {
        srs_error("Failed, %s", srs_error_desc(err).c_str());
    }
    
    int ret = srs_error_code(err);
    srs_freep(err);
    return ret;
}
```

如上，实际的启动代码为`do_main`

```
srs_error_t do_main(int argc, char** argv, char** envp)
{
	... 
	// 参数解析 配置文件解析
	
	// 实际启动代码
	err = run_directly_or_daemon();
	
	...
	return err;
}


srs_error_t run_directly_or_daemon()
{
    srs_error_t err = srs_success;

    // Load daemon from config, disable it for docker.
    // @see https://github.com/ossrs/srs/issues/1594
    bool run_as_daemon = _srs_config->get_daemon();
    if (run_as_daemon && _srs_in_docker && _srs_config->disable_daemon_for_docker()) {
        srs_warn("disable daemon for docker");
        run_as_daemon = false;
    }
    
    // If not daemon, directly run hybrid server.
    if (!run_as_daemon) {
        if ((err = run_in_thread_pool()) != srs_success) {
            return srs_error_wrap(err, "run thread pool");
        }
        return srs_success;
    }
    
    srs_trace("start daemon mode...");
    
    ... // daemon逻辑
    
    // 实际启动代码
    if ((err = run_in_thread_pool()) != srs_success) {
        return srs_error_wrap(err, "daemon run thread pool");
    }
    
    return err;
}

```

run_directly_or_daemon主要是判断是否daemon启动，使用子进程，然后父进程退出，子进程被init接管。

```
srs_error_t run_in_thread_pool()
{
#ifdef SRS_SINGLE_THREAD
    srs_trace("Run in single thread mode");
    return run_hybrid_server(NULL); // 如果单线程，则直接调用run_hybrid_server 否则使用线程池
#else
    srs_error_t err = srs_success;

    // Initialize the thread pool.
    // 初始化线程池
    if ((err = _srs_thread_pool->initialize()) != srs_success) {
        return srs_error_wrap(err, "init thread pool");
    }

    // Start the hybrid service worker thread, for RTMP and RTC server, etc.
    // 将run_hybrid_server任务放入线程池执行，实际就是执行run_hybrid_server函数
    if ((err = _srs_thread_pool->execute("hybrid", run_hybrid_server, (void*)NULL)) != srs_success) {
        return srs_error_wrap(err, "start hybrid server thread");
    }

    srs_trace("Pool: Start threads primordial=1, hybrids=1 ok");

	// 线程池启动
    return _srs_thread_pool->run();
#endif
}
```

下面分析一下线程池

```
srs_error_t SrsThreadPool::execute(string label, srs_error_t (*start)(void* arg), void* arg)
{
    srs_error_t err = srs_success;

    SrsThreadEntry* entry = new SrsThreadEntry();

    // Update the hybrid thread entry for circuit breaker.
    if (label == "hybrid") {
        hybrid_ = entry;
        hybrids_.push_back(entry);
    }

    // To protect the threads_ for executing thread-safe.
    if (true) {
        SrsThreadLocker(lock_);
        threads_.push_back(entry);
    }

	// 记录函数指针，上下文，供start调用 以及参数
    entry->pool = this;
    entry->label = label;
    entry->start = start;
    entry->arg = arg;

    // The id of thread, should equal to the debugger thread id.
    // For gdb, it's: info threads
    // For lldb, it's: thread list
    static int num = entry_->num + 1;
    entry->num = num++;

    char buf[256];
    snprintf(buf, sizeof(buf), "srs-%s-%d", entry->label.c_str(), entry->num);
    entry->name = buf;

    // https://man7.org/linux/man-pages/man3/pthread_create.3.html
    // 开启线程 SrsThreadPool::start即调用entry->start = start;指针
    pthread_t trd;
    int r0 = pthread_create(&trd, NULL, SrsThreadPool::start, entry); // 记录返回参数
    if (r0 != 0) {
        entry->err = srs_error_new(ERROR_THREAD_CREATE, "create thread %s, r0=%d", label.c_str(), r0);
        return srs_error_copy(entry->err);
    }

    entry->trd = trd;

    return err;
}

void* SrsThreadPool::start(void* arg)
{
    srs_error_t err = srs_success;

    SrsThreadEntry* entry = (SrsThreadEntry*)arg;

    // Initialize thread-local variables.
    if ((err = SrsThreadPool::setup_thread_locals()) != srs_success) {
        entry->err = err;
        return NULL;
    }

    // Set the thread local fields.
    entry->tid = gettid();

#ifndef SRS_OSX
    // https://man7.org/linux/man-pages/man3/pthread_setname_np.3.html
    pthread_setname_np(pthread_self(), entry->name.c_str());
#else
    pthread_setname_np(entry->name.c_str());
#endif

    srs_trace("Thread #%d: run with tid=%d, entry=%p, label=%s, name=%s", entry->num, (int)entry->tid, entry, entry->label.c_str(), entry->name.c_str());

    if ((err = entry->start(entry->arg)) != srs_success) { // 调用传入的参数
        entry->err = err;
    }

    // We use a special error to indicates the normally done.
    if (entry->err == srs_success) {
        entry->err = srs_error_new(ERROR_THREAD_FINISHED, "finished normally");
    }

    // We do not use the return value, the err has been set to entry->err.
    return NULL;
}

srs_error_t SrsThreadPool::run()
{
    srs_error_t err = srs_success;

    while (true) {
        vector<SrsThreadEntry*> threads;
        if (true) {
            SrsThreadLocker(lock_);
            threads = threads_;
        }

        // Check the threads status fastly.
        int loops = (int)(interval_ / SRS_UTIME_SECONDS);
        for (int i = 0; i < loops; i++) {
            for (int j = 0; j < (int)threads.size(); j++) {
                SrsThreadEntry* entry = threads.at(j);
                if (entry->err != srs_success) { // 对一些返回值的处理 以及运行时间runtime的一些mertics的打印
                    // Quit with success.
                    if (srs_error_code(entry->err) == ERROR_THREAD_FINISHED) {
                        srs_trace("quit for thread #%d(%s) finished", entry->num, entry->label.c_str());
                        srs_freep(err);
                        return srs_success;
                    }

                    // Quit with specified error.
                    err = srs_error_copy(entry->err);
                    err = srs_error_wrap(err, "thread #%d(%s)", entry->num, entry->label.c_str());
                    return err;
                }
            }

            srs_usleep(1 * SRS_UTIME_SECONDS);
        }

        // Show statistics for RTC server.
        SrsProcSelfStat* u = srs_get_self_proc_stat();
        // Resident Set Size: number of pages the process has in real memory.
        int memory = (int)(u->rss * 4 / 1024);

        srs_trace("Process: cpu=%.2f%%,%dMB, threads=%d", u->percent * 100, memory, (int)threads_.size());
    }

    return err;
}
```

