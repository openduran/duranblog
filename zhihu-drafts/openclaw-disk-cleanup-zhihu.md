# OpenClaw 磁盘满排查实录：tmpfs 日志清理方案

> 昨晚 11 点，队友突然问我："磁盘空间满了？"
> 我检查后发现 `/tmp` 目录被 OpenClaw 日志撑爆到 100%，单个日志文件占了 1.9GB。
> 记录完整排查过程和解决方案，供大家参考。

## 问题现象

昨晚队友在执行 `sudo apt update` 时遇到报错：

```
E: You don't have enough free space in /var/cache/apt/archives/
```

他怀疑是磁盘空间问题，问我："磁盘空间满了？"

我第一反应是检查根分区，但 `df -h` 显示根分区还有空间。然后我看到了这个：

```bash
$ df -h
文件系统        大小  已用  可用 已用% 挂载点
tmpfs           2.0G  2.0G     0  100% /tmp
```

**`/tmp` 满了！**

## 排查过程

### 第一步：定位大文件

```bash
$ du -sh /tmp/*
1.9G  /tmp/openclaw
```

OpenClaw 的日志目录占用了几乎全部空间。

### 第二步：分析具体文件

```bash
$ ls -lh /tmp/openclaw/
-rw-rw-r 1 warwick warwick 1.9G  2月10日 17:14 openclaw-2026-02-08.log
-rw-rw-r  1 warwick warwick  24K  2月11日 13:45 openclaw-2026-02-10.log
-rw-rw-r  1 warwick warwick    0  2月11日 19:09 openclaw-2026-02-11.log
```

**问题很明显：单个日志文件占用了 1.9GB**，而其他日志只有几十 KB。

### 第三步：紧急清理

先解决燃眉之急，删除异常日志：

```bash
$ rm /tmp/openclaw/openclaw-2026-02-08.log
```

验证清理效果：

```bash
$ df -h /tmp
tmpfs           2.0G   24M  2.0G   2% /tmp
```

磁盘空间恢复正常！

### 第四步：根因分析

不能就这样结束，必须找到根本原因。我查看了那个 1.9GB 的日志文件内容：

```bash
$ head -n 20 /tmp/openclaw/openclaw-2026-02-08.log
[2026-02-08 05:51:47] Starting heartbeat check...
[2026-02-08 05:51:47] HEARTBEAT_OK
[2026-02-08 05:51:47] Starting heartbeat check...
[2026-02-08 05:51:47] HEARTBEAT_OK
[2026-02-08 05:51:47] Starting heartbeat check...
...
```

**发现问题：同一个时间戳（05:51:47）出现了成千上万次！**

判断是 **2026-02-08 凌晨的定时任务异常** 导致日志无限循环写入。可能是某个 heartbeat 任务卡住了，每秒都在写日志。

## 解决方案

### 短期方案：清理脚本

创建清理脚本 `/usr/local/bin/cleanup-openclaw-logs.sh`：

```bash
#!/bin/bash
# OpenClaw 日志清理脚本
# 保留最近7天日志，删除超过1GB的大文件

LOG_DIR="/tmp/openclaw"

# 删除7天前的日志
find "$LOG_DIR" -name "openclaw-*.log" -mtime +7 -delete

# 删除超过1GB的大文件（排除当天）
find "$LOG_DIR" -name "openclaw-*.log" -size +1G ! -name "openclaw-$(date +%Y-%m-%d).log" -delete

echo "[$(date)] 日志清理完成"
```

赋予执行权限并添加到定时任务：

```bash
chmod +x /usr/local/bin/cleanup-openclaw-logs.sh

# 添加到 crontab
crontab -e
# 添加以下行：
0 3 * * * /usr/local/bin/cleanup-openclaw-logs.sh >> /var/log/cleanup.log 2>&1
```

### 长期方案：日志大小监控

创建限制脚本 `/usr/local/bin/limit-log-size.sh`：

```bash
#!/bin/bash
# 日志大小限制脚本

LOG_FILE="/tmp/openclaw/openclaw-$(date +%Y-%m-%d).log"
MAX_SIZE=1073741824  # 1GB

if [ -f "$LOG_FILE" ]; then
    FILE_SIZE=$(stat -c%s "$LOG_FILE")
    if [ $FILE_SIZE -gt $MAX_SIZE ]; then
        # 超过1GB，截断保留最后1000行
        tail -n 1000 "$LOG_FILE" > "$LOG_FILE.tmp"
        mv "$LOG_FILE.tmp" "$LOG_FILE"
        echo "[$(date)] 日志超过1GB，已截断保留最后1000行" >> /var/log/log-limit.log
    fi
fi
```

添加到定时任务（每小时检查一次）：

```bash
chmod +x /usr/local/bin/limit-log-size.sh
crontab -e
# 添加：
0 * * * * /usr/local/bin/limit-log-size.sh
```

## 关键知识点

### 1. tmpfs 是什么？

tmpfs 是 Linux 的**内存文件系统**：
- ✅ 读写速度极快（在内存中操作）
- ✅ 重启后自动清空
- ❌ **容量受限于内存大小**（默认是物理内存的一半）
- ❌ **不能依赖重启清理**（生产环境不能随意重启）

### 2. 为什么 OpenClaw 日志在 /tmp？

OpenClaw 默认将日志写入 `/tmp`，因为：
- 日志是临时文件，不需要持久化
- tmpfs 速度快，不影响系统性能
- 重启后自动清理，省去维护成本

**但缺点也很明显**：如果不加限制，很容易撑满。

### 3. 更好的日志管理方案

对于生产环境，建议：
1. **修改 OpenClaw 日志路径**到独立的磁盘分区
2. **配置 logrotate** 自动轮转
3. **设置监控告警**，/tmp 使用率超过 80% 时通知

## 总结

| 方案 | 作用 | 执行频率 | 是否必须 |
|------|------|---------|---------|
| 清理脚本 | 删除旧日志 | 每天一次 | ✅ 必须 |
| 大小限制 | 防止单文件过大 | 每小时一次 | ✅ 必须 |
| 路径迁移 | 移到独立分区 | 一次性 | ⚠️ 生产环境建议 |

**这次排查的经验教训**：
1. 任何写到 `/tmp` 的程序都要考虑大小限制
2. 日志文件必须设置轮转策略
3. 定期检查磁盘使用率（建议自动化监控）

从发现问题到彻底解决，包括写脚本、测试、部署，总共用了不到 30 分钟。现在可以高枕无忧了！

---

**如果你也在用 OpenClaw 或类似的 Agent 系统，建议检查一下 `/tmp` 目录，防患于未然。**

**相关问题可以评论区交流，我会及时回复。**

---

**参考链接：**
- [tmpfs 官方文档](https://www.kernel.org/doc/html/latest/filesystems/tmpfs.html)
- [Linux 日志管理最佳实践](https://linux.die.net/man/8/logrotate)
