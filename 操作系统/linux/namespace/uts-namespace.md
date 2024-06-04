## uts namespace

uts(UNIX Time-Sharing System) namespace可隔离hostname和NIS Domain name资源，使得一个宿主机可拥有多个主机名或Domain Name。
换句话说，可让不同namespace中的进程看到不同的主机名。

```shell
# 这是机器原本的主机名
root@ubuntu-server:~# hostname
ubuntu-server

# -u或--uts表示创建一个uts namespace 进入了新的namespace中的shell
root@ubuntu-server:~# unshare -u /bin/bash

# 这个namespace中的hostname也是继承了上级namespace,所以也是"ubuntu-server"
root@ubuntu-server:~# hostname
ubuntu-server

@ hostname hello-world
@ hostname
hello-world

root@ubuntu-server:~# exit
exit
root@ubuntu-server:~# hostname
ubuntu-server

root@ubuntu-server:~# lsns --list |grep uts
4026531838 uts       299       1 root             /lib/systemd/systemd --system --deserialize 48
4026532368 uts         1     670 root             /lib/systemd/systemd-udevd
4026532370 uts         1   57136 systemd-timesync /lib/systemd/systemd-timesyncd
4026532387 uts         1 3928826 root             /bin/sh
4026532491 uts         1   57640 root             /lib/systemd/systemd-machined
4026532525 uts         1     950 root             /lib/systemd/systemd-logind

# 可以看到3928826这个就是我们创建的
```

我们可以看到在我们自己创建的uts namesapce中设置的hostname与宿主机是不一样的，上文也提到了namespace的功能就是为了让进程有一个“私密”空间。

> 总结一下namespace的作用：namespace可以将系统资源（如进程、网络、文件系统等）隔离到不同的命名空间中，使得在一个命名空间中的进程只能看到该命名空间中的资源，而无法看到其他命名空间中的资源