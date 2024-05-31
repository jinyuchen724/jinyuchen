# kubernetes


#一、环境初始化

环境说明：

| 主机   |   角色   |   操作系统 |   软件版本  |    备注  |
| ------ | ----- | ----- | ------- | ------ |
| 10.37.51.1 | master | Centos 7.2(x86-64) | 1.kubelet-1.11.0-0.x86_64 2.kubeadm-1.11.0-0.x86_64 3.kubernetes-cni-0.6.0-0.x86_64 4.kubectl-1.11.0-0.x86_64 |  主节点|
| 10.37.50.2	  | node | Centos 7.2(x86-64) |  同上 |  客户节点|
| 10.37.51.2	  | node | Centos 7.2(x86-64) |  同上 |  客户节点|

>v1.11.0版本推荐使用docker v17.03，v1.11,v1.12,v1.13, 也可以使用，再高版本的docker可能无法正常使用。

1.安装dockerv17.03

```
# 移除以前安装的docker
# yum remove -y docker-ce docker-ce-selinux container-selinux
# rm -rf /var/lib/docker

# 配置docker源及安装

# yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# yum -y install https://download.docker.com/linux/centos/7/x86_64/stable/Packages/docker-ce-selinux-17.03.2.ce-1.el7.centos.noarch.rpm 
# yum -y install docker-ce-17.03.2.ce*
```
 
 
2.配置kubenets yum源

``` 
# cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

3.安装k8s相关软件

```
# yum -y install kubelet-1.11.0-0
# yum -y install kubectl-1.11.0-0
# yum -y install kubeadm-1.11.0-0
```

4.关闭selinux、防火墙

```
# setenforce 0

# 开启转发，否则会启动报错
# vim /etc/sysctl.conf
net.ipv4.ip_forward = 1
# sysctl -p /etc/sysctl.conf

# cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
vm.swappiness=0
EOF

# sysctl --system
```

5.启动docker

```
# systemctl enable docker && systemctl start docker
```

#二、k8s master配置

1.启动master节点

```
# systemctl enable kubelet && systemctl start kubelet
```

2.查看版本所依赖的镜像版本

```
# kubeadm config images list --kubernetes-version=v1.11.0
k8s.gcr.io/kube-apiserver-amd64:v1.11.0
k8s.gcr.io/kube-controller-manager-amd64:v1.11.0
k8s.gcr.io/kube-scheduler-amd64:v1.11.0
k8s.gcr.io/kube-proxy-amd64:v1.11.0
k8s.gcr.io/pause-amd64:3.1
k8s.gcr.io/etcd-amd64:3.2.18
k8s.gcr.io/coredns:1.1.3
```

3.下载依赖的镜像

```
# cat kube.sh 
#!/bin/bash
images=(kube-proxy-amd64:v1.11.0
        kube-scheduler-amd64:v1.11.0
        kube-controller-manager-amd64:v1.11.0
        kube-apiserver-amd64:v1.11.0
        etcd-amd64:3.2.18
        coredns:1.1.3
        pause-amd64:3.1
        kubernetes-dashboard-amd64:v1.8.3
        k8s-dns-sidecar-amd64:1.14.8
        k8s-dns-kube-dns-amd64:1.14.8
        k8s-dns-dnsmasq-nanny-amd64:1.14.8 )
for imageName in ${images[@]} ; do
docker pull registry.cn-shenzhen.aliyuncs.com/duyj/$imageName
docker tag registry.cn-shenzhen.aliyuncs.com/duyj/$imageName k8s.gcr.io/$imageName
docker rmi registry.cn-shenzhen.aliyuncs.com/duyj/$imageName
done

docker pull quay.io/coreos/flannel:v0.10.0-amd64
docker tag da86e6ba6ca1 k8s.gcr.io/pause:3.1
```

4.查看镜像

```
# docker images
REPOSITORY                                 TAG                 IMAGE ID            CREATED             SIZE
k8s.gcr.io/kube-controller-manager-amd64   v1.11.0             55b70b420785        4 weeks ago         155 MB
k8s.gcr.io/kube-scheduler-amd64            v1.11.0             0e4a34a3b0e6        4 weeks ago         56.8 MB
k8s.gcr.io/kube-proxy-amd64                v1.11.0             1d3d7afd77d1        4 weeks ago         97.8 MB
k8s.gcr.io/kube-apiserver-amd64            v1.11.0             214c48e87f58        4 weeks ago         187 MB
k8s.gcr.io/coredns                         1.1.3               b3b94275d97c        2 months ago        45.6 MB
k8s.gcr.io/etcd-amd64                      3.2.18              b8df3b177be2        3 months ago        219 MB
k8s.gcr.io/kubernetes-dashboard-amd64      v1.8.3              0c60bcf89900        5 months ago        102 MB
k8s.gcr.io/k8s-dns-sidecar-amd64           1.14.8              9d10ba894459        5 months ago        42.2 MB
k8s.gcr.io/k8s-dns-dnsmasq-nanny-amd64     1.14.8              ac4746d72dc4        5 months ago        40.9 MB
k8s.gcr.io/k8s-dns-kube-dns-amd64          1.14.8              6ceab6c8330d        5 months ago        50.5 MB
quay.io/coreos/flannel                     v0.10.0-amd64       f0fad859c909        6 months ago        44.6 MB
k8s.gcr.io/pause-amd64                     3.1                 da86e6ba6ca1        7 months ago        742 kB
k8s.gcr.io/pause                           3.1                 da86e6ba6ca1        7 months ago        742 kB 
```

5.查看服务启动情况

```
# echo "export KUBECONFIG=/etc/kubernetes/admin.conf " >> ~/.bashrc
# source ~/.bashrc
#kubectl get nodes
NAME                                STATUS     ROLES     AGE       VERSION
dc07-daily-k8s-bj01host-103750247   NotReady   master    2m        v1.11.0

```

6.执行master节点初始化

```
# kubeadm init --kubernetes-version=v1.11.0 --pod-network-cidr=10.244.0.0/16
......
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

kubeadm join 10.37.51.1:6443 --token tn4bz8.1681s784faygnave --discovery-token-ca-cert-hash sha256:41c18d5387838471e48863b6109c66fc9c5f97a95646f0ca952ffbdcf7b847d6
# 这段很重要要保存下来，后面node节点加入集群需要用到
``` 

7.查看初始化是否成功
  
```
# kubectl get pods --all-namespaces
NAMESPACE     NAME                                                        READY     STATUS    RESTARTS   AGE
kube-system   coredns-78fcdf6894-fzt7n                                    0/1       Pending   0          2m
kube-system   coredns-78fcdf6894-zwsdv                                    0/1       Pending   0          2m
kube-system   etcd-dc07-daily-k8s-bj01host-103750247                      1/1       Running   0          1m
kube-system   kube-apiserver-dc07-daily-k8s-bj01host-103750247            1/1       Running   0          1m
kube-system   kube-controller-manager-dc07-daily-k8s-bj01host-103750247   1/1       Running   0          1m
kube-system   kube-proxy-4tvsh                                            1/1       Running   0          2m
kube-system   kube-scheduler-dc07-daily-k8s-bj01host-103750247            1/1       Running   0          1m
```

8.配置master的网络

 
```
# wget https://raw.githubusercontent.com/coreos/flannel/v0.10.0/Documentation/kube-flannel.yml

# kubectl apply -f  kube-flannel.yml
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.extensions/kube-flannel-ds created  
```
 
9.再次确认初始化是否成功

```
# kubectl get pods --all-namespaces
NAMESPACE     NAME                                                      READY     STATUS              RESTARTS   AGE
kube-system   coredns-78fcdf6894-lqncf                                  0/1       ContainerCreating   0          19m
kube-system   coredns-78fcdf6894-mxzrg                                  0/1       ContainerCreating   0          15m
kube-system   etcd-dc07-daily-k8s-bj01host-1037511                      1/1       Running             0          18m
kube-system   kube-apiserver-dc07-daily-k8s-bj01host-1037511            1/1       Running             0          19m
kube-system   kube-controller-manager-dc07-daily-k8s-bj01host-1037511   1/1       Running             0          19m
kube-system   kube-flannel-ds-t48wx                                     1/1       Running             0          18m
kube-system   kube-proxy-s4768                                          1/1       Running             0          19m
kube-system   kube-scheduler-dc07-daily-k8s-bj01host-1037511            1/1       Running             0          18m
```
 
>发现2个coredns都有问题，都是类似这个错误，这个问题纠结了很久

查看pod日志：

```
# kubectl get pods --namespace=kube-system -l k8s-app=kube-dns -o name
# kubectl describe pods coredns-78fcdf6894-v677n -n kube-system
Warning  FailedCreatePodSandBox  5m (x478 over 15m)   kubelet, dc07-daily-k8s-bj01host-1037511  (combined from similar events): Failed create pod sandbox: rpc error: code = Unknown desc = [failed to set up sandbox container "7581496d8839926e3df6f9f9d095cd5c74c067a5a1485b2d0f31cb359132d556" network for pod "coredns-78fcdf6894-mxzrg": NetworkPlugin cni failed to set up pod "coredns-78fcdf6894-mxzrg_kube-system" network: open /proc/sys/net/ipv6/conf/eth0/accept_dad: no such file or directory, failed to clean up sandbox container "7581496d8839926e3df6f9f9d095cd5c74c067a5a1485b2d0f31cb359132d556" network for pod "coredns-78fcdf6894-mxzrg": NetworkPlugin cni failed to teardown pod "coredns-78fcdf6894-mxzrg_kube-system" network: failed to get IP addresses for "eth0": <nil>]

```

终于在这git这里查到：https://github.com/containernetworking/cni/issues/569

```
# 需要关闭ipv6
# vim /etc/default/grub
GRUB_CMDLINE_LINUX="true ipv6.disable=0" #修改成这个
# grub2-mkconfig -o /boot/grub2/grub.cfg 
# reboot
```

```
kubectl get pods --all-namespaces
NAMESPACE     NAME                                                      READY     STATUS    RESTARTS   AGE
kube-system   coredns-78fcdf6894-lqncf                                  1/1       Running   0          37m
kube-system   coredns-78fcdf6894-mxzrg                                  1/1       Running   0          32m
kube-system   etcd-dc07-daily-k8s-bj01host-1037511                      1/1       Running   1          36m
kube-system   kube-apiserver-dc07-daily-k8s-bj01host-1037511            1/1       Running   1          36m
kube-system   kube-controller-manager-dc07-daily-k8s-bj01host-1037511   1/1       Running   1          36m
kube-system   kube-flannel-ds-t48wx                                     1/1       Running   1          35m
kube-system   kube-proxy-s4768                                          1/1       Running   1          37m
kube-system   kube-scheduler-dc07-daily-k8s-bj01host-1037511            1/1       Running   1          36m
```

#三、添加Node节点
1.完成第一步环境初始化

2.下载镜像

```
# cat nodekube.sh 
#!/bin/bash
images=(kube-proxy-amd64:v1.11.0
        pause-amd64:3.1
        kubernetes-dashboard-amd64:v1.8.3
    heapster-influxdb-amd64:v1.3.3
    heapster-grafana-amd64:v4.4.3
    heapster-amd64:v1.4.2 )
for imageName in ${images[@]} ; do
docker pull registry.cn-shenzhen.aliyuncs.com/duyj/$imageName
docker tag registry.cn-shenzhen.aliyuncs.com/duyj/$imageName k8s.gcr.io/$imageName
docker rmi registry.cn-shenzhen.aliyuncs.com/duyj/$imageName
done

docker pull quay.io/coreos/flannel:v0.10.0-amd64
docker tag k8s.gcr.io/pause-amd64:3.1 k8s.gcr.io/pause:3.1

# bash nodekube.sh
```

3.查看镜像

```
# docker images
REPOSITORY                              TAG                 IMAGE ID            CREATED             SIZE
k8s.gcr.io/kube-proxy-amd64             v1.11.0             1d3d7afd77d1        6 months ago        97.8 MB
k8s.gcr.io/kubernetes-dashboard-amd64   v1.8.3              0c60bcf89900        11 months ago       102 MB
quay.io/coreos/flannel                  v0.10.0-amd64       f0fad859c909        12 months ago       44.6 MB
k8s.gcr.io/pause-amd64                  3.1                 da86e6ba6ca1        13 months ago       742 kB
k8s.gcr.io/pause                        3.1                 da86e6ba6ca1        13 months ago       742 kB
k8s.gcr.io/heapster-influxdb-amd64      v1.3.3              577260d221db        16 months ago       12.5 MB
k8s.gcr.io/heapster-grafana-amd64       v4.4.3              8cb3de219af7        16 months ago       152 MB
k8s.gcr.io/heapster-amd64               v1.4.2              d4e02f5922ca        17 months ago       73.4 MB
```

4.加入master节点

```
# kubeadm join 10.37.51.1:6443 --token tn4bz8.1681s784faygnave --discovery-token-ca-cert-hash sha256:41c18d5387838471e48863b6109c66fc9c5f97a95646f0ca952ffbdcf7b847d6
```
5.在master节点查看node

```
# kubectl get nodes
NAME                              STATUS    ROLES     AGE       VERSION
dc07-daily-k8s-bj01host-1037502   Ready     <none>    3m        v1.11.0
dc07-daily-k8s-bj01host-1037511   Ready     master    1h        v1.11.0

# kubectl get pods -n kube-system -o wide
NAME                                                      READY     STATUS    RESTARTS   AGE       IP            NODE
coredns-78fcdf6894-lqncf                                  1/1       Running   0          1h        10.244.0.37   dc07-daily-k8s-bj01host-1037511
coredns-78fcdf6894-mxzrg                                  1/1       Running   0          1h        10.244.0.36   dc07-daily-k8s-bj01host-1037511
etcd-dc07-daily-k8s-bj01host-1037511                      1/1       Running   1          1h        10.37.51.1    dc07-daily-k8s-bj01host-1037511
kube-apiserver-dc07-daily-k8s-bj01host-1037511            1/1       Running   1          1h        10.37.51.1    dc07-daily-k8s-bj01host-1037511
kube-controller-manager-dc07-daily-k8s-bj01host-1037511   1/1       Running   1          1h        10.37.51.1    dc07-daily-k8s-bj01host-1037511
kube-flannel-ds-fmlrz                                     1/1       Running   0          3m        10.37.50.2    dc07-daily-k8s-bj01host-1037502
kube-flannel-ds-t48wx                                     1/1       Running   1          1h        10.37.51.1    dc07-daily-k8s-bj01host-1037511
kube-proxy-n5b7p                                          1/1       Running   0          3m        10.37.50.2    dc07-daily-k8s-bj01host-1037502
kube-proxy-s4768                                          1/1       Running   1          1h        10.37.51.1    dc07-daily-k8s-bj01host-1037511
kube-scheduler-dc07-daily-k8s-bj01host-1037511            1/1       Running   1          1h        10.37.51.1    dc07-daily-k8s-bj01host-1037511
```


6.node节点显示的role为none,修改label

```
# kubectl label node dc07-daily-k8s-bj01host-1037502 node-role.kubernetes.io/node=node-1037502
node/dc07-daily-k8s-bj01host-1037502 labeled

# kubectl get node --show-labels
NAME                              STATUS    ROLES     AGE       VERSION   LABELS
dc07-daily-k8s-bj01host-1037502   Ready     node      13m       v1.11.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=dc07-daily-k8s-bj01host-1037502,node-role.kubernetes.io/node=node-1037502
dc07-daily-k8s-bj01host-1037511   Ready     master    1h        v1.11.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=dc07-daily-k8s-bj01host-1037511,node-role.kubernetes.io/master=
```

# 四、外部访问

由于Pod和Service是kubernetes集群范围内的虚拟概念，所以集群外的客户端系统无法通过Pod的IP地址或者Service的虚拟IP地址和虚拟端口号访问到它们。

> Kubernetes 暴露服务的方式目前只有三种：LoadBlancer Service、NodePort Service、Ingress

官方地址：https://github.com/kubernetes/ingress-nginx/tree/master/deploy

### 一、模式介绍

#####1.端口映射的方式
- 设置容器级别的hostPort，将容器应用的端口号映射到机器(docker的host模式）
- 设置Pod级别的hostNetwork=true，该Pod中所有容器的端口号都将被直接映射到宿主机上
- 设置nodePort映射到宿主机，同时设置Service的类型为NodePort(Nodeport模式，此时集群会在每一个节点都监听设置nodePort）

如果设置了Service的nodePort，那么集群会在每一个节点都监听设置的nodePort，外部客户端可以通过任意 nodeIP:Port的方式对集群服务进行访问。但是当集群中服务较多，那么需要管理的端口也会比较多，各个端口之间不能冲突，比较麻烦；另外，因为方式访问形式为nodeIP:Port的方式，那么对于一些HTTP服务，这种方式是无法做到根据URL路径进行转发的。

#####2.LoadBlancer Service
- 设置LoadBalancer映射到云服务商提供的LoadBalancer地址

#####3.Ingress

ingress是kubernetes V1.1版本之后新增的资源对象，用于实现HTTP层业务路由机制。 
实现ingress路由机制主要包括3个组件： 

- ingress是kubernetes的一个资源对象，用于编写定义规则。 
- 反向代理负载均衡器，通常以Service的Port方式运行，接收并按照ingress定义的规则进行转发，通常为nginx，haproxy，traefik等，这里使用nginx。 
- ingress-controller，监听apiserver，获取服务新增，删除等变化，并结合ingress规则动态更新到反向代理负载均衡器上，并重载配置使其生效。


### 二、安装配置

1.获取部署文件

```
# git clone https://github.com/kubernetes/ingress-nginx.git
# cat /root/ingress/ingress-nginx/deploy/mandatory.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx

---

apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: default-http-backend
  labels:
    app.kubernetes.io/name: default-http-backend
    app.kubernetes.io/part-of: ingress-nginx
  namespace: ingress-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: default-http-backend
      app.kubernetes.io/part-of: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: default-http-backend
        app.kubernetes.io/part-of: ingress-nginx
    spec:
      terminationGracePeriodSeconds: 60
      containers:
        - name: default-http-backend
          # Any image is permissible as long as:
          # 1. It serves a 404 page at /
          # 2. It serves 200 on a /healthz endpoint
          image: registry.cn-qingdao.aliyuncs.com/kubernetes_xingej/defaultbackend-amd64:1.5 #原来是k8s.gcr.io/defaultbackend-amd64
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
              scheme: HTTP
            initialDelaySeconds: 30
            timeoutSeconds: 5
          ports:
            - containerPort: 8080
          resources:
            limits:
              cpu: 10m
              memory: 20Mi
            requests:
              cpu: 10m
              memory: 20Mi

---
apiVersion: v1
kind: Service
metadata:
  name: default-http-backend
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: default-http-backend
    app.kubernetes.io/part-of: ingress-nginx
spec:
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app.kubernetes.io/name: default-http-backend
    app.kubernetes.io/part-of: ingress-nginx

---

kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---

kind: ConfigMap
apiVersion: v1
metadata:
  name: tcp-services
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---

kind: ConfigMap
apiVersion: v1
metadata:
  name: udp-services
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---

apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress-serviceaccount
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: nginx-ingress-clusterrole
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - endpoints
      - nodes
      - pods
      - secrets
    verbs:
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - services
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - "extensions"
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - events
    verbs:
      - create
      - patch
  - apiGroups:
      - "extensions"
    resources:
      - ingresses/status
    verbs:
      - update

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role
metadata:
  name: nginx-ingress-role
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - pods
      - secrets
      - namespaces
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - configmaps
    resourceNames:
      # Defaults to "<election-id>-<ingress-class>"
      # Here: "<ingress-controller-leader>-<nginx>"
      # This has to be adapted if you change either parameter
      # when launching the nginx-ingress-controller.
      - "ingress-controller-leader-nginx"
    verbs:
      - get
      - update
  - apiGroups:
      - ""
    resources:
      - configmaps
    verbs:
      - create
  - apiGroups:
      - ""
    resources:
      - endpoints
    verbs:
      - get

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: nginx-ingress-role-nisa-binding
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: nginx-ingress-role
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: nginx-ingress-clusterrole-nisa-binding
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: nginx-ingress-clusterrole
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx

---

apiVersion: extensions/v1beta1
kind: DaemonSet                  
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  #replicas: 1
  #selector:
  #  matchLabels:
  #    app.kubernetes.io/name: ingress-nginx
  #    app.kubernetes.io/part-of: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
      annotations:
        prometheus.io/port: "10254"
        prometheus.io/scrape: "true"
    spec:
      nodeSelector:
        LB: ingress
      hostNetwork: true
      serviceAccountName: nginx-ingress-serviceaccount
      containers:
        - name: nginx-ingress-controller
          image: registry.cn-qingdao.aliyuncs.com/kubernetes_xingej/nginx-ingress-controller:0.20.0  #原来是quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.20.0
          args:
            - /nginx-ingress-controller
            - --default-backend-service=$(POD_NAMESPACE)/default-http-backend
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
            - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
            - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
            - --publish-service=$(POD_NAMESPACE)/ingress-nginx
            - --annotations-prefix=nginx.ingress.kubernetes.io
          securityContext:
            capabilities:
              drop:
                - ALL
              add:
                - NET_BIND_SERVICE
            # www-data -> 33
            runAsUser: 33
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - name: http
              containerPort: 80
              hostPort: 80
            - name: https
              containerPort: 443
              hostPort: 443
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 1
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 1

---
```

这里改了几个地方：在结尾以'#'方式标注:

- image

| 原配置   |   新配置	 |   
| ------  |   -----    |
| k8s.gcr.io/defaultbackend-amd64	 | registry.cn-qingdao.aliyuncs.com/kubernetes_xingej/defaultbackend-amd64:1.5 |
| quay.io/kubernetes-ingress-controller/nginx-ingress-controller | registry.cn-qingdao.aliyuncs.com/kubernetes_xingej/nginx-ingress-controller:0.20.0|

> 国外镜像无法下载，所以这里改成国内阿里云镜像源
 
2.kind

原配置:

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/part-of: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
      annotations:
        prometheus.io/port: "10254"
        prometheus.io/scrape: "true"
    spec:
      serviceAccountName: nginx-ingress-serviceaccount
```


新配置：

```
apiVersion: extensions/v1beta1
kind: DaemonSet                  
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  #replicas: 1
  #selector:
  #  matchLabels:
  #    app.kubernetes.io/name: ingress-nginx
  #    app.kubernetes.io/part-of: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
      annotations:
        prometheus.io/port: "10254"
        prometheus.io/scrape: "true"
    spec:
      nodeSelector:
        LB: ingress
      hostNetwork: true
      serviceAccountName: nginx-ingress-serviceaccount
```


DaemonSet能够让所有（或者特定）的节点运行同一个pod。

当节点加入到K8S集群中，pod会被（DaemonSet）调度到该节点上运行，当节点从K8S集群中被移除，被DaemonSet调度的pod会被移除，如果删除DaemonSet，所有跟这个DaemonSet相关的pods都会被删除。

在某种程度上，DaemonSet承担了RC的部分功能，它也能保证相关pods持续运行，如果一个DaemonSet的Pod被杀死、停止、或者崩溃，那么DaemonSet将会重新创建一个新的副本在这台计算节点上。

一般应用于日志收集、监控采集、分布式存储守护进程、ingress等。

Deployment和DaemonSet的区别 
Deployment 部署的副本 Pod 会分布在各个 Node 上，每个 Node 都可能运行好几个副本。DaemonSet 的不同之处在于：每个 Node 上最多只能运行一个副本。

主要区别：

The scalability is much better when using a Deployment, because you will have a Single-Pod-per-Node model when using the DaemonSet.
It is possible to exclusively run a Service on a dedicated set of machines using taints and tolerations with a DaemonSet.
On the other hand the DaemonSet allows you to access any Node directly on Port 80 and 443, where you have to setup a Service object with a Deployment.

lb这层需要固定部署在特定的机器以便后续做dns解析、绑定lvs等。

这里使用标签方式，将ingress control固定node。

将要使用的node节点打上标签：

```
# kubectl label node  dc07-daily-k8s-bj01host-1037502 LB=ingress
```

3.hostNetwork

设置Pod级别的hostNetwork=true，该Pod中所有容器的端口号都将被直接映射到宿主机上

4.执行配置文件

```
# kubectl apply -f mandatory.yaml
```

5.查看组件运行状态

```
#kubectl get pods -n ingress-nginx -owide
NAME                                    READY     STATUS    RESTARTS   AGE       IP            NODE
default-http-backend-8498ffcc8c-qmk9m   1/1       Running   0          1h        10.244.2.54   dc07-daily-k8s-bj01host-1037512
nginx-ingress-controller-5cgn6          1/1       Running   0          1h        10.37.50.2    dc07-daily-k8s-bj01host-1037502

#kubectl get all -n ingress-nginx
NAME                                        READY     STATUS    RESTARTS   AGE
pod/default-http-backend-8498ffcc8c-qmk9m   1/1       Running   0          1h
pod/nginx-ingress-controller-5cgn6          1/1       Running   0          1h

NAME                           TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
service/default-http-backend   ClusterIP   10.99.79.90   <none>        80/TCP    1h

NAME                                      DESIRED   CURRENT   READY     UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/nginx-ingress-controller   1         1         1         1            1           LB=ingress      1h

NAME                                   DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/default-http-backend   1         1         1            1           1h

NAME                                              DESIRED   CURRENT   READY     AGE
replicaset.apps/default-http-backend-8498ffcc8c   1         1         1         1h
```

6.查看service状态

```
# kubectl get service -n ingress-nginx
NAME                   TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
default-http-backend   ClusterIP   10.99.79.90   <none>        80/TCP    1h
```
