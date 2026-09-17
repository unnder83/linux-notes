# MySQL 运维方向系统课程 · 学习笔记（第 1~3 课）

> **学习环境**：VMware + Rocky Linux 9（最小化安装）+ MySQL 8.4 LTS
> **客户端工具**：DataGrip（主力）/ Navicat Premium 16（备选）
> **参考资料**： MySQL（基础篇 / 进阶篇上·下 / 运维篇），版本信息已更新至 2026 现状

---

## 目录

- 一、MySQL 概述与安装部署（第 1 课）
- 二、GUI 远程连接 + SQL 概述 + DDL（第 2 课上）
- 三、DML 增删改 + DQL 查询（第 2 课下）
- 四、DCL 用户与权限 + 函数（第 3 课）
- 五、易错点与运维安全红线汇总

---

## 一、MySQL 概述与安装部署（第 1 课）

### 1.1 数据库相关概念

| 名称                | 全称                       | 通俗理解                               |
| ------------------- | -------------------------- | -------------------------------------- |
| 数据库 DB           | DataBase                   | 存储数据的仓库，数据**有组织**地存放   |
| 数据库管理系统 DBMS | DataBase Management System | 操纵和管理数据库的大型软件（如 MySQL） |
| SQL                 | Structured Query Language  | 操作关系型数据库的**统一语言/标准**    |

**三者关系**：通过 **DBMS（MySQL）** 操作 **DB（数据库）**，操作语言是 **SQL**。

主流关系型数据库：Oracle（大型收费）、**MySQL**（开源免费，互联网首选）、SQL Server、PostgreSQL、MariaDB（MySQL 衍生分支）。

### 1.2 版本选择（★ 黑马资料已过时，已更新）

**MySQL 版本现状**：

| 版本系列      | 状态                     | 说明                               |
| ------------- | ------------------------ | ---------------------------------- |
| MySQL 8.0     | ❌ EOL（2026-04）         | 黑马用的 8.0.26 属于它，**弃用**   |
| **MySQL 8.4** | ✅ LTS 长期支持（8.4.11） | 8.0 直接后继，**✅ 本课程采用**     |
| MySQL 9.7     | ✅ LTS（2026-04）         | 最新 LTS，但改动大、与教材匹配度低 |
| MySQL 26.x    | 🔬 Innovation            | 新特性预览，生产勿用               |

**Linux 发行版现状**：

| 发行版                     | 状态             | 说明                          |
| -------------------------- | ---------------- | ----------------------------- |
| CentOS 7                   | ❌ EOL（2024-06） | 黑马运维篇用的，**弃用**      |
| **Rocky Linux 9**          | ✅ 推荐           | CentOS 正统"续作"，本课程采用 |
| AlmaLinux 9 / Ubuntu 24.04 | ✅ 可选           | 同类替代                      |

**选 8.4 LTS 的理由**：8.0 的直接 LTS 后继（教材命令 95%+ 可用）、生产主流、避免 9.7 的破坏性变更（如移除 mysql_native_password）。

### 1.3 VMware 安装 Rocky Linux 9

**关键参数**：CPU 2 核、内存 2~4 GB、NAT 网络、40 GB 磁盘、**Software Selection 选 Minimal Install**（命令行）、固定 root 密码。

装好后的基本功：

```bash
ping -c 3 baidu.com        # 确认联网
ip a                       # 查看本机 IP
cat /etc/os-release        # 看系统版本
```

### 1.4 安装 MySQL 8.4 LTS（官方 yum 源）

```bash
# ① 下载并安装官方仓库（el9 = Rocky 9，文件名以官网实际为准）
wget https://dev.mysql.com/get/mysql97-community-release-el9-1.noarch.rpm
sudo dnf install -y mysql97-community-release-el9-1.noarch.rpm

# ② 切换子仓库（★ 关键：默认启用 9.7，要切到 8.4）
sudo dnf config-manager --disable mysql-9.7-lts-community
sudo dnf config-manager --enable mysql84-community
dnf repolist enabled | grep mysql          # 验证

# ③ 安装
sudo dnf install -y mysql-community-server

# ④ 启动 + 开机自启
sudo systemctl start mysqld
sudo systemctl enable mysqld

# ⑤ 拿临时密码 + 安全初始化
sudo grep 'temporary password' /var/log/mysqld.log
sudo mysql_secure_installation

# ⑥ 登录
mysql -uroot -p
```

> 若提示 `dnf config-manager: command not found`，先 `sudo dnf install -y dnf-plugins-core`。

### 1.5 启动停止 & 客户端连接

| 操作               | 命令                                      |
| ------------------ | ----------------------------------------- |
| 启动 / 停止 / 重启 | `systemctl start / stop / restart mysqld` |
| 开机自启           | `systemctl enable mysqld`                 |
| 查看状态           | `systemctl status mysqld`                 |

连接语法：

```bash
mysql [-h 主机IP] [-P 端口] -u 用户名 -p   # -p 后不直接写密码，回车交互输入更安全
```

### 1.6 数据模型

MySQL 是关系型数据库，基于二维表存数据（行=记录，列=字段）。

```
MySQL 服务器
  └── 数据库 db01
        ├── 表 tb_user
        ├── 表 tb_order
  └── 数据库 db02
```

---

## 二、GUI 远程连接 + SQL 概述 + DDL（第 2 课上）

### 2.1 工具选型

| 工具               | 定位                         | 建议                       |
| ------------------ | ---------------------------- | -------------------------- |
| Navicat Premium 16 | 界面友好、运维日常操作顺手   | 日常首选                   |
| **DataGrip**       | SQL 智能提示强、结果展示清晰 | **本课程主力**（黑马也用） |
| DBeaver            | 免费开源                     | 免费备选                   |

> 核心认知：**GUI 只是客户端载体，"会不会写 SQL"才是运维核心能力**（写脚本、自动化、排障都得靠 SQL）。

### 2.2 远程连接配置（4 步，★ 运维基本功）

**第 1 步：创建远程专用用户（不开 root）**

```sql
CREATE USER 'dev'@'%' IDENTIFIED BY 'Dev@123456';
GRANT ALL PRIVILEGES ON itcast.* TO 'dev'@'%';
FLUSH PRIVILEGES;
```

**第 2 步：防火墙放行 3306**

```bash
sudo firewall-cmd --zone=public --add-port=3306/tcp --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports
```

**第 3 步：确认监听地址**（一般不用改）

```bash
ss -tlnp | grep 3306   # 看是 0.0.0.0:3306 还是 127.0.0.1:3306
```

**第 4 步：客户端连接**：主机填虚拟机 IP（非 localhost）、端口 3306、用户名 dev。

> 常见坑 `Public Key Retrieval is not allowed`：MySQL 8.4 默认认证插件 `caching_sha2_password` 导致，客户端勾选「Allow Public Key Retrieval」即可。

### 2.3 SQL 概述

**通用语法**：分号结尾；不区分大小写（关键字建议大写）；注释 `--`/`#`/`/* */`。

**四大分类**：

| 分类 | 全称                       | 作用                   | 关键字               |
| ---- | -------------------------- | ---------------------- | -------------------- |
| DDL  | Data Definition Language   | 定义对象（库/表/字段） | CREATE/ALTER/DROP    |
| DML  | Data Manipulation Language | 数据增删改             | INSERT/UPDATE/DELETE |
| DQL  | Data Query Language        | 查询记录               | SELECT               |
| DCL  | Data Control Language      | 管用户、控权限         | GRANT/REVOKE         |

记忆：**DDL 管结构，DML 管数据，DQL 查数据，DCL 管权限**。

### 2.4 DDL

**数据库操作**：

```sql
SHOW DATABASES;
SELECT DATABASE();
CREATE DATABASE [IF NOT EXISTS] 库名 [DEFAULT CHARSET 字符集];
DROP DATABASE [IF EXISTS] 库名;
USE 库名;
```

**表操作**：

```sql
SHOW TABLES;                     -- 查所有表
DESC 表名;                       -- 查表结构
SHOW CREATE TABLE 表名;          -- 查建表语句
CREATE TABLE 表名(字段 类型 [COMMENT '注释'], ...) [COMMENT '表注释'];
DROP TABLE [IF EXISTS] 表名;     -- 删表
TRUNCATE TABLE 表名;             -- 清空数据保留结构
```

**改表（ALTER）**：

| 操作      | 语法                                      |
| --------- | ----------------------------------------- |
| 加字段    | `ALTER TABLE 表 ADD 字段 类型;`           |
| 改类型    | `ALTER TABLE 表 MODIFY 字段 新类型;`      |
| 改名+类型 | `ALTER TABLE 表 CHANGE 旧名 新名 新类型;` |
| 删字段    | `ALTER TABLE 表 DROP 字段;`               |
| 改表名    | `ALTER TABLE 表 RENAME TO 新名;`          |

**数据类型**：

- **数值**：TINYINT（1B）/ INT（4B）/ BIGINT（8B）/ FLOAT / DOUBLE / **DECIMAL(M,D)（精确小数，钱必须用它）**
- **字符串**：CHAR(n) 定长（手机号/性别）/ VARCHAR(n) 变长（姓名/地址）/ TEXT / BLOB
- **日期**：DATE（日期）/ DATETIME（日期+时间，最常用）/ TIMESTAMP

> char 定长性能高、varchar 变长省空间。字符集生产一律 `utf8mb4`（支持中文和 emoji）。

---

## 三、DML 增删改 + DQL 查询（第 2 课下）

### 3.1 DML

```sql
-- 增（①指定字段 ②全部字段 ③批量）
INSERT INTO 表 (字段1, 字段2) VALUES (值1, 值2);
INSERT INTO 表 VALUES (值1, 值2, ...);
INSERT INTO 表 (字段1, 字段2) VALUES (值1,值2), (值1,值2), ...;   -- ★批量效率高

-- 改
UPDATE 表 SET 字段1=值1, 字段2=值2 [WHERE 条件];

-- 删
DELETE FROM 表 [WHERE 条件];
```

> 🔴 **安全红线**：UPDATE/DELETE 漏写 WHERE 会改/删整张表！铁律：改删前先 `SELECT ... WHERE 同样条件` 确认 + 先备份。

### 3.2 DQL（核心）

**完整语法（编写顺序）**：

```sql
SELECT 字段列表
FROM 表名列表
WHERE 条件列表
GROUP BY 分组字段列表
HAVING 分组后条件列表
ORDER BY 排序字段列表
LIMIT 分页参数;
```

**⭐ 执行顺序**（与编写顺序不同，面试必考）：

```
FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT
```

> WHERE 在 SELECT 前执行 → WHERE 不能用 SELECT 起的别名；ORDER BY 在 SELECT 后 → 可以用别名。

**基础查询**：`SELECT 字段`、`SELECT *`、`AS 别名`、`DISTINCT 去重`。

**条件查询运算符**：

| 类型 | 运算符                                         |
| ---- | ---------------------------------------------- |
| 比较 | `>` `>=` `<` `<=` `=` `<>` `!=`                |
| 范围 | `BETWEEN a AND b`（含两端）、`IN(...)`         |
| 模糊 | `LIKE`（`_` 单字符、`%` 任意字符）             |
| 判空 | `IS NULL` / `IS NOT NULL`（**不能用 = NULL**） |
| 逻辑 | `AND`/`&&`、`OR`/`                             |

**聚合函数**：`count` `max` `min` `avg` `sum`（**NULL 值不参与运算**）。

- `count(*)` = 总行数；`count(字段)` = 该字段非 NULL 行数。

**分组**：`GROUP BY 字段 [HAVING 分组后条件]`

**WHERE vs HAVING**：

| 对比           | WHERE    | HAVING   |
| -------------- | -------- | -------- |
| 执行时机       | 分组之前 | 分组之后 |
| 能否用聚合函数 | ❌        | ✅        |

**排序**：`ORDER BY 字段 [ASC|DESC]`（ASC 默认，可省略；多字段依次比较）。

**分页**：`LIMIT 起始索引, 记录数`（起始索引从 0 起，`= (页码-1)*每页数`）。

**emp 员工表（16 条测试数据，建表语句）**：

```sql
CREATE TABLE emp(
    id          INT             COMMENT '编号',
    workno      VARCHAR(10)     COMMENT '工号',
    name        VARCHAR(10)     COMMENT '姓名',
    gender      CHAR(1)         COMMENT '性别',
    age         TINYINT UNSIGNED COMMENT '年龄',
    idcard      CHAR(18)        COMMENT '身份证号',
    workaddress VARCHAR(50)     COMMENT '工作地址',
    entrydate   DATE            COMMENT '入职时间'
) COMMENT '员工表';
```

---

## 四、DCL 用户与权限 + 函数（第 3 课）

### 4.1 DCL 用户与权限管理（★ 运维核心）

**管理用户**：

```sql
SELECT * FROM mysql.user;                       -- 查用户
CREATE USER '用户名'@'主机名' IDENTIFIED BY '密码';  -- 建用户
ALTER USER '用户名'@'主机名' IDENTIFIED BY '新密码'; -- 改密码
DROP USER '用户名'@'主机名';                       -- 删用户
```

> **⭐ 用户唯一标识 = `用户名@主机名`**，二者缺一不可。`%` 是主机通配符。
> **8.4 版本差异**：改密码直接 `IDENTIFIED BY`，别用已弃用的 `mysql_native_password` 插件。

**权限控制**：

```sql
SHOW GRANTS FOR '用户名'@'主机名';                       -- 查权限
GRANT 权限列表 ON 数据库.表 TO '用户名'@'主机名';          -- 授权
REVOKE 权限列表 ON 数据库.表 FROM '用户名'@'主机名';       -- 撤权
```

常用权限：`ALL` / `SELECT` / `INSERT` / `UPDATE` / `DELETE` / `CREATE` / `ALTER` / `DROP`。

**权限范围（从小到大）**：`*.*`（全局）→ `库.*` → `库.表` → `库.表(列)`。

> 最小权限原则：按需授权，别动不动 `ALL ON *.*`；生产环境主机名指定具体 IP，不用 `%`；不开放 root 远程。

### 4.2 函数（四类）

**字符串函数**：

| 函数          | 功能       |
| ------------- | ---------- |
| CONCAT        | 拼接       |
| LOWER / UPPER | 转大小写   |
| LPAD / RPAD   | 左/右填充  |
| TRIM          | 去首尾空格 |
| SUBSTRING     | 截取子串   |

**数值函数**：CEIL（向上取整）/ FLOOR（向下取整）/ MOD（取模）/ RAND（随机）/ ROUND（四舍五入）。

**日期函数**：CURDATE / CURTIME / NOW / YEAR / MONTH / DAY / DATE_ADD / DATEDIFF。

**流程函数**：`IF(v,t,f)` / `IFNULL(v1,v2)` / `CASE WHEN ... THEN ... ELSE ... END`。

**经典案例**：

```sql
-- 工号补 5 位
UPDATE emp SET workno = LPAD(workno, 5, '0');

-- 入职天数
SELECT name, DATEDIFF(CURDATE(), entrydate) AS days FROM emp ORDER BY days DESC;

-- 年龄分档
SELECT name, age,
       CASE WHEN age < 35 THEN '青年'
            WHEN age < 60 THEN '中年'
            ELSE '老年'
       END AS '年龄段'
FROM emp;
```

---

## 五、易错点与运维安全红线汇总

### 5.1 易错点（来自作业批改）

1. **DML 不含"查"**：增 INSERT、删 DELETE、改 UPDATE；查 SELECT 单独归 DQL。
2. **SQL 是"语言"**，不是"标准"（SQL 定义了一套统一标准，但本质是语言）。
3. **小数用 DECIMAL**，不能用 TINYINT/INT（存不了小数）、不用 FLOAT/DOUBLE（有精度误差，钱会算错）。
4. **NULL 判断用 `IS NULL`**，不能用 `= NULL`。
5. **反引号 `` ` `` 只用于标识符**（表名/列名），不能包 `count(*)` 这种表达式；字符串用单引号 `'`。
6. **`count(*)` 是"总行数"**，不是"所有列的个数"。
7. **聚合函数 NULL 不参与**：`count(*)` 与 `count(字段)` 结果可能不同。
8. **ALTER 三兄弟**：ADD 加字段、MODIFY 只改类型、CHANGE 改名+类型。

### 5.2 运维安全红线

- **UPDATE/DELETE 必须带 WHERE**，改删前先 SELECT 确认 + 备份。
- **不开放 root 远程登录**，用专用账号 + 最小权限。
- **生产环境主机名不用 `%`**，指定具体来源 IP。
- **防火墙用放行端口方式**，不要直接关 firewalld。
- **生产字符集用 utf8mb4**，不是 utf8（阉割版）。

---

*笔记整理截止：第三课（DCL + 函数）。第四课起将进入「约束」章节。*
