# Dockerfile 与镜像构建

## 学习目标

- [ ] 掌握 Dockerfile 常用指令
- [ ] 能构建自己的镜像并运行
- [ ] 理解镜像分层和构建缓存
- [ ] 了解多阶段构建

## 常用指令

| 指令 | 作用 |
| --- | --- |
| FROM | 基础镜像 |
| WORKDIR | 工作目录 |
| COPY / ADD | 复制文件 |
| RUN | 构建时执行命令 |
| ENV | 环境变量 |
| EXPOSE | 声明端口 |
| CMD / ENTRYPOINT | 启动命令 |

## 示例

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
EXPOSE 8000
CMD ["python", "app.py"]
```

```bash
docker build -t myapp:v1 .
```

## 笔记

> 边学边补充（CMD 和 ENTRYPOINT 的区别要搞清）

## 踩坑记录

> 待补充
