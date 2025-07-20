# DevOps 面试项目完整指南

## 项目概述

这是一个专为 DevOps 工程师面试准备的完整项目案例，展示了两种主流的 DevOps 实践方案：

1. **GitOps 方案** - 基于 GitLab + ArgoCD + Kubernetes 的现代化 GitOps 实践
2. **Jenkins 方案** - 基于 Jenkins + Kubernetes 的传统 CI/CD 实践

## 项目架构

```
devops-sample/
├── gitops-project/          # GitOps 实践项目
│   ├── src/                # Spring Boot 应用源码
│   ├── Dockerfile          # 容器化配置
│   ├── kustomization.yaml  # Kustomize 配置
│   └── README.md           # GitOps 详细部署指南
├── jenkins-project/         # Jenkins CI/CD 实践项目
│   ├── src/                # Spring Boot 应用源码
│   ├── Jenkinsfile         # Jenkins Pipeline 配置
│   └── HELP.md             # 项目帮助文档
└── k8s/                    # Kubernetes 配置文件
    ├── deploy.yaml         # 应用部署配置
    ├── ingress-argocd.yaml # Ingress 和 ArgoCD 配置
    └── metallb-conf.yaml   # MetalLB 负载均衡配置
```

## 技术栈对比

| 组件 | GitOps 方案 | Jenkins 方案 |
|------|-------------|--------------|
| **CI/CD 工具** | GitLab CI + ArgoCD | Jenkins Pipeline |
| **容器编排** | Kubernetes | Kubernetes |
| **镜像仓库** | Harbor | Harbor |
| **配置管理** | Kustomize | Kubernetes YAML |
| **部署方式** | GitOps (声明式) | 传统 CI/CD (命令式) |
| **优势** | 自动化程度高、审计友好 | 灵活性高、生态成熟 |

## 快速开始

### 方案一：GitOps 实践

GitOps 是一种现代化的 DevOps 实践，通过 Git 作为单一真实来源来管理应用部署。

**核心特点：**
- 基于 Git 的声明式部署
- 自动化的应用同步
- 完整的审计追踪
- 快速回滚能力

**技术栈：**
- GitLab (代码仓库 + CI)
- Harbor (镜像仓库)
- ArgoCD (持续部署)
- Kubernetes (容器编排)
- Kustomize (配置管理)

**详细指南：** [GitOps 项目文档](./gitops-project/README.md)

### 方案二：Jenkins CI/CD 实践

Jenkins 是业界最成熟的 CI/CD 工具之一，提供强大的流水线能力。

**核心特点：**
- 灵活的流水线配置
- 丰富的插件生态
- 强大的脚本能力
- 详细的构建日志

**技术栈：**
- Jenkins (CI/CD 服务器)
- Harbor (镜像仓库)
- Kubernetes (容器编排)
- Docker (容器化)

**详细指南：** [Jenkins 项目文档](./jenkins-project/)

## 环境要求

### 硬件要求
- **CPU**: 至少 4 核心
- **内存**: 至少 8GB RAM
- **存储**: 至少 50GB 可用空间
- **网络**: 稳定的网络连接

### 软件要求
- **操作系统**: Ubuntu 20.04+ 或 CentOS 7+
- **Docker**: 20.10+
- **Kubernetes**: 1.25+
- **Git**: 2.30+

## 部署步骤

### 1. 基础环境准备
```bash
# 克隆项目
git clone <repository-url>
cd devops-sample

# 检查环境
kubectl version
docker --version
git --version
```

### 2. 选择部署方案

#### GitOps 方案部署
```bash
# 进入 GitOps 项目目录
cd gitops-project

# 按照 README.md 中的步骤进行部署
# 1. 搭建 Kubernetes 集群
# 2. 安装 Harbor 镜像仓库
# 3. 配置 GitLab + Runner
# 4. 部署 ArgoCD
# 5. 配置应用部署
```

#### Jenkins 方案部署
```bash
# 进入 Jenkins 项目目录
cd jenkins-project

# 按照项目文档进行部署
# 1. 安装 Jenkins
# 2. 配置 Jenkins Pipeline
# 3. 配置 Harbor 集成
# 4. 部署应用到 Kubernetes
```

## 项目特色

### 🚀 现代化实践
- 采用最新的 DevOps 工具和理念
- 完整的容器化部署方案
- 自动化的 CI/CD 流程

### 📚 学习友好
- 详细的部署文档
- 清晰的架构说明
- 完整的配置示例

### 🎯 面试准备
- 涵盖主流 DevOps 技术栈
- 展示实际项目经验
- 体现技术深度和广度

### 🔧 生产就绪
- 完整的错误处理
- 安全配置考虑
- 可扩展的架构设计

## 常见问题

### Q: 两个方案有什么区别？
A: GitOps 方案更现代化，自动化程度高；Jenkins 方案更传统，灵活性更强。建议根据团队技术栈和需求选择。

### Q: 需要多少台服务器？
A: 最小配置需要 3 台服务器（1 台 Master + 2 台 Worker），生产环境建议更多节点。

### Q: 如何自定义应用？
A: 修改 `src/` 目录下的 Spring Boot 应用代码，然后重新构建和部署。

### Q: 支持哪些编程语言？
A: 当前示例使用 Java Spring Boot，但架构支持任何可以容器化的应用。

## 贡献指南

欢迎提交 Issue 和 Pull Request 来改进这个项目！

## 许可证

本项目采用 MIT 许可证，详见 [LICENSE](LICENSE) 文件。

## 联系方式

如有问题或建议，请通过以下方式联系：
- 提交 GitHub Issue
- 发送邮件至：[1518483751@qq.com]

---

**注意**: 这是一个用于学习和面试准备的项目，生产环境使用前请进行充分测试和配置调整。 