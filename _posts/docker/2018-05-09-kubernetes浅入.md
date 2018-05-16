---
layout: post
title: kubernetes浅入(一)
categories: [Docker, Kubernetes]
description: 浅入kubernetes,基本的安装和使用
keywords: kubernetes安装,kubernetes浅入
---

---

公司需要搭建一个k8s集群用于爬虫快捷部署,临时看了mritd[](https://mritd.me/2018/04/19/set-up-kubernetes-1.10.1-cluster-by-hyperkube/)的搭建教程,基本算是可以使用了.

本文仅做个人笔记使用,搭建过程参考自:[mritd的博文](https://mritd.me/2018/04/19/set-up-kubernetes-1.10.1-cluster-by-hyperkube/){:target="_blank"}

为了快速搭建,本次搭建使用了mritd的脚本

---

## 系统和软件
1. Ubuntu16.04 server
2. Kubernetes1.10.1
3. Docker 18.03.1-ce

## 软件和背景
1. 需要提前安装docker和docker-compose
2. etcd [etcd简介参考文档](https://yq.aliyun.com/articles/11035) ETCD是用于共享配置和服务发现的分布式，一致性的KV存储系统。
3. cfssl 用于生成ectd证书[GitHub官网下载](https://github.com/cloudflare/cfssl/releases)
4. hyperkube [官方介绍](https://github.com/kubernetes/kubernetes/blob/master/cluster/images/hyperkube/README.md)(用于安装kubelet)
5. 本文使用4台机子搭建(ip地址根据自己需要修改)


| IP   | TYPE  |
| ------ | ------ |
| 192.168.1.30  | master node etcd  | 
| 192.168.1.31  | node etcd  | 
| 192.168.1.32  | node etcd  | 
| 192.168.1.33  | node   | 



## 开始搭建 
切换到root用户,在root家目录新建个文件夹用户存放后续需要用到的文件和配置,

记得对从节点配置ssh免登陆

我新建的目录为 k8s,后续安装不特别说明都基于 /root/k8s 此目录

先查看安装目录结构<br>
![](https://stone-upyun.b0.upaiyun.com/blog20180509130102.png!700x999)
<br>
### 一 安装cfssl
使用mritd的cnd下载二进制安装包,如果不能用,可到[GitHub下载](https://github.com/cloudflare/cfssl/releases)

#### 1.1 安装

```
wget https://mritdftp.b0.upaiyun.com/cfssl/cfssl.tar.gz
tar -zxvf cfssl.tar.gz
mv cfssl cfssljson /usr/local/bin
chmod +x /usr/local/bin/cfssl /usr/local/bin/cfssljson
rm -f cfssl.tar.gz
```
#### 1.2 生成etcd证书
mkdir ssl 
cd ssl
##### etcd-csr.json
修改hosts相关配置为自己的ip
```
{
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "O": "etcd",
      "OU": "etcd Security",
      "L": "Beijing",
      "ST": "Beijing",
      "C": "CN"
    }
  ],
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "localhost",
    "192.168.1.61",
    "192.168.1.62",
    "192.168.1.63"
  ]
}

```

##### etcd-gencert.json
```
{
  "signing": {
    "default": {
        "usages": [
          "signing",
          "key encipherment",
          "server auth",
          "client auth"
        ],
        "expiry": "87600h"
    }
  }
}
```

##### etcd-root-ca-csr.json
```
{
  "key": {
    "algo": "rsa",
    "size": 4096
  },
  "names": [
    {
      "O": "etcd",
      "OU": "etcd Security",
      "L": "Beijing",
      "ST": "Beijing",
      "C": "CN"
    }
  ],
  "CN": "etcd-root-ca"
}
```

##### 生成证书
```
cfssl gencert --initca=true etcd-root-ca-csr.json | cfssljson --bare etcd-root-ca
cfssl gencert --ca etcd-root-ca.pem --ca-key etcd-root-ca-key.pem --config etcd-gencert.json etcd-csr.json | cfssljson --bare etcd
```
生成后结果如下<br>
![](https://stone-upyun.b0.upaiyun.com/blog20180509144247.png!700x999)
<br>

### 二 安装etcd
Etcd 这里采用最新的 3.2.18 版本，安装方式直接复制二进制文件、systemd service 配置
cd ~/k8s/
mkdir systemd && cd systemd
touch etcd.service
#### etcd.service
```
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
EnvironmentFile=-/etc/etcd/etcd.conf
User=etcd
## set GOMAXPROCS to number of processors
ExecStart=/bin/bash -c "GOMAXPROCS=$(nproc) /usr/local/bin/etcd --name=\"${ETCD_NAME}\" --data-dir=\"${ETCD_DATA_DIR}\" --listen-client-urls=\"${ETCD_LISTEN_CLIENT_URLS}\""
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

```

touch etcd.conf
#### etcd.conf
```
## [member]
ETCD_NAME=etcd1
ETCD_DATA_DIR="/var/lib/etcd/etcd1.etcd"
ETCD_WAL_DIR="/var/lib/etcd/wal"
ETCD_SNAPSHOT_COUNT="100"
ETCD_HEARTBEAT_INTERVAL="100"
ETCD_ELECTION_TIMEOUT="1000"
ETCD_LISTEN_PEER_URLS="https://192.168.1.61:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.1.61:2379,http://127.0.0.1:2379"
ETCD_MAX_SNAPSHOTS="5"
ETCD_MAX_WALS="5"
#ETCD_CORS=""

## [cluster]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.1.61:2380"
## if you use different ETCD_NAME (e.g. test), set ETCD_INITIAL_CLUSTER value for this name, i.e. "test=http://..."
ETCD_INITIAL_CLUSTER="etcd1=https://192.168.1.61:2380,etcd2=https://192.168.1.62:2380,etcd3=https://192.168.1.63:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.1.61:2379"
#ETCD_DISCOVERY=""
#ETCD_DISCOVERY_SRV=""
#ETCD_DISCOVERY_FALLBACK="proxy"
#ETCD_DISCOVERY_PROXY=""
#ETCD_STRICT_RECONFIG_CHECK="false"
#ETCD_AUTO_COMPACTION_RETENTION="0"

## [proxy]
#ETCD_PROXY="off"
#ETCD_PROXY_FAILURE_WAIT="5000"
#ETCD_PROXY_REFRESH_INTERVAL="30000"
#ETCD_PROXY_DIAL_TIMEOUT="1000"
#ETCD_PROXY_WRITE_TIMEOUT="5000"
#ETCD_PROXY_READ_TIMEOUT="0"

## [security]
ETCD_CERT_FILE="/etc/etcd/ssl/etcd.pem"
ETCD_KEY_FILE="/etc/etcd/ssl/etcd-key.pem"
ETCD_CLIENT_CERT_AUTH="true"
ETCD_TRUSTED_CA_FILE="/etc/etcd/ssl/etcd-root-ca.pem"
ETCD_AUTO_TLS="true"
ETCD_PEER_CERT_FILE="/etc/etcd/ssl/etcd.pem"
ETCD_PEER_KEY_FILE="/etc/etcd/ssl/etcd-key.pem"
ETCD_PEER_CLIENT_CERT_AUTH="true"
ETCD_PEER_TRUSTED_CA_FILE="/etc/etcd/ssl/etcd-root-ca.pem"
ETCD_PEER_AUTO_TLS="true"

## [logging]
#ETCD_DEBUG="false"
## examples for -log-package-levels etcdserver=WARNING,security=DEBUG
#ETCD_LOG_PACKAGE_LEVELS=""
```

touch install.sh
#### install .sh
这里解释下,此处下载etcd安装包,为系统添加etcd用户,并赋值etcd用户权限,安装下载来的安装包,拷贝相关证书到/etc/etcd/目录中给etcd使用
```
#!/bin/bash

set -e

ETCD_VERSION="3.2.18"
#下载etcd安装包
function download(){
    if [ ! -f "etcd-v${ETCD_VERSION}-linux-amd64.tar.gz" ]; then
        wget https://github.com/coreos/etcd/releases/download/v${ETCD_VERSION}/etcd-v${ETCD_VERSION}-linux-amd64.tar.gz
        tar -zxvf etcd-v${ETCD_VERSION}-linux-amd64.tar.gz
    fi
}

## 添加ectd相关用户和权限
function preinstall(){
    getent group etcd >/dev/null || groupadd -r etcd
    getent passwd etcd >/dev/null || useradd -r -g etcd -d /var/lib/etcd -s /sbin/nologin -c "etcd user" etcd
}

function install(){
    echo -e "\033[32mINFO: Copy etcd...\033[0m"
    tar -zxvf etcd-v${ETCD_VERSION}-linux-amd64.tar.gz
    cp etcd-v${ETCD_VERSION}-linux-amd64/etcd* /usr/local/bin
    rm -rf etcd-v${ETCD_VERSION}-linux-amd64

    echo -e "\033[32mINFO: Copy etcd config...\033[0m"
    cp -r conf /etc/etcd
    chown -R etcd:etcd /etc/etcd
    chmod -R 755 /etc/etcd/ssl

    echo -e "\033[32mINFO: Copy etcd systemd config...\033[0m"
    cp systemd/*.service /lib/systemd/system
    systemctl daemon-reload
}

function postinstall(){
    if [ ! -d "/var/lib/etcd" ]; then
        mkdir /var/lib/etcd
        chown -R etcd:etcd /var/lib/etcd
    fi
}


download
preinstall
install
postinstall
```

- download : 从GitHub官网下载二进制文件并解压
- preinstall : 创建etcd用户,并制定家目录登录shell,为Etcd做准备
- install :  将下载来的二进制压缩包解压并复制到/usr/local/bin,后到conf中的文件复制到 /etc/etcd
- postinstall : 安装后收尾工作，比如检测 /var/lib/etcd 是否存在，纠正权限等

执行 ./install.sh,得到如下结果:<br>
![](https://stone-upyun.b0.upaiyun.com/blog20180510110948.png!700x999)

<font color='red'>注意:完后install后,将etcd目录scp到每台需要安装etcd的节点,修改etcd.conf 中的相关配置(ETCD_NAME,IP等),在每个etcd再执行install.sh </font>

### 三 安装kubernetes
由于 kubelet 和 kube-proxy 用到的 kubeconfig 配置文件需要借助 kubectl 来生成，所以需要先安装一下 kubect
```
wget https://storage.googleapis.com/kubernetes-release/release/v1.10.1/bin/linux/amd64/hyperkube -O hyperkube_1.10.1
chmod +x hyperkube_1.10.1
cp hyperkube_1.10.1 /usr/local/bin/hyperkube
ln -s /usr/local/bin/hyperkube /usr/local/bin/kubectl
```
#### 3.1生成k8s证书
##### admin-csr.json
```
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
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
```

##### k8s-gencert.json
```
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 4096
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```

##### kube-apiserver-csr.json
注意修改为自己的ip,10.254.0.1这个ip我暂时不听清楚具体是啥,貌似是k8s集群里类似路由地址的东西
```
{
    "CN": "kubernetes",
    "hosts": [
        "127.0.0.1",
        "10.254.0.1",
        "192.168.1.30",
        "192.168.1.31",
        "192.168.1.32",
        "192.168.1.33",
        "*.kubernetes.master",
        "localhost",
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
            "ST": "BeiJing",
            "L": "BeiJing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
```

##### kube-proxy-csr.json
```
{
  "CN": "system:kube-proxy",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```

##### 生成证书操作:
可将此脚本写到一个 sh中,然后执行
```
## 生成 CA
cfssl gencert --initca=true k8s-root-ca-csr.json | cfssljson --bare k8s-root-ca

## 依次生成其他组件证书
for targetName in kube-apiserver admin kube-proxy; do
    cfssl gencert --ca k8s-root-ca.pem --ca-key k8s-root-ca-key.pem --config k8s-gencert.json --profile kubernetes $targetName-csr.json | cfssljson --bare $targetName
done

## 地址默认为 127.0.0.1:6443
## 如果在 master 上启用 kubelet 请在生成后的 kubeconfig 中
## 修改该地址为 当前MASTER_IP:6443
KUBE_APISERVER="https://127.0.0.1:6443"
BOOTSTRAP_TOKEN=$(head -c 16 /dev/urandom | od -An -t x | tr -d ' ')
echo "Tokne: ${BOOTSTRAP_TOKEN}"

## 不要质疑 system:bootstrappers 用户组是否写错了，有疑问请参考官方文档
## https://kubernetes.io/docs/admin/kubelet-tls-bootstrapping/
cat > token.csv <<EOF
${BOOTSTRAP_TOKEN},kubelet-bootstrap,10001,"system:bootstrappers"
EOF

echo "Create kubelet bootstrapping kubeconfig..."
## 设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=k8s-root-ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=bootstrap.kubeconfig
## 设置客户端认证参数
kubectl config set-credentials kubelet-bootstrap \
  --token=${BOOTSTRAP_TOKEN} \
  --kubeconfig=bootstrap.kubeconfig
## 设置上下文参数
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kubelet-bootstrap \
  --kubeconfig=bootstrap.kubeconfig
## 设置默认上下文
kubectl config use-context default --kubeconfig=bootstrap.kubeconfig

echo "Create kube-proxy kubeconfig..."
## 设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=k8s-root-ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=kube-proxy.kubeconfig
## 设置客户端认证参数
kubectl config set-credentials kube-proxy \
  --client-certificate=kube-proxy.pem \
  --client-key=kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig
## 设置上下文参数
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig
## 设置默认上下文
kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig

## 创建高级审计配置
cat >> audit-policy.yaml <<EOF
## Log all requests at the Metadata level.
apiVersion: audit.k8s.io/v1beta1
kind: Policy
rules:
- level: Metadata
EOF
```

生成完成后如下截图:<br>
![](https://stone-upyun.b0.upaiyun.com/blog20180510112658.png!700x999)

#### 3.2配置systemd
安装k8s都是用二进制文件的,所以需要手动创建systemd
```
cd k8s && mkdir systemd  && cd systemd
```
如果已经创建了这个目录就不需要了

##### vim kube-apiserver.service
```
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target
After=etcd.service

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/apiserver
User=kube
ExecStart=/usr/local/bin/hyperkube apiserver \
            $KUBE_LOGTOSTDERR \
            $KUBE_LOG_LEVEL \
            $KUBE_ETCD_SERVERS \
            $KUBE_API_ADDRESS \
            $KUBE_API_PORT \
            $KUBELET_PORT \
            $KUBE_ALLOW_PRIV \
            $KUBE_SERVICE_ADDRESSES \
            $KUBE_ADMISSION_CONTROL \
            $KUBE_API_ARGS
Restart=on-failure
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```
##### vim kube-controller-manager.service
```
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/controller-manager
User=kube
ExecStart=/usr/local/bin/hyperkube controller-manager \
            $KUBE_LOGTOSTDERR \
            $KUBE_LOG_LEVEL \
            $KUBE_MASTER \
            $KUBE_CONTROLLER_MANAGER_ARGS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

##### vim kubelet.service
```
[Unit]
Description=Kubernetes Kubelet Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/kubelet
ExecStart=/usr/local/bin/hyperkube kubelet \
            $KUBE_LOGTOSTDERR \
            $KUBE_LOG_LEVEL \
            $KUBELET_API_SERVER \
            $KUBELET_ADDRESS \
            $KUBELET_PORT \
            $KUBELET_HOSTNAME \
            $KUBE_ALLOW_PRIV \
            $KUBELET_ARGS
Restart=on-failure
KillMode=process

[Install]
WantedBy=multi-user.target

```

##### vim kube-proxy.service

```
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/proxy
ExecStart=/usr/local/bin/hyperkube proxy \
            $KUBE_LOGTOSTDERR \
            $KUBE_LOG_LEVEL \
            $KUBE_MASTER \
            $KUBE_PROXY_ARGS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

##### vim kube-scheduler.service
```
[Unit]
Description=Kubernetes Scheduler Plugin
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/scheduler
User=kube
ExecStart=/usr/local/bin/hyperkube scheduler \
            $KUBE_LOGTOSTDERR \
            $KUBE_LOG_LEVEL \
            $KUBE_MASTER \
            $KUBE_SCHEDULER_ARGS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```
至此 systemd创建完成,后续通过脚本会将此此目录中是的文件 copy到系统的systemd目录中的<br>
返回上层目录,回到k8s目录,继续创建相关配置
```
## cd .. && pwd
/root/k8s
```

#### 3.3k8s-配置master节点
/root/k8s目录下进行<br>
Master 节点主要会运行 3 各组件: kube-apiserver、kube-controller-manager、kube-scheduler，其中用到的配置文件如下

##### config
config 是一个通用配置文件，值得注意的是由于安装时对于 Node、Master 节点都会包含该文件，在 Node 节点上请注释掉 KUBE_MASTER 变量，因为 Node 节点需要做 HA，要连接本地的 6443 加密端口；而这个变量将会覆盖 kubeconfig 中指定的 127.0.0.1:6443 地址

```
###
## kubernetes system config
#
## The following values are used to configure various aspects of all
## kubernetes services, including
#
##   kube-apiserver.service
##   kube-controller-manager.service
##   kube-scheduler.service
##   kubelet.service
##   kube-proxy.service
## logging to stderr means we get it in the systemd journal
KUBE_LOGTOSTDERR="--logtostderr=true"

## journal message level, 0 is debug
KUBE_LOG_LEVEL="--v=2"

## Should this cluster be allowed to run privileged docker containers
KUBE_ALLOW_PRIV="--allow-privileged=true"

## How the controller-manager, scheduler, and proxy find the apiserver
KUBE_MASTER="--master=http://127.0.0.1:8080"

```

##### apiserver
apiserver 配置相对于 1.8 略有变动，其中准入控制器(admission control)选项名称变为了 --enable-admission-plugins，控制器列表也有相应变化，这里采用官方推荐配置，具体请参考 [官方文档](https://kubernetes.io/docs/admin/admission-controllers/#is-there-a-recommended-set-of-admission-controllers-to-use)<br>
注意,此处直接复制了mritd的文档.其中ip需要改为自己的
```
###
## kubernetes system config
#
## The following values are used to configure the kube-apiserver
#

## The address on the local server to listen to.
KUBE_API_ADDRESS="--advertise-address=192.168.1.61 --bind-address=192.168.1.61"

## The port on the local server to listen on.
KUBE_API_PORT="--secure-port=6443"

## Port minions listen on
## KUBELET_PORT="--kubelet-port=10250"

## Comma separated list of nodes in the etcd cluster
KUBE_ETCD_SERVERS="--etcd-servers=https://192.168.1.61:2379,https://192.168.1.62:2379,https://192.168.1.63:2379"

## Address range to use for services
KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"

## default admission control policies
KUBE_ADMISSION_CONTROL="--enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota,NodeRestriction"

## Add your own!
KUBE_API_ARGS=" --anonymous-auth=false \
                --apiserver-count=3 \
                --audit-log-maxage=30 \
                --audit-log-maxbackup=3 \
                --audit-log-maxsize=100 \
                --audit-log-path=/var/log/kube-audit/audit.log \
                --audit-policy-file=/etc/kubernetes/audit-policy.yaml \
                --authorization-mode=Node,RBAC \
                --client-ca-file=/etc/kubernetes/ssl/k8s-root-ca.pem \
                --enable-bootstrap-token-auth \
                --enable-garbage-collector \
                --enable-logs-handler \
                --enable-swagger-ui \
                --etcd-cafile=/etc/etcd/ssl/etcd-root-ca.pem \
                --etcd-certfile=/etc/etcd/ssl/etcd.pem \
                --etcd-keyfile=/etc/etcd/ssl/etcd-key.pem \
                --etcd-compaction-interval=5m0s \
                --etcd-count-metric-poll-period=1m0s \
                --event-ttl=48h0m0s \
                --kubelet-https=true \
                --kubelet-timeout=3s \
                --log-flush-frequency=5s \
                --token-auth-file=/etc/kubernetes/token.csv \
                --tls-cert-file=/etc/kubernetes/ssl/kube-apiserver.pem \
                --tls-private-key-file=/etc/kubernetes/ssl/kube-apiserver-key.pem \
                --service-node-port-range=30000-50000 \
                --service-account-key-file=/etc/kubernetes/ssl/k8s-root-ca.pem \
                --storage-backend=etcd3 \
                --enable-swagger-ui=true"
```

##### controller-manager 
controller manager 配置默认开启了证书轮换能力用于自动签署 kueblet 证书，并且证书时间也设置了 10 年，可自行调整；增加了 --controllers 选项以指定开启全部控制器
```
###
## The following values are used to configure the kubernetes controller-manager

## defaults from config and apiserver should be adequate

## Add your own!
KUBE_CONTROLLER_MANAGER_ARGS="  --bind-address=0.0.0.0 \
                                --cluster-name=kubernetes \
                                --cluster-signing-cert-file=/etc/kubernetes/ssl/k8s-root-ca.pem \
                                --cluster-signing-key-file=/etc/kubernetes/ssl/k8s-root-ca-key.pem \
                                --controllers=*,bootstrapsigner,tokencleaner \
                                --deployment-controller-sync-period=10s \
                                --experimental-cluster-signing-duration=86700h0m0s \
                                --leader-elect=true \
                                --node-monitor-grace-period=40s \
                                --node-monitor-period=5s \
                                --pod-eviction-timeout=5m0s \
                                --terminated-pod-gc-threshold=50 \
                                --root-ca-file=/etc/kubernetes/ssl/k8s-root-ca.pem \
                                --service-account-private-key-file=/etc/kubernetes/ssl/k8s-root-ca-key.pem \
                                --feature-gates=RotateKubeletServerCertificate=true"
```

##### scheduler
```
###
## kubernetes scheduler config

## default config should be adequate

## Add your own!
KUBE_SCHEDULER_ARGS="   --address=0.0.0.0 \
                        --leader-elect=true \
                        --algorithm-provider=DefaultProvider"
```
#### 3.4k8s-配置node节点
Node 节点上主要有 kubelet、kube-proxy 组件，用到的配置如下

##### kubelet
kubeket 默认也开启了证书轮换能力以保证自动续签相关证书，同时增加了 --node-labels 选项为 node 打一个标签，关于这个标签最后部分会有讨论，如果在 master 上启动 kubelet，请将 node-role.kubernetes.io/k8s-node=true 修改为 node-role.kubernetes.io/k8s-master=true
```
###
## kubernetes kubelet (minion) config

## The address for the info server to serve on (set to 0.0.0.0 or "" for all interfaces)
KUBELET_ADDRESS="--node-ip=192.168.1.61"

## The port for the info server to serve on
## KUBELET_PORT="--port=10250"

## You may leave this blank to use the actual hostname
KUBELET_HOSTNAME="--hostname-override=k1.node"

## location of the api-server
## KUBELET_API_SERVER=""

## Add your own!
KUBELET_ARGS="  --bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig \
                --cert-dir=/etc/kubernetes/ssl \
                --cgroup-driver=cgroupfs \
                --cluster-dns=10.254.0.2 \
                --cluster-domain=cluster.local. \
                --fail-swap-on=false \
                --feature-gates=RotateKubeletClientCertificate=true,RotateKubeletServerCertificate=true \
                --node-labels=node-role.kubernetes.io/k8s-node=true \
                --image-gc-high-threshold=70 \
                --image-gc-low-threshold=50 \
                --kube-reserved=cpu=500m,memory=512Mi,ephemeral-storage=1Gi \
                --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \
                --system-reserved=cpu=1000m,memory=1024Mi,ephemeral-storage=1Gi \
                --serialize-image-pulls=false \
                --sync-frequency=30s \
                --pod-infra-container-image=k8s.gcr.io/pause-amd64:3.0 \
                --resolv-conf=/etc/resolv.conf \
                --rotate-certificates"
```
##### proxy
```
###
## kubernetes proxy config
## default config should be adequate
## Add your own!
KUBE_PROXY_ARGS="--bind-address=0.0.0.0 \
                 --hostname-override=k1.node \
                 --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig \
                 --cluster-cidr=10.254.0.0/16"

```

##### 安装集群组件
完成上述后,会得到如下一个目录结构(需要保证这个目录结构,才能使用接下来的安装脚本)
```
k8s
├── conf
│   ├── apiserver
│   ├── audit-policy.yaml
│   ├── bootstrap.kubeconfig
│   ├── config
│   ├── controller-manager
│   ├── kubelet
│   ├── kube-proxy.kubeconfig
│   ├── proxy
│   ├── scheduler
│   ├── ssl
│   │   ├── admin.csr
│   │   ├── admin-csr.json
│   │   ├── admin-key.pem
│   │   ├── admin.pem
│   │   ├── k8s-gencert.json
│   │   ├── k8s-root-ca.csr
│   │   ├── k8s-root-ca-csr.json
│   │   ├── k8s-root-ca-key.pem
│   │   ├── k8s-root-ca.pem
│   │   ├── kube-apiserver.csr
│   │   ├── kube-apiserver-csr.json
│   │   ├── kube-apiserver-key.pem
│   │   ├── kube-apiserver.pem
│   │   ├── kube-proxy.csr
│   │   ├── kube-proxy-csr.json
│   │   ├── kube-proxy-key.pem
│   │   └── kube-proxy.pem
│   └── token.csv
├── hyperkube_1.10.1
├── install.sh
└── systemd
    ├── kube-apiserver.service
    ├── kube-controller-manager.service
    ├── kubelet.service
    ├── kube-proxy.service
    └── kube-scheduler.service

```

##### install.sh
```
#!/bin/bash

set -e

KUBE_VERSION="1.10.1"

function download_k8s(){
    if [ ! -f "hyperkube_${KUBE_VERSION}" ]; then
        wget https://storage.googleapis.com/kubernetes-release/release/v${KUBE_VERSION}/bin/linux/amd64/hyperkube -O hyperkube_${KUBE_VERSION}
        chmod +x hyperkube_${KUBE_VERSION}
    fi
}

function preinstall(){
    getent group kube >/dev/null || groupadd -r kube
    getent passwd kube >/dev/null || useradd -r -g kube -d / -s /sbin/nologin -c "Kubernetes user" kube
}

function install_k8s(){
    echo -e "\033[32mINFO: Copy hyperkube...\033[0m"
    cp hyperkube_${KUBE_VERSION} /usr/local/bin/hyperkube

    echo -e "\033[32mINFO: Create symbolic link...\033[0m"
    ln -sf /usr/local/bin/hyperkube /usr/local/bin/kubectl

    echo -e "\033[32mINFO: Copy kubernetes config...\033[0m"
    cp -r conf /etc/kubernetes
    if [ -d "/etc/kubernetes/ssl" ]; then
        chown -R kube:kube /etc/kubernetes/ssl
    fi

    echo -e "\033[32mINFO: Copy kubernetes systemd config...\033[0m"
    cp systemd/*.service /lib/systemd/system
    systemctl daemon-reload
}

function postinstall(){
    if [ ! -d "/var/log/kube-audit" ]; then
        mkdir /var/log/kube-audit
    fi

    if [ ! -d "/var/lib/kubelet" ]; then
        mkdir /var/lib/kubelet
    fi

    if [ ! -d "/usr/libexec" ]; then
        mkdir /usr/libexec
    fi
    chown -R kube:kube /var/log/kube-audit /var/lib/kubelet /usr/libexec
}


download_k8s
preinstall
install_k8s
postinstall
```

- 脚本解释如下:
> download_k8s: 下载 hyperkube 二进制文件
> preinstall: 安装前处理，同 etcd 一样创建 kube 普通用户指定家目录、shell 等
> install_k8s: 复制 hyperkube 到安装目录，为 kubectl 创建软连接(为啥创建软连接就能执行请自行阅读 源码)，复制相关配置到对应目录，并处理权限
> postinstall: 收尾工作，创建日志目录等，并处理权限

<font color='red'> <h5>最后执行此脚本安装即可，将k8s目录scp到每个节点 执行install.sh 此外，应确保每个节点安装了 ipset、conntrack 两个包，因为 kube-proxy 组件会使用其处理 iptables 规则等</h5></font>

### 四 启动集群

#### 4.1 启动master
对于 master 节点启动无需做过多处理，多个 master 只要保证 apiserver 等配置中的 ip 地址监听没问题后直接启动即可
```
systemctl daemon-reload
systemctl start kube-apiserver
systemctl start kube-controller-manager
systemctl start kube-scheduler
systemctl enable kube-apiserver
systemctl enable kube-controller-manager
systemctl enable kube-scheduler
```

#### 4.2 启动node
```
systemctl daemon-reload
systemctl start kubelet
systemctl start kube-proxy
systemctl enable kubelet
systemctl enable kube-proxy
```
其中,如果master也需要跑任务的话, kubelet 和 kube-proxy 也需要在master节点启动<br>

启动完后如图<br>
![](https://stone-upyun.b0.upaiyun.com/blog20180510114912.png!700x999)

### 五 配置集群网络相关
由于 HA 等功能需要，对于 Node 需要做一些处理才能启动，主要有以下两个地方需要处理
#### 5.1 nginx-proxy
在启动 kubelet、kube-proxy 服务之前，需要在本地启动 nginx 来 tcp 负载均衡 apiserver 6443 端口，nginx-proxy 使用 docker + systemd 启动，配置如下<br>
注意: 对于在 master 节点启动 kubelet 来说，不需要 nginx 做负载均衡；可以跳过此步骤，并修改 kubelet.kubeconfig、kube-proxy.kubeconfig 中的 apiserver 地址为当前 master ip 6443 端口即可
##### nginx-proxy.service
```
[Unit]
Description=kubernetes apiserver docker wrapper
Wants=docker.socket
After=docker.service

[Service]
User=root
PermissionsStartOnly=true
ExecStart=/usr/bin/docker run -p 127.0.0.1:6443:6443 \
                              -v /etc/nginx:/etc/nginx \
                              --name nginx-proxy \
                              --net=host \
                              --restart=on-failure:5 \
                              --memory=512M \
                              nginx:1.13.12-alpine
ExecStartPre=-/usr/bin/docker rm -f nginx-proxy
ExecStop=/usr/bin/docker stop nginx-proxy
Restart=always
RestartSec=15s
TimeoutStartSec=30s

[Install]
WantedBy=multi-user.target

```


##### nginx.conf
注意修改ip
```
error_log stderr notice;

worker_processes auto;
events {
        multi_accept on;
        use epoll;
        worker_connections 1024;
}

stream {
    upstream kube_apiserver {
        least_conn;
        server 192.168.1.30:6443;
        server 192.168.1.31:6443;
        server 192.168.1.32:6443;
    }

    server {
        listen        0.0.0.0:6443;
        proxy_pass    kube_apiserver;
        proxy_timeout 10m;
        proxy_connect_timeout 1s;
    }
}
```

##### 启动 apiserver 的本地负载均衡 
```
mkdir /etc/nginx
cp nginx.conf /etc/nginx
cp nginx-proxy.service /lib/systemd/system

systemctl daemon-reload
systemctl start nginx-proxy
systemctl enable nginx-proxy
```

##### TLS bootstrapping
创建好 nginx-proxy 后不要忘记为 TLS Bootstrap 创建相应的 RBAC 规则，这些规则能实现证自动签署 TLS Bootstrap 发出的 CSR 请求，从而实现证书轮换(创建一次即可)；详情请参考 [Kubernetes TLS bootstrapping 那点事](https://mritd.me/2018/01/07/kubernetes-tls-bootstrapping-note/)

###### tls-bootstrapping-clusterrole.yaml
```
## A ClusterRole which instructs the CSR approver to approve a node requesting a
## serving cert matching its client cert.
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: system:certificates.k8s.io:certificatesigningrequests:selfnodeserver
rules:
- apiGroups: ["certificates.k8s.io"]
  resources: ["certificatesigningrequests/selfnodeserver"]
  verbs: ["create"]
```

上述yaml需要再master节点执行创建

```
## 给与 kubelet-bootstrap 用户进行 node-bootstrapper 的权限
kubectl create clusterrolebinding kubelet-bootstrap \
    --clusterrole=system:node-bootstrapper \
    --user=kubelet-bootstrap

kubectl create -f tls-bootstrapping-clusterrole.yaml

## 自动批准 system:bootstrappers 组用户 TLS bootstrapping 首次申请证书的 CSR 请求
kubectl create clusterrolebinding node-client-auto-approve-csr \
        --clusterrole=system:certificates.k8s.io:certificatesigningrequests:nodeclient \
        --group=system:bootstrappers

## 自动批准 system:nodes 组用户更新 kubelet 自身与 apiserver 通讯证书的 CSR 请求
kubectl create clusterrolebinding node-client-auto-renew-crt \
        --clusterrole=system:certificates.k8s.io:certificatesigningrequests:selfnodeclient \
        --group=system:nodes

## 自动批准 system:nodes 组用户更新 kubelet 10250 api 端口证书的 CSR 请求
kubectl create clusterrolebinding node-server-auto-renew-crt \
        --clusterrole=system:certificates.k8s.io:certificatesigningrequests:selfnodeserver \
        --group=system:nodes
```


##### 执行启动
多节点部署时先启动好 nginx-proxy，然后修改好相应配置的 ip 地址等配置，最终直接启动即可(master 上启动 kubelet 不要忘了修改 kubeconfig 中的 apiserver 地址，还有对应的 kubelet 的 node label)
```
systemctl daemon-reload
systemctl restart kubelet
systemctl restart kube-proxy
systemctl enable kubelet
systemctl enable kube-proxy

```
启动成功后得到如下图<br>
![](https://stone-upyun.b0.upaiyun.com/blog20180510115842.png!700x999)
<br>
截图来自mritd,由于是公司机器,不能直接暴露,后续个人搭建时,再来替换为自己的



#### 5.2 Calico
Calico 安装仍然延续以前的方案，使用 Daemonset 安装 cni 组件，使用 systemd 控制 calico-node 以确保 calico-node 能正确的拿到主机名等
##### 修改calico配置
```
wget https://docs.projectcalico.org/v3.1/getting-started/kubernetes/installation/hosted/calico.yaml -O calico.example.yaml

ETCD_CERT=`cat /etc/etcd/ssl/etcd.pem | base64 | tr -d '\n'`
ETCD_KEY=`cat /etc/etcd/ssl/etcd-key.pem | base64 | tr -d '\n'`
ETCD_CA=`cat /etc/etcd/ssl/etcd-root-ca.pem | base64 | tr -d '\n'`
ETCD_ENDPOINTS="https://192.168.1.61:2379,https://192.168.1.62:2379,https://192.168.1.63:2379"

cp calico.example.yaml calico.yaml

sed -i "s@.*etcd_endpoints:.*@\ \ etcd_endpoints:\ \"${ETCD_ENDPOINTS}\"@gi" calico.yaml

sed -i "s@.*etcd-cert:.*@\ \ etcd-cert:\ ${ETCD_CERT}@gi" calico.yaml
sed -i "s@.*etcd-key:.*@\ \ etcd-key:\ ${ETCD_KEY}@gi" calico.yaml
sed -i "s@.*etcd-ca:.*@\ \ etcd-ca:\ ${ETCD_CA}@gi" calico.yaml

sed -i 's@.*etcd_ca:.*@\ \ etcd_ca:\ "/calico-secrets/etcd-ca"@gi' calico.yaml
sed -i 's@.*etcd_cert:.*@\ \ etcd_cert:\ "/calico-secrets/etcd-cert"@gi' calico.yaml
sed -i 's@.*etcd_key:.*@\ \ etcd_key:\ "/calico-secrets/etcd-key"@gi' calico.yaml

## 注释掉 calico-node 部分(由 Systemd 接管)
sed -i '123,219s@.*@#&@gi' calico.yaml
```


##### 创建systemd
<h5>注意: 创建 systemd service 配置文件要在每个节点上都执行</h5>
```
K8S_MASTER_IP="192.168.1.61"
HOSTNAME=`cat /etc/hostname`
ETCD_ENDPOINTS="https://192.168.1.61:2379,https://192.168.1.62:2379,https://192.168.1.63:2379"

cat > /lib/systemd/system/calico-node.service <<EOF
[Unit]
Description=calico node
After=docker.service
Requires=docker.service

[Service]
User=root
Environment=ETCD_ENDPOINTS=${ETCD_ENDPOINTS}
PermissionsStartOnly=true
ExecStart=/usr/bin/docker run   --net=host --privileged --name=calico-node \\
                                -e ETCD_ENDPOINTS=\${ETCD_ENDPOINTS} \\
                                -e ETCD_CA_CERT_FILE=/etc/etcd/ssl/etcd-root-ca.pem \\
                                -e ETCD_CERT_FILE=/etc/etcd/ssl/etcd.pem \\
                                -e ETCD_KEY_FILE=/etc/etcd/ssl/etcd-key.pem \\
                                -e NODENAME=${HOSTNAME} \\
                                -e IP= \\
                                -e IP_AUTODETECTION_METHOD=can-reach=${K8S_MASTER_IP} \\
                                -e AS=64512 \\
                                -e CLUSTER_TYPE=k8s,bgp \\
                                -e CALICO_IPV4POOL_CIDR=10.20.0.0/16 \\
                                -e CALICO_IPV4POOL_IPIP=always \\
                                -e CALICO_LIBNETWORK_ENABLED=true \\
                                -e CALICO_NETWORKING_BACKEND=bird \\
                                -e CALICO_DISABLE_FILE_LOGGING=true \\
                                -e FELIX_IPV6SUPPORT=false \\
                                -e FELIX_DEFAULTENDPOINTTOHOSTACTION=ACCEPT \\
                                -e FELIX_LOGSEVERITYSCREEN=info \\
                                -e FELIX_IPINIPMTU=1440 \\
                                -e FELIX_HEALTHENABLED=true \\
                                -e CALICO_K8S_NODE_REF=${HOSTNAME} \\
                                -v /etc/calico/etcd-root-ca.pem:/etc/etcd/ssl/etcd-root-ca.pem \\
                                -v /etc/calico/etcd.pem:/etc/etcd/ssl/etcd.pem \\
                                -v /etc/calico/etcd-key.pem:/etc/etcd/ssl/etcd-key.pem \\
                                -v /lib/modules:/lib/modules \\
                                -v /var/lib/calico:/var/lib/calico \\
                                -v /var/run/calico:/var/run/calico \\
                                quay.io/calico/node:v3.1.0
ExecStop=/usr/bin/docker rm -f calico-node
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF
``` 
<font color='red'> 对于以上脚本中的 K8S_MASTER_IP 变量，只需要填写一个 master ip 即可，这个变量用于 calico 自动选择 IP 使用；在宿主机有多张网卡的情况下，calcio node 会自动获取一个 IP，获取原则就是尝试是否能够联通这个 master ip 由于 calico 需要使用 etcd 存储数据，所以需要复制 etcd 证书到相关目录，/etc/calico 需要在每个节点都有</font>

```
cp -r /etc/etcd/ssl /etc/calico
```

##### 修改kubelet配置
使用 Calico 后需要修改 kubelet 配置增加 CNI 设置(--network-plugin=cni)，修改后配置如下 <br>
增加了 --network-plugin=cni \    这个配置
```
###
## kubernetes kubelet (minion) config

## The address for the info server to serve on (set to 0.0.0.0 or "" for all interfaces)
KUBELET_ADDRESS="--node-ip=192.168.1.30"

## The port for the info server to serve on
## KUBELET_PORT="--port=10250"

## You may leave this blank to use the actual hostname
KUBELET_HOSTNAME="--hostname-override=k1.node"

## location of the api-server
## KUBELET_API_SERVER=""

## Add your own!
KUBELET_ARGS="  --bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig \
                --cert-dir=/etc/kubernetes/ssl \
                --cgroup-driver=cgroupfs \
                --network-plugin=cni \
                --cluster-dns=10.254.0.2 \
                --cluster-domain=cluster.local. \
                --fail-swap-on=false \
                --feature-gates=RotateKubeletClientCertificate=true,RotateKubeletServerCertificate=true \
                --node-labels=node-role.kubernetes.io/k8s-master=true \
                --image-gc-high-threshold=70 \
                --image-gc-low-threshold=50 \
                --kube-reserved=cpu=500m,memory=512Mi,ephemeral-storage=1Gi \
                --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \
                --system-reserved=cpu=1000m,memory=1024Mi,ephemeral-storage=1Gi \
                --serialize-image-pulls=false \
                --sync-frequency=30s \
                --pod-infra-container-image=k8s.gcr.io/pause-amd64:3.0 \
                --resolv-conf=/etc/resolv.conf \
                --rotate-certificates"
```

##### 创建 Calico Daemonset
```
## 先创建 RBAC
kubectl apply -f \
https://docs.projectcalico.org/v3.1/getting-started/kubernetes/installation/rbac.yaml

## 再创建 Calico Daemonset
kubectl create -f calico.yaml
```

##### 启动 Calico Node
```
systemctl daemon-reload
systemctl restart calico-node
systemctl enable calico-node

## 等待 20s 拉取镜像
sleep 20
systemctl restart kubelet
```

##### 测试网络
```
## 创建 deployment
cat << EOF >> demo.deploy.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-deployment
spec:
  replicas: 5
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      containers:
      - name: demo
        image: mritd/demo
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
EOF
kubectl create -f demo.deploy.yml
```
截图引用自mritd<br>
![](https://stone-upyun.b0.upaiyun.com/blog20180510120356.png!700x999)
<br>


#### 5.3 coreDNS
此处使用的mritd的脚本自动安装的<br>

```
cd ~ 
git clone https://github.com/mritd/ktool
cd /root/ktool/k8s/addons/coredns

## 执行上面的替换脚本
./deploy.sh

## 创建 CoreDNS
kubectl create -f coredns.yaml
```

替换脚本内容为:
```
#!/bin/bash

## Deploys CoreDNS to a cluster currently running Kube-DNS.

SERVICE_CIDR=${1:-10.254.0.0/16}
POD_CIDR=${2:-10.20.0.0/16}
CLUSTER_DNS_IP=${3:-10.254.0.2}
CLUSTER_DOMAIN=${4:-cluster.local}
YAML_TEMPLATE=${5:-`pwd`/coredns.yaml.sed}

sed -e s/CLUSTER_DNS_IP/$CLUSTER_DNS_IP/g -e s/CLUSTER_DOMAIN/$CLUSTER_DOMAIN/g -e s?SERVICE_CIDR?$SERVICE_CIDR?g -e s?POD_CIDR?$POD_CIDR?g $YAML_TEMPLATE > coredns.yaml
```


##### 部署DNS自动扩容 
vim dns-horizontal-autoscaler.yaml
```
## Copyright 2016 The Kubernetes Authors.
#
## Licensed under the Apache License, Version 2.0 (the "License");
## you may not use this file except in compliance with the License.
## You may obtain a copy of the License at
#
##     http://www.apache.org/licenses/LICENSE-2.0
#
## Unless required by applicable law or agreed to in writing, software
## distributed under the License is distributed on an "AS IS" BASIS,
## WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
## See the License for the specific language governing permissions and
## limitations under the License.

kind: ServiceAccount
apiVersion: v1
metadata:
  name: kube-dns-autoscaler
  namespace: kube-system
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: system:kube-dns-autoscaler
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["list"]
  - apiGroups: [""]
    resources: ["replicationcontrollers/scale"]
    verbs: ["get", "update"]
  - apiGroups: ["extensions"]
    resources: ["deployments/scale", "replicasets/scale"]
    verbs: ["get", "update"]
## Remove the configmaps rule once below issue is fixed:
## kubernetes-incubator/cluster-proportional-autoscaler#16
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "create"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: system:kube-dns-autoscaler
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
subjects:
  - kind: ServiceAccount
    name: kube-dns-autoscaler
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: system:kube-dns-autoscaler
  apiGroup: rbac.authorization.k8s.io

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kube-dns-autoscaler
  namespace: kube-system
  labels:
    k8s-app: kube-dns-autoscaler
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  selector:
    matchLabels:
      k8s-app: kube-dns-autoscaler
  template:
    metadata:
      labels:
        k8s-app: kube-dns-autoscaler
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ''
    spec:
      priorityClassName: system-cluster-critical
      containers:
      - name: autoscaler
        image: k8s.gcr.io/cluster-proportional-autoscaler-amd64:1.1.2-r2
        resources:
            requests:
                cpu: "20m"
                memory: "10Mi"
        command:
          - /cluster-proportional-autoscaler
          - --namespace=kube-system
          - --configmap=kube-dns-autoscaler
          ## Should keep target in sync with cluster/addons/dns/kube-dns.yaml.base
          - --target=Deployment/kube-dns
          ## When cluster is using large nodes(with more cores), "coresPerReplica" should dominate.
          ## If using small nodes, "nodesPerReplica" should dominate.
          - --default-params={"linear":{"coresPerReplica":256,"nodesPerReplica":16,"preventSinglePointFailure":true}}
          - --logtostderr=true
          - --v=2
      tolerations:
      - key: "CriticalAddonsOnly"
        operator: "Exists"
      serviceAccountName: kube-dns-autoscaler
```
执行yaml创建即可完成DNS的自动扩容
```
kubectl create -f dns-horizontal-autoscaler.yaml 
```

### 六 k8s-监控heapster
直接yaml创建即可
```
kubectl create -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/grafana.yaml
kubectl create -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/heapster.yaml
kubectl create -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/influxdb.yaml
kubectl create -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/rbac/heapster-rbac.yaml
```
至此,搭建完成 ,集群已可以正常使用.如需要管理页面的可翻看下面的内容,不需要就不用了
### 七 k8s-图形管理页面 Dashboard
因为本次搭建没有部署Dashboard,具体搭建可看mritd的 [部署 Dashboard](https://mritd.me/2018/04/19/set-up-kubernetes-1.10.1-cluster-by-hyperkube/#%E5%85%AB%E9%83%A8%E7%BD%B2-dashboard)
<br>
<br>

**本文参考来自mritd的[Kubernetes 1.10.1 集群搭建](https://mritd.me/2018/04/19/set-up-kubernetes-1.10.1-cluster-by-hyperkube) ,其中大量脚本使用mritd提供**
