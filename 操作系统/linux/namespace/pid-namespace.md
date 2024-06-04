## pid namespace

pid namespace表示隔离一个具有独立PID的运行环境。
在每一个pid namespace中，进程的pid都从1开始，且和其他pid namespace中的PID互不影响。
这意味着，不同pid namespace中可以有相同的PID值。

> 在介绍pid namespace之前，先创建其他类型的namespace然后查看进程关系:

```shell
#在root namesapce查看当前进程的id是4048028
root@ubuntu-server:~# echo $$
4048028

root@ubuntu-server:~# pstree -p |grep 4048028
|-sshd(3944519)---sshd(4047935)---bash(4048028)-+-grep(4074182)

#进程的关系:
#systemd(1) |-sshd(3944519 这个是sshd服务server)|-sshd(4047935,这个是ssh连接的进程)|---bash(4048028)-+-grep(4074182)

root@ubuntu-server:~# ps -ef|grep 3944519
root     3944519       1  0 Jan15 ?        00:00:00 sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups
```

> 创建一个uts namespace

```shell
root@ubuntu-server:~# unshare -u /bin/bash
```

> 在uts namespace中查看当前进程

```shell
root@ubuntu-server:~# pstree -p |grep grep
|-sshd(3944519)---sshd(4047935)---bash(4048028)---bash(4074716)-+-grep(4074764)

root@ubuntu-server:# ls -l /proc/4074716/ns|grep uts
lrwxrwxrwx 1 root root 0 Jan 16 15:07 uts -> uts:[4026532372]
root@ubuntu-server:# ls -l /proc/4048028/ns|grep uts
lrwxrwxrwx 1 root root 0 Jan 16 11:17 uts -> uts:[4026531838]
root@ubuntu-server:~# ls -l /proc/4047935/ns|grep uts
lrwxrwxrwx 1 root root 0 Jan 16 11:17 uts -> uts:[4026531838]

#从上面输出的结果可知，创建uts类型namespace时，unshare进程会在创建新的namespace后被该namespace中的第一个进程给替换掉。
#并且我们可以看到unshare创建的bash(4074716)进程与其父进程(4048028),2者的uts namespace是不一致的。
```

> 创建pid namespace：

```shell
unshare --pid --fork [--mount-proc] <CMD>
--pid(-p):    表示创建pid namespace
--mount-proc: 表示创建pid namespace时重新挂载procfs
--fork(-f):   表示创建pid namespace时，不是直接替换unshare进程，而是fork unshare进程，并使用CMD替换fork出来的子进程
```
    
> 我们使用fork的参数来验证一下:
```shell
root@ubuntu-server:~# echo $$
4048028

root@ubuntu-server:# unshare -p -f  --mount-proc /bin/bash
root@ubuntu-server:# pstree -p |grep grep
bash(1)-+-grep(9)

#终端2，重新打开一个终端
root@ubuntu-server:~# pstree -p 4048028
bash(4048028)───unshare(4090558)───bash(4090559)

root@ubuntu-server:# ls -l /proc/4048028/ns/pid
lrwxrwxrwx 1 root root 0 Jan 16 11:17 /proc/4048028/ns/pid -> 'pid:[4026531836]'
root@ubuntu-server:# ls -l /proc/4090558/ns/pid
lrwxrwxrwx 1 root root 0 Jan 16 17:27 /proc/4090558/ns/pid -> 'pid:[4026531836]'
root@ubuntu-server:~# ls -l /proc/4090559/ns/pid
lrwxrwxrwx 1 root root 0 Jan 16 17:27 /proc/4090559/ns/pid -> 'pid:[4026532379]'

#通过这个实验，我们可以看到unshare(4090558)这个进程还是在原来的namespace,并不会加入新的namespace，
而fork出来的bash(4090559)进程,他是加入到了一个新的namespace当中。
```

> 嵌套pid namespace

这个很容易理解，就是不同namespace之间可以相互嵌套，所有的子孙pid namespace中的进程信息都会保存在父级以及祖先级namespace中，
只不过在不同嵌套层级中，同一个进程对应的PID不同。
```shell
#我们这里在namespace1之中在创建一个namespace2看看进程的关系是什么样的
#------第一个终端------
#namespace1
root@ubuntu-server:~# unshare -p -f -u --mount-proc /bin/bash
root@ubuntu-server:~# hostname namespace1
root@ubuntu-server:~# exec bash

#在namespace1中创建namespace2
root@namespace1:~# unshare -p -f -u --mount-proc /bin/bash
root@namespace1:~# hostname namespace2
root@namespace1:~# exec bash
root@namespace2:~# pstree -p |grep grep
bash(1)-+-grep(16)

##------第二个终端------，也就是root namespace下面
root@ubuntu-server:~# pstree -p 4048028
bash(4048028)───unshare(4097920)───bash(4097921)───unshare(4098083)───bash(4098084)

#在上面fork测试中我们知道unshare进程还在原来的namespace，而unshare创建的bash是在新的namespace中。
所以bash(4048028)───unshare(4097920)在同一个ns，bash(4097921)───unshare(4098083)在同一个ns，bash(4098084)在一个ns

root@ubuntu-server:~# ls -l -h  /proc/4048028/ns/pid
lrwxrwxrwx 1 root root 0 Jan 16 11:17 /proc/4048028/ns/pid -> 'pid:[4026531836]'
root@ubuntu-server:~# ls -l -h  /proc/4097920/ns/pid
lrwxrwxrwx 1 root root 0 Jan 16 18:31 /proc/4097920/ns/pid -> 'pid:[4026531836]'
root@ubuntu-server:~# ls -l -h  /proc/4097921/ns/pid
lrwxrwxrwx 1 root root 0 Jan 16 18:31 /proc/4097921/ns/pid -> 'pid:[4026532386]'
root@ubuntu-server:~# ls -l -h  /proc/4098083/ns/pid
lrwxrwxrwx 1 root root 0 Jan 16 18:31 /proc/4098083/ns/pid -> 'pid:[4026532386]'
root@ubuntu-server:~# ls -l -h  /proc/4098084/ns/pid
lrwxrwxrwx 1 root root 0 Jan 16 18:31 /proc/4098084/ns/pid -> 'pid:[4026532391]'
```

> pid namespace和procfs

/proc目录是内核对外暴露的可供用户查看或修改的内核中所记录的信息，包括内核自身的部分信息以及每个进程的信息。
比如对于pid=N的进程来说，它的信息保存在/proc/目录下。在操作系统启动的过程中，会挂载procfs到/proc目录，它存在于root namespace中。
但是，创建新的pid namespace时不会自动重新挂载procfs，而是直接拷贝父级namespace的挂载点信息。
这使得在新的pid namespace中仍然保留了父级namespace的/proc目录，也就是在新创建的这个pid namespace中仍然保留了父级的进程信息。

```shell
#不挂载 --mount-proc此时namespace还是继承宿主机的进程树
root@ubuntu-server:~# unshare -p -u -f /bin/bash
root@ubuntu-server:~# hostname namepsace1
root@ubuntu-server:~# exec bash
root@namepsace1:~# pstree
systemd─┬─ModemManager───2*[{ModemManager}]
        ├─agetty
        ├─cron
        ├─dbus-daemon
        ├─dnsmasq───dnsmasq
        ├─irqbalance───{irqbalance}
        ......
```

之所以有上述问题，其原因是在pid namespace中保留了root namespace中的/proc目录，而不是属于pid namespace自己的/proc。
但用户创建pid namespace时希望的是有完全独立的进程运行环境。
这时，需要在pid namespace中重新挂载procfs，或者在创建pid namespace时指定–mount-proc选项。

```shell
# 注意这儿添加了mount-proc参数
root@ubuntu-server:~# unshare -p -u -f --mount-proc /bin/bash
root@ubuntu-server:~# hostname namespace1
root@ubuntu-server:~# exec bash

# 进程树中展示了bash 为init进程
[root@namespace1 ~]# pstree -p
bash(1)───pstree(24)
```

> pid namespace信号量

pid=1的进程是每一个pid namespace的核心进程(init进程)，它不仅负责收养其所在pid namespace中的孤儿进程，还影响整个pid namespace。

当pid namespace中pid=1的进程退出或终止，内核默认会发送SIGKILL信号给该pid namespace中的所有进程以便杀掉它们(如果该pid namespace中有子孙namespace，也会直接被杀)。

在创建pid namespace时可以通过–kill-child选项指定pid=1的进程终止后内核要发送给pid namespace中进程的信号，其默认信号便是SIGKILL。

```shell
root@ubuntu-server:~# lsns
4026532379 pid         1 4099754 root             bash

root@ubuntu-server:~# kill -9 4099754
#在namespace1的进程，kill掉这个进程后这个namespace1整个namespace都被干掉了
root@namepsace1:~# Killed

#也可以指定为其他信号
unshare -p -f -m -u --mount-proc --kill-child=SIGHUP /bin/bash
```

我们在这里通过一些简单的例子来熟悉了一下pid namespace是如何工作的，当以后我们在玩k8s和docker的时候看到容器中的进程
是如何与宿主机隔离的就不会感到疑惑了。