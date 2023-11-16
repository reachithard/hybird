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



