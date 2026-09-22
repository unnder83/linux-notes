# Docker 基础

## 学习目标

- [ ] 理解容器和虚拟机的区别
- [ ] 掌握 Docker 的安装和换源（国内加速）
- [ ] 理解镜像、容器、仓库三个核心概念
- [ ] 会跑第一个容器并进入交互

## 核心概念

| 概念 | 说明 | 类比 |
| --- | --- | --- |
| 镜像 image | 只读模板 | 类 |
| 容器 container | 镜像的运行实例 | 对象 |
| 仓库 registry | 存放镜像的地方 | 应用商店 |

## 常用命令

```bash
docker run -d -p 80:80 --name web nginx
docker ps -a
docker exec -it web bash
docker stop web && docker rm web
```

## 笔记

> 边学边补充（数据卷、网络模式是重点）

## 踩坑记录

> 待补充
