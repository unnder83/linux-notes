# Linux 运维十课复习笔记 · 上篇（L1 ~ L3）

> 学员环境：VMware + Ubuntu 26.04
> 本篇覆盖：L1 环境认知与文件操作 / L2 日志排查基本功 / L3 文本三剑客与查找
> 每个知识点下方附「踩过的坑」，是本人实操中真实犯过的错，务必重点看。

---

## L1 环境认知与文件操作

### 1.1 Linux 是什么

- **Linux = 内核（kernel）+ 发行版（distribution）**
  - 内核：管理 CPU、内存、硬盘、网络的底层"发动机"
  - 发行版：内核 + 工具 + 包管理器的"整车"，Ubuntu 属于 Debian 系
- **一切皆文件**：硬盘是 `/dev/sda`、网卡是 `/dev/eth0`、进程是目录 `/proc/[pid]/`。好处是操作任何东西都用同一套文件命令。

### 1.2 目录树（Linux 的"公司组织架构"）

| 目录 | 类比 | 装什么 |
|---|---|---|
| `/` | 公司总部 | 一切从这里开始 |
| `/home/用户名` | 你的工位 | 你自己的文件（家目录，缩写 `~`） |
| `/etc` | 制度文件柜 | 所有软件的配置文件 |
| `/var/log` | 保安监控录像 | 系统和服务的日志（排障主战场） |
| `/tmp` | 茶水间 | 临时文件，系统会定期清理 |
| `/usr/bin` | 工具库 | 系统命令本体 |

### 1.3 路径：绝对 vs 相对

- **绝对路径**：从根开始，`/home/zhangsan/a.txt`，任何时候都指同一个地方
- **相对路径**：从当前位置开始。`.` = 当前目录，`..` = 上一级，`~` = 家目录

```bash
cd /etc      # 绝对路径
cd ..        # 上一级
cd ~         # 回家目录
cd -         # 回到上一次所在的目录（好用）
```

### 1.4 核心命令详解

#### pwd —— 我在哪
```bash
pwd          # print working directory，打印当前目录的绝对路径
```

#### ls —— 有什么
```bash
ls           # 列出当前目录（不显示隐藏文件）
ls -l        # 长格式：权限/属主/属组/大小/时间/文件名
ls -lh       # -h 把大小转成人类可读（K/M/G）
ls -la       # -a 显示隐藏文件（以 . 开头的文件）
ls -ld 目录  # -d 只看目录本身，不展开里面的内容
```

**`ls -l` 每一列的含义**：
```
-rw-r--r--  1 zhangsan zhangsan  12 Aug 13 10:00 a.txt
└┬┘└──┬──┘ └┬┘  └┬──┘  └─┬──┘
 类型  权限  链接  属主  属组   大小  时间  文件名
```

#### cd —— 去哪
```bash
cd /path/to/dir
```
> 配合 **Tab 键补全**，防手滑。

#### cp —— 复制
```bash
cp 源 目标           # 复制文件
cp -r 源目录 目标目录  # -r 递归复制目录
cp -i 源 目标        # -i 覆盖前询问
cp file file.bak     # 改配置前先备份（保命习惯）
```

#### mv —— 移动 / 改名
```bash
mv 源 目标           # 同分区内=改名（瞬间完成），跨分区=复制+删除
mv -i 源 目标        # -i 覆盖前询问
```

#### rm —— 删除（高危！）
```bash
rm 文件              # 删除（无回收站！）
rm -r 目录           # -r 递归删除目录
rm -f 文件           # -f 强制删除，不询问
rm -rf 目录          # ⚠️ 最危险组合，慎之又慎
rm -i 文件           # -i 删除前逐个询问
```

### 1.5 vim 生存四件套

| 操作 | 命令 | 说明 |
|---|---|---|
| 进入插入模式 | `i` | 在光标前开始输入 |
| 退回命令模式 | `Esc` | 停止输入 |
| 保存退出 | `:wq` | 写入并退出 |
| 不保存退出 | `:q!` | 放弃修改强制退出 |
| 搜索 | `/关键字` | 回车后按 `n` 跳下一个 |
| 删一行 | `dd` | 命令模式下 |
| 撤销 | `u` | 命令模式下 |

---

### 🕳 L1 踩过的坑

1. **目录名 `lab/l1` 写成了 `lab/11`**——字母 `l` 和数字 `1` 太像。→ 用 Tab 补全，别手敲。
2. **rm 无回收站**——删了就没了。→ 生产删任何东西前先 `ls` 看清楚，再想 3 秒；建议 `alias rm='rm -i'`。
3. **命令替换写错**：
   - ❌ `cp test.txt test"date +%F".txt.bak`（引号=字面量，shell 不执行）
   - ❌ `cp test.txt test-'date +%F'.txt.bak`（单引号同样是字面量）
   - ✅ `cp test.txt test-$(date +%F).txt.bak`（`$()` 才执行命令并替换输出）
4. **unlink 概念错误**：以为"Linux 和 Windows 一样，正在使用的文件删不掉"。
   - ✅ 真相：Linux **可以删除正在使用的文件**，`rm` 只是把文件名从目录摘掉（unlink），数据要等进程关闭文件才释放。所以 `rm` 后空间不释放是正常现象。
5. **"释放内存"说法错误**：`kill` 进程后释放的是 file descriptor / inode 引用 → 磁盘块回收，**不是内存（RAM）**。

---

## L2 日志排查基本功

### 2.1 日志是什么、在哪

- 日志 = 服务自己记录的"黑匣子"。排障第一动作永远是看日志。
- Ubuntu 日志位置：

| 文件 | 内容 |
|---|---|
| `/var/log/syslog` | 系统整体日志 |
| `/var/log/auth.log` | 登录记录 |
| `/var/log/kern.log` | 内核日志 |
| `/var/log/apt/history.log` | 装过什么软件 |

### 2.2 查看命令详解

#### cat —— 看全文（只适合小文件）
```bash
cat 文件          # 输出全文
cat -n 文件       # -n 带行号
```
> ⚠️ 大日志别 cat，会刷爆终端。

#### head / tail —— 看头尾
```bash
head 文件         # 默认前 10 行
head -20 文件     # 前 20 行
tail 文件         # 默认后 10 行
tail -20 文件     # 后 20 行
tail -f 文件      # -f 实时跟踪新写入的行（生产神器，Ctrl+C 退出）
tail -F 文件      # -F 文件被删除/轮转后自动重连（比 -f 更稳）
```

#### grep —— 过滤（核心中的核心）
```bash
grep ERROR 文件           # 只显示含 ERROR 的行
grep -i error 文件        # -i 忽略大小写
grep -c ERROR 文件        # -c 只输出匹配行数（count）
grep -v INFO 文件         # -v 反向，排除含 INFO 的行
grep -n ERROR 文件        # -n 带行号（定位位置必用！）
grep -A3 -B2 ERROR 文件   # -A 显示后 3 行，-B 显示前 2 行（看上下文）
grep -E "err|fail" 文件   # -E 扩展正则（多关键词）
grep -w error 文件        # -w 整词匹配（避免匹配到 error2）
grep -r "xxx" 目录        # -r 递归搜索目录
grep -m1 ERROR 文件       # -m 只取前 1 个匹配
```

#### wc / sort / uniq —— 统计
```bash
wc -l 文件           # -l 数行数
wc -c 文件           # -c 数字节
sort                 # 默认字典序排序
sort -n              # -n 按数字排序（字典序会把 10 排在 2 前）
sort -r              # -r 倒序
sort -rn             # 数字倒序
sort -V              # -V 版本序（正确处理 user2 < user10）
sort -u              # -u 去重
uniq                 # 去重（只能去相邻重复，常配合 sort）
uniq -c              # -c 统计每项出现次数
```

**经典组合**：统计出现次数排行
```bash
cat 文件 | sort | uniq -c | sort -rn
# 排序 → 去重计数 → 按次数倒序
```

#### 管道 `|`
把前一个命令的输出"喂"给下一个命令。日志排查几乎全靠管道串起来。

**统计错误最多的来源 IP**：
```bash
grep ERROR app.log | awk '{print $NF}' | sort | uniq -c | sort -rn
```

---

### 🕳 L2 踩过的坑

1. **`sort` 字典序坑**：`user10` 排在 `user2` 前面（逐字符比，'1' < '2'）。
   - 纯数字用 `sort -n`；混合名用 `sort -V`。
2. **"排行"漏了 `sort -rn`**：`uniq -c` 之后要再加一次 `sort -rn` 才是按数量倒序排行。
3. **作业文件整理混乱**：答案放错标题、旧内容没清、粘贴串行。
   - ✅ 每次提交新建文件、题目间加分隔线、提交前 `cat` 检查。
4. **"最后 8 行"理解偏差**：`grep INFO | tail -8` 是"最后 8 条 INFO"，`tail -8 文件` 才是"文件最后 8 行"。生产场景"看日志最后几行"要直接 tail，别自作主张加过滤。

---

## L3 文本三剑客与查找

### 3.1 find —— 文件系统的"搜索引擎"

```bash
find <从哪里找> <条件>

find ~/lab -name "*.log"          # -name 按名字（支持通配符 * ?）
find ~/lab -iname "*.LOG"         # -iname 忽略大小写
find ~/lab -type f                # -type f=文件，d=目录，l=链接
find ~/lab -size +100M            # -size +大于，-小于，不带号=约等于（c=字节，k/M/G）
find ~/lab -mtime -7              # -mtime 按修改时间：-7=7天内，+7=7天前，7=恰好
find ~/lab -name "*.tmp" -delete  # -delete 找到即删 ⚠️ 高危
find ~/lab -name "*.log" -exec ls -l {} \;   # -exec 对每个结果执行命令，{} 是占位符
find ~/lab -name "*.log" -exec rm {} +      # + 比 \; 更高效（一次传多个）
find ~/lab -perm 777              # -perm 按权限位查找
```

**关键规则**：
- `find` **不走管道**！它从参数拿条件，不从 stdin 读。条件写进**同一个** find 里（默认 AND 关系）。
- 遇到不可读目录是**静默跳过**（Permission denied），结果会不完整 → 查系统目录用 `sudo`。

### 3.2 awk —— 按"列"处理文本

```bash
awk '{print $1}'           # $1 第一列，$2 第二列... $NF 最后一列，$0 整行
awk '{print $1, $NF}'      # 有逗号=加空格分隔；无逗号=硬拼接（注意！）
awk -F: '{print $1}'       # -F 指定分隔符（处理 /etc/passwd 用冒号）
awk '$3=="ERROR"'          # 按条件过滤（第 3 列等于 ERROR 的行）
awk '$3 >= 1000 {print $1}'  # 条件 + 动作
awk '{sum+=$1} END {print sum}'  # 求和（END 块在所有行处理完后执行）
```

- 默认按**空白**（空格/Tab）切列。
- 分析冒号分隔文件：`/etc/passwd`（第 1 列用户名、第 7 列 shell）、`/etc/shadow`。

**取 /etc/passwd 里 UID>=1000 的真实用户**：
```bash
awk -F: '$3 >= 1000 {print $1}' /etc/passwd
```

### 3.3 sed —— 文本的"批量查找替换"

```bash
sed 's/old/new/'           # 替换每行第一个匹配
sed 's/old/new/g'          # g 全局替换
sed 's/old/new/2'          # 只替换每行第 2 个
sed -n '10,20p'            # -n 关闭默认输出，p 打印第 10-20 行
sed '3,5d'                 # d 删除第 3-5 行
sed '/pattern/d'           # 删除匹配 pattern 的行
sed -i 's/old/new/g' 文件  # -i 原地修改 ⚠️ 直接改文件，先备份
sed -i.bak 's/old/new/g' 文件  # -i.bak 改前自动生成 .bak 备份（推荐）
sed 's|/a/b|/c/d|g'        # 分隔符可换，处理含 / 的路径时用 |
```

### 3.4 命令的"输入来源"（重要概念）

- **接管道（读 stdin）**：`grep / awk / sed / sort / uniq / wc`（文本处理类）
- **不接管道（按参数行动）**：`find / ls / cp / mv / rm`（文件操作类）

> 类比：管道是"递货传送带"，grep 是"等着接货的工人"，find 是"自己进仓库找货的人"。`find A | find B` 里的第二个 find 根本不理传送带。

---

### 🕳 L3 踩过的坑

1. **`find /var/log -mtime +7 | find -name "*log"` 是错的**：第二个 find 不看管道，且没路径默认在当前目录找。→ 条件合并进同一个 find。
2. **awk `print $1 $7` 粘连**：少了逗号，输出 `root/bin/bash`。→ `print $1, $7`。
3. **两次 awk 串联冗余**：`awk ... | awk ...` 能一次写完就一次。
4. **sed -i.bak 把 `/etc/hosts` 的 `0` 全换成 `o`**：破坏 IP（127.0.0.1 → 127.o.o.1）。→ 改系统文件前想清楚替换目标，改完用 `diff` 验证。
5. **重定向写错目标会创建意外文件**：`echo ... > 文件名` 目标写岔，悄悄多出个文件。→ 命令敲完看 `ls`。
6. **sed -i 完整安全闭环**（改配置铁律）：
   ```bash
   cat 文件                       # 1. 改前看
   sudo cp 文件 文件.bak          # 2. 备份
   sudo sed -i.bak 's/旧/新/' 文件  # 3. 改（想清楚目标）
   diff 文件.bak 文件             # 4. 验证只改了该改的
   sudo cp 文件.bak 文件          # 5. 改错回滚
   ```

---

## 上篇速查表

| 需求 | 命令 |
|---|---|
| 我在哪 | `pwd` |
| 看目录（含隐藏/大小易读） | `ls -lah` |
| 复制目录 | `cp -r 源 目标` |
| 删除前逐个确认 | `rm -i 文件` |
| 实时盯日志 | `tail -f 文件` |
| 数错误条数 | `grep -c ERROR 文件` |
| 带行号+上下文 | `grep -n -A3 -B2 ERROR 文件` |
| 统计排行 | `... | sort | uniq -c | sort -rn` |
| 找 7 天前日志 | `find 目录 -mtime +7 -name "*.log"` |
| 取第 3 列 | `awk '{print $3}'` |
| 全局替换 | `sed 's/旧/新/g'` |
| 安全改文件 | `sed -i.bak 's/旧/新/g' 文件` + `diff` |
