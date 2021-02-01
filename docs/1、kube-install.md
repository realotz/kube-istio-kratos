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
kubectl apply -f deploy/nfs-client/client.yaml
```
### openebs
kubekey会默认安装openebs作为默认StorageClass



