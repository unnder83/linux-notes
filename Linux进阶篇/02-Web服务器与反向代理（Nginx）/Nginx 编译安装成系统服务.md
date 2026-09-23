# 推荐流程：

### 1. 先安装编译依赖

CentOS/Rocky/Alma：

```bash
yum install -y gcc gcc-c++ make pcre pcre-devel zlib zlib-devel openssl openssl-devel
```

Ubuntu/Debian：

```bash
apt update
apt install -y build-essential libpcre3 libpcre3-dev zlib1g zlib1g-dev libssl-dev
```

### 2. 上传/下载并解压

```bash
scp nginx-1.24.0.tar.gz root@服务器:/usr/local/src/
cd /usr/local/src
tar -zxvf nginx-1.24.0.tar.gz
cd nginx-1.24.0
```

### 3. 配置、编译、安装

```bash
./configure --prefix=/usr/local/nginx --with-http_ssl_module --with-http_v2_module
make -j$(nproc)
make install
```

安装后主要文件：

```bash
/usr/local/nginx/sbin/nginx
/usr/local/nginx/conf/nginx.conf
/usr/local/nginx/logs/nginx.pid
```

测试：

```bash
/usr/local/nginx/sbin/nginx -t
```

### 4. 手动启动测试（可选）

```bash
/usr/local/nginx/sbin/nginx
curl -I http://127.0.0.1
```

如果测试完要交给 systemd，先停掉：

```bash
/usr/local/nginx/sbin/nginx -s stop
```

如果你没手动启动过，就不用停。

### 5. 配置 systemd 服务

创建：

```bash
vim /etc/systemd/system/nginx.service
```

内容示例：

```ini
[Unit]
Description=nginx
After=network-online.target
Wants=network-online.target

[Service]
Type=forking
PIDFile=/usr/local/nginx/logs/nginx.pid
ExecStartPre=/usr/local/nginx/sbin/nginx -t
ExecStart=/usr/local/nginx/sbin/nginx
ExecReload=/usr/local/nginx/sbin/nginx -s reload
ExecStop=/usr/local/nginx/sbin/nginx -s quit
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

### 6. 重载并启动

```bash
systemctl daemon-reload
systemctl enable --now nginx
systemctl status nginx
```

之后就用：

```bash
systemctl start nginx
systemctl stop nginx
systemctl reload nginx
systemctl restart nginx
```

### 总结：

**装依赖 → 上传解压 → ./configure → make && make install → 写 nginx.service → daemon-reload → enable/start**。



## Ubuntu 用 `apt install nginx` 后，大致位置：

```bash
/usr/sbin/nginx                 # nginx 主程序
/etc/nginx/                     # 配置文件
/var/log/nginx/                 # 日志
/var/www/html/                  # 默认网站目录
/lib/systemd/system/nginx.service  # systemd 服务
```
