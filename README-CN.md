[English](README.md) | 简体中文

## 1. 什么是kube-eye
kube-eye是一个专门为kubernetes平台设计的辅助工具，提供多种实用功能帮助您快速定位集群中的问题。

kube-eye中目前包含以及将来可能加入的功能清单如下：
- 收集集群内所有Pod日志，并提供全文搜索功能。
- 查看指定容器的实时日志，并支持搜索词高亮。
- 使用[KubeBench](https://github.com/aquasecurity/kube-bench)对集群节点进行安全扫描，并给出修复建议。
- 使用[DockerBench](https://hub.docker.com/r/docker/docker-bench-security)对集群节点上的Docker容器进行CIS基准扫描。（待加入）
- 使用[Clair](https://coreos.com/clair)对集群中的容器进行漏洞分析。（待加入）

注意，在当前版本中，为了提供日志检索服务，kube-eye会临时存储Pod日志，但不保证日志可以被永久存储，在您删除或重新部署kube-eye时，之前存储的日志也将被清空。因此，如果您需要永久存储某些Pod日志，可能需要寻找其他方案。在后续版本中，我们有计划支持日志的永久存储，以及数据的导入导出功能。

## 2. 前提条件
- 确认本地已经安装并配置kubectl。
- 确保集群中至少有两个节点的可用内存大于3GB。

## 3. 公网IP安装方式
该安装方式适用于在云提供商中部署kube-eye，例如：AKS、EKS、GKS等。安装过程中将会向云提供商申请一个公网IP，以便访问kube-eye服务，这可能产生额外的费用。

### 3.1 安装kube-eye
打开您的本地终端窗口，执行以下命令来安装kube-eye。
```
kubectl create ns kube-eye
kubectl create -f http://cdn.jsdelivr.net/gh/ivyentcn/ivyentcn.github.io/kube-eye/deploy/kube-eye-k8s1.18-loadbalance.yaml
```

### 3.2 检查安装状态
接下来等待所有Pod到达Running状态，使用以下命令查看Pod状态，如果一切顺利，将看到类似下列输出。
```
kubectl -n kube-eye get pods
NAME                        READY   STATUS    RESTARTS   AGE
elasticsearch-logging-0     1/1     Running   0          1m46s
elasticsearch-logging-1     1/1     Running   0          1m30s
fluentd-es-v3.1.0-2js9n     1/1     Running   0          1m46s
fluentd-es-v3.1.0-7mqj7     1/1     Running   0          1m46s
kube-eye-7b97c8d7c5-6f6fr   1/1     Running   0          1m45s
```

### 3.3 访问kube-eye
执行以下命令，并在输出中找到kube-eye的外部IP，在以下实例中为`10.100.100.100`。
```
kubectl -n kube-eye get svc
NAME                          TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)             AGE
kube-eye                      LoadBalancer   10.0.248.249   10.100.100.100   80:32610/TCP        1m
elasticsearch-logging         ClusterIP      10.0.251.238   <none>           9200/TCP,9300/TCP   1m
elasticsearch-logging-inner   ClusterIP      None           <none>           9200/TCP,9300/TCP   1m
```

现在您可以从浏览器中访问kube-eye页面：`http://10.100.100.100`

## 4. Ingress安装方式
如果您的集群中部署了[Ingress控制器](https://kubernetes.io/zh/docs/concepts/services-networking/ingress-controllers)，则无需单独为kube-eye分配公网IP。

### 4.1 安装kube-eye
打开您的本地终端窗口，执行以下命令来安装kube-eye。
```
kubectl create ns kube-eye
kubectl create -f http://cdn.jsdelivr.net/gh/ivyentcn/ivyentcn.github.io/kube-eye/deploy/kube-eye-k8s1.18-ingress.yaml
```

### 4.2 检查安装状态
接下来等待所有Pod到达Running状态，使用以下命令查看Pod状态，如果一切顺利，将看到类似下列输出。
```
kubectl -n kube-eye get pods
NAME                        READY   STATUS    RESTARTS   AGE
elasticsearch-logging-0     1/1     Running   0          1m46s
elasticsearch-logging-1     1/1     Running   0          1m30s
fluentd-es-v3.1.0-2js9n     1/1     Running   0          1m46s
fluentd-es-v3.1.0-7mqj7     1/1     Running   0          1m46s
kube-eye-7b97c8d7c5-6f6fr   1/1     Running   0          1m45s
```

### 4.3 创建入口
最后创建一个[Ingress资源](https://kubernetes.io/zh/docs/concepts/services-networking/ingress/#the-ingress-resource)以便访问kube-eye，您可能需要将以下模板中的`kubeeye.k8s.local`修改为您自己的域名。然后执行它。

适用于`1.18`或更早的k8s：
```
kubectl create -f - <<< '
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kube-eye
  namespace: kube-eye
  annotations:
      kubernetes.io/ingress.class: nginx
      nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
spec:
  rules:
  - host: kubeeye.k8s.local
    http:
      paths:
      - backend:
          serviceName: kube-eye
          servicePort: 80
'
```

适用于`1.19`或更新的k8s：
```
kubectl create -f - <<< '
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kube-eye
  namespace: kube-eye
  annotations:
      kubernetes.io/ingress.class: nginx
      nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
spec:
  rules:
  - host: kubeeye.k8s.local
    http:
      paths:
      - pathType: ImplementationSpecific
        backend:
          service:
            name: kube-eye
            port:
              number: 80
'
```

## 5. 卸载kube-eye
### 5.1 删除kube-eye所有组件
```
kubectl delete -f http://cdn.jsdelivr.net/gh/ivyentcn/ivyentcn.github.io/kube-eye/deploy/kube-eye-k8s1.18-loadbalance.yaml
kubectl -n kube-eye delete ingress/kube-eye
```

### 5.2 删除命名空间
最后您可以选择删除`kube-eye`命名空间，删除前请确认该命名空间内没有其他资源。
```
kubectl delete ns kube-eye
```
