---
name: excel-work
description: >
  读写并处理 Excel 表格（.xlsx/.xls/.csv）。可用于读取某范围/整表数据、筛选与排序、透视统计、批量公式计算、把多表合并，或根据你的理解结果生成 Excel。适合数据查询、报表汇总、CSV 处理。
  当用户提供 .xlsx/.xls/.csv、要查表、算数据、整理报表、合并表格时触发。
license: MIT
compatibility: python3 + openpyxl (xlsx) / pandas (数据操作) / xlrd (旧xls读取)
---

# Excel 表格处理

angles-cli 读 Excel、算数据、生成报表。**读取喂模型做理解，写入用 openpyxl/pandas 精确落地。**

## When to use

- 用户给 Excel/CSV 问"表里有什么"、"平均/合计是多少"、"帮我筛选"
- 要跨多表合并、透视汇总、生成报表
- 要程序化写一个新表

## 依赖检查

```bash
python3 -c "import openpyxl" 2>/dev/null && echo "openpyxl OK" || echo "need: pip install openpyxl"
python3 -c "import pandas" 2>/dev/null && echo "pandas OK" || echo "need: apk add py3-pandas 或 pip install pandas"
```

## 读表（理解基础）

### openpyxl 读 .xlsx

```python
import openpyxl
wb = openpyxl.load_workbook("数据.xlsx", data_only=True)
for ws in wb.worksheets:
    print("Sheet:", ws.title, "尺寸:", ws.dimensions)
    for i, row in enumerate(ws.iter_rows(values_only=True)):
        if i > 20:  # 限制行数，避免爆上下文
            print("...(截断)")
            break
        print("\t".join("" if c is None else str(c) for c in row))
```

### pandas 读（自带类型/统计）

```python
import pandas as pd
df = pd.read_excel("数据.xlsx", sheet_name=0)
print(df.head(15).to_string())          # 前15行
print("\n列:", list(df.columns))
print(df.describe().to_string())        # 数值统计
```

### 读 .csv（含中文编码注意）

```python
import pandas as pd
df = pd.read_csv("数据.csv", encoding="utf-8-sig")  # 防 BOM 中文乱码
print(df.head(20).to_string())
# 若还乱码试: encoding="gbk"
```

### 旧 .xls
```python
import pandas as pd
df = pd.read_excel("旧表.xls", engine="xlrd")
```

## 数据操作

```python
import pandas as pd
df = pd.read_excel("销售.xlsx")

# 筛选
print(df[df["销售额"] > 10000].to_string())
# 排序
print(df.sort_values("日期").head(10).to_string())
# 分组汇总
print(df.groupby("地区")["销售额"].sum().to_string())
# 透视
print(pd.pivot_table(df, values="销售额", index="地区", columns="月份", aggfunc="sum").to_string())
```

## 合并多个表

```python
import pandas as pd
dfs = [pd.read_excel(f) for f in ["1月.xlsx", "2月.xlsx", "3月.xlsx"]]
merged = pd.concat(dfs, ignore_index=True)
merged.to_excel("全年.xlsx", index=False)
print("合并后行数:", len(merged))
```

## 写表（生成 Excel 交付）

```python
from openpyxl import Workbook
wb = Workbook()
ws = wb.active
ws.title = "汇总"

ws.append(["地区", "销售额", "占比"])
data = [["华东", 3200, "42%"], ["华南", 2500, "33%"], ["华北", 1900, "25%"]]
for row in data:
    ws.append(row)

# 简单样式：表头加粗
from openpyxl.styles import Font
for cell in ws[1]:
    cell.font = Font(bold=True)

wb.save("汇总表.xlsx")
print("已保存 汇总表.xlsx")
```

**或一键存 pandas df 为 Excel：**
```python
df.to_excel("汇总.xlsx", index=False, sheet_name="结果")
```

## 交给模型回答

- 理解"表里有什么"：把 `head(15).to_string()` + 列名 + 行数给模型
- 统计类问题：让 pandas 算好数值再给模型，**不要靠模型心算**
- 大表：pandas 聚合后再贴结果，别整表喂爆上下文
- CSV 中文：总带上 `encoding` 参数

## 常见坑

- `load_workbook` 默认读到公式文本而非结果，要统计结果务必 `data_only=True`
- CSV 中文第一列带 `\ufeff` = BOM，用 `utf-8-sig`
- xlsx 有合并单元格时 `iter_rows` 只在首格有值，要合并语义得手动推断
- 日期列 pandas 可能读成字符串，需 `pd.to_datetime(df["日期"])`
- 空单元格返回 None，转字符串前先判空

## 原则

计算交给 pandas（可靠），判断和表述交给模型（灵活）。凡涉及"加起来、平均、比大小、筛选"都用 pandas 出数字，再让模型解读。
