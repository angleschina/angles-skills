---
name: log-analyze
description: >
  分析日志文件并定位问题。可按时间/级别/模块过滤，统计错误出现的频次与分布，提取堆栈异常，聚合同类报错，找出时间窗口内的关键事件，并把分析结果归纳成给管理者或用户的结论。适合排查"服务为什么挂""哪个接口报错最多""最近有异常吗""帮我看看日志"等。
  当用户提供 .log/.out 日志文件、要求排查、统计错误、找异常原因、做日志摘要时触发。
license: MIT
compatibility: python3(内置) / awk/sed/grep；无额外依赖
---

# 日志分析与问题定位

对日志文件做过滤、统计、聚合，定位异常。核心：**用 grep/awk/python 精确挖数据，把因果链交给模型归纳，别靠肉眼扫大文件。**

## When to use

- 用户给日志说"看看有没有问题"、"为什么报错"、"哪个错最多"
- 需要统计错误频次、按时段分布、按模块聚合
- 排查服务崩溃 / 接口 5xx / 慢请求

## 1. 先看规模再动手

```bash
ls -lh app.log && wc -l app.log      # 别上来就全量读
head -50 app.log                      # 看格式先
tail -100 app.log                     # 最近的（崩溃常见于尾部）
```

## 2. 提关键行

```bash
# 全部错误/异常
grep -iE "error|exception|failed|fatal|panic" app.log | tail -200
# 只看 ERROR 级以上（按常见级别关键字）
grep -E "\b(WARN|ERROR|FATAL)\b" app.log | tail -200
# 某时间段（按行内时间戳前缀过滤）
grep "^2026-08-28 10:0" app.log | grep ERROR
```

## 3. 频次统计（找"最多"）

```bash
# 错误消息前 N：先归一化(去掉时间戳/具体ID)再排序
grep -E "error|exception" app.log \
  | sed 's/^[0-9T: +-]\{20\}//' \
  | sort | uniq -c | sort -rn | head -20
```

```python
# 更稳：python 聚合（可去 ID 变量后归并）
import re, collections
freq = collections.Counter()
for line in open("app.log", errors="ignore"):
    if re.search(r"error|exception|failed", line, re.I):
        msg = re.sub(r"\b\w{8}-\w{4}-\w{4}-\w{4}-\w{12}\b", "<UUID>", line).strip()  # 归一化唯一ID
        freq[msg] += 1
for msg, n in freq.most_common(10):
    print(f"{n:5d}  {msg[:120]}")
```

## 4. 按模块 / 接口 / 级别聚合

```bash
# 某模块的报错
grep "module=api" app.log | grep ERROR | tail -50
# 按"接口/URL"统计报错次数（假设行内含 /path）
grep -oE '"/[a-z_/-]+"' app.log | sort | uniq -c | sort -rn | head
# 按时间(小时)分布，看高峰
grep ERROR app.log | grep -oE "^[0-9-]+ [0-9]{2}:" | uniq -c | tail -24
```

## 5. 提取多行堆栈

多行异常（traceback/堆栈）按"行首时间戳是否连续"切块：

```python
import re
blocks, cur = [], []
for line in open("app.log", errors="ignore"):
    if re.match(r"^\d{4}-\d{2}-\d{2}", line):     # 行首是时间戳=新记录
        if cur: blocks.append(cur)
        cur = [line.rstrip()]
    else:
        cur.append(line.rstrip())                  # 续行(堆栈)
if cur: blocks.append(cur)
# 打印含 Traceback 的完整块给模型
for b in blocks:
    if any("Traceback" in x or "Error" in x for x in b):
        print("\n".join(b[:40]))
        print("=" * 40)
```

## 6. 时间窗口对比 / 慢请求

```bash
# 某接口平均次数
grep -c "POST /api/orders" app.log
# 慢请求(>3s，假设行内有 ms/time=)
grep -E "time=[0-9]{4,}" app.log | tail -50
```

## 交给模型回答

把聚合后的**事实**（错误Top、模块分布、时间窗、典型堆栈）喂给模型，让它：
- 概括"什么问题为主、影响多大"
- 推理可能的因果（哪些先发、哪些伴随）
- 给出排查/修复建议

**铁律**：结论必须源自日志里真实出现的行。日志没有的别脑补；不确定的明确说"这里还没有证据，需要再查某段"。

## 常见坑

- **多行堆栈当单行**：直接 grep 会把 traceback 中间行漏掉，用上面分块法
- **编码/中文**：日志可能非 utf-8，open 加 `errors="ignore"` 或在命令里 `iconv`
- 时间戳格式各异，先看 head 确认再写正则；别假设通用
- 大型日志直接 `cat` 会刷爆终端/上下文 —— 永远先 wc/grep 缩小
- 唯一 ID 让 uniq 难聚同一错：先归一化(替换 UUID/时间/连接号)再计数
- **区分"症状"和"根因"**：别把第一处报错当根因，看前后关联

## 原则

**先统计再读全文，用事实(频次/分布/堆栈)说话。** 交给模型的是经过提炼的证据链，不是原始日志大海捞针。
