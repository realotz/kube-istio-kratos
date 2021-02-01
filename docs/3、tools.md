# 常用工具部署

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
