# K8s 核心概念（简单学）

## 学习目标

- [ ] 理解 K8s 解决什么问题（容器编排）
- [ ] 掌握核心概念：Pod、Deployment、Service
- [ ] 能搭一个单机环境（minikube 或 k3s）
- [ ] 会部署一个应用并暴露访问

## 核心概念

| 概念 | 说明 |
| --- | --- |
| Pod | 最小调度单位，一个或多个容器 |
| Deployment | 管理 Pod 的副本和滚动更新 |
| Service | 给 Pod 提供稳定访问入口 |
| Namespace | 资源隔离 |
| ConfigMap / Secret | 配置和密钥管理 |

## 常用命令

```bash
kubectl get pods -A
kubectl get nodes
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl describe pod xxx
kubectl logs xxx
```

## 笔记

> 边学边补充（学习阶段不用深入，知道是什么、会跑起来即可）

## 踩坑记录

> 待补充
