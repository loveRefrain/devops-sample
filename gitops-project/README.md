# GitOps DevOps 项目完整部署指南

本项目是一个基于 GitOps 理念的 Java Spring Boot 应用 DevOps 实践项目，采用 Kubernetes + Harbor + GitLab + ArgoCD 的完整技术栈。

## 项目架构

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   GitLab CI     │    │   Harbor        │    │   Kubernetes    │
│   + Runner      │───▶│   Registry      │───▶│   Cluster       │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Kustomize     │    │   ArgoCD        │    │   Application   │
│   (GitOps)      │    │   (CD)          │    │   Deployment    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

## 第一步：Kubernetes 集群搭建

### 1.1 环境准备
```bash
# 系统要求
- Ubuntu 20.04+ 或 CentOS 7+
- 至少 2GB RAM
- 至少 2 CPU 核心
- 网络连接

# 关闭防火墙和 SELinux
systemctl stop firewalld
systemctl disable firewalld
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```


### 1.2 安装 Kubernetes
```bash
hostnamectl set-hostname k8s01
hostnamectl set-hostname k8s02
hostnamectl set-hostname k8s03

# 卸载 Docker
sudo yum remove -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin
sudo rm -rf /var/lib/docker /etc/docker /etc/containerd

# 清理残留配置
sudo rm -rf /etc/systemd/system/docker.service.d
sudo rm -rf ~/.docker

sudo kubeadm reset -f
sudo yum remove -y kubelet kubectl kubeadm
sudo rm -rf /etc/kubernetes ~/.kube

sudo ip link delete cni0
sudo ip link delete flannel.1
sudo rm -rf /var/lib/cni/

wget https://github.com/labring/sealos/releases/download/v4.3.0/sealos_4.3.0_linux_amd64.tar.gz
tar -zxvf sealos_4.3.0_linux_amd64.tar.gz sealos
sudo mv sealos /usr/local/bin && chmod +x /usr/local/bin/sealos

sealos run labring/kubernetes:v1.25.0 \
  labring/helm:v3.8.2 \
  labring/calico:v3.24.1 \
  --masters <MASTER_IP> \
  --nodes <NODE_IPS> \
  --passwd <PASSWORD>
  
sealos run labring/kubernetes:v1.25.0 \
  labring/helm:v3.8.2 \
  labring/calico:v3.24.1 \
  --masters 192.168.31.106 \
  --nodes 192.168.31.107,192.168.31.108 \
  --passwd  123456
```

### 1.3 验证集群
```bash
# 检查节点状态
kubectl get nodes

# 检查 Pod 状态
kubectl get pods --all-namespaces
```

### 1.4 安装组件-ingress
```bash
ingress
# 安装ingress + metallb提供对外访问
curl -k https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/baremetal/deploy.yaml  -o deploy.yaml

# 修改配置文件
vim deploy.yaml
```
```
[root@k8s-master01 ~]# vim deploy.yaml

334 apiVersion: v1
335 kind: Service
336 metadata:
337   labels:
338     app.kubernetes.io/component: controller
339     app.kubernetes.io/instance: ingress-nginx
340     app.kubernetes.io/name: ingress-nginx
341     app.kubernetes.io/part-of: ingress-nginx
342     app.kubernetes.io/version: 1.3.0
343   name: ingress-nginx-controller
344   namespace: ingress-nginx
345 spec:
346   ipFamilies:
347   - IPv4
348   ipFamilyPolicy: SingleStack
349   ports:
350   - appProtocol: http
351     name: http
352     port: 80
353     protocol: TCP
354     targetPort: http
355   - appProtocol: https
356     name: https
357     port: 443
358     protocol: TCP
359     targetPort: https
360   selector:
361     app.kubernetes.io/component: controller
362     app.kubernetes.io/instance: ingress-nginx
363     app.kubernetes.io/name: ingress-nginx
364   type: LoadBalancer

把364行修改为LoadBalancer
365 ---
366 apiVersion: v1
367 kind: Service
```
```
[root@k8s-master01 ~]# kubectl apply -f deploy.yaml

验证部署：
[root@k8s-master01 ~]# kubectl get pod -n ingress-nginx
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-rksbr        0/1     Completed   0          104s
ingress-nginx-admission-patch-zg8f8         0/1     Completed   3          104s
ingress-nginx-controller-6dc865cd86-7rvfx   0/1     Running     0          104s
[root@k8s-master01 ~]# kubectl get all -n ingress-nginx
NAME                                            READY   STATUS      RESTARTS   AGE
pod/ingress-nginx-admission-create-rksbr        0/1     Completed   0          111s
pod/ingress-nginx-admission-patch-zg8f8         0/1     Completed   3          111s
pod/ingress-nginx-controller-6dc865cd86-7rvfx   1/1     Running     0          111s
NAME                                         TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)                      AGE
service/ingress-nginx-controller             LoadBalancer   10.96.2.211   <pending>     80:32210/TCP,443:32004/TCP   111s
service/ingress-nginx-controller-admission   ClusterIP      10.96.0.20    <none>        443/TCP                      111s
NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ingress-nginx-controller   1/1     1            1           111s
NAME                                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/ingress-nginx-controller-6dc865cd86   1         1         1       111s
NAME                                       COMPLETIONS   DURATION   AGE
job.batch/ingress-nginx-admission-create   1/1           43s        111s
job.batch/ingress-nginx-admission-patch    1/1           68s        111s
```

### 1.5 安装组件-metallb
```
[root@k8s-master01 ~]# kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.12.1/manifests/namespace.yaml
[root@k8s-master01 ~]# kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.12.1/manifests/metallb.yaml

[root@k8s-master01 ~]# vim metallb-conf.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.10.90-192.168.10.100
192.168.10.90-192.168.10.100是集群节点服务器IP同一段。      
[root@k8s-master01 ~]# kubectl apply -f metallb-conf.yaml
```
```
[root@k8s-master01 ~]# kubectl get svc -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.96.2.211   192.168.10.90   80:32210/TCP,443:32004/TCP   7m49s
ingress-nginx-controller-admission   ClusterIP      10.96.0.20    <none>          443/TCP                      7m49s
```

## 第二步：Harbor 镜像仓库搭建

### 2.1 安装 Harbor
```bash
# 下载 Harbor
wget https://github.com/goharbor/harbor/releases/download/v2.8.0/harbor-offline-installer-v2.8.0.tgz
tar -xzf harbor-offline-installer-v2.8.0.tgz
cd harbor

# 配置 Harbor
cp harbor.yml.tmpl harbor.yml
```

### 2.2 配置 Harbor
```yaml
# harbor.yml 配置示例
hostname: harbor.yourdomain.com
https:
  port: 443
  certificate: /your/certificate/path
  private_key: /your/private/key/path
harbor_admin_password: Harbor12345
database:
  password: root123
  max_idle_conns: 50
  max_open_conns: 100
```

### 2.3 启动 Harbor
```bash
# 执行安装脚本
./prepare
./install.sh

# 检查服务状态
docker-compose ps
```

### 2.4 配置 Docker 客户端
```bash
# 配置 Docker 信任 Harbor 仓库
cat > /etc/docker/daemon.json << EOF
{
  "insecure-registries": ["harbor.yourdomain.com"]
}
EOF
systemctl restart docker
```

## 第三步：GitLab + Runner + Kustomize

### 3.1 安装 GitLab
```bash
# 使用 Docker 安装 GitLab
docker run --detach \
  --hostname gitlab.yourdomain.com \
  --publish 443:443 --publish 80:80 --publish 22:22 \
  --name gitlab \
  --restart always \
  --volume $GITLAB_HOME/config:/etc/gitlab \
  --volume $GITLAB_HOME/logs:/var/log/gitlab \
  --volume $GITLAB_HOME/data:/var/lib/gitlab \
  gitlab/gitlab-ce:latest
```

### 3.2 配置 GitLab Runner
```bash
# 注册 GitLab Runner
gitlab-runner register \
  --url "http://gitlab.yourdomain.com/" \
  --registration-token "YOUR_REGISTRATION_TOKEN" \
  --executor "docker" \
  --docker-image "alpine:latest" \
  --description "docker-runner" \
  --tag-list "docker,aws" \
  --run-untagged="true" \
  --locked="false" \
  --access-level="not_protected"
```

### 3.3 安装 Kustomize
```bash
# 安装 Kustomize
[root@gitlab ~]# curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh"  | bash
{Version:kustomize/v4.5.7 GitCommit:56d82a8378dfc8dc3b3b1085e5a6e67b82966bd7 BuildDate:2022-08-02T16:35:54Z GoOs:linux GoArch:amd64}
kustomize installed to //root/kustomize
[root@gitlab ~]# mv /root/kustomize /usr/bin/
```

### 3.4 配置 GitLab 变量
在 GitLab 项目设置中配置以下变量：
- `ci_registry`: Harbor 仓库地址
- `ci_registry_name`: Harbor 用户名
- `ci_registry_passwd`: Harbor 密码
- `CI_USERNAME`: GitLab 用户名
- `CI_PASSWORD`: GitLab 密码

## 第四步：GitLab CI 配置

### 4.1 项目结构
```
gitops-project/
├── .gitlab-ci.yml          # CI/CD 配置文件
├── Dockerfile              # Docker 镜像构建文件
├── pom.xml                 # Maven 项目配置
├── kustomization.yaml      # Kustomize 配置
├── gitops-deployment.yaml  # Kubernetes Deployment
├── gitops-service.yaml     # Kubernetes Service
└── gitops-ingress.yaml     # Kubernetes Ingress
```

### 4.2 CI/CD 流程说明
```yaml
# .gitlab-ci.yml 主要阶段
stages:
  - build    # 构建阶段：编译代码、构建镜像、推送镜像
  - deploy   # 部署阶段：更新 Kustomize 配置、提交代码
```

### 4.3 构建阶段详解
1. **build code**: 使用 Maven 编译 Java 项目
2. **docker build**: 构建 Docker 镜像
3. **docker tag**: 为镜像打标签
4. **docker push**: 推送镜像到 Harbor

### 4.4 部署阶段详解
1. **deploy dev**: 使用 Kustomize 更新镜像版本
2. 自动提交更新后的配置文件
3. 触发 ArgoCD 同步

## 第五步：ArgoCD 配置

### 5.1 安装 ArgoCD
```bash
# 创建 ArgoCD 命名空间
kubectl create namespace argocd

# 安装 ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 获取管理员密码
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### 5.2 配置 ArgoCD 应用
```yaml
# gitops-project-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gitops-project
  namespace: argocd
spec:
  project: default
  source:
    repoURL: http://gitlab.yourdomain.com/your-group/gitops-project.git
    targetRevision: main
    path: .
  destination:
    server: https://kubernetes.default.svc
    namespace: gitops-project
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

### 5.3 应用配置
```bash
# 应用 ArgoCD 配置
kubectl apply -f gitops-project-app.yaml

# 检查应用状态
kubectl get applications -n argocd
```

## 使用说明

### 开发流程
1. 开发人员在本地修改代码
2. 提交代码到 GitLab 的 main 分支
3. GitLab CI 自动触发构建流程
4. 构建完成后自动更新 Kustomize 配置
5. ArgoCD 检测到配置变更，自动部署到 Kubernetes

### 监控和日志
```bash
# 查看应用状态
kubectl get pods -n gitops-project

# 查看应用日志
kubectl logs -f deployment/gitops-project -n gitops-project

# 查看 ArgoCD 同步状态
kubectl get applications -n argocd
```

## 故障排除

### 常见问题
1. **镜像拉取失败**: 检查 Harbor 配置和网络连接
2. **构建失败**: 检查 Maven 依赖和代码语法
3. **部署失败**: 检查 Kubernetes 资源配置
4. **同步失败**: 检查 ArgoCD 配置和 Git 仓库权限

### 日志查看
```bash
# GitLab Runner 日志
gitlab-runner --debug run

# ArgoCD 日志
kubectl logs -f deployment/argocd-server -n argocd

# 应用日志
kubectl logs -f deployment/gitops-project -n gitops-project
```

## 安全建议

1. **网络安全**: 使用 HTTPS 和防火墙
2. **访问控制**: 配置 RBAC 和网络策略
3. **镜像安全**: 定期扫描镜像漏洞
4. **密钥管理**: 使用 Kubernetes Secrets 管理敏感信息

---

**注意**: 请根据实际环境修改域名、IP 地址和配置参数。 