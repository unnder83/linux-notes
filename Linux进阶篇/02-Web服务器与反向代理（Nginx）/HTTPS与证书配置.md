# HTTPS 与证书配置

## 学习目标

- [ ] 理解 HTTPS 握手流程和证书的作用
- [ ] 会用 openssl 生成自签名证书
- [ ] 掌握 Nginx 配置 HTTPS（证书 + 跳转）
- [ ] 了解 Let's Encrypt 免费证书申请

## 核心配置

```nginx
server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate     /etc/nginx/ssl/example.com.crt;
    ssl_certificate_key /etc/nginx/ssl/example.com.key;

    ssl_protocols TLSv1.2 TLSv1.3;
}

server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;   # HTTP 跳 HTTPS
}
```

## 自签名证书

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout example.key -out example.crt
```

## 笔记

> 边学边补充

## 踩坑记录

> 待补充
