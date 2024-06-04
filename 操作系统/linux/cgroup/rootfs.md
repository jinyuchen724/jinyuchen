## rootfs文件隔离

其实rootfs很容易理解，本质上就是在机器上建一个目录，然后让容器（或者我们这里测试的进程）知道这个目录就是自己根目录，别人看不到。其实和namespace的作用差不多。隔离的是文件系统的环境。

我们还是用具体的例子来了解一下：
```shell
root@ubuntu-server:# mkdir /tmp/rootfstest

root@ubuntu-server:# chroot /tmp/rootfstest/ /bin/bash
chroot: failed to run command ‘/bin/bash’: No such file or directory

#准备bash和bash的依赖关系，因为/tmp/ttt将会被设定为/，按照此关系将bash与其依赖关系的相对路径设定好。否则就会找不到bash

root@ubuntu-server:# mkdir /tmp/rootfstest/bin /tmp/rootfstest/lib64

root@ubuntu-server:# which bash
/usr/bin/bash

root@ubuntu-server:# ldd  /usr/bin/bash
linux-vdso.so.1 =>  (0x00007fff0b1ca000)
libtinfo.so.5 => /lib64/libtinfo.so.5 (0x00007f45d8787000)
libdl.so.2 => /lib64/libdl.so.2 (0x00007f45d8583000)
libc.so.6 => /lib64/libc.so.6 (0x00007f45d81b5000)
/lib64/ld-linux-x86-64.so.2 (0x00007f45d89b1000)

#将bash依赖库拷贝到待测试待rootfs目录下
root@ubuntu-server:# cp /lib64/libtinfo.so.5 /lib64/libdl.so.2 /lib64/libc.so.6 /lib64/ld-linux-x86-64.so.2 /tmp/rootfstest/lib64

#将bash拷贝到待测试待rootfs/bin目录下
root@ubuntu-server:# cp /user/bin/bash /tmp/rootfstest/bin/bash
```

```shell
#将/tmp/rootfstest/作为chroot的根目录
root@ubuntu-server:# chroot /tmp/rootfstest/ /bin/bash
root@ubuntu-server:# pwd
/
```

因为我们隔离的chroot中没有其他命令环境，所以我们直接用busbox这个工具来测试
```shell
root@ubuntu-server:# wget --no-check-certificate https://busybox.net/downloads/binaries/1.35.0-x86_64-linux-musl/busybox
root@ubuntu-server:# chmod 777 busybox

#创建的chroot目录下面是一个隔离的'文件系统'
@# ./busybox ls -l
total 3576
drwxr-xr-x    2 0        0             4096 Feb  4 08:45 bin
-rwxrwxrwx    1 0        0          1131168 Jan 17  2022 busybox
drwxr-xr-x    2 0        0             4096 Feb  4 08:45 lib64

@# ./busybox echo "choroot test" > test.a

#退出chroot环境
#exit 

#数据还在
@# ll /tmp/rootfstest
total 3580
drwxr-xr-x 2 root root    4096 Feb  4 16:45 bin
-rwxrwxrwx 1 root root 1131168 Jan 18  2022 busybox
-rw-r--r-- 1 root root      13 Feb  4 17:14 test.a

#创建的文件也正常读取
[root@ubuntu-server:# /tmp/rootfstest]
cat test.a
choroot test
```

经过这个小例子,应该可以联想到docker创建的容器进程为什么是一个独立的文件系统，当然docker还做了联合文件等事情，不过底层的原理都是利用了chroot
来对每个docker容器进行文件的隔离。
    


    

    
