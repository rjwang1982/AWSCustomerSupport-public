<img src="aws-partner.png" height="50" /> &nbsp; <img src="vstecs-logo-horizontal.png" height="50" />

# CloudShell 管理 EKS 集群 SOP

**作者**: RJ.Wang | 高级工程师  
**公司**: 伟仕佳杰 (VSTECS) | AWS Partner  
**邮箱**: renjun.wang@vstecs.com  
**创建时间**: 2026-04-21  
**交付客户**: 阳光电源 (Sungrow) HEMS 项目  
**适用集群**: hems-eu-k8s (eu-central-1, 账号 872515264289)

---

## 一、打开 CloudShell

1. 登录 AWS 控制台（账号 872515264289）
2. 确认右上角区域为 **eu-central-1（法兰克福）**
3. 点击右上角 `>_` 图标，或直接访问：
   ```
   https://eu-central-1.console.aws.amazon.com/cloudshell/home?region=eu-central-1
   ```
4. 等待终端加载完成（首次约 30 秒）

> 💡 CloudShell 自带 `aws cli`、`kubectl`、`helm`，无需安装。
> 💡 CloudShell 使用你当前登录控制台的 IAM 身份，无需配置 AKSK。

---

## 二、首次连接集群（只需做一次）

```bash
# 配置 kubectl 连接到 EKS 集群
aws eks update-kubeconfig --name hems-eu-k8s --region eu-central-1

# 验证连接
kubectl get svc
```

**预期输出**：
```
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.100.0.1   <none>        443/TCP   xxd
```

> ⚠️ CloudShell 会话超时后 kubeconfig 仍保留在 `$HOME` 目录，下次打开无需重新配置。
> 但如果 CloudShell 环境被回收（长时间未使用），需要重新运行上述命令。

---

## 三、日常运维命令

### 3.1 查看集群状态

```bash
# 查看所有节点
kubectl get nodes -o wide

# 查看节点详情（实例类型、角色标签）
kubectl get nodes -o custom-columns=\
NAME:.metadata.name,\
STATUS:.status.conditions[-1].type,\
INSTANCE:.metadata.labels.node\\.kubernetes\\.io/instance-type,\
ROLE:.metadata.labels.node-role,\
AGE:.metadata.creationTimestamp

# 查看集群信息
kubectl cluster-info
```

### 3.2 查看 Pod

```bash
# 查看所有命名空间的 Pod
kubectl get pods -A

# 查看指定命名空间的 Pod
kubectl get pods -n <命名空间>

# 查看 Pod 详情（排查问题用）
kubectl describe pod <pod名称> -n <命名空间>

# 查看 Pod 日志
kubectl logs <pod名称> -n <命名空间>

# 查看 Pod 日志（实时跟踪）
kubectl logs -f <pod名称> -n <命名空间>

# 查看上一次崩溃的日志
kubectl logs <pod名称> -n <命名空间> --previous
```

### 3.3 查看 Service / Ingress

```bash
# 查看所有 Service
kubectl get svc -A

# 查看 Ingress
kubectl get ingress -A

# 查看 Service 详情
kubectl describe svc <service名称> -n <命名空间>
```

### 3.4 查看资源使用情况

```bash
# 查看节点资源使用（需要 metrics-server）
kubectl top nodes

# 查看 Pod 资源使用
kubectl top pods -A

# 查看某个节点上运行的所有 Pod
kubectl get pods -A --field-selector spec.nodeName=<节点名称>
```

### 3.5 查看命名空间

```bash
# 列出所有命名空间
kubectl get namespaces

# 查看某个命名空间下的所有资源
kubectl get all -n <命名空间>
```

---

## 四、应用部署与管理

### 4.1 部署应用（YAML 方式）

```bash
# 方法 1: 从文件部署
kubectl apply -f deployment.yaml

# 方法 2: 直接在 CloudShell 中创建 YAML 并部署
cat > my-app.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: nginx:latest
        ports:
        - containerPort: 80
EOF

kubectl apply -f my-app.yaml
```

### 4.2 扩缩容

```bash
# 手动扩缩容 Deployment
kubectl scale deployment <deployment名称> -n <命名空间> --replicas=3

# 查看扩缩容状态
kubectl rollout status deployment <deployment名称> -n <命名空间>
```

### 4.3 更新镜像

```bash
# 更新 Deployment 的容器镜像
kubectl set image deployment/<deployment名称> \
  <容器名>=<新镜像:tag> -n <命名空间>

# 查看更新状态
kubectl rollout status deployment/<deployment名称> -n <命名空间>

# 查看更新历史
kubectl rollout history deployment/<deployment名称> -n <命名空间>

# 回滚到上一个版本
kubectl rollout undo deployment/<deployment名称> -n <命名空间>
```

### 4.4 删除资源

```bash
# 删除 Deployment
kubectl delete deployment <deployment名称> -n <命名空间>

# 删除 Service
kubectl delete svc <service名称> -n <命名空间>

# 通过 YAML 文件删除
kubectl delete -f my-app.yaml
```

---

## 五、故障排查

### 5.1 Pod 状态异常排查流程

```
Pod 状态不正常？
├── Pending     → 资源不足或调度问题
├── CrashLoopBackOff → 应用启动失败
├── ImagePullBackOff → 镜像拉取失败
├── Error       → 容器运行出错
└── Evicted     → 节点资源不足被驱逐
```

```bash
# 第 1 步: 查看 Pod 状态
kubectl get pods -n <命名空间>

# 第 2 步: 查看 Pod 事件（最重要！）
kubectl describe pod <pod名称> -n <命名空间>

# 第 3 步: 查看容器日志
kubectl logs <pod名称> -n <命名空间>
```

### 5.2 常见问题速查

**Pod 一直 Pending**：
```bash
# 检查节点资源
kubectl describe node <节点名> | grep -A 5 "Allocated resources"
# 原因: CPU/内存不足，需要扩容节点或减少 Pod 资源请求
```

**ImagePullBackOff**：
```bash
# 检查镜像地址是否正确
kubectl describe pod <pod名称> -n <命名空间> | grep "Image:"
# 原因: 镜像不存在、ECR 权限不足、或网络不通
```

**CrashLoopBackOff**：
```bash
# 查看上一次崩溃日志
kubectl logs <pod名称> -n <命名空间> --previous
# 原因: 应用代码错误、配置错误、依赖服务不可用
```

**节点 NotReady**：
```bash
# 查看节点状态详情
kubectl describe node <节点名>
# 检查节点组状态
aws eks describe-nodegroup --cluster-name hems-eu-k8s \
  --nodegroup-name <节点组名> --region eu-central-1
```

### 5.3 查看集群事件

```bash
# 查看最近的集群事件（按时间排序）
kubectl get events -A --sort-by='.lastTimestamp' | tail -20

# 查看某个命名空间的事件
kubectl get events -n <命名空间> --sort-by='.lastTimestamp'
```

---

## 六、EKS 集群管理（AWS CLI）

### 6.1 查看集群信息

```bash
# 集群详情
aws eks describe-cluster --name hems-eu-k8s --region eu-central-1

# 节点组列表
aws eks list-nodegroups --cluster-name hems-eu-k8s --region eu-central-1

# 节点组详情
aws eks describe-nodegroup --cluster-name hems-eu-k8s \
  --nodegroup-name hems-business --region eu-central-1
```

### 6.2 节点组扩缩容

```bash
# 修改节点组大小（例如从 3 扩到 5）
aws eks update-nodegroup-config \
  --cluster-name hems-eu-k8s \
  --nodegroup-name hems-business \
  --scaling-config minSize=5,maxSize=5,desiredSize=5 \
  --region eu-central-1
```

### 6.3 查看附加组件

```bash
# 列出已安装的附加组件
aws eks list-addons --cluster-name hems-eu-k8s --region eu-central-1

# 查看附加组件详情
aws eks describe-addon --cluster-name hems-eu-k8s \
  --addon-name vpc-cni --region eu-central-1
```

---

## 七、实用技巧

### 7.1 设置别名（提高效率）

```bash
# 添加到 ~/.bashrc（CloudShell 会保留）
echo 'alias k=kubectl' >> ~/.bashrc
echo 'alias kgp="kubectl get pods"' >> ~/.bashrc
echo 'alias kga="kubectl get pods -A"' >> ~/.bashrc
echo 'alias kgn="kubectl get nodes -o wide"' >> ~/.bashrc
echo 'alias kd="kubectl describe"' >> ~/.bashrc
echo 'alias kl="kubectl logs"' >> ~/.bashrc
source ~/.bashrc
```

之后可以用简写：
```bash
k get pods          # 等同于 kubectl get pods
kga                 # 查看所有 Pod
kgn                 # 查看所有节点
kd pod <名称>       # 查看 Pod 详情
kl <pod名称>        # 查看日志
```

### 7.2 kubectl 自动补全

```bash
# 开启 kubectl 命令自动补全
echo 'source <(kubectl completion bash)' >> ~/.bashrc
echo 'complete -o default -F __start_kubectl k' >> ~/.bashrc
source ~/.bashrc
```

之后按 `Tab` 键可以自动补全命令和资源名称。

### 7.3 上传/下载文件

CloudShell 支持文件上传下载：
- **上传**: 点击右上角「操作」→「上传文件」
- **下载**: 点击右上角「操作」→「下载文件」，输入文件路径

### 7.4 切换命名空间

```bash
# 设置默认命名空间（避免每次都加 -n）
kubectl config set-context --current --namespace=<命名空间>

# 查看当前默认命名空间
kubectl config view --minify | grep namespace
```

---

## 八、CloudShell 注意事项

| 项目 | 说明 |
|------|------|
| 存储 | `$HOME` 目录有 1GB 持久存储，重启后保留 |
| 超时 | 20 分钟无操作自动断开，重新点击即可恢复 |
| 环境回收 | 120 天不使用会删除 `$HOME`，需重新配置 |
| 并发 | 同一区域最多 10 个 CloudShell 会话 |
| 网络 | 有公网访问能力，可以 curl 外部 API |
| 限制 | 不支持长时间运行的后台进程 |

---

> 本文档由伟仕佳杰 (VSTECS) 作为 AWS Partner 提供
