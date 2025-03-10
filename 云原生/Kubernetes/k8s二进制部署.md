## 一、k8s环境部署

### 2.1 搭建etcd集群

#### 2.1.1 安装签发证书工具cfssl

```shell
[root@k8s-host /root]# wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
[root@k8s-host /root]# wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
[root@k8s-host /root]# wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
[root@k8s-host /root]# chmod +x cfssl*
[root@k8s-host /root]# mv cfssl_linux-amd64 /usr/local/bin/cfssl 
[root@k8s-host /root]# mv cfssljson_linux-amd64 /usr/local/bin/cfssljson 
[root@k8s-host /root]# mv cfssl-certinfo_linux-amd64 /usr/local/bin/cfssl-certinfo
```

#### 2.1.2 配置ca证书

>- 生成ca证书请求文件
```shell
[root@k8s-host /root]# cat ca-csr.json
{
  "CN": "kubernetes",
  "key": {
      "algo": "rsa",
      "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Zhejiang",
      "L": "Hangzhou",
      "O": "k8s",
      "OU": "system"
    }
  ],
  "ca": {
          "expiry": "876000h"
  }
}

[root@k8s-host /root]# cfssl gencert -initca ca-csr.json  | cfssljson -bare ca
2022/08/10 13:37:04 [INFO] generating a new CA key and certificate from CSR
2022/08/10 13:37:04 [INFO] generate received request
2022/08/10 13:37:04 [INFO] received CSR
2022/08/10 13:37:04 [INFO] generating key: rsa-2048
2022/08/10 13:37:04 [INFO] encoded CSR
2022/08/10 13:37:04 [INFO] signed certificate with serial number 277004694657827998333236437935739196848568561695

[root@k8s-host /data/ssl]# ls
ca.csr  ca-csr.json  ca-key.pem  ca.pem
```

>- 生成ca证书配置文件,主要注意过期时间,这里过期时间设置的是100年
```shell
[root@k8s-host /root]# cat ca-config.json
{
  "signing": {
      "default": {
          "expiry": "876000h"
        },
      "profiles": {
          "kubernetes": {
              "usages": [
                  "signing",
                  "key encipherment",
                  "server auth",
                  "client auth"
              ],
              "expiry": "876000h"
          }
      }
  }
}
```


#### 2.1.3 生成etcd证书
>- 主要关注host里的配置,这里最好用域名,ip万一变了证书失效,会比较麻烦

```shell
[root@k8s-host /root]# cat etcd-csr.json
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "10.160.65.28",
    "10.160.69.230",
    "10.160.65.147",
    "bigdata-spark-k8s1.idcvdian.com",
    "bigdata-spark-k8s2.idcvdian.com",
    "bigdata-spark-k8s3.idcvdian.com"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [{
    "C": "CN",
    "ST": "Zhejiang",
    "L": "Hangzhou",
    "O": "k8s",
    "OU": "system"
  }]
}
```

```shell
[root@k8s-host /root]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes etcd-csr.json | cfssljson  -bare etcd
2022/08/10 13:49:29 [INFO] generate received request
2022/08/10 13:49:29 [INFO] received CSR
2022/08/10 13:49:29 [INFO] generating key: rsa-2048
2022/08/10 13:49:29 [INFO] encoded CSR
2022/08/10 13:49:29 [INFO] signed certificate with serial number 580147585393953277974184358784294478670114176750
2022/08/10 13:49:29 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
# 查看一下
[root@k8s-host ]# ls etcd*.pem 
etcd-key.pem  etcd.pem
```


#### 2.1.4 部署 etcd 集群

>- 下载安装包
```shell
ETCD_VER=v3.5.15
GOOGLE_URL=https://storage.googleapis.com/etcd
GITHUB_URL=https://github.com/etcd-io/etcd/releases/download
DOWNLOAD_URL=${GOOGLE_URL}
wget ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz
tar -xvf etcd-v3.5.15-linux-amd64.tar.gz
cp etcd-v3.5.15-linux-amd64/etcd* /usr/bin/
@#chmod +x /usr/bin/etcd*
@#mkdir -p /etc/etcd
```

>- 创建配置文件
```shell
[root@k8s-host ]# /root]# /data/work]# cat etcd.conf
#[Member]
ETCD_NAME="etcd1"
ETCD_DATA_DIR="/home/www/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://10.160.65.28:2380"
ETCD_LISTEN_CLIENT_URLS="https://10.160.65.28:2379,http://127.0.0.1:2379"
#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://10.160.65.28:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://10.160.65.28:2379"
ETCD_INITIAL_CLUSTER="etcd1=https://10.160.65.28:2380,etcd2=https://10.160.69.230:2380,etcd3=https://10.160.65.147:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
#注：

ETCD_NAME：节点名称，集群中唯一
ETCD_DATA_DIR：数据目录
ETCD_LISTEN_PEER_URLS：集群通信监听地址
ETCD_LISTEN_CLIENT_URLS：客户端访问监听地址
ETCD_INITIAL_ADVERTISE_PEER_URLS：集群通告地址
ETCD_ADVERTISE_CLIENT_URLS：客户端通告地址
ETCD_INITIAL_CLUSTER：集群节点地址
ETCD_INITIAL_CLUSTER_TOKEN：集群 Token
ETCD_INITIAL_CLUSTER_STATE：加入集群的当前状态，new 是新集群，existing 表示加入已有集群
```

>- 创建启动服务文件
```shell
[root@k8s-host /lib/systemd/system/]#vim etcd.service
# cat etcd.service
[Unit]
Description=etcd - highly-available key value store
Documentation=https://etcd.io/docs
Documentation=man:etcd
After=network.target
Wants=network-online.target

[Service]
Environment=DAEMON_ARGS=
Environment=ETCD_NAME=%H
Environment=ETCD_DATA_DIR=/var/lib/etcd/default
EnvironmentFile=-/etc/etcd/etcd.conf
Type=notify
User=etcd
PermissionsStartOnly=true
#ExecStart=/bin/sh -c "GOMAXPROCS=$(nproc) /usr/bin/etcd $DAEMON_ARGS"
ExecStart=/usr/bin/etcd \
--cert-file=/etc/etcd/ssl/etcd.pem \
--key-file=/etc/etcd/ssl/etcd-key.pem \
--trusted-ca-file=/etc/etcd/ssl/ca.pem \
--peer-cert-file=/etc/etcd/ssl/etcd.pem \
--peer-key-file=/etc/etcd/ssl/etcd-key.pem \
--peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
--peer-client-cert-auth=true \
--client-cert-auth=true

Restart=on-abnormal
#RestartSec=10s
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
Alias=etcd2.service
```

```shell
[root@k8s-host]# mkdir -p /etc/etcd/ssl
[root@k8s-host /data/ssl]# cp etcd.pem etcd-key.pem ca.pem /etc/etcd/ssl/
[root@k8s-host /data/ssl]# ls /etc/etcd/ssl/
ca.pem  etcd-key.pem  etcd.pem
[root@k8s-host]# mkdir -p /var/lib/etcd/default.etcd
[root@k8s-host]# chown etcd:etcd /home/www/etcd/default.etcd/
[root@k8s-host]# chown -R etcd:etcd /etc/etcd/ssl/*
```

其他2个节点操作类似，不赘述。。。

>- 启动 etcd 集群
```shell
[root@k8s-host]# systemctl daemon-reload && systemctl enable etcd.service && systemctl start etcd.service
```

>- 查看 etcd 集群

[root@k8s-host]# ETCDCTL_API=3 && /usr/bin/etcdctl --write-out=table --cacert=/etc/etcd/ssl/ca.pem --cert=/etc/etcd/ssl/etcd.pem --key=/etc/etcd/ssl/etcd-key.pem --endpoints=https://10.160.65.28:2379,https://10.160.69.230:2379,https://10.160.65.147:2379 endpoint health+---------------------------+--------+-------------+-------+
+----------------------------+--------+-------------+-------+
|          ENDPOINT          | HEALTH |    TOOK     | ERROR |
+----------------------------+--------+-------------+-------+
| https://10.160.65.147:2379 |   true |  13.83799ms |       |
|  https://10.160.65.28:2379 |   true | 14.669041ms |       |
| https://10.160.69.230:2379 |   true | 20.950313ms |       |
+----------------------------+--------+-------------+-------+

### 2.2 安装kubernetes组件

下载安装包,二进制包所在的 github 地址如下：
https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/

```shell
[root@k8s-host /data]# wget https://storage.googleapis.com/kubernetes-release/release/v1.23.2/kubernetes-server-linux-amd64.tar.gz
[root@k8s-host /data]# tar -xvf kubernetes-server-linux-amd64.tar.gz
[root@k8s-host /data]# cd kubernetes/server/bin/
[root@k8s-host /data/kubernetes/server/bin/]# cp kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/bin/
[root@k8s-host ]# mkdir -p /etc/kubernetes/ssl
[root@k8s-host ]# mkdir -p /var/log/kubernetes
```

#### 2.2.1 部署 apiserver 组件

启动 TLS Bootstrapping 机制

```shell
Master apiserver 启用 TLS 认证后，每个节点的 kubelet 组件都要使用由 apiserver 使用的CA 签发的有效证书才能与 apiserver 通讯，当 Node 节点很多时，这种客户端证书颁发需要大量工作，同样也会增加集群扩展复杂度。
为了简化流程，Kubernetes 引入了 TLS bootstraping 机制来自动颁发客户端证书，kubelet 会以一个低权限用户自动向 apiserver 申请证书，kubelet 的证书由 apiserver 动态签署。
Bootstrap 是很多系统中都存在的程序，比如 Linux 的 bootstrap，bootstrap 一般都是作为预先配置在开启或者系统启动的时候加载，这可以用来生成一个指定环境。Kubernetes 的 kubelet 在启动时同样可以加载一个这样的配置文件，这个文件的内容类似如下形式：

apiVersion: v1 clusters: null contexts:
- context:
cluster: kubernetes
user: kubelet-bootstrap
name: default
current-context: default
kind: Config preferences: {}
users:
-name: kubelet-bootstrap user: {}
```

- TLS bootstrapping 具体引导过程
TLS 作用
TLS 的作用就是对通讯加密，防止中间人窃听；同时如果证书不信任的话根本就无法与 apiserver建立连接，更不用提有没有权限向 apiserver 请求指定内容。

- RBAC 作用

当 TLS 解决了通讯问题后，那么权限问题就应由 RBAC 解决(可以使用其他权限模型，如ABAC)；RBAC 中规定了一个用户或者用户组(subject)具有请求哪些 api 的权限；在配合 TLS 加密的时候，实际上 apiserver 读取客户端证书的 CN 字段作为用户名，读取 O 字段作为用户组.
以上说明：第一，想要与 apiserver 通讯就必须采用由 apiserver CA 签发的证书，这样才能形成信任关系，建立 TLS 连接；第二，可以通过证书的 CN、O 字段来提供 RBAC 所需的用户与用户组。
kubelet 首次启动流程
TLS bootstrapping 功能是让 kubelet 组件去 apiserver 申请证书，然后用于连接apiserver；那么第一次启动时没有证书如何连接 apiserver ?
在 apiserver 配置中指定了一个 token.csv 文件，该文件中是一个预设的用户配置；同时该用户的Token 和 由 apiserver 的 CA 签发的用户被写入了 kubelet 所使用的 bootstrap.kubeconfig 配置文件中；这样在首次请求时，kubelet 使用 bootstrap.kubeconfig 中被 apiserver CA 签发证书时信任的用户来与 apiserver 建立 TLS 通讯，使用 bootstrap.kubeconfig 中的用户 Token 来向apiserver 声明自己的 RBAC 授权身份.
token.csv 格式:
3940fd7fbb391d1b4d861ad17a1f0613,kubelet-bootstrap,10001,"system:kubelet- bootstrap"
首次启动时，可能与遇到 kubelet 报 401 无权访问 apiserver 的错误；这是因为在默认情况下，kubelet 通过 bootstrap.kubeconfig 中的预设用户 Token 声明了自己的身份，然后创建 CSR请求；但是不要忘记这个用户在我们不处理的情况下他没任何权限的，包括创建 CSR 请求；所以需要创建一个ClusterRoleBinding，将预设用户 kubelet-bootstrap 与内置的 ClusterRole system:node-bootstrapper 绑定到一起，使其能够发起 CSR 请求。稍后安装 kubelet 的时候演示。

- 创建 token.csv 文件
```shell
[root@k8s-host ]# cd /data/
[root@k8s-host ]# cat > token.csv << EOF
$(head -c 16 /dev/urandom | od -An -t x | tr -d ' '),kubelet-bootstrap,10001,"system:kubelet-bootstrap"
EOF
```


- 格式：token，用户名，UID，用户组
创建 csr 请求文件，替换为自己机器的 IP
注： 如果 hosts 字段不为空则需要指定授权使用该证书的 IP 或域名列表。 由于该证书后续被 kubernetes master 集群使用，需要将 master 节点的 IP 都填上，
同时还需要填写 service 网络的首个 IP。(一般是 kube-apiserver 指定的 service-cluster-ip-range 网段的第一个 IP，如 10.255.0.1)

```shell
[root@k8s-host ]# cat kube-apiserver-csr.json
{
    "CN": "kubernetes",
    "hosts": [
    "10.160.65.28",
    "10.160.69.230",
    "10.160.65.147",
    "bigdata-spark-k8s1.idcvdian.com",
    "bigdata-spark-k8s2.idcvdian.com",
    "bigdata-spark-k8s3.idcvdian.com",
    "10.39.128.64",
    "bigdata-spark-k8s.idcvdian.com",
    "127.0.0.1",
    "10.96.0.1",
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local"
],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
    {
        "C": "CN",
        "ST": "Zhejiang",
        "L": "Hangzhou",
        "O": "k8s",
        "OU": "system"
        }
    ]
}
```

>- 生成k8s-api证书
```shell
[root@k8s-host ]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-apiserver-csr.json | cfssljson -bare kube-apiserver
2022/08/10 16:34:32 [INFO] generate received request
2022/08/10 16:34:32 [INFO] received CSR
2022/08/10 16:34:32 [INFO] generating key: rsa-2048
2022/08/10 16:34:32 [INFO] encoded CSR
2022/08/10 16:34:32 [INFO] signed certificate with serial number 477144676195233738114152251729193994277731824828
2022/08/10 16:34:32 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
```

创建apiserver的配置文件

创建 api-server 的配置文件，替换成自己的 ip
```shell
[root@k8s-host ]# cat /etc/kubernetes/kube-apiserver.conf
KUBE_APISERVER_OPTS="--enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \
  --anonymous-auth=false \
  --bind-address=10.160.65.28 \
  --secure-port=6443 \
  --advertise-address=10.160.65.28 \
  --insecure-port=0 \
  --authorization-mode=Node,RBAC \
  --runtime-config=api/all=true \
  --enable-bootstrap-token-auth \
  --service-cluster-ip-range=10.96.0.0/12 \
  --token-auth-file=/etc/kubernetes/token.csv \
  --service-node-port-range=30000-50000 \
  --tls-cert-file=/etc/kubernetes/ssl/kube-apiserver.pem  \
  --tls-private-key-file=/etc/kubernetes/ssl/kube-apiserver-key.pem \
  --client-ca-file=/etc/kubernetes/ssl/ca.pem \
  --kubelet-client-certificate=/etc/kubernetes/ssl/kube-apiserver.pem \
  --kubelet-client-key=/etc/kubernetes/ssl/kube-apiserver-key.pem \
  --service-account-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --service-account-signing-key-file=/etc/kubernetes/ssl/ca-key.pem  \
  --service-account-issuer=https://kubernetes.default.svc.cluster.local \
  --etcd-cafile=/etc/etcd/ssl/ca.pem \
  --etcd-certfile=/etc/etcd/ssl/etcd.pem \
  --etcd-keyfile=/etc/etcd/ssl/etcd-key.pem \
  --etcd-servers=https://bigdata-spark-k8s1.idcvdian.com:2379,https://bigdata-spark-k8s2.idcvdian.com:2379,https://bigdata-spark-k8s3.idcvdian.com:2379 \
  --enable-swagger-ui=true \
  --allow-privileged=true \
  --apiserver-count=3 \
  --audit-log-maxage=30 \
  --audit-log-maxbackup=3 \
  --audit-log-maxsize=100 \
  --audit-log-path=/home/www/logs/kubernetes-audit \
  --event-ttl=1h \
  --alsologtostderr=true \
  --logtostderr=false \
  --requestheader-client-ca-file=/etc/kubernetes/ssl/ca.pem \
  --requestheader-allowed-names= \
  --requestheader-extra-headers-prefix=X-Remote-Extra- \
  --requestheader-group-headers=X-Remote-Group \
  --requestheader-username-headers=X-Remote-User \
  --log-dir=/var/log/kubernetes \
  --v=4"
  
# 注解
--logtostderr：启用日志
--v：日志等级
--log-dir：日志目录
--etcd-servers：etcd 集群地址
--bind-address：监听地址
--secure-port：https 安全端口
--advertise-address：集群通告地址
--allow-privileged：启用授权
--service-cluster-ip-range：Service 虚拟 IP 地址段
--enable-admission-plugins：准入控制模块
--authorization-mode：认证授权，启用 RBAC 授权和节点自管理
--enable-bootstrap-token-auth：启用 TLS bootstrap 机制
--token-auth-file：bootstrap token 文件
--service-node-port-range：Service nodeport 类型默认分配端口范围
--kubelet-client-xxx：apiserver 访问 kubelet 客户端证书
--tls-xxx-file：apiserver https 证书
--etcd-xxxfile：连接 Etcd 集群证书
–-audit-log-xxx：审计日志  
```

创建服务启动文件
```shell
[root@k8s-host ]# vim kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
After=etcd.service
Wants=etcd.service

[Service]
EnvironmentFile=-/etc/kubernetes/kube-apiserver.conf
ExecStart=/usr/bin/kube-apiserver $KUBE_APISERVER_OPTS
Restart=on-failure
RestartSec=5
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

其他2个节点操作类似,这里不赘述

启动kube-apiserver
```shell
[root@k8s-host ]# systemctl daemon-reload && systemctl enable kube-apiserver && systemctl start kube-apiserver
```


### 2.3 部署 kubectl 组件
#### 2.3.1 kubectl组件介绍
Kubectl 是客户端工具，操作k8s 资源的，如增删改查等。

Kubectl 操作资源的时候，怎么知道连接到哪个集群，需要一个文件/etc/kubernetes/admin.conf，kubectl 会根据这个文件的配置，去访问 k8s 资源。

/etc/kubernetes/admin.conf 文件记录了访问的 k8s 集群，和用到的证书。可以设置一个环境变量 KUBECONFIG

[root@ k8s-master1 ~]# export KUBECONFIG=/etc/kubernetes/admin.conf
这样在操作 kubectl，就会自动加载 KUBECONFIG 来操作要管理哪个集群的 k8s 资源了也可以按照下面方法，这个是在 kubeadm 初始化 k8s 的时候会提示我们要用的一个方法

[root@ k8s-master1 ~]# cp /etc/kubernetes/admin.conf /root/.kube/config
这样我们在执行 kubectl，就会加载/root/.kube/config 文件，去操作 k8s 资源了

如果设置了 KUBECONFIG，那就会先找到 KUBECONFIG 去操作 k8s，如果没有 KUBECONFIG变量，那就会使用/root/.kube/config 文件决定管理哪个 k8s 集群的资源注意：admin.conf 还没创建，下面步骤创建

#### 2.3.2 创建 csr 请求文件
```shell
[root@k8s-host ]# cat admin-csr.json
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Zhejiang",
      "L": "Hangzhou",
      "O": "system:masters",
      "OU": "system"
    }
  ]
}
```


说明： 后续 kube-apiserver 使用 RBAC 对客户端(如 kubelet、kube-proxy、Pod)请求进行授权； kube-apiserver 预定义了一些 RBAC 使用的 RoleBindings，如 cluster-admin 将 Group system:masters 与 Role cluster-admin 绑定，
该 Role 授予了调用 kube-apiserver 的所有 API 的权限； O 指定该证书的 Group 为 system:masters，kubelet 使用该证书访问 kube-apiserver 时 ，由于证书被 CA 签名，所以认证通过，同时由于证书用户组为经过预授权的system:masters，所以被授予访问所有 API 的权限；

注： 这个 admin 证书，是将来生成管理员用的 kube config 配置文件用的，现在我们一般建议使用 RBAC 来对 kubernetes 进行角色权限控制， kubernetes 将证书中的 CN 字段 作为 User，

O 字段作为 Group； "O": "system:masters", 必须是 system:masters，否则后面 kubectl create

clusterrolebinding 报错。

证书 O 配置为 system:masters 在集群内部 cluster-admin 的 clusterrolebinding 将system:masters 组和cluster-admin clusterrole 绑定在一起

#### 2.3.3 生成客户端的证书

```shell
[root@k8s-host ]#  cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin
2025/03/07 17:36:10 [INFO] generate received request
2025/03/07 17:36:10 [INFO] received CSR
2025/03/07 17:36:10 [INFO] generating key: rsa-2048
2025/03/07 17:36:10 [INFO] encoded CSR
2025/03/07 17:36:10 [INFO] signed certificate with serial number 402809480764048501083220148180998524053207784129
2025/03/07 17:36:10 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
```


[root@k8s-host ]# cp admin*.pem /etc/kubernetes/ssl/

#### 2.3.4 配置安全上下文

#创建 kubeconfig 配置文件，比较重要

kubeconfig 为 kubectl 的配置文件，包含访问 apiserver 的所有信息，如 apiserver 地址、 CA 证书和自身使用的证书(这里如果报错找不到 kubeconfig 路径，请手动复制到相应路径下，没有则忽略)

1.设置集群参数

```shell
[root@k8s-host ]# kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=https://bigdata-spark-k8s.idcvdian.com:443 --kubeconfig=kube.config
Cluster "kubernetes" set
```

#查看 kube.config 内容
```shell
[root@k8s-host ]# cat kube.config
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBD...
    server: https://bigdata-spark-k8s.idcvdian.com:443
  name: kubernetes
contexts: null
current-context: ""
kind: Config
preferences: {}
users: null
```

2.设置客户端认证参数

```shell
[root@k8s-host ]# kubectl config set-credentials admin --client-certificate=admin.pem --client-key=admin-key.pem --embed-certs=true --kubeconfig=kube.config
User "admin" set

[root@k8s-host ]# cat kube.config
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBD...
    server: https://bigdata-spark-k8s.idcvdian.com:443
  name: kubernetes
contexts: null
current-context: ""
kind: Config
preferences: {}
users:
- name: admin
  user:
    client-certificate-data: LS0tLS1CRUdJTiBD...
    client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJ...
```


3.设置上下文参数
```shell
[root@k8s-host ]# kubectl config set-context kubernetes --cluster=kubernetes --user=admin --kubeconfig=kube.config
Context "kubernetes" created
```


4.设置当前上下文

```shell
[root@k8s-host ]# kubectl config use-context kubernetes --kubeconfig=kube.config
Switched to context "kubernetes".
[root@k8s-host ]# mkdir ~/.kube -p
[root@k8s-host ]# cp kube.config ~/.kube/config
[root@k8s-host ]# cp kube.config /etc/kubernetes/admin.conf
[root@k8s-host ]# echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile
```


5.授权 kubernetes 证书访问kubelet api 权限
```shell
[root@k8s-host ]# kubectl create clusterrolebinding kube-apiserver:kubelet-apis --clusterrole=system:kubelet-api-admin --user kubernetes
clusterrolebinding.rbac.authorization.k8s.io/kube-apiserver:kubelet-apis created

查看集群组件状态
[root@k8s-host ]# kubectl cluster-info

Kubernetes control plane is running at https://192.168.7.10:6443
To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

执行下面命令查看集群信息
[root@k8s-host ]# kubectl get componentstatuses
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS      MESSAGE              ERROR
controller-manager   Unhealthy   Get "https://127.0.0.1:10257/healthz": dial tcp 127.0.0.1:10257: connect: connection refused   
scheduler            Unhealthy   Get "https://127.0.0.1:10259/healthz": dial tcp 127.0.0.1:10259: connect: connection refused   
etcd-0               Healthy     {"health":"true"}                                                                         
etcd-1               Healthy     {"health":"true"}

[root@k8s-host ]# kubectl get all --all-namespaces
NAMESPACE   NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
default     service/kubernetes   ClusterIP   10.255.0.1   <none>        443/TCP   36m
```


### 2.4 部署 kube-controller-manager 组件
#### 2.4.1 创建 kube-controller-manager csr 请求文件
```shell
[root@k8s-host ]# cat kube-controller-manager-csr.json
{
  "CN": "system:kube-controller-manager",
  "hosts": [
    "10.160.65.28",
    "10.160.69.230",
    "10.160.65.147",
    "bigdata-spark-k8s1.idcvdian.com",
    "bigdata-spark-k8s2.idcvdian.com",
    "bigdata-spark-k8s3.idcvdian.com",
    "10.39.128.64",
    "bigdata-spark-k8s.idcvdian.com",
    "127.0.0.1",
    "10.96.0.1",
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Zhejiang",
      "L": "Hangzhou",
      "O": "system:kube-controller-manager",
      "OU": "system"
    }
  ]
}
```

>- 注： hosts 列表包含所有 kube-controller-manager 节点 IP；

CN 为 system:kube-controller-manager
O 为 system:kube-controller-manager，
kubernetes 内置的 ClusterRoleBindings system:kube-controller-manager 赋予 kube-controller-manager 工作所需的权限

#### 2.4.2 生成 kube-controller-manager证书
```shell
[root@k8s-host ]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
2025/03/10 14:28:11 [INFO] generate received request
2025/03/10 14:28:11 [INFO] received CSR
2025/03/10 14:28:11 [INFO] generating key: rsa-2048
2025/03/10 14:28:11 [INFO] encoded CSR
2025/03/10 14:28:11 [INFO] signed certificate with serial number 98376623204756928012295020163527494055697541890
2025/03/10 14:28:11 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
```

#### 2.4.3 创建 kube-controller-manager 的 kubeconfig

1.设置集群参数
```shell
[root@k8s-host ]# kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=https://bigdata-spark-k8s.idcvdian.com:443 --kubeconfig=kube-controller-manager.kubeconfig
Cluster "kubernetes" set

[root@k8s-host ]# cat kube-controller-manager.kubeconfig
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CR...
    server: https://bigdata-spark-k8s.idcvdian.com:443
  name: kubernetes
contexts: null
current-context: ""
kind: Config
preferences: {}
users: null
```

2.设置客户端认证参数
```shell
[root@k8s-host ]# kubectl config set-credentials system:kube-controller-manager --client-certificate=kube-controller-manager.pem --client-key=kube-controller-manager-key.pem --embed-certs=true --kubeconfig=kube-controller-manager.kubeconfig
User "system:kube-controller-manager" set.

@#cat kube-controller-manager.kubeconfig
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1...
    server: https://bigdata-spark-k8s.idcvdian.com:443
  name: kubernetes
contexts: null
current-context: ""
kind: Config
preferences: {}
users:
- name: system:kube-controller-manager
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRV...
    client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJV...
```

3.设置上下文参数
```shell
[root@k8s-host ]# kubectl config set-context system:kube-controller-manager --cluster=kubernetes --user=system:kube-controller-manager --kubeconfig=kube-controller-manager.kubeconfig
Context "system:kube-controller-manager" created.
```

4.设置当前上下文
```shell
[root@k8s-host ]# kubectl config use-context system:kube-controller-manager --kubeconfig=kube-controller-manager.kubeconfig
Switched to context "system:kube-controller-manager"
```

#### 2.4.4 创建kube-controller-manager配置文件
```shell
[root@k8s-host ]# cat kube-controller-manager.conf
KUBE_CONTROLLER_MANAGER_OPTS="--port=0 \
  --secure-port=10257 \
  --bind-address=10.160.65.28 \
  --kubeconfig=/etc/kubernetes/kube-controller-manager.kubeconfig \
  --service-cluster-ip-range=10.96.0.0/12 \
  --cluster-name=kubernetes \
  --cluster-signing-cert-file=/etc/kubernetes/ssl/ca.pem \
  --cluster-signing-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --allocate-node-cidrs=true \
  --cluster-cidr=10.0.0.0/16 \
  --experimental-cluster-signing-duration=876000h \
  --root-ca-file=/etc/kubernetes/ssl/ca.pem \
  --service-account-private-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --leader-elect=true \
  --feature-gates=RotateKubeletServerCertificate=true \
  --controllers=*,bootstrapsigner,tokencleaner \
  --horizontal-pod-autoscaler-sync-period=10s \
  --tls-cert-file=/etc/kubernetes/ssl/kube-controller-manager.pem \
  --tls-private-key-file=/etc/kubernetes/ssl/kube-controller-manager-key.pem \
  --use-service-account-credentials=true \
  --alsologtostderr=true \
  --logtostderr=false \
  --log-dir=/var/log/kubernetes \
  --kube-api-qps=500 \
  --kube-api-burst=1500 \
  --endpoint-updates-batch-period=10s \
  --large-cluster-size-threshold=200 \
  --secondary-node-eviction-rate=0 \
  --v=2"
```

#### 2.4.5 创建kube-controller-manager启动文件
```shell
[root@k8s-host ]# cat kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
After=kube-apiserver.service
Requires=kube-apiserver.service
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=/etc/kubernetes/kube-controller-manager.conf
ExecStart=/usr/bin/kube-controller-manager $KUBE_CONTROLLER_MANAGER_OPTS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

#### 2.4.6 启动 kube-controller-manager服务
```shell
[root@k8s-host ]# cp kube-controller-manager*.pem /etc/kubernetes/ssl/
[root@k8s-host ]# cp kube-controller-manager.kubeconfig /etc/kubernetes/
[root@k8s-host ]# cp kube-controller-manager.conf /etc/kubernetes/
[root@k8s-host ]# cp kube-controller-manager.service /usr/lib/systemd/system/

[root@k8s-host ]# systemctl daemon-reload  &&systemctl enable kube-controller-manager && systemctl start kube-controller-manager && systemctl status kube-controller-manager
```

其他节点类似,这里不赘述...

### 2.5 部署 kube-scheduler 组件
#### 2.5.1 创建kube-scheduler的csr 请求

```shell
[root@k8s-host ]# cat kube-scheduler-csr.json
{
  "CN": "system:kube-scheduler",
  "hosts": [
    "10.160.65.28",
    "10.160.69.230",
    "10.160.65.147",
    "bigdata-spark-k8s1.idcvdian.com",
    "bigdata-spark-k8s2.idcvdian.com",
    "bigdata-spark-k8s3.idcvdian.com",
    "10.39.128.64",
    "bigdata-spark-k8s.idcvdian.com",
    "127.0.0.1",
    "10.96.0.1",
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Zhejiang",
      "L": "Hangzhou",
      "O": "system:kube-scheduler",
      "OU": "system"
    }
  ]
}
```

注： hosts 列表包含所有 kube-scheduler 节点 IP； CN 为 system:kube-scheduler、O 为 system:kube-scheduler，kubernetes 内置的 ClusterRoleBindings system:kube-scheduler 将赋予  kube-scheduler 工作所需的权限。

#### 2.5.2 生成kube-scheduler证书
```shell
[root@k8s-host ]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-scheduler-csr.json | cfssljson -bare kube-scheduler
2025/03/10 15:33:27 [INFO] generate received request
2025/03/10 15:33:27 [INFO] received CSR
2025/03/10 15:33:27 [INFO] generating key: rsa-2048
2025/03/10 15:33:27 [INFO] encoded CSR
2025/03/10 15:33:27 [INFO] signed certificate with serial number 335995241342672053519971700060333494482582643094
2025/03/10 15:33:27 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
```

#### 2.5.3 创建 kube-scheduler 的 kubeconfig文件

1.设置集群参数
```shell
[root@k8s-host ]# kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=https://bigdata-spark-k8s.idcvdian.com:443 --kubeconfig=kube-scheduler.kubeconfig
Cluster "kubernetes" set

[root@k8s-host ]# cat kube-scheduler.kubeconfig
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CR...
    server: https://bigdata-spark-k8s.idcvdian.com:443
  name: kubernetes
contexts: null
current-context: ""
kind: Config
preferences: {}
users: null
```

2.设置客户端认证参数
```shell
[root@k8s-host ]# kubectl config set-credentials system:kube-scheduler --client-certificate=kube-scheduler.pem --client-key=kube-scheduler-key.pem --embed-certs=true --kubeconfig=kube-scheduler.kubeconfig
User "system:kube-scheduler" set.

[root@k8s-host ]# cat kube-scheduler.kubeconfig
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS...
    server: https://bigdata-spark-k8s.idcvdian.com:443
  name: kubernetes
contexts: null
current-context: ""
kind: Config
preferences: {}
users:
- name: system:kube-scheduler
  user:
    client-certificate-data: LS0tLS1CRUdJT...
    client-key-data: LS0tLS1CRUdJTiBSU0Eg...
```

3.设置上下文参数
```shell
[root@k8s-host ]# kubectl config set-context system:kube-scheduler --cluster=kubernetes --user=system:kube-scheduler --kubeconfig=kube-scheduler.kubeconfig
Context "system:kube-scheduler" created
```

4.设置当前上下文
```shell
[root@k8s-host ]# kubectl config use-context system:kube-scheduler --kubeconfig=kube-scheduler.kubeconfig
Switched to context "system:kube-scheduler"
```

#### 2.5.4 创建配置文件 kube-scheduler的配置文件
```shell
[root@k8s-host ]# cat kube-scheduler.conf
KUBE_SCHEDULER_OPTS="--address=10.160.65.28 \
--kubeconfig=/etc/kubernetes/kube-scheduler.kubeconfig \
--leader-elect=true \
--alsologtostderr=true \
--logtostderr=false \
--log-dir=/var/log/kubernetes \
--v=2"
```

#### 2.5.5 创建kube-scheduler的服务启动文件
```shell
[root@k8s-host ]# cat kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=-/etc/kubernetes/kube-scheduler.conf
ExecStart=/usr/bin/kube-scheduler $KUBE_SCHEDULER_OPTS
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

#### 2.5.6 拷贝文件到master2节点并启动服务
```shell
[root@k8s-host ]# cp kube-scheduler*.pem /etc/kubernetes/ssl/
[root@k8s-host ]# cp kube-scheduler.kubeconfig /etc/kubernetes/
[root@k8s-host ]# cp kube-scheduler.conf /etc/kubernetes/
[root@k8s-host ]# cp kube-scheduler.service /usr/lib/systemd/system/
[root@k8s-host ]# systemctl daemon-reload &&  systemctl enable kube-scheduler && systemctl start kube-scheduler && systemctl status kube-scheduler
[root@k8s-master2]# systemctl daemon-reload && systemctl enable kube-scheduler && systemctl start kube-scheduler && systemctl status kube-scheduler
```

### 2.6 部署docker组件
注:这里docker使用的devicemapper存储模式,另外镜像也是本地harbor仓库拉好的
```shell
apt-get remove -y docker-ce docker-ce-cli containerd.io 1>/dev/null 2>&1
    rm -rf /var/lib/docker
    wget -P /tmp/ http://apt.idcvdian.com/k8s/docker-ce_20.10.13~3-0~ubuntu-jammy_amd64.deb
    wget -P /tmp/ http://apt.idcvdian.com/k8s/docker-ce-cli_20.10.17~3-0~ubuntu-jammy_amd64.deb
    wget -P /tmp/ http://apt.idcvdian.com/k8s/containerd.io_1.6.9-1_amd64.deb
    wget -P /tmp/ http://apt.idcvdian.com/k8s/docker-scan-plugin_0.17.0~ubuntu-jammy_amd64.deb
    dpkg -i /tmp/containerd.io_1.6.9-1_amd64.deb
    dpkg -i /tmp/docker-ce-cli_20.10.17~3-0~ubuntu-jammy_amd64.deb
    dpkg -i /tmp/docker-scan-plugin_0.17.0~ubuntu-jammy_amd64.deb
    dpkg -i /tmp/docker-ce_20.10.13~3-0~ubuntu-jammy_amd64.deb
    mkdir -p /etc/docker
    cat <<EOF > /etc/docker/daemon.json
{
    "storage-driver": "devicemapper",
    "insecure-registries":["nexus.vdian.net:8082","nexus.vdian.net:8083","nexus.vdian.net:8081","harbor.vdian.net"],
    "storage-opts": [
      "dm.directlvm_device=/dev/$device",
      "dm.thinp_percent=95",
      "dm.thinp_metapercent=1",
      "dm.thinp_autoextend_threshold=80",
      "dm.thinp_autoextend_percent=20",
      "dm.directlvm_device_force=false",
      "dm.basesize=20G"
    ],
    "exec-opts": ["native.cgroupdriver=systemd"],
    "data-root": "/data/docker",
	   "log-driver": "json-file",
	     "log-opts": {
	       "max-size": "100m",
	       "max-file": "3"
	     }  
}
EOF
    systemctl daemon-reload 1>/dev/null 2>&1
    systemctl restart docker 1>/dev/null 2>&1
    systemctl enable docker 1>/dev/null 2>&1
    #安装完通过docker images确保以下镜像已经存在,不然docker调度上来再通过公网拉会很慢
    #异常情况docker启动失败这步会失败
    docker pull harbor.vdian.net/k8s-base/pause:3.1 1>/dev/null 2>&1
    docker tag harbor.vdian.net/k8s-base/pause:3.1 k8s.gcr.io/pause:3.1 1>/dev/null 2>&1
    docker rmi harbor.vdian.net/k8s-base/pause:3.1 1>/dev/null 2>&1
    docker pull harbor.vdian.net/library/prom/node-exporter 1>/dev/null 2>&1
    docker tag harbor.vdian.net/library/prom/node-exporter prom/node-exporter 1>/dev/null 2>&1
    docker rmi harbor.vdian.net/library/prom/node-exporter 1>/dev/null 2>&1
```

### 2.7 部署 kubelet 组件
kubelet： 每个 Node 节点上的 kubelet 定期就会调用 API Server 的 REST 接口报告自身状态， API Server 接收这些信息后，将节点状态信息更新到 etcd 中。kubelet 也通过 API Server 监听 Pod信息，从而对 Node 机器上的 POD 进行管理，如创建、删除、更新 Pod

#### 2.7.1 创建 kubelet-bootstrap.kubeconfig
```shell
[root@k8s-host ]# BOOTSTRAP_TOKEN=$(awk -F "," '{print $1}' /etc/kubernetes/token.csv)
[root@k8s-host ]# kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=https://bigdata-spark-k8s.idcvdian.com:443 --kubeconfig=kubelet-bootstrap.kubeconfig
Cluster "kubernetes" set.
[root@k8s-host ]# cat kubelet-bootstrap.kubeconfig
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1...
    server: https://bigdata-spark-k8s.idcvdian.com:443
  name: kubernetes
contexts: null
current-context: ""
kind: Config
preferences: {}
users: null

[root@k8s-host ]# kubectl config set-credentials kubelet-bootstrap --token=${BOOTSTRAP_TOKEN} --kubeconfig=kubelet-bootstrap.kubeconfig
[root@k8s-host ]# cat kubelet-bootstrap.kubeconfig
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1C...
    server: https://bigdata-spark-k8s.idcvdian.com:443
  name: kubernetes
contexts: null
current-context: ""
kind: Config
preferences: {}
users:
- name: kubelet-bootstrap
  user:
    token: 6b26208b9f1bf17912afb2dc0ac4c8d1

[root@k8s-host ]# kubectl config set-context default --cluster=kubernetes --user=kubelet-bootstrap --kubeconfig=kubelet-bootstrap.kubeconfig
[root@k8s-host ]# cat kubelet-bootstrap.kubeconfig
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiB...
    server: https://bigdata-spark-k8s.idcvdian.com:443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubelet-bootstrap
  name: default
current-context: ""
kind: Config
preferences: {}
users:
- name: kubelet-bootstrap
  user:
    token: 6b26208b9f1bf17912afb2dc0ac4c8d1
[root@k8s-host ]# kubectl config use-context default --kubeconfig=kubelet-bootstrap.kubeconfig
[root@k8s-host ]# kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap
```

#### 2.7.2 创建配置文件 kubelet.json
"cgroupDriver": "systemd"要和 docker 的驱动一致。

address 替换为自己 k8s-node1 的 IP 地址。
```shell
[root@k8s-host ]# cat kubelet.json
{
  "kind": "KubeletConfiguration",
  "apiVersion": "kubelet.config.k8s.io/v1beta1",
  "authentication": {
    "x509": {
      "clientCAFile": "/etc/kubernetes/ssl/ca.pem"
    },
    "webhook": {
      "enabled": true,
      "cacheTTL": "2m0s"
    },
    "anonymous": {
      "enabled": false
    }
  },
  "authorization": {
    "mode": "Webhook",
    "webhook": {
      "cacheAuthorizedTTL": "5m0s",
      "cacheUnauthorizedTTL": "30s"
    }
  },
  "address": "10.16.6.158",
  "port": 10250,
  "readOnlyPort": 10255,
  "cgroupDriver": "systemd",
  "hairpinMode": "promiscuous-bridge",
  "serializeImagePulls": false,
  "featureGates": {
    "RotateKubeletServerCertificate": true
  }
}
```

#### 2.7.3 创建kubelet服务启动文件
```shell
[root@k8s-host ]# cat kubelet.service
[Unit]
Description=kubelet: The Kubernetes Node Agent
Documentation=https://kubernetes.io/docs/
After=docker.service
Requires=docker.service
[Service]
ExecStartPre=/usr/bin/mkdir -p /sys/fs/cgroup/pids/system.slice
ExecStartPre=/usr/bin/mkdir -p /sys/fs/cgroup/cpu/system.slice
ExecStartPre=/usr/bin/mkdir -p /sys/fs/cgroup/cpuacct/system.slice
ExecStartPre=/usr/bin/mkdir -p /sys/fs/cgroup/cpuset/system.slice
ExecStartPre=/usr/bin/mkdir -p /sys/fs/cgroup/memory/system.slice
ExecStartPre=/usr/bin/mkdir -p /sys/fs/cgroup/devices/system.slice
ExecStartPre=/usr/bin/mkdir -p /sys/fs/cgroup/blkio/system.slice
ExecStartPre=/usr/bin/mkdir -p /sys/fs/cgroup/hugetlb/system.slice
ExecStartPre=/usr/bin/mkdir -p /sys/fs/cgroup/systemd/system.slice
ExecStartPre=/usr/bin/mkdir -p /sys/fs/cgroup/pids/kubelet.slice
ExecStartPre=/usr/bin/mkdir -p /sys/fs/cgroup/cpu/kubelet.slice
ExecStartPre=/usr/bin/mkdir -p /sys/fs/cgroup/cpuacct/kubelet.slice
ExecStartPre=/usr/bin/mkdir -p /sys/fs/cgroup/cpuset/kubelet.slice
ExecStartPre=/usr/bin/mkdir -p /sys/fs/cgroup/memory/kubelet.slice
ExecStartPre=/usr/bin/mkdir -p /sys/fs/cgroup/devices/kubelet.slice
ExecStartPre=/usr/bin/mkdir -p /sys/fs/cgroup/blkio/kubelet.slice
ExecStartPre=/usr/bin/mkdir -p /sys/fs/cgroup/hugetlb/kubelet.slice
WorkingDirectory=/var/lib/kubelet

ExecStart=/usr/bin/kubelet \
   --hostname-override=10.16.6.158 \
   --bootstrap-kubeconfig=/etc/kubernetes/kubelet-bootstrap.kubeconfig \
   --cert-dir=/etc/kubernetes/ssl \
   --kubeconfig=/etc/kubernetes/kubelet-bootstrap.kubeconfig \
   --config=/etc/kubernetes/kubelet.json \
   --network-plugin=cni \
   --cni-conf-dir=/etc/cni/net.d \
   --cni-bin-dir=/opt/cni/bin \
   --image-pull-progress-deadline=5m0s \
   --root-dir=/data/docker/kubelet \
   --resolv-conf=/etc/resolv.conf \
   --serialize-image-pulls=false \
   --max-pods=35 \
   --alsologtostderr=true \
   --logtostderr=false \
   --log-dir=/var/log/kubernetes \
   --v=2

Restart=always
StartLimitInterval=0
RestartSec=10
[Install]
WantedBy=multi-user.target


#注： –hostname-override：显示名称，集群中唯一
–network-plugin：启用 CNI
–kubeconfig：空路径，会自动生成，后面用于连接 apiserver
–bootstrap-kubeconfig：首次启动向 apiserver 申请证书
–config：配置参数文件
–cert-dir：kubelet 证书生成目录
#注：kubelete.json 配置文件 address 改为各个节点的 ip 地址，在各个 work 节点上启动服务
```

#### 2.7.4 拷贝kubelet的证书及配置文件到node节点
```shell
[root@k8s-node1 ~]# mkdir /etc/kubernetes/ssl -p
[root@k8s-host ]# scp kubelet-bootstrap.kubeconfig kubelet.json k8s-node1:/etc/kubernetes/
[root@k8s-host ]# scp ca.pem k8s-node1:/etc/kubernetes/ssl/
[root@k8s-host ]# scp  kubelet.service k8s-node1:/usr/lib/systemd/system/
```

#### 2.7.5 在node节点上启动 kubelet 服务
```shell
[root@k8s-node1 ~]# mkdir /var/log/kubernetes
[root@k8s-node1 ~]# systemctl daemon-reload && systemctl enable kubelet &&  systemctl start kubelet && systemctl status kubelet
```

确认 kubelet 服务启动成功后，接着到 k8s-master1 节点上 Approve 一下 bootstrap 请求。
执行如下命令可以看到一个 worker 节点发送了一个 CSR 请求：
```shell
[root@k8s-host ]# kubectl get csr
NAME                                                   AGE   SIGNERNAME                                    REQUESTOR           REQUESTEDDURATION   CONDITION
node-csr-UIwBToi0tiFkb6wLdk81y4LC_rVjSiUpDr3qCZBNDo0   8s    kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   <none>              Pending
```

#### 2.7.6 在master节点审批node节点的请求
```shell
[root@k8s-host ]# kubectl certificate approve node-csr-UIwBToi0tiFkb6wLdk81y4LC_rVjSiUpDr3qCZBNDo0
certificatesigningrequest.certificates.k8s.io/node-csr-UIwBToi0tiFkb6wLdk81y4LC_rVjSiUpDr3qCZBNDo0 approved
```

# 再次查看申请
```shell
[root@k8s-host ]# kubectl get csr
NAME                                                   AGE   SIGNERNAME                                    REQUESTOR           REQUESTEDDURATION   CONDITION
node-csr-UIwBToi0tiFkb6wLdk81y4LC_rVjSiUpDr3qCZBNDo0   67s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   <none>              Approved,Issued
```

在master上查看一下node节点是否已经正常加入进来了

```shell
[root@k8s-host ]# kubectl get node
NAME          STATUS   ROLES    AGE   VERSION
10.16.6.158   Ready    <none>   50s   v1.23.2
```

#### 2.7.7 安装网络插件

这里用的macvlan,每个公司所用的网络模式有区别。需要自行考虑

安装cni插件

```shell
[root@k8s-host ]# rpm -qa |grep cni
kubernetes-cni-0.6.0-0.x86_64

安装完就是一些二进制文件
[root@k8s-host ]# ls /opt/cni/bin/
bridge  dhcp  flannel  host-local  ipvlan  loopback  macvlan  portmap  ptp  sample  tuning  vlan
```

配置macvlan
```shell
[root@k8s-host ]# cat /etc/cni/net.d/10-maclannet.conf
{
    "cniVersion":"0.3.1",
    "name": "macvlannet",
    "type": "macvlan",
    "master": "bond0",
    "isGateway": true,
    "ipMasq": false,
    "ipam": {
        "type": "host-local",
        "subnet": "10.160.64.0/19",
        "rangeStart": "10.160.68.171",
        "rangeEnd": "10.160.68.250",
        "gateway": "10.160.64.1",
        "routes": [
            { "dst": "0.0.0.0/0" }
        ]
    }
}
```

### 2.8 部署 kube-proxy 组件
#### 2.8.1 创建kube-proxy的 csr 请求
[root@k8s-host ]# cat kube-proxy-csr.json
```shell
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Zhejiang",
      "L": "Hangzhou",
      "O": "system:kube-proxy",
      "OU": "system"
    }
  ]
}
```

#### 2.7.2 生成证书
```shell
[root@k8s-host ]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy
2025/03/10 17:11:25 [INFO] generate received request
2025/03/10 17:11:25 [INFO] received CSR
2025/03/10 17:11:25 [INFO] generating key: rsa-2048
2025/03/10 17:11:25 [INFO] encoded CSR
2025/03/10 17:11:25 [INFO] signed certificate with serial number 141280875917381566079599045896645784629924909508
2025/03/10 17:11:25 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
```

#### 2.7.3 创建 kubeconfig 文件
```shell
[root@k8s-host ]# kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=https://bigdata-spark-k8s.idcvdian.com:443 --kubeconfig=kube-proxy.kubeconfig
[root@k8s-host ]# cat kube-proxy.kubeconfig
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRU...
    server: https://bigdata-spark-k8s.idcvdian.com:443
  name: kubernetes
contexts: null
current-context: ""
kind: Config
preferences: {}
users: null
[root@k8s-host ]# kubectl config set-credentials kube-proxy --client-certificate=kube-proxy.pem --client-key=kube-proxy-key.pem --embed-certs=true --kubeconfig=kube-proxy.kubeconfig
[root@k8s-host ]# cat kube-proxy.kubeconfig
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJT...
    server: https://bigdata-spark-k8s.idcvdian.com:443
  name: kubernetes
contexts: null
current-context: ""
kind: Config
preferences: {}
users:
- name: kube-proxy
  user:
    client-certificate-data: LS0tLS1CRUdJTiB...
    client-key-data: LS0tLS1CRUd..
[root@k8s-host ]# kubectl config set-context default --cluster=kubernetes --user=kube-proxy --kubeconfig=kube-proxy.kubeconfig
[root@k8s-host ]# kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

#### 2.7.4 创建 kube-proxy 配置文件
```shell
[root@k8s-host ]# cat kube-proxy.yaml
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 10.16.6.158
clientConnection:
  kubeconfig: /etc/kubernetes/kube-proxy.kubeconfig
clusterCIDR: 172.1.0.0/16
healthzBindAddress: 10.16.6.158:10256
kind: KubeProxyConfiguration
metricsBindAddress: 10.16.6.158:10249
mode: "ipvs"
```

#### 2.7.5 创建kube-proxy服务启动文件
```shell
[root@k8s-host ]# cat kube-proxy.service
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
WorkingDirectory=/var/lib/kube-proxy
ExecStart=/usr/bin/kube-proxy \
--config=/etc/kubernetes/kube-proxy.yaml \
--alsologtostderr=true \
--logtostderr=false \
--log-dir=/var/log/kubernetes \
--v=2
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

#### 2.7.6 拷贝kube-proxy文件到node节点上
```shell
[root@k8s-host ]# scp kube-proxy.kubeconfig kube-proxy.yaml 10.16.6.158:/etc/kubernetes/
[root@k8s-host ]# /root]# work]#scp  kube-proxy.service 10.16.6.158:/usr/lib/systemd/system/
```

#### 2.7.7 启动kube-proxy服务
```shell
[root@k8s-node1 ~]# mkdir -p /var/lib/kube-proxy
[root@k8s-node1 ~]# systemctl daemon-reload && systemctl enable kube-proxy && systemctl start kube-proxy && systemctl status kube-proxy
```

## END
```shell
[root@k8s-host ]# kubectl get node
NAME          STATUS   ROLES    AGE   VERSION
10.16.6.158   Ready    <none>   55m   v1.23.2
[root@k8s-host ]# kubectl get all --all-namespaces
NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   3d2h
```
