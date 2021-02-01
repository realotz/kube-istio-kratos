# kubernetes环境搭建

本文主要分线下单节点开发环境与阿里云ack托管集群为生产环境的搭建。
非专业运维同学，不建议以自建k8s做生产环境。
k8s安装有很多方式，单纯以学习来看最好是以二进制方式安装etcd、apiserver等组件。有需求等可以google搜索网上有很多的文章
本次以kubekey做安装工具来快速搭建内网或者本地的单机k8s环境，主要是做测试与devops环境 其他同类工具放下面可供大家自行查看
1、minikube https://github.com/kubernetes/minikube
2、kubeasz https://github.com/easzlab/kubeasz （国内友好）
3、kubekey https://github.com/kubesphere/kubekey（国内友好）

## 本地开发/内网集群
提前需求
1、1台或以上的liunx环境服务器 os > centos7 | ubuntu 16.04
2、建议至少500G以上的硬盘，并配ssd

笔者开发环境
1、cpu amd R7 4800Hcpu bios开启amd-v 
2、vm 虚拟机 centos8

此环境为本机开发环境搭建，内网环境也是差不多的。
修改主机名

```
sudo hostnamectl set-hostname k8s-master
```
安装docker, centos8安装docker需要升级containerd
```
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://mirrors.aliyun.com/dockerce/linux/centos/docker-ce.repo
yum install -y docker-ce docker-ce-cli  containerd.io
sudo systemctl enable docker
```

```
export KKZONE=cn && \
curl -sfL https://get-kk.kubesphere.io | VERSION=v1.0.1 sh - && \
chmod +x kk
```
可以增加 --with-kubesphere v3.0.0 参数安装kubesphere面板，关于kubesphere后面篇幅会讲到一些应用
此脚本会安装一个All-in-One的Kubernetes集群
```
./kk create cluster --with-kubernetes v1.18.6 
```
查看安装过程
```
kubectl logs -n kubesphere-system $(kubectl get pod -n kubesphere-system -l app=ks-install -o jsonpath='{.items[0].metadata.name}') -f
```
过几分钟后脚本执行完毕没有错误后，查看安装结果
```
# 列出当前集群下的节点
kubectl get nodes
# 输出
NAME         STATUS   ROLES           AGE   VERSION
k8s-master   Ready    master,worker   34d   v1.18.6
```
自此一个单机开发环境的k8s安装完毕，如多机器情况下可以参考kubekey文档进行配置
https://github.com/kubesphere/kubekey/blob/master/README_zh-CN.md

## 存储 SC PV PVC 
1、sc(StorageClass) 是存储类型的抽象，可以看出pv的模版，创建好后可以根据sc快速创建pv存储声明
2、pv(Persistent Volume) pv是持久化存储券，是实际存储（云硬、oss、本地硬盘等）抽象的存储单元。比如nfs中会以pv-xxxxxx的名字声明一个目录存放在nfs服务器上
3、pvc(Persistent Volume Claim) pod实际使用pv的实例，描述pod怎么去使用一个pv
### nfs
如本地环境有nas一类存储，可以用nfs为k8s环境提供sc
```
yum install nfs-common  nfs-utils -y 
kubectl apply -f deploy/nfs-client.yaml
```
### openebs
kubekey会默认安装openebs作为默认StorageClass

## cert-manage 证书管理
cert-manage 是一个自动证书签发工具，可以简单的利用cert-manage做一个永不过期（到期前自动续期）的Let’s Encrypt免费证书管理工具。

### cert-manage
```shell
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.1.0/cert-manager.yaml
```
### 阿里云DNS01
开源webhook  https://github.com/pragkent/alidns-webhook
在阿里云创建一个有dns指向权限的access-key

安装webhook

```
kubectl apply -f https://raw.githubusercontent.com/pragkent/alidns-webhook/master/deploy/bundle.yaml
```

阿里云密钥

```
cat >> alidns-secret.yaml << EOF
apiVersion: v1
kind: Secret
metadata:
  name: alidns-secret
  namespace: cert-manager
dataString:
  access-key: "你的阿里云appkey"
  secret-key: "你的阿里云secretKey"
EOF	
```

```
kubectl apply -f -n cert-manager alidns-secret.yaml
```

ClusterIssuer 证书发行者

```
cat >> letsencrypt-staging.yaml << EOF
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    # 你的邮箱
    email: certmaster@example.com
    # 测试证书申请地址
    server: https://acme-v02.api.letsencrypt.org/directory
    # 测试证书申请地址 请根据环境选择
    # server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-staging-account-key
    solvers:
    - dns01:
        webhook:
          groupName: acme.yourcompany.com
          solverName: alidns
          config:
            region: ""
            accessKeySecretRef:
              name: alidns-secret
              key: access-key
            secretKeySecretRef:
              name: alidns-secret
              key: secret-key
EOF
```

```
kubectl apply -f -n cert-manager letsencrypt-staging.yaml
```

证书 注意，证书申请每日有上限，尤其泛域名证书，如果cert-manager 日志中显示429错误，具体参考letsencrypt官网

```
cat >> example-tls.yaml << EOF
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  name: example-tls
spec:
  secretName: example-com-tls
  commonName: example.com
  dnsNames:
  - "*.example.com"
  issuerRef:
    name: letsencrypt-staging
    kind: ClusterIssuer
EOF
```

```
kubectl apply -f example-tls.yaml
```

稍等1-2分钟查看证书是否颁发

```

kubectl get secret

NAME                                              TYPE                                  DATA   AGE
example-tls 			                               kubernetes.io/tls                     2      34d
```
错误排查

```
# 确认cert正常运行
kubectl -n cert-manager get pods

alidns-webhook-645b8d5c47-wfn9r            1/1     Running   24         34d
cert-manager-58c8844bb8-cl7gt              1/1     Running   10         34d
cert-manager-cainjector-745768f6ff-tb8gc   1/1     Running   30         34d
cert-manager-webhook-67cc76975b-l7484      1/1     Running   8          34d
# 正常运行请查看cert日志
kubectl -n cert-manager logs cert-manager-58c8844bb8-cl7gt
```

## ingress gateway
ingress作为k8s自带的路由资源，允许部署各种网关来进行7层负载均衡。

以下以nginx ingress为例，当然也可以用traefik等其他gateway作为默认网关



### 部署ingress

###  L4 Balancer 与nodeport 还是 80 443

