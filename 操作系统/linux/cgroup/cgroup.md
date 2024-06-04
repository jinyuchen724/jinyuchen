## cgroup资源限制

我们来研究一下linux的“限制”问题。我们还是拿一个进程来举例子。我们在不同的pid namesapce中第一个进程的pid就是1，但是宿主机上，2个pid namesapce中的进程与其他所有进程之间依然是平等的竞争关系。
这就意味着，虽然pid namesapce中的进程表面上被隔离了起来，但是它所能够使用到的资源（比如 CPU、内存），却是可以随时被宿主机上的其他进程（或者其他容器）占用的。
当然，这个pid namespace中的进程自己也可能把所有资源吃光。这些情况和我们理解的“盒子”的作用还是有区别的。

而在linux中cgroup就是用来进行资源限制的一个重要功能。

Linux Cgroups 的全称是 Linux Control Group。它最主要的作用，就是限制一个进程组能够使用的资源上限，包括 CPU、内存、磁盘、网络带宽等等。

目前已经有了cgroup2，我们这里还是使用的cgroup1(关闭了cgroup2)来进行验证。环境是ubuntu22.04,5.15的内核：
```shell
ubuntu@ubuntu-server# mount|grep cgroup
mount -t cgroup
cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,cpu,cpuacct)
cgroup on /sys/fs/cgroup/net_cls type cgroup (rw,nosuid,nodev,noexec,relatime,net_cls)
cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,cpuset)
cgroup on /sys/fs/cgroup/pids type cgroup (rw,nosuid,nodev,noexec,relatime,pids)
cgroup on /sys/fs/cgroup/blkio type cgroup (rw,nosuid,nodev,noexec,relatime,blkio)
cgroup on /sys/fs/cgroup/hugetlb type cgroup (rw,nosuid,nodev,noexec,relatime,hugetlb)
```

linux支持的cgroup种类比较多，归纳一下其实也还好，有些基本也很少用到，下面我们具体来分析一下

| 名称                         | 隔离资源                                                 |
|:---------------------------| :--------------------------------------------------- |
| /sys/fs/cgroup/memory      | 这个 Cgroup 用于限制和控制内存资源的使用情况，可以设置内存限制、内存交换限制等。         |
| /sys/fs/cgroup/cpu,cpuacct | 这个 Cgroup 用于限制和控制 CPU 资源的使用情况，可以设置 CPU 时间片、CPU 份额等。  |
| /sys/fs/cgroup/net_cls     | 这个 Cgroup 用于限制和控制网络资源的使用情况，可以设置网络带宽、优先级等。            |
| /sys/fs/cgroup/cpuset      | 这个 Cgroup 用于限制和控制 CPU 核心的使用情况，可以将进程绑定到特定的 CPU 核心上。   |
| /sys/fs/cgroup/pids        | 这个 Cgroup 用于限制和控制进程数量的使用情况。                          |
| /sys/fs/cgroup/blkio       | 这个 Cgroup 用于限制和控制块设备（例如硬盘、SSD）的使用情况，可以设置块设备的 I/O 限制。 |
| /sys/fs/cgroup/hugetlb     | 这个 Cgroup 用于限制和控制大页内存资源的使用情况。          