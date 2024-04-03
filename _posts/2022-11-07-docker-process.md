---
toc: true
title: "docker 的1号进程可以被杀死吗?"
categories:
  - devops
tags:
  - linux
  - docker
---

我想修改容器镜像里的一个 bug，但因为网路配置的问题，这个同学又不想为了重建 pod 去改变 pod IP。让 pod 做个原地重启了

```bash
git clone https://github.com/chengyli/training.git
 
docker stop sig-proc;docker rm sig-proc
docker run --name sig-proc -d registry/sig-proc:v1 /init.sh
docker exec -it sig-proc bash
```

执行kill 1 , kill -9 1 都无法让进程终止

其他语言的init进程能否杀掉？

C语言

```bash
docker stop sig-proc;docker rm sig-proc
docker run --name sig-proc -d registry/sig-proc:v1 /c-init-nosig
docker exec -it sig-proc bash
```

Golang

```bash
docker stop sig-proc;docker rm sig-proc
docker run --name sig-proc -d registry/sig-proc:v1 /go-init
docker exec -it sig-proc bash
```

kill 命令下达后, linux 究竟发生了什么？

![](https://raw.githubusercontent.com/xiaoshenwei/xiaoshenwei.github.io/master/assets/images/image2022-11-7_17-30-25.png)

内核在决定把信号发送给init进程的时候会调用sig_task_ignored() 这个函数来做判断

```c
kernel/signal.c
static bool sig_task_ignored(struct task_struct *t, int sig, bool force)
{
        void __user *handler;
        handler = sig_handler(t, sig);
 
        /* SIGKILL and SIGSTOP may not be sent to the global init */
        if (unlikely(is_global_init(t) && sig_kernel_only(sig)))
 
                return true;
 
        if (unlikely(t->signal->flags & SIGNAL_UNKILLABLE) &&
            handler == SIG_DFL && !(force && sig_kernel_only(sig)))
          SIGTREM = 1
                return true;
 
        /* Only allow kernel generated signals to this kthread */
        if (unlikely((t->flags & PF_KTHREAD) &&
                     (handler == SIG_KTHREAD_KERNEL) && !force))
                return true;
 
        return sig_handler_ignored(handler, sig);
}
```

handler == SIG_DFL

SIG_DEL: 对于每个信号， 如果用户自己不注册一个handler， 就会有一个缺省的handler

Linux 内核针对每个Namespace的init进程，把只有default handler的信号都给忽略了。

换而言之， 如果我们自己注册了信号的handler， 那么信号的handle就不再是SIG_DEF， 因此init进程接受到也是可以退出的。

注意：SIGKILL SIGSTOP 是不允许用户自己注册的

所以上述中SIGTERM可以杀掉init进程。

```bash
### golang init
# cat /proc/1/status | grep -i SigCgt
SigCgt:     fffffffe7fc1feff
 
### C init
# cat /proc/1/status | grep -i SigCgt
SigCgt:     0000000000000000
 
### bash init
# cat /proc/1/status | grep -i SigCgt
SigCgt:     0000000000010002
 
# https://qastack.cn/unix/85364/how-can-i-check-what-signals-a-process-is-listening-to
```

Golang程序中，很多信号注册了自己的handler, 当然也包括SIGTERM(15), 也就是bit 15

C 缺省状态下，一个都没有

Bash注册了两个，bit2 bit 17 也就是SIGINT, SIGCHILD



结论

1号进程, 永远不会响应SIGKILL, SIGSTOP两个进程，对于其他信号， 1号进程是可以响应的