## cgroup之内存

- /sys/fs/cgroup/memory
```shell
root@ubuntu-server:# cd /sys/fs/cgroup/memory
root@ubuntu-server:# ls -l -h /sys/fs/cgroup/memory
-rw-r--r--   1 root root 0 Aug  1  2022 cgroup.clone_children
--w--w--w-   1 root root 0 Aug  2  2022 cgroup.event_control
-rw-r--r--   1 root root 0 Aug  2  2022 cgroup.procs
-r--r--r--   1 root root 0 Aug  2  2022 cgroup.sane_behavior
-rw-r--r--   1 root root 0 Aug  1  2022 memory.failcnt
--w-------   1 root root 0 Aug  2  2022 memory.force_empty
-rw-r--r--   1 root root 0 Aug  1  2022 memory.kmem.failcnt
-rw-r--r--   1 root root 0 Aug  1  2022 memory.kmem.limit_in_bytes
-rw-r--r--   1 root root 0 Aug  1  2022 memory.kmem.max_usage_in_bytes
-r--r--r--   1 root root 0 Aug  2  2022 memory.kmem.slabinfo
-rw-r--r--   1 root root 0 Aug  1  2022 memory.kmem.tcp.failcnt
-rw-r--r--   1 root root 0 Aug  1  2022 memory.kmem.tcp.limit_in_bytes
-rw-r--r--   1 root root 0 Aug  1  2022 memory.kmem.tcp.max_usage_in_bytes
-r--r--r--   1 root root 0 Aug  1  2022 memory.kmem.tcp.usage_in_bytes
-r--r--r--   1 root root 0 Aug  1  2022 memory.kmem.usage_in_bytes
-rw-r--r--   1 root root 0 Aug  1  2022 memory.limit_in_bytes
-rw-r--r--   1 root root 0 Aug  1  2022 memory.max_usage_in_bytes
-r--r--r--   1 root root 0 Aug  2  2022 memory.meminfo
-rw-r--r--   1 root root 0 Aug  1  2022 memory.memsw.failcnt
-rw-r--r--   1 root root 0 Aug  1  2022 memory.memsw.limit_in_bytes
-rw-r--r--   1 root root 0 Aug  1  2022 memory.memsw.max_usage_in_bytes
-r--r--r--   1 root root 0 Aug  1  2022 memory.memsw.usage_in_bytes
-rw-r--r--   1 root root 0 Aug  2  2022 memory.move_charge_at_immigrate
-r--r--r--   1 root root 0 Aug  2  2022 memory.numa_stat
-rw-r--r--   1 root root 0 Aug  2  2022 memory.oom_control
----------   1 root root 0 Aug  2  2022 memory.pressure_level
-rw-r--r--   1 root root 0 Aug  1  2022 memory.soft_limit_in_bytes
-r--r--r--   1 root root 0 Aug  1  2022 memory.stat
-rw-r--r--   1 root root 0 Aug  2  2022 memory.swappiness
-r--r--r--   1 root root 0 Aug  1  2022 memory.usage_in_bytes
-rw-r--r--   1 root root 0 Aug  1  2022 memory.use_hierarchy
-r--r--r--   1 root root 0 Aug  2  2022 memory.vmstat
-rw-r--r--   1 root root 0 Aug  2  2022 notify_on_release
-rw-r--r--   1 root root 0 Aug  2  2022 release_agent
drwxr-xr-x 211 root root 0 Feb  4 15:58 system.slice
-rw-r--r--   1 root root 0 Aug  2  2022 tasks
drwxr-xr-x   2 root root 0 Aug  1  2022 user.slice
```
```shell
root@ubuntu-server:# mkdir /sys/fs/cgroup/memory/memorytest
root@ubuntu-server:# ls -l -h /sys/fs/cgroup/memory/memorytest
-rw-r--r-- 1 root root 0 Feb  4 16:05 cgroup.clone_children
--w--w--w- 1 root root 0 Feb  4 16:05 cgroup.event_control
-rw-r--r-- 1 root root 0 Feb  4 16:05 cgroup.procs
-rw-r--r-- 1 root root 0 Feb  4 16:05 memory.failcnt
--w------- 1 root root 0 Feb  4 16:05 memory.force_empty
-rw-r--r-- 1 root root 0 Feb  4 16:05 memory.kmem.failcnt
-rw-r--r-- 1 root root 0 Feb  4 16:05 memory.kmem.limit_in_bytes
-rw-r--r-- 1 root root 0 Feb  4 16:05 memory.kmem.max_usage_in_bytes
-r--r--r-- 1 root root 0 Feb  4 16:05 memory.kmem.slabinfo
-rw-r--r-- 1 root root 0 Feb  4 16:05 memory.kmem.tcp.failcnt
-rw-r--r-- 1 root root 0 Feb  4 16:05 memory.kmem.tcp.limit_in_bytes
-rw-r--r-- 1 root root 0 Feb  4 16:05 memory.kmem.tcp.max_usage_in_bytes
-r--r--r-- 1 root root 0 Feb  4 16:05 memory.kmem.tcp.usage_in_bytes
-r--r--r-- 1 root root 0 Feb  4 16:05 memory.kmem.usage_in_bytes
-rw-r--r-- 1 root root 0 Feb  4 16:05 memory.limit_in_bytes
-rw-r--r-- 1 root root 0 Feb  4 16:05 memory.max_usage_in_bytes
-r--r--r-- 1 root root 0 Feb  4 16:05 memory.meminfo
-rw-r--r-- 1 root root 0 Feb  4 16:05 memory.memsw.failcnt
-rw-r--r-- 1 root root 0 Feb  4 16:05 memory.memsw.limit_in_bytes
-rw-r--r-- 1 root root 0 Feb  4 16:05 memory.memsw.max_usage_in_bytes
-r--r--r-- 1 root root 0 Feb  4 16:05 memory.memsw.usage_in_bytes
-rw-r--r-- 1 root root 0 Feb  4 16:05 memory.move_charge_at_immigrate
-r--r--r-- 1 root root 0 Feb  4 16:05 memory.numa_stat
-rw-r--r-- 1 root root 0 Feb  4 16:05 memory.oom_control
---------- 1 root root 0 Feb  4 16:05 memory.pressure_level
-rw-r--r-- 1 root root 0 Feb  4 16:05 memory.soft_limit_in_bytes
-r--r--r-- 1 root root 0 Feb  4 16:05 memory.stat
-r--r--r-- 1 root root 0 Feb  4 16:05 memory.stats_isolated
-rw-r--r-- 1 root root 0 Feb  4 16:05 memory.swappiness
-r--r--r-- 1 root root 0 Feb  4 16:05 memory.usage_in_bytes
-rw-r--r-- 1 root root 0 Feb  4 16:05 memory.use_hierarchy
-r--r--r-- 1 root root 0 Feb  4 16:05 memory.vmstat
-rw-r--r-- 1 root root 0 Feb  4 16:05 notify_on_release
-rw-r--r-- 1 root root 0 Feb  4 16:05 tasks
```

> 查看当前内存限制,默认是无限制,注意内存这里不是-1，无限制是给了一个很大的值
```shell
root@ubuntu-server:# cat memory.limit_in_bytes
9223372036854771712
```

> 使用stress模拟内存使用
```shell
root@ubuntu-server:# stress --vm 1 --vm-bytes 256M &
[1] 16387

root@ubuntu-server:# ps -ef|grep stress|grep -v grep
root     16390 16387 99 16:08 pts/0    00:03:58 stress --vm 8 --vm-bytes 256M
```

> 将这个进程id加入到cgroup下面，并将cgroup限制改成200M
```shell
root@ubuntu-server:# echo 16390 > /sys/fs/cgroup/memory/memorytest/tasks
root@ubuntu-server:# echo 209637376 > /sys/fs/cgroup/memory/memorytest/memory.limit_in_bytes
```

> 查看日志被oom kill了
```shell
#运行stress的终端显示
#stress: FAIL: [16387] (415) <-- worker 16390 got signal 9
stress: WARN: [16387] (417) now reaping child worker processes
stress: FAIL: [16387] (451) failed run completed in 252s

#查看内核日志:
Feb  4 16:13:01 dc07-daily-k8s2-node-BJ01host-732165 kernel: [ pid ]   uid  tgid total_vm      rss nr_ptes nr_pmds swapents oom_score_adj name
Feb  4 16:13:01 dc07-daily-k8s2-node-BJ01host-732165 kernel: [16387]     0 16387     1831      219       8       3        0             0 stress
Feb  4 16:13:01 dc07-daily-k8s2-node-BJ01host-732165 kernel: [16390]     0 16390    67368    51165     109       3        0             0 stress
Feb  4 16:13:01 dc07-daily-k8s2-node-BJ01host-732165 kernel: Memory cgroup out of memory: Kill process 16390 (stress) score 971 or sacrifice child
Feb  4 16:13:01 dc07-daily-k8s2-node-BJ01host-732165 kernel: Killed process 16390 (stress) total-vm:269472kB, anon-rss:204168kB, file-rss:492kB, shmem-rss:0kB
Feb  4 16:13:01 dc07-daily-k8s2-node-BJ01host-732165 kernel: oom_reaper: reaped process 16390 (stress), now anon-rss:0kB, file-rss:0kB, shmem-rss:0kB
```

> 与cpu类似，内存中哪个指标体现内存不足呢
```shell
root@ubuntu-server:# cat memory.failcnt
59390293

#这个指标的含义是cgroup中的任务是否在使用超过其限制的资源，如果这个数量上升很快，说明内存有瓶颈了。
#另外容器中的内存cache也是算到使用的一部分，如果容器中memory_usage+cache超过分配的内存，也是可能会存在被oom的风险。
#所以我们时刻关注整体的内存使用情况！！！
```

- /sys/fs/cgroup/hugetlb

这个用的比较少，主要是用来限制大页内存的使用，至于大页内存的作用可以参考
