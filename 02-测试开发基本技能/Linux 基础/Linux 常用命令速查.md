---
tags:
  - 速查
  - Linux
---
# Linux 常用命令速查

> 来源：`source/_posts/linux_cmd.md`  
> 整理日期：2026-06-16  
> 适用场景：日常测试环境排查、日志分析、批量操作、性能监控

---

## 一、文件与目录操作

| 命令 | 常用参数/示例 | 测试场景 |
|---|---|---|
| `ls` | `ls -la`、`ls -lh` | 查看测试产物权限与大小 |
| `find` | `find . -name "*.log" -mtime +7` | 清理过期日志 |
| `cp/mv/rm` | `cp -r`、`rm -rf` | 备份/清理测试数据 |
| `tar` | `tar -zcvf logs.tar.gz logs/` | 打包测试日志 |
| `scp` | `scp -r user@host:/var/log ./` | 远程拷贝测试日志 |
| `du` | `du -sh *` | 查看目录占用空间 |
| `df` | `df -h` | 检查磁盘空间 |

---

## 二、文本处理三剑客

### 2.1 grep：文本过滤

```bash
# 在日志中搜索错误
grep -i "error" app.log

# 显示匹配行前后 5 行
grep -C 5 "exception" app.log

# 排除某些行
grep -v "healthcheck" app.log

# 递归搜索目录
grep -r "timeout" /var/log/

# 统计出现次数
grep -c "ERROR" app.log
```

### 2.2 awk：列处理与统计

```bash
# 打印第 1、4 列
awk '{print $1, $4}' access.log

# 按条件过滤后统计
awk '$9 >= 500 {count++} END {print count}' access.log

# 求和
awk '{sum += $2} END {print sum}' data.txt

# 按列分组计数
awk '{count[$1]++} END {for(k in count) print k, count[k]}' access.log
```

**测试应用**：分析接口 access.log，统计状态码分布、慢请求数量。

### 2.3 sed：文本替换与编辑

```bash
# 替换文件中的 IP
sed -i 's/192.168.1.1/192.168.1.2/g' config.yaml

# 删除空行
sed -i '/^$/d' file.txt

# 提取第 10-20 行
sed -n '10,20p' file.txt
```

### 2.4 wc：行数/字数统计

```bash
wc -l test_cases.csv      # 统计用例行数
wc -c response_body.json  # 统计响应体字节数
```

---

## 三、网络排查

| 命令 | 示例 | 测试场景 |
|---|---|---|
| `ip` | `ip a`、`ip r` | 查看网卡 IP、路由 |
| `ping` | `ping -c 4 8.8.8.8` | 基础连通性 |
| `ss` | `ss -tlnp` | 查看监听端口及进程 |
| `netstat` | `netstat -anp \| grep 8080` | 端口占用排查 |
| `curl` | `curl -v http://localhost:8080/health` | 接口可达性检查 |
| `telnet` | `telnet host port` | 端口连通性 |

```bash
# 查看端口被哪个进程占用
ss -tlnp | grep 3306
lsof -i :3306

# 抓包（需 root）
tcpdump -i eth0 -w capture.pcap port 8080
```

---

## 四、进程与性能

| 命令 | 示例 | 测试场景 |
|---|---|---|
| `ps` | `ps -ef \| grep java` | 查找服务进程 |
| `top` | `top -p PID` | 监控单个进程资源 |
| `free` | `free -h` | 查看内存使用 |
| `vmstat` | `vmstat 1 5` | 采样系统负载 |
| `iostat` | `iostat -x 1` | 磁盘 IO 分析 |
| `nproc` | `nproc` | 查看 CPU 核心数 |

```bash
# 找出 CPU 占用最高的 10 个进程
ps -aux --sort=-%cpu | head -n 11

# 找出内存占用最高的 10 个进程
ps -aux --sort=-%mem | head -n 11
```

---

## 五、系统与服务

| 命令 | 示例 | 测试场景 |
|---|---|---|
| `systemctl` | `systemctl status nginx` | 查看服务状态 |
| `journalctl` | `journalctl -u nginx -f` | 查看服务日志 |
| `crontab` | `crontab -l` | 查看定时任务 |
| `nohup` | `nohup pytest > run.log 2>&1 &` | 后台执行测试 |
| `watch` | `watch -n 5 'df -h'` | 周期性监控 |

```bash
# 后台执行测试套件，忽略挂起信号
nohup python -m pytest tests/ --html=report.html > pytest.log 2>&1 &
echo $! > pytest.pid  # 记录 PID，便于后续停止
```

---

## 六、常用组合命令

### 6.1 日志中按小时统计错误数

```bash
grep "ERROR" app.log | awk -F' ' '{print $2}' | cut -d: -f1 | sort | uniq -c
```

### 6.2 批量替换配置后检查

```bash
find . -type f -name "*.yaml" | xargs sed -i 's/old-domain/new-domain/g'
grep -r "old-domain" . || echo "替换完成"
```

### 6.3 批量重命名测试文件

```bash
for f in test_*.log; do mv "$f" "bak_$f"; done
```

### 6.4 快速生成指定大小测试文件

```bash
dd if=/dev/zero of=test_100m.bin bs=1M count=100
```

---

## 七、实践建议

1. **日志分析优先用 `grep + awk + sed`**：80% 的排查场景不需要写脚本。
2. **大文件避免 `cat`**：用 `less`、`grep`、`awk` 流式处理。
3. **批量操作先 `echo` 验证**：避免误删/误改，如 `find ... | xargs echo rm`。
4. **后台任务记录 PID**：方便测试结束后清理。
5. **性能数据保留时间戳**：便于与测试执行时间关联。

---

## 八、关联文档

- [[Linux 核心概念]]
- [[Shell 脚本编程]]
- [[TCP-IP 与 HTTP-HTTPS|TCP/IP 与 HTTP/HTTPS]]
- 性能测试