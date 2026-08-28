---
name: json-processing
description: >
  查询、转换、校验和生成 JSON 数据。可用 jq/python 按条件过滤、取字段、合并/嵌套/扁平化、按 key 排序，把杂乱的 JSON 整理成清晰结构，或把表格/文本转成 JSON。适合从 API 响应、配置文件、数据 dump 里提取信息或重构结构，供进一步分析。
  当用户要求处理 JSON、从 JSON 里取数据、查某个字段、转换格式、合并两个 JSON 时触发。
license: MIT
compatibility: python3(内置 json+jq 二选一自动用) / jq 命令(若可用)；无额外依赖
---

# JSON 数据处理

对结构化 JSON 做查询、转换、整理。核心：**重活交给 jq/python，你把结果解读给用户。** JSON 是 agent 数据处理的"母语"，处理要精确不靠猜。

## When to use

- 用户给一段 JSON（或 API 响应/配置文件）要提取/查询/改结构
- 需要按条件过滤、按字段聚合、合并两个 JSON
- 需要把 CSV/文本 → JSON，或 JSON → 表格/Markdown
- 校验 JSON 是否合法、压缩/格式化

## 依赖检查

```bash
which jq && echo "jq OK" || echo "no jq (用 python fallback)"
python3 -c "import json; print('python json OK')"
```

有 jq 用 jq（一行解决问题），没有用 python。下面两套都给出。

## 查询 / 取字段

```bash
# jq
jq '.name' data.json                    # 取顶层字段
jq '.users[] | {name, email}' data.json # 取数组每项的部分字段
jq '.orders | map(select(.total > 100))' data.json  # 按条件过滤
jq 'length' data.json                   # 长度/元素数
```

```python
import json
data = json.load(open("data.json"))
print([{"name": u.get("name"), "email": u.get("email")} for u in data.get("users", [])])
```

## 深链查询（嵌套）

```bash
jq '.data.items[]? | .title' data.json   # ? 防取不存在的字段报错
```

```python
# 安全取值（防 KeyError）
def get(d, path):
    for p in path.split("."):
        if isinstance(d, dict): d = d.get(p)
        elif isinstance(d, list) and p.isdigit(): d = d[int(p)] if int(p) < len(d) else None
        else: return None
        if d is None: return None
    return d
print(get(data, "data.items.0.title"))
```

## 合并 / 排序 / 扁平化

```python
import json
a = json.load(open("a.json")); b = json.load(open("b.json"))

# 合并两个 dict（b 覆盖 a）
merged = {**a, **b}
# 合并 list
merged_list = (a.get("items") or []) + (b.get("items") or [])

# 按字段排序
sorted_items = sorted(merged_list, key=lambda x: x.get("date", ""), reverse=True)

# 嵌套扁平化
def flatten(d, prefix=""):
    out = {}
    for k, v in d.items():
        if isinstance(v, dict):
            out.update(flatten(v, f"{prefix}{k}."))
        else:
            out[f"{prefix}{k}"] = v
    return out
print(json.dumps(flatten(data), ensure_ascii=False, indent=2))
```

## 提供 JSON 给用户 / 给模型

- **格式化输出**：`python3 -m json.tool data.json` 或 `json.dumps(..., indent=2, ensure_ascii=False)`
- **紧凑**：给脚本/API 用 `json.dumps(..., separators=(",", ":"))`
- **喂给模型**：长 JSON 先抽取关键字段再给，别把整坨 dump 灌进上下文
- **转 Markdown**：若是数组对象，把字段抽成表给用户看更友好

```python
import json
data = json.load(open("data.json"))
rows = data if isinstance(data, list) else [data]
if rows and isinstance(rows[0], dict):
    keys = list(rows[0].keys())
    print("| " + " | ".join(keys) + " |")
    print("|" + "---|" * len(keys))
    for r in rows:
        print("| " + " | ".join(str(r.get(k, "")) for k in keys) + " |")
```

## 校验 + 修复常见问题

```python
import json
try:
    data = json.loads(open("x.json").read())
    print("合法 JSON")
except json.JSONDecodeError as e:
    print("非法:", e)   # 常见：尾逗号/Trailing comma、单引号、注释
```

一键压缩/格式化：
```bash
python3 -m json.tool x.json > x_pretty.json      # 格式化
jq -c . x.json > x_min.json                      # 压成一行
```

## 常见坑

- ensure_ascii 默认 True 会把中文转 \uXXXX，输出带中文一定要 `ensure_ascii=False`
- JSON 里 bool 是小写 `true/false`、null（不是 None/TRUE），python parse 后变 True/False/None，往**写回**时注意转换
- 尾逗号/注释/单引号 会让标准 json 报错 —— 用 `json5` 或先清洗再 parse
- `jq` 取不存在的字段会报错，加 `?` 容错
- 超大 JSON 先 `head -c` 看体积，别整包吃掉内存/上下文

## 原则

**JSON 是精确数据，处理结果必须是真实结构而非"大概"。** 查询给准确路径，改动给可复现的 python/jq 命令，解读交给模型但数据来自程序，不靠模型编造结构。
