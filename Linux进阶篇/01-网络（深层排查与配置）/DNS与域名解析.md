# DNS 与域名解析

## 学习目标

- [ ] 理解域名解析的完整流程
- [ ] 掌握 /etc/hosts、/etc/resolv.conf、nsswitch.conf 的作用
- [ ] 会用 dig 排查解析问题
- [ ] 了解 DNS 服务器搭建（可选）

## 核心概念

| 记录类型 | 说明 |
| --- | --- |
| A | 域名 → IPv4 |
| AAAA | 域名 → IPv6 |
| CNAME | 别名 |
| MX | 邮件 |
| NS | 域名服务器 |

## 排查命令

```bash
dig example.com            # 完整解析过程
dig +short example.com     # 只看结果
dig @8.8.8.8 example.com   # 指定 DNS 服务器
nslookup example.com
cat /etc/resolv.conf
```

## 笔记

> 边学边补充

## 踩坑记录

> 待补充
