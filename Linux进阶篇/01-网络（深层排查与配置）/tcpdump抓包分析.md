# tcpdump 抓包分析

## 学习目标

- [ ] 理解 tcpdump 抓包原理和适用场景
- [ ] 掌握常用过滤表达式（host、port、协议）
- [ ] 能把抓包结果存成 pcap 用 Wireshark 分析
- [ ] 能通过抓包定位连接失败、丢包问题

## 常用参数

```bash
tcpdump -i eth0                    # 指定网卡
tcpdump -nn                        # 不解析域名和端口名
tcpdump -w cap.pcap                # 保存到文件
tcpdump -r cap.pcap                # 读取文件
tcpdump -c 100                     # 抓 100 个包后停止
```

## 过滤表达式

```bash
tcpdump -nn host 192.168.1.10            # 指定主机
tcpdump -nn port 80                      # 指定端口
tcpdump -nn 'tcp and port 443'           # 组合条件
tcpdump -nn 'src 192.168.1.10'           # 指定源地址
tcpdump -nn -A port 80                   # 以 ASCII 显示内容
```

## 笔记

> 边学边补充

## 实操记录

> 待补充

## 踩坑记录

> 待补充
