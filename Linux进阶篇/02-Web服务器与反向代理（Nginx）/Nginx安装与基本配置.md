# Nginx 安装与基本配置

## 学习目标

- [ ] 掌握 Nginx 的安装方式（yum/apt/源码）
- [ ] 理解主配置文件结构和 include 机制
- [ ] 掌握 server、location 的匹配规则
- [ ] 能配置虚拟主机（多站点）

## 配置文件结构

```
/etc/nginx/
├── nginx.conf              # 主配置
├── conf.d/                 # 子配置（推荐放这里）
└── sites-available/        # Ubuntu 风格
```

## 核心配置块

```nginx
server {
    listen 80;
    server_name example.com;
    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

## 常用命令

```bash
nginx -t              # 检查配置语法
nginx -s reload       # 平滑重载
systemctl reload nginx
```

## 笔记

> 边学边补充（location 匹配优先级是重点）

## 踩坑记录

> 待补充
