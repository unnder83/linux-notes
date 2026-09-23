# Linux 运维十课复习笔记 · 中篇（L4 ~ L7）

> 本篇覆盖：L4 用户与权限 / L5 进程与服务管理 / L6 网络排查 / L7 磁盘与存储
> 每节末尾附「踩过的坑」。

---

## L4 用户与权限（安全基线）

### 4.1 权限模型：三组人 × 三种权限

| 身份 | 含义 |
|---|---|
| **owner（属主）** | 文件主人，通常是创建者 |
| **group（属组）** | 同组用户共享一套权限 |
| **other（其他人）** | 以上两类以外的所有人 |

三种权限：`r` 读（4）、`w` 写（2）、`x` 执行（1）。

### 4.2 读权限位

```
-rw-r--r--    →    644   （文件最常见）
 rwxr-xr-x    →    755   （目录/脚本最常见）
 rwxrwxrwx    →    777   （安全红线！）
```
```
-   rw-   r--   r--
类型 属主  属组  其他
```

### 4.3 核心命令

```bash
chmod 755 文件          # 数字模式：一次性设三段
chmod 640 文件          # 属主 rw，组 r，其他无
chmod u+x 文件          # 符号模式：给属主加执行（u=owner g=group o=other a=all）
chmod g-w,o= 文件       # 组去写，其他人清空
chmod -R 755 目录       # -R 递归 ⚠️ 高危，慎用

chown 用户 文件             # 改属主
chown 用户:组 文件          # 同时改属主+属组
chgrp 组 文件               # 只改组
```

### 4.4 用户与组管理

```bash
sudo useradd -m -s /bin/bash 用户   # 建用户（-m 建家目录，-s 指定 shell）
sudo passwd 用户                    # 设密码
sudo userdel -r 用户                # 删用户（-r 连家目录一起删）
sudo groupadd 组                    # 建组
sudo usermod -aG 组 用户            # 把用户加进组（-a 追加，不加会清掉原组）
id 用户                             # 查看用户的 UID、组
whoami                              # 我是谁
su - 用户                           # 切换到该用户（带环境变量）
sudo -u 用户 命令                   # 以该用户身份执行命令（测权限）
```

**用户信息存放文件**（都是冒号分隔，配合 awk -F: 使用）：
- `/etc/passwd`：用户名、UID、家目录、shell
- `/etc/shadow`：密码哈希（不存明文）
- `/etc/group`：组名、GID、成员

### 4.5 umask —— 新文件的默认权限

- Ubuntu 默认 `022`：新文件 = 666 - 022 = **644**，新目录 = 777 - 022 = **755**
- `umask` 查看，`umask 022` 修改

### 4.6 sudo 与 visudo

- `sudo` = 临时提权，用户加入 `sudo` 组即可用
- 改 sudo 配置**必须用 `visudo`**（有语法检查），禁止 `vim /etc/sudoers`（写错一个字符就锁死整个 sudo）

### 4.7 目录权限的特殊含义（重点！）

| 权限 | 文件 | 目录 |
|---|---|---|
| r | 读内容 | 列出文件名（ls 看得到） |
| w | 改内容 | 在里面创建/删除文件 |
| x | 执行 | 进入目录、访问里面文件（cd + 读文件） |

**关键结论**：
1. 要读目录里的文件，**必须 r + x**（光 r 只能看文件名，进不去）。
2. **删除文件由"目录的 w"决定，不是文件本身权限**（rm 删的是目录项/门牌，回想 unlink）。
3. "只读目录" = `r-x`（5），不是 `r--`（4）。

**正确设计"只读"访问（组方案）**：
```bash
sudo groupadd appreaders
sudo chgrp -R appreaders /srv/app
sudo chmod -R 750 /srv/app        # 属主 rwx，组 r-x，其他人 0
sudo usermod -aG appreaders 新同事
```

---

### 🕳 L4 踩过的坑

1. **`chmod 764` 给了组写权限**：764 = rwx rw- r--，属组的"6"= rw-，违背"只读"。→ 只读方案用 750，组给 r-x（5）。
2. **"新同事大概不在组里"靠猜**：权限设计要确定性，用 groupadd + usermod -aG 明确管理，用 `id` 验证。
3. **目录只读写成 r--（444）**：能 ls 但 cat 报 Permission denied。→ 必须 r-x（555）。
4. **组名前后不一致**：`groupadd new` 却 `chgrp -R appdev`，导致方案静默失效。→ 组名全程统一，最后 `id 用户` 验证。
5. **删除文件由目录 w 决定**（易忽略）：想"任何人能读、只有属主能删"→ 目录给 755。

---

## L5 进程与服务管理（核心维稳）

### 5.1 进程基础

- 进程 = 程序在内存中的运行实例，有唯一 **PID**、父进程 **PPID**
- 系统第一个进程是 **systemd（PID 1）**，是所有进程的"老祖宗"

### 5.2 查看进程

```bash
ps aux               # BSD 风格：USER/PID/%CPU/%MEM/命令（最常用）
ps -ef               # System V 风格：多 PPID 列
ps -p $$             # 看当前 shell 自己的进程（$$ 是当前 shell PID）
ps -p $$ -o pid,ppid,cmd   # -o 指定输出列（查 PPID 用这个）
ps aux | grep 关键字  # 过滤找进程
pstree               # 树状图显示父子关系
top                  # 实时动态（P 按 CPU 排序，M 按内存，q 退出）
htop                 # 彩色增强版（需 apt install htop）
```

### 5.3 信号（kill 的本质）

`kill` 是"发信号"，不是"杀进程"：

| 信号 | 编号 | 含义 |
|---|---|---|
| SIGTERM | 15 | 优雅退出（进程自己清理现场再退，默认信号） |
| SIGKILL | 9 | 强制杀死（没机会收尾，可能损坏数据） |
| SIGINT | 2 | 相当于 Ctrl+C |
| SIGHUP | 1 | 通知进程重读配置 |

```bash
kill PID              # 默认发 SIGTERM(15)
kill -9 PID           # 强制 SIGKILL（最后手段）
kill -TERM PID        # 显式指定
pkill -f 关键字        # 按名字匹配杀
pgrep -l 关键字        # 按名字找 PID
jobs                  # 看后台任务
```

### 5.4 systemd 与 systemctl

```bash
systemctl status 服务    # 状态（是否 running / enabled、最近日志）
systemctl start 服务     # 启动
systemctl stop 服务      # 停止
systemctl restart 服务   # 重启（会中断服务）
systemctl reload 服务    # 热重载配置（不中断，优先用）
systemctl enable 服务    # 开机自启
systemctl disable 服务   # 取消自启
systemctl list-units --type=service --state=running   # 所有运行中的服务
```

### 5.5 journalctl —— systemd 日志

```bash
journalctl -xe             # 最近系统日志（-e 跳到末尾，-x 带解释）
journalctl -u 服务 -n 20   # 某服务最近 20 行
journalctl -u 服务 -f      # 实时跟踪
journalctl --since "1 hour ago"   # 时间段过滤
```

---

### 🕳 L5 踩过的坑

1. **PPID 不会查**：`ps -p $$ -o pid,ppid,cmd` 或 `echo $PPID`。
2. **盲目 restart 三连的危害**：①治标不治本（配置错/端口占/磁盘满，重启照样挂）②销毁证据（现场日志/内存状态没了）③丢请求（服务可能只是卡住而非死）。→ 正确顺序：看日志找根因 → 修根因 → 再重启 → 验证。
3. **SSH 连着时别 `systemctl restart ssh`**：会断自己的连接。演示用无害服务（如 cron）。
4. **kill 前先 `ps`/`pgrep` 确认 PID**：PID 会复用，杀错 = 事故。

---

## L6 网络排查（占比极高）

### 6.1 核心思想：按"链路分段"定位（端口不通四段式）

```
① 服务进程      ② 本机端口      ③ 防火墙       ④ 网络连通      ⑤ 对方服务
ps/status  →  ss -tlnp    →  iptables/ufw → ping/route  →  telnet/curl
```

**四段式（背下来）**：
1. 服务监听了吗：`ss -tlnp | grep 端口`
2. 本机通吗：`curl 127.0.0.1:端口` 或 `telnet 127.0.0.1 端口`
3. 防火墙挡了吗：`sudo ufw status` / `sudo iptables -L -n`
4. 外部通吗：从另一台机器 `telnet IP 端口`

### 6.2 命令详解

```bash
ip addr                  # 看网卡 IP、状态（UP/DOWN），替代 ifconfig
ip route                 # 看路由表，找默认网关（default via ...）
ip link                  # 看网卡链路层状态
ping -c 4 目标           # -c 限制次数（否则一直 ping）
ss -tlnp                 # 看本机监听 TCP 端口和进程（-t TCP -l 监听 -n 数字 -p 进程）
ss -tunlp                # 加 -u 含 UDP
ss -tlnp | grep 22       # 查某端口
telnet IP 端口           # 测远程端口通不通
nc -zv IP 端口           # netcat 测端口（-z 扫描 -v 详细）
curl -I http://IP:端口   # HTTP 测试（-I 只看响应头）
curl -v http://...       # 详细过程
sudo iptables -L -n      # 看防火墙规则
sudo ufw status          # Ubuntu 防火墙前端状态
dig 域名                 # DNS 解析排查
nslookup 域名            # DNS 排查（老工具）
```

### 6.3 三个易误解点

1. **`ping` 不通 ≠ 服务不通**：服务器常禁 ICMP，ping 被丢但服务正常。ping 测"机器活没活"（ICMP），服务通不通要测"端口"（TCP）。
2. **`0.0.0.0:22` vs `127.0.0.1:22`**：0.0.0.0 = 监听所有网卡（全网可访问）；127.0.0.1 = 只监听回环（仅本机可访问）。"外部访问不了"常因服务只听 127.0.0.1。
3. **`ss` 看不到端口 = 服务没监听**（没起/配置错/监听地址错）。

---

### 🕳 L6 踩过的坑

1. **`ss -tlnp` 看不到进程名**：`-p` 只能显示**自己**的进程，别人（root/mysql 用户）的进程看不到。→ 加 `sudo ss -tlnp`。
2. **"ping 不通"和"服务不通"分不清**：一个是 ICMP 层，一个是 TCP 层，两回事。

---

## L7 磁盘与存储

### 7.1 磁盘三层结构

```
物理磁盘 /dev/sda
   → 分区 /dev/sda1、/dev/sda2...
       → 格式化（文件系统 ext4/xfs）
           → 挂载（mount）到目录树某点（如 / 或 /home）
```
Linux 没有 C 盘 D 盘，所有分区挂载在 `/` 这棵树上。

### 7.2 命令详解

```bash
df -h                  # 文件系统用量（容量/已用/可用/挂载点），重点看 Use%
df -i                  # inode 用量（文件"数量"，重点看 IUse%）
df -B1M ~              # 指定单位 1MB（观察小变化用，比 -h 精确）
du -sh 目录            # 某目录总共占多大
du -h --max-depth=1 目录 | sort -rh | head   # 分层找大目录（钻取法）
lsblk                  # 块设备层级（NAME/SIZE/TYPE/MOUNTPOINTS）
sudo fdisk -l          # 磁盘分区表
mount                  # 当前挂载情况
find 目录 -size +10M 2>/dev/null   # 找大文件
```

### 7.3 两个经典"对不上"

1. **df 和 du 对不上**：`df` 已用 10G，`du` 加起来只有 6G → 有"幽灵文件"（文件被 rm 但进程还占着 inode）。`df` 按文件系统算，`du` 按目录遍历（名字没了遍历不到）。
2. **空间够但建不了文件**：inode 耗尽（小文件太多）。用 `df -i` 看 IUse% 是否 100%。

### 7.4 磁盘满排查"钻取法"

```bash
df -h                                        # 1. 找满的分区（如 /）
du -h --max-depth=1 / 2>/dev/null | sort -rh | head   # 2. 看 / 下哪些一级目录大
du -h --max-depth=1 /var/lib 2>/dev/null | sort -rh | head  # 3. 对最大的继续下钻
```
逐层钻到具体文件，**不要预设某个目录**（可能是 /home、/var/lib、/var/cache 等）。

### 7.5 幽灵文件（unlink 的完整故事）

- `lsof +L1`：列出"已删除但仍被占用"的文件（link count = 0，NAME 列带 `(deleted)`）
- `lsof +L1` 输出关键列：**NLINK=0**（硬证据）+ `(deleted)`（注释）
- 释放方法：`kill`/重启占用进程，或 `> /proc/PID/fd/N` 清空（不杀进程），或配 logrotate

---

### 🕳 L7 踩过的坑

1. **`df` 和 `du` 搞混**：下钻找大目录用 `du --max-depth`，`df` 没有 `--max-depth` 参数。口诀：df 看"分区"，du 看"目录"。
2. **预设元凶是 /var/log**：磁盘满可能是 /home、/var/lib 等。→ 从满的分区顶层钻取。
3. **inode 耗尽答成 `fdisk -l`**：`fdisk -l` 是看分区表，inode 要看 `df -i`。
4. **"直接 kill PID"释放幽灵文件**：占用幽灵文件的进程往往是服务本身，kill = 停服。→ 先确认身份（ps -p PID），能用 logrotate/`/proc/fd` 清空就别 kill。
5. **50MB 太小 `df -h` 看不出来**：0.05G 被四舍五入。→ 验证空间变化用 `df -B1M` 或把文件造大（300MB）。
6. **lsof 按路径查已删文件**：`lsof 文件` 报 No such file（名字没了），要用 `lsof +L1`。

---

## 中篇速查表

| 需求 | 命令 |
|---|---|
| 建用户（家目录+bash） | `sudo useradd -m -s /bin/bash 用户` |
| 用户加进组 | `sudo usermod -aG 组 用户` |
| 设权限 | `chmod 644/755 文件` |
| 查进程+PPID | `ps -p $$ -o pid,ppid,cmd` |
| 优雅结束进程 | `kill PID`（默认 TERM） |
| 查服务状态 | `systemctl status 服务` |
| 看网卡/网关 | `ip addr` / `ip route` |
| 看监听端口 | `sudo ss -tlnp` |
| 测端口通不通 | `nc -zv IP 端口` |
| 看磁盘用量 | `df -h` / `df -i` |
| 找大目录 | `du -h --max-depth=1 / | sort -rh | head` |
| 找幽灵文件 | `lsof +L1` |
