# Docker 镜像与容器操作

## 学习目标

- [ ] 掌握镜像的拉取、查看、删除、导出导入
- [ ] 掌握容器的生命周期管理
- [ ] 理解数据卷挂载（-v）和端口映射（-p）
- [ ] 会查看容器日志和资源占用

## 命令速查

```bash
# 镜像
docker pull nginx:1.25
docker images
docker rmi nginx:1.25
docker save/load

# 容器
docker start/stop/restart/rm
docker logs -f --tail 100 web
docker inspect web
docker stats

# 数据卷
docker run -v /host/data:/container/data ...
```

## 笔记

> 边学边补充

## 实操记录

> 待补充

## 踩坑记录

> 待补充
