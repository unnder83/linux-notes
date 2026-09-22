# Docker Compose

## 学习目标

- [ ] 理解 Compose 解决的问题（多容器编排）
- [ ] 掌握 docker-compose.yml 基本写法
- [ ] 能用 Compose 一键启动 Web + 数据库
- [ ] 掌握常用命令

## 示例

```yaml
services:
  web:
    image: nginx
    ports:
      - "80:80"
    volumes:
      - ./html:/usr/share/nginx/html
    depends_on:
      - db

  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: 123456
    volumes:
      - db_data:/var/lib/mysql

volumes:
  db_data:
```

## 常用命令

```bash
docker compose up -d
docker compose ps
docker compose logs -f
docker compose down
```

## 笔记

> 边学边补充

## 实操记录

> 待补充

## 踩坑记录

> 待补充
