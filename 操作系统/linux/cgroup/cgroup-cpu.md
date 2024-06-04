## cgroup之cpu

- /sys/fs/cgroup/cpu,cpuacct

我们先来看一下linux中这个目录下面都有哪些东西:
```shell
root@ubuntu-server:~# ls -l -h /sys/fs/cgroup/cpu,cpuacct/
total 0
-rw-r--r--   1 root root 0 Aug  1  2022 cgroup.clone_children
-rw-r--r--   1 root root 0 Aug  1  2022 cgroup.procs
-r--r--r--   1 root root 0 Aug  1  2022 cgroup.sane_behavior
-r--r--r--   1 root root 0 Aug  1  2022 cpuacct.stat
-r--r--r--   1 root root 0 Aug  1  2022 cpuacct.uptime
-rw-r--r--   1 root root 0 Aug  1  2022 cpuacct.usage
-r--r--r--   1 root root 0 Aug  1  2022 cpuacct.usage_all
-r--r--r--   1 root root 0 Aug  1  2022 cpuacct.usage_percpu
-r--r--r--   1 root root 0 Aug  1  2022 cpuacct.usage_percpu_sys
-r--r--r--   1 root root 0 Aug  1  2022 cpuacct.usage_percpu_user
-r--r--r--   1 root root 0 Aug  1  2022 cpuacct.usage_sys
-r--r--r--   1 root root 0 Aug  1  2022 cpuacct.usage_user
-rw-r--r--   1 root root 0 Aug  1  2022 cpu.cfs_period_us
-rw-r--r--   1 root root 0 Aug  1  2022 cpu.cfs_quota_us
-rw-r--r--   1 root root 0 Aug  1  2022 cpu.rt_period_us
-rw-r--r--   1 root root 0 Aug  1  2022 cpu.rt_runtime_us
-rw-r--r--   1 root root 0 Aug  1  2022 cpu.shares
-r--r--r--   1 root root 0 Aug  1  2022 cpu.stat
-rw-r--r--   1 root root 0 Aug  1  2022 notify_on_release
-rw-r--r--   1 root root 0 Aug  1  2022 release_agent
drwxr-xr-x 211 root root 0 Jan 30 17:22 system.slice
-rw-r--r--   1 root root 0 Aug  1  2022 tasks
drwxr-xr-x   2 root root 0 Aug  1  2022 user.slice
```
```shell
#接着我们就在这个cgroup下创建一个目录
root@ubuntu-server:~# mkdir cputest
root@ubuntu-server:~# ls -h -l cputest
-rw-r--r-- 1 root root 0 Jan 30 17:29 cgroup.clone_children
-rw-r--r-- 1 root root 0 Jan 30 17:29 cgroup.procs
-r--r--r-- 1 root root 0 Jan 30 17:29 cpuacct.stat
-r--r--r-- 1 root root 0 Jan 30 17:29 cpuacct.stats_isolated
-r--r--r-- 1 root root 0 Jan 30 17:29 cpuacct.uptime
-rw-r--r-- 1 root root 0 Jan 30 17:29 cpuacct.usage
-r--r--r-- 1 root root 0 Jan 30 17:29 cpuacct.usage_all
-r--r--r-- 1 root root 0 Jan 30 17:29 cpuacct.usage_percpu
-r--r--r-- 1 root root 0 Jan 30 17:29 cpuacct.usage_percpu_sys
-r--r--r-- 1 root root 0 Jan 30 17:29 cpuacct.usage_percpu_user
-r--r--r-- 1 root root 0 Jan 30 17:29 cpuacct.usage_sys
-r--r--r-- 1 root root 0 Jan 30 17:29 cpuacct.usage_user
-rw-r--r-- 1 root root 0 Jan 30 17:29 cpu.cfs_period_us
-rw-r--r-- 1 root root 0 Jan 30 17:29 cpu.cfs_quota_us
-r--r--r-- 1 root root 0 Jan 30 17:29 cpu.quota_aware
-rw-r--r-- 1 root root 0 Jan 30 17:29 cpu.rt_period_us
-rw-r--r-- 1 root root 0 Jan 30 17:29 cpu.rt_runtime_us
-rw-r--r-- 1 root root 0 Jan 30 17:29 cpu.shares
-r--r--r-- 1 root root 0 Jan 30 17:29 cpu.stat
-rw-r--r-- 1 root root 0 Jan 30 17:29 notify_on_release
-rw-r--r-- 1 root root 0 Jan 30 17:29 tasks
```

我们可以看到在cpu这个cgroup下面创建一个目录,linux就自动帮我们创建了对应限制资源的文件

这么多文件，到底是干嘛的？这么快速理解呢？
```shell
cpu.cfs_quota_us          #用来设置在一个CFS调度时间周期(cfs_period_us)内，允许此控制组执行的时间。默认值为-1表示不限制时间。
cpu.cfs_period_us         #用来设置一个CFS调度时间周期长度，默认值是100000us(100ms)，一般cpu.cfs_period_us作为系统默认值我们不会去修改它。 
cpu.shares                #用来设置cpu cgroup子系统对于控制组之间的cpu分配比例
```

我们继续通过实验，来进行更深的了解:
```shell
#我们在后台执行一个死循环
root@ubuntu-server:~# while true;do : ;done &
[1] 25692

#我们可以看到这个死循环占了1核心的资源(机器是多核心的,100%就是1核心)
root@ubuntu-server:~#top -p 25692
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
25692 root      20   0  115104   2676    832 R 100.0  0.0   0:56.70 bash

root@ubuntu-server:#cat cpu.cfs_quota_us
-1
root@ubuntu-server:#cat cpu.cfs_period_us
100000

#这里可能有疑问了，-1不是不限制吗，为什么这个进程只用了1核心呢？这个shell是个单线程的所以最多只能跑满1核心，如果有兴趣，
# 可以使用sysbench去跑多核心，或者多跑几个这种死循环都可以实现占用多个cpu的场景。
```

> 我们现在将这个进程分配0.5核心
```shell
root@ubuntu-server:# echo 50000 > cpu.cfs_quota_us
```

> 在将之前的那个task id 25962加入到这个控制组的task文件中
```shell
root@ubuntu-server:# echo 25692 > tasks
```

> 现在文件结果如下
```shell
root@ubuntu-server:#cat cpu.cfs_quota_us
50000
root@ubuntu-server:#cat cpu.cfs_period_us
100000
root@ubuntu-server:#cat tasks
25692
```

> 在看看cpu
```shell
root@ubuntu-server:~#top -p 25692
PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
25692 root      20   0  115104   2676    832 R  50.2  0.0  21:04.41 bash
```

现在限制的效果和我们预期完全一样，那么我们再去理解一下这2个参数的含义

cpu.cfs_period_us代表的是一个内核计算cpu时间的一个周期，单位是ns,默认100000就是100毫秒。

cpu.cfs_quota_us代表的是在cfs_period_us周期内，可以使用的cpu的时间是多少，单位也是ns。

那么连起来意思就是这个cgroup下面在100ms的周期内，可以使用50ms的cpu时间，那就是0.5核心啦～

同理我们是不是也可以通过调整cfs_period_us的时间来控制使用的cpu大小呢？

```shell
#cpu.shares
对于cpu.shares分配比例的使用，例如有两个cpu控制组foo和bar，foo的cpu.shares是1024，bar的cpu.shares是3072，它们的比例就是1:3。 
在一台8个CPU的主机上，如果foo和bar设置的都是需要4个CPU的话(cfs_quota_us/cfs_period_us=4)，根据它们的CPU分配比例，控制组foo得到的是2个，而bar控制组得到的是6个。
需要注意cpu.shares是在多个cpu控制组之间的分配比例，且只有到整个主机的所有CPU都打满时才会起作用。 

这里需要注意一下，这里是必须cpu打满才会生效，但是我们尽量不要去使用这个，因为我们节点一定要做资源预留不要让他打满，如果cpu打满很可能就已经出现问题了。
```

- /sys/fs/cgroup/cpuset

这个cgroup的作用是可以让进程绑定到某些cpu上去执行，为什么要让进程绑定到cpu上呢？
简单的说就是现在的服务器都是大核心96C，128C，256C，过多的cpu过造成cache抖动的问题，而进程绑定到同一个L3中的cpu这样可以让性能更稳定，
有兴趣的可以看下我的这篇[文章](https://jinyuchen724.gitbook.io/docs/cao-zuo-xi-tong/linux/cpu/cpu-zong-jie)中关于cpu的分析。

那我们这里还是继续通过例子来熟悉：

> 查看cpuset cgroup下面的文件内容及创建目录
```shell
root@ubuntu-server:#ls -l -h /sys/fs/cgroup/cpuset
total 0
-rw-r--r-- 1 root root 0 Aug  1  2022 cgroup.clone_children
-rw-r--r-- 1 root root 0 Aug  1  2022 cgroup.procs
-r--r--r-- 1 root root 0 Aug  1  2022 cgroup.sane_behavior
-rw-r--r-- 1 root root 0 Aug  1  2022 cpuset.cpu_exclusive
-r--r--r-- 1 root root 0 Aug  1  2022 cpuset.cpuinfo
-rw-r--r-- 1 root root 0 Aug  1  2022 cpuset.cpus
-r--r--r-- 1 root root 0 Aug  1  2022 cpuset.effective_cpus
-r--r--r-- 1 root root 0 Aug  1  2022 cpuset.effective_mems
-r--r--r-- 1 root root 0 Aug  1  2022 cpuset.loadavg
-rw-r--r-- 1 root root 0 Aug  1  2022 cpuset.mem_exclusive
-rw-r--r-- 1 root root 0 Aug  1  2022 cpuset.mem_hardwall
-rw-r--r-- 1 root root 0 Aug  1  2022 cpuset.memory_migrate
-r--r--r-- 1 root root 0 Aug  1  2022 cpuset.memory_pressure
-rw-r--r-- 1 root root 0 Aug  1  2022 cpuset.memory_pressure_enabled
-rw-r--r-- 1 root root 0 Aug  1  2022 cpuset.memory_spread_page
-rw-r--r-- 1 root root 0 Aug  1  2022 cpuset.memory_spread_slab
-rw-r--r-- 1 root root 0 Aug  1  2022 cpuset.mems
-rw-r--r-- 1 root root 0 Aug  1  2022 cpuset.sched_load_balance
-rw-r--r-- 1 root root 0 Aug  1  2022 cpuset.sched_relax_domain_level
-r--r--r-- 1 root root 0 Aug  1  2022 cpuset.stat
-rw-r--r-- 1 root root 0 Aug  1  2022 notify_on_release
-rw-r--r-- 1 root root 0 Aug  1  2022 release_agent
drwxr-xr-x 2 root root 0 Nov 30 14:32 system.slice
-rw-r--r-- 1 root root 0 Aug  1  2022 tasks

#在cpuset下默认将所有cpu都加入进来了（机器是24核心的）
root@ubuntu-server:#cat cpuset.cpus
0-23

root@ubuntu-server:#mkdir cpusettest
root@ubuntu-server:#ls -l -h
-rw-r--r-- 1 root root 0 Jan 30 18:38 cgroup.clone_children
-rw-r--r-- 1 root root 0 Jan 30 18:38 cgroup.procs
-rw-r--r-- 1 root root 0 Jan 30 18:38 cpuset.cpu_exclusive
-r--r--r-- 1 root root 0 Jan 30 18:38 cpuset.cpuinfo
-rw-r--r-- 1 root root 0 Jan 30 18:38 cpuset.cpus
-r--r--r-- 1 root root 0 Jan 30 18:38 cpuset.effective_cpus
-r--r--r-- 1 root root 0 Jan 30 18:38 cpuset.effective_mems
-r--r--r-- 1 root root 0 Jan 30 18:38 cpuset.loadavg
-rw-r--r-- 1 root root 0 Jan 30 18:38 cpuset.mem_exclusive
-rw-r--r-- 1 root root 0 Jan 30 18:38 cpuset.mem_hardwall
-rw-r--r-- 1 root root 0 Jan 30 18:38 cpuset.memory_migrate
-r--r--r-- 1 root root 0 Jan 30 18:38 cpuset.memory_pressure
-rw-r--r-- 1 root root 0 Jan 30 18:38 cpuset.memory_spread_page
-rw-r--r-- 1 root root 0 Jan 30 18:38 cpuset.memory_spread_slab
-rw-r--r-- 1 root root 0 Jan 30 18:38 cpuset.mems
-rw-r--r-- 1 root root 0 Jan 30 18:38 cpuset.sched_load_balance
-rw-r--r-- 1 root root 0 Jan 30 18:38 cpuset.sched_relax_domain_level
-r--r--r-- 1 root root 0 Jan 30 18:38 cpuset.stat
-r--r--r-- 1 root root 0 Jan 30 18:38 cpuset.stats_isolated
-rw-r--r-- 1 root root 0 Jan 30 18:38 notify_on_release
-rw-r--r-- 1 root root 0 Jan 30 18:38 tasks

#我们创建的cpusettest目录下面是空的
root@ubuntu-server:#cat cpuset.cpus
0-23
```

> 接下来我们将需要绑定的cpu加入到cpuset
```shell
root@ubuntu-server:#echo 8 > cpuset.cpus
#没开启numa设置成0即可
root@ubuntu-server:#echo 0 > cpuset.mems 
```

> 我们还是重新运行一下上面的while循环的例子
```shell
root@ubuntu-server:#while true;do : ;done &
[1] 9420
root@ubuntu-server:#echo 9420 > tasks
```

> 可以看到现在cpu在8上
```shell
top - 18:47:07 up 547 days, 19 min,  1 user,  load average: 4.47, 4.22, 4.12
Tasks:   1 total,   1 running,   0 sleeping,   0 stopped,   0 zombie
%Cpu0  :  1.7 us,  0.7 sy,  0.0 ni, 97.7 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
...
%Cpu8  :100.0 us,  0.0 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
...
%Cpu23 :  2.7 us,  1.3 sy,  0.3 ni, 95.7 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem : 98507360 total,  9036152 free, 71255192 used, 18216020 buff/cache
KiB Swap:        0 total,        0 free,        0 used. 29094680 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 9420 root      20   0  115104   2444    576 R 100.0  0.0   4:27.92 bash
```

> 我们在修改一下,将需要绑定的cpu改成6
```shell
root@ubuntu-server:#echo 6 > cpuset.cpus
root@ubuntu-server:#cat cpuset.cpus
6
top - 18:48:08 up 547 days, 20 min,  1 user,  load average: 3.69, 4.06, 4.07
Tasks:   1 total,   1 running,   0 sleeping,   0 stopped,   0 zombie
%Cpu0  :  2.4 us,  1.7 sy,  0.0 ni, 95.9 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
...
%Cpu6  :100.0 us,  0.0 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
...
%Cpu23 :  5.0 us,  2.7 sy,  1.0 ni, 91.3 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem : 98507360 total,  9526108 free, 71058384 used, 17922872 buff/cache
KiB Swap:        0 total,        0 free,        0 used. 29292136 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 9420 root      20   0  115104   2444    576 R 100.0  0.0   5:29.09 bash

#切到cpu6上了，说明我们绑核生效了。
#当然我们也可以将多个cpu绑定到这个cgroup下面，可以自己尝试一下
```

> 关于cgroup重要的指标，哪个指标可以提现cpu到底够不够用？我们回忆一下cgroup的作用是什么，限制资源，为什么会限制资源，说明资源用超了，那么哪个指标可以提现cpu用超了呢？
```shell
root@ubuntu-server:#cat cpu.stat
...
nr_throttled 5982                  #这是一个累计值，代表被限流的总次数
throttled_time 299016546065        #这是一个累计值，代表被限流的总时间

#通过这2个指标我们就可以清晰的算出来在一段时间内被限流了多少次、被限流了多长时间
#另外这里提一下在容器中如果cpu使用率非常高，但是没有被限流，说明资源利用的比较高，不一定代表有问题。
#但如果cpu使用率不高，但是被限流的次数、时间都很高，那说明在一个cpu周期(cfs_period_us),cpu是不够用的，就需要看看是哪里出了问题！
```



