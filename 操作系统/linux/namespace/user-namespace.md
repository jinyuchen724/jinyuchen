## user namespace

User namespace用来隔离user权限相关的Linux资源，包括[user IDs and group IDs](https://link.segmentfault.com/?enc=NDSuJAP3OeXuVDCkJofo9A%3D%3D.bBvCLnOyhYEEUI4LX4l8c2%2F5dPKZ%2F9xa%2FJrfilcOmW2WpH%2F0gU5E%2FZ9DY%2FNmbjsCtbnMvAhgygLfeINECT1v1g%3D%3D)，[keys](https://link.segmentfault.com/?enc=ToEqw2nAKSo0vc7EnE%2BQEg%3D%3D.TJqcnxUkK2cuFD58fVEwVXWKyKaoI4Szt4VoOJxfcSi%2B3a9njqm2ZhAlATYYdULrDZ65ywx2eJQq8D3obq7Nmg%3D%3D), 和[capabilities](https://link.segmentfault.com/?enc=jYVMEpd7Z9jJ6vzLC9UKAA%3D%3D.K6m8vTpnsJ%2Fl2d%2Bp%2B%2FqNqrgZPMT3biK5qzkUBAj8CgGQTcT7uGUjMUJEOTIb9BhqU7P9DPDUHS6C%2B%2BAIuUrpWA%3D%3D)。

这是目前实现的namespace中最复杂的一个，因为user和权限息息相关，而权限又事关容器的安全，所以稍有不慎，就会出安全问题。

user namespace可以嵌套（目前内核控制最多32层），除了系统默认的user namespace外，所有的user namespace都有一个父user namespace，每个user namespace都可以有零到多个子user namespace。 
当在一个进程中调用unshare或者clone创建新的user namespace时，当前进程原来所在的user namespace为父user namespace，新的user namespace为子user namespace.

在不同的user namespace中，同样一个用户的user ID 和group ID可以不一样，换句话说，一个用户可以在父user namespace中是普通用户，在子user namespace中是超级用户（超级用户只相对于子user namespace所拥有的资源，无法访问其他user namespace中需要超级用户才能访问资源）。

从Linux 3.8开始，创建新的user namespace不需要root权限。

> 创建user namesapce
```shell
#当前id,gid,usernamespace
ubuntu@ubuntu-server:~$ id
uid=1000(ubuntu) gid=1000(ubuntu) groups=1000(ubuntu)

ubuntu@ubuntu-server:~$ readlink /proc/$$/ns/user
user:[4026531837]

#创建新的user namespace
ubuntu@ubuntu-server: unshare --user /bin/bash
nobody@ubuntu-server: id
uid=65534(nobody) gid=65534(nogroup) groups=65534(nogroup)

可以看到上面例子中显示uid、gid、groups 都是65534，因为我们还没有映射父user namespace的user ID和group ID到子user namespace中来，这一步是必须的，因为这样系统才能控制一个user namespace里的用户在其他user namespace中的权限。（比如给其他user namespace中的进程发送信号，或者访问属于其他user namespace挂载的文件）

如果没有映射的话，当在新的user namespace中用getuid()和getgid()获取user id和group id时，系统将返回文件/proc/sys/kernel/overflowuid中定义的user ID以及proc/sys/kernel/overflowgid中定义的group ID，它们的默认值都是65534。也就是说如果没有指定映射关系的话，会默认映射到ID65534。
```

```shell
nobody@ubuntu-server:~$ cat /proc/sys/kernel/overflowuid
65534
nobody@ubuntu-server:~$ cat /proc/sys/kernel/overflowgid
65534
```

> 下面看看这个user能干些什么

```shell
#ls的结果显示/root目录属于nobody
nobody@ubuntu-server:~$ ls -l / |grep root
drwx------  12 nobody nogroup       4096 Jan 16 19:38 root

#但是当前的nobody账号访问不了，说明这两个nobody不是一个ID，他们之间没有映射关系
nobody@ubuntu-server:~$ ls -l /root
ls: cannot open directory '/root': Permission denied

#这里显示/home/ubuntu目录属于nobody
nobody@ubuntu-server:~$ ls -l /home/
total 4
drwxr-x--- 4 nobody nogroup 4096 Jul  3  2023 ubuntu

#ls /home/ubuntu成功，说明虽然没有显式的映射ID，但还是能访问父user namespace里ubuntu账号拥有的资源
#说明他们背后还是有映射关系
nobody@ubuntu-server:~$ ls -l /home/ubuntu/
total 4728
-rwxrwxrwx 1 nobody nogroup      50 Jul  3  2023 cron.sh
-rw-r--r-- 1 nobody nogroup 4831757 Jan 16 20:51 error.log

```

> 映射user id和group id

通常情况下，创建新的user namespace后，第一件事就是映射user和group ID， 映射ID的方法是添加配置到/proc/PID/uid_map和/proc/PID/gid_map（这里的PID是新user namespace中的进程ID，刚开始时这两个文件都是空的）

这两个文件里面的配置格式如下（可以有多条）：
```shell
ID-inside-ns   ID-outside-ns   length

举个例子, 0 1000 256这条配置就表示父user namespace中的1000~1256映射到新user namespace中的0~256。

# 系统默认的user namespace没有父user namespace，但为了保持一致，kernel提供了一个虚拟的uid和gid map文件，看起来是这样子的:
ubuntu@ubuntu-server:~$ cat /proc/$$/uid_map
         0          0 4294967295
```
 

那么谁可以向这个文件中写配置呢？

/proc/PID/uid_map和/proc/PID/gid_map的拥有者是创建新user namespace的这个user，所以和这个user在一个user namespace的root账号可以写。
但这个user自己有没有写map文件权限还要看它有没有CAP_SETUID和CAP_SETGID的capability。

> **注意**：只能向map文件写一次数据，但可以一次写多条，并且最多只能5条

关于capability的详细介绍可以参考[这里](https://link.segmentfault.com/?enc=nhgAkynVKqtxb5hXojy%2Bbg%3D%3D.UBFIJIGnFEWoKi53BLlzKaHMSs%2FYrb2p1SvCLnJOeywZ%2Fy8kyMjHTCPsACaUJEUxip%2Bu1wy9SWl6iZZLStH6lg%3D%3D)，简单点说，原来的Linux就分root和非root，很多操作只能root完成，比如修改一个文件的owner，后来Linux将root的一些权限分解了，变成了各种capability，只要拥有了相应的capability，就能做相应的操作，不需要root账户的权限。

下面我们来看看如何用ubuntu账号映射uid和gid
```shell
#------第一个终端------
#获取当前bash的pid(这个是上面我门使用unshare --user /bin/bash创建的ns的终端)
nobody@ubuntu-server:~$ echo $$
46215

#------第二个终端------
ubuntu@ubuntu-server:~$ ls -l /proc/46215/uid_map /proc/46215/gid_map
-rw-r--r-- 1 ubuntu ubuntu 0 Jan 17 10:15 /proc/46215/gid_map
-rw-r--r-- 1 ubuntu ubuntu 0 Jan 17 10:15 /proc/46215/uid_map

ubuntu@ubuntu-server:~$ echo '0 1000 100' > /proc/46215/uid_map
-bash: echo: write error: Operation not permitted
ubuntu@ubuntu-server:~$ echo '0 1000 100' > /proc/46215/gid_map
-bash: echo: write error: Operation not permitted

#当前用户运行的bash进程没有CAP_SETUID和CAP_SETGID的权限
ubuntu@ubuntu-server:~$ cat /proc/$$/status |egrep Cap
CapInh:	0000000000000000
CapPrm:	0000000000000000
CapEff:	0000000000000000
CapBnd:	000001ffffffffff
CapAmb:	0000000000000000

#为/bin/bash设置capability，
ubuntu@ubuntu-server:~$ sudo setcap cap_setgid,cap_setuid+ep /bin/bash
[sudo] password for ubuntu:
ubuntu@ubuntu-server:~$ exec bash
ubuntu@ubuntu-server:~$ cat /proc/$$/status |egrep Cap
CapInh:	0000000000000000
CapPrm:	00000000000000c0
CapEff:	00000000000000c0
CapBnd:	000001ffffffffff
CapAmb:	0000000000000000

#再试一次写map文件，成功了
ubuntu@ubuntu-server:~$ echo '0 1000 100' > /proc/46215/uid_map
ubuntu@ubuntu-server:~$ echo '0 1000 100' > /proc/46215/gid_map

#再写一次就失败了，因为这个文件只能写一次
ubuntu@ubuntu-server:~$ echo '0 1000 100' > /proc/46215/gid_map
bash: echo: write error: Operation not permitted

#后续测试不需要CAP_SETUID了，将/bin/bash的capability恢复到原来的设置
ubuntu@ubuntu-server:~$ getcap /bin/bash
/bin/bash cap_setgid,cap_setuid=ep
ubuntu@ubuntu-server:~$ sudo setcap cap_setgid,cap_setuid-ep /bin/bash
[sudo] password for ubuntu:
ubuntu@ubuntu-server:~$ getcap /bin/bash
/bin/bash =
```

```shell
#------第一个终端------
#回到第一个窗口，id已经变成0了，说明映射成功
nobody@ubuntu-server:~$ id
uid=0(root) gid=0(root) groups=0(root),65534(nogroup)

#------第二个终端------
#回到第二个窗口，确认map文件的owner，(这个是上面我门使用unshare --user /bin/bash创建的ns的终端,id是46215)
dev@ubuntu:~$ ls -l /proc/46215/
......
-rw-r--r--  1 ubuntu ubuntu 0 Jan 17 14:57 gid_map
dr-x--x--x  2 ubuntu ubuntu 0 Jan 17 14:57 ns
-rw-r--r--  1 ubuntu ubuntu 0 Jan 17 14:57 uid_map
......

#------第一个终端------
#重新加载bash，提示有root权限了
nobody@ubuntu-server:~$ exec bash
root@ubuntu-server:~#
root@ubuntu-server:~# cat /proc/$$/status |grep Cap
CapInh:	0000000000000000
CapPrm:	000001ffffffffff
CapEff:	000001ffffffffff
CapBnd:	000001ffffffffff
CapAmb:	0000000000000000

#------第二个终端------
ubuntu@ubuntu-server:~$ ls -l -h /proc/46215/
...
-rw-r--r--  1 ubuntu ubuntu 0 Jan 17 14:57 gid_map
dr-x--x--x  2 ubuntu ubuntu 0 Jan 17 14:57 ns
-rw-r--r--  1 ubuntu ubuntu 0 Jan 17 14:57 uid_map
...

ubuntu@ubuntu-server:~$ ls -l -h /proc/46215/ns
total 0
lrwxrwxrwx 1 ubuntu ubuntu 0 Jan 17 15:07 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 ubuntu ubuntu 0 Jan 17 15:07 ipc -> 'ipc:[4026531839]'
lrwxrwxrwx 1 ubuntu ubuntu 0 Jan 17 15:07 mnt -> 'mnt:[4026531841]'
lrwxrwxrwx 1 ubuntu ubuntu 0 Jan 17 15:07 net -> 'net:[4026531840]'
lrwxrwxrwx 1 ubuntu ubuntu 0 Jan 17 15:07 pid -> 'pid:[4026531836]'
lrwxrwxrwx 1 ubuntu ubuntu 0 Jan 17 15:07 pid_for_children -> 'pid:[4026531836]'
lrwxrwxrwx 1 ubuntu ubuntu 0 Jan 17 15:07 time -> 'time:[4026531834]'
lrwxrwxrwx 1 ubuntu ubuntu 0 Jan 17 15:07 time_for_children -> 'time:[4026531834]'
lrwxrwxrwx 1 ubuntu ubuntu 0 Jan 17 15:07 user -> 'user:[4026532372]'
lrwxrwxrwx 1 ubuntu ubuntu 0 Jan 17 15:07 uts -> 'uts:[4026531838]'

ubuntu@ubuntu-server:~$ readlink /proc/46215/ns/user
user:[4026532372]

#------第一个终端------
root@ubuntu-server:~# ls -l /proc/46215/ns
total 0
lrwxrwxrwx 1 root root 0 Jan 17 15:07 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 Jan 17 15:07 ipc -> 'ipc:[4026531839]'
lrwxrwxrwx 1 root root 0 Jan 17 15:07 mnt -> 'mnt:[4026531841]'
lrwxrwxrwx 1 root root 0 Jan 17 15:07 net -> 'net:[4026531840]'
lrwxrwxrwx 1 root root 0 Jan 17 15:07 pid -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root 0 Jan 17 15:07 pid_for_children -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root 0 Jan 17 15:07 time -> 'time:[4026531834]'
lrwxrwxrwx 1 root root 0 Jan 17 15:07 time_for_children -> 'time:[4026531834]'
lrwxrwxrwx 1 root root 0 Jan 17 15:07 user -> 'user:[4026532372]'
lrwxrwxrwx 1 root root 0 Jan 17 15:07 uts -> 'uts:[4026531838]'

root@ubuntu-server:~# readlink /proc/46215/ns/user
user:[4026532372]

#------第二个终端------
#仍然不能访问/root目录，因为他的拥有着是nobody
root@ubuntu-server:~# ls -l /|grep root
drwx------  12 nobody nogroup       4096 Jan 17 14:51 root
root@ubuntu-server:~# ls /root/
ls: cannot open directory '/root/': Permission denied

#对于原来/home/ubuntu下的内容，显示的owner已经映射过来了，由dev变成了新namespace中的root，
#当前root用户可以访问他里面的内容
root@ubuntu-server:~# ls -l /home/
drwxr-x--- 4 root   root   4096 Jul  3  2023 ubuntu
root@ubuntu-server:~# touch /home/ubuntu/1

root@ubuntu-server:~# hostname container001
hostname: you must be root to change the host name
#修改失败，说明这个新user namespace中的root账号在父user namespace里面不好使
#这也正是user namespace所期望达到的效果，当访问其他user namespace里的资源时，
#是以其他user namespace中的相应账号的权限来执行的，
#比如这里root对应父user namespace的账号是ubuntu，所以改不了系统的hostname
```

> 经过上面的实验可以看到usernamespace 还是比较复杂的，不过我们也能通过这个实验清楚的体会到usernamespace对用户隔离的效果