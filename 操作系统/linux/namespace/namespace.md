## 一、准备
为了对容器有一个更清晰、本质的了解我们需要对容器底层用到的一些技术有一个整体的认知，做到知其然知其所以然。
在聊容器之前,我们先回顾一下linux,linux的作用是管理硬件的资源分配，对于使用操作系统的人来说，我们就是为了让其运行我们的程序，这样就可以理解成我们将想运行的程序交给操作系统，操作系统来给我们的程序分配对应的资源。

### 1.1 进程

那么我们的程序本身就是一个安安静静待在磁盘上的一个可执行文件，并不能达到我们想要的目地，只有程序运行起来了，才有价值。那么这个程序是怎么运行的呢？

*   程序加载：内核将可执行文件从磁盘加载到内存中。
*   进程创建：内核为新进程分配一个唯一的进程 ID（PID），并为其创建一个进程描述符（process descriptor）数据结构，该数据结构包含了进程的所有信息，如进程状态、进程优先级、进程运行时间等等。
*   虚拟地址空间分配：内核为新进程分配一个虚拟地址空间，该地址空间包含了进程所需的代码、数据和堆栈等区域。
*   资源分配：内核为新进程分配所需的资源，如文件描述符、信号处理器、定时器等等。
*   初始化：内核初始化进程的状态和环境，如设置进程的初始状态、清空进程的内存空间等等。
*   执行进程：内核将 CPU 的控制权交给新进程，使其开始执行。

那进程和容器之间的关系是什么呢，其实我们可以这么理解：容器就是一个“盒子”，然后把进程塞进“盒子”里面，那每个“盒子”之间是隔离的，
大家都有自己的隐私。
- namespace: linux就是通过“namespace”进行隔离，来保护各自的隐私；
- cgroup: 那这个盒子里的进程如果无限制的使用资源，是不是会影响其他“盒子”中的进程？
一个操作系统资源就这么多，所以我们是不是还要对“盒子”里的进程占用的资源进行限制？
Linux就是通过“cgroup”来进行资源限制的。到了这里这个“盒子”是不是已经模子都形成了？
- rootfs: 但是进程运行中的数据怎么隔离呢？既然这个盒子都隔离了，那“盒子”之间不能看到的文件目录都一样吧？
所以是不是不同的"盒子"之间都有自己的独立的根目录，大家互不影响。Linux就通过"rootfs"这个东西可以让不同的"盒子"拥有自己的根目录。

### 1.2 namespace资源隔离

Linux namespace 是一种内核级别的资源隔离机制，用来让运行在同一个操作系统上的进程互相不会干扰。

**namespace 目的就是隔离**，要做到的效果是：如果某个 namespace 中有进程在里面运行，它们只能看到该 namespace 的信息，无法看到 namespace 以外的东西。我们来想一下：一个进程在运行的时候，它会知道哪些信息？

*   看到系统的 hostname
*   可用的网络资源（bridge、interface、网络端口的时候情况……）
*   进程的关系（有哪些进程，进程之间的父子关系等）
*   系统的用户信息（有哪些用户、组，它们的权限是怎么样的）
*   文件系统（有哪些可用的文件系统，使用情况）
*   IPC（怎么实现进程间通信）
*   ……

| 名称   | 宏定义            | 隔离资源                                                                                                                                                  | 内核版本   |
| :--- | :------------- | :---------------------------------------------------------------------------------------------------------------------------------------------------- | :----- |
| mnt  | CLONE\_NEWNS   | Mount point                                                                                                                                           | 2.4.19 |
| ipc  | CLONE\_NEWIPC  | System V IPC, POSIX message queue                                                                                                                     | 2.6.19 |
| net  | CLONE\_NEWNET  | network device interface, IPv4 and IPv6 protocol stack, IP routing table, firewall rule, the /proc/net and /sys/class/net directory tree, socket, etc | 2.6.24 |
| pid  | CLONE\_NEWPID  | Process ID                                                                                                                                            | 2.6.24 |
| user | CLONE\_NEWUSER | User and group ID                                                                                                                                     | 3.8    |
| UTS  | CLONE\_NEWUTS  | Hostname and NIS domain name                                                                                                                          | 2.6.19 |

在[namespace man](https://man7.org/linux/man-pages/man7/namespaces.7.html)上提供的系统调用包括clone, setns, unshare

- clone: 创建一个新的进程并把他放到新的namespace中
```
int clone(int (*child_func)(void *), void *child_stack, int flags, void *arg);
```

- setns：将当前进程加入到已有的namespace中, (ip netns命令调用的就是setns系统调用)
```
int setns(int fd, int nstype);
```

- unshare: 让进程离开当前的 namespace，加入到新建的 namespace 中。（unshare命令）
```
int unshare(int flags);
```

**clone和unshare的区别**

- unshare是使当前进程加入新的namespace
- clone是创建一个新的进程，然后让进程加入新的namespace

看到这里大家会云里雾里，这些东西到底是啥玩意。我们动手实践玩一玩，(这里需要一个linux的机器,文中使用的是ubuntu22.04,5.15的内核)，就能有更深刻的认识。