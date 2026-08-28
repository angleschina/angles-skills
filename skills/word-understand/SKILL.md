---
name: word-understand
description: >
  理解并处理 Microsoft Word 文档（.docx，含 .doc）。可用于提取正文、标题层级、表格、图片说明和批注；用你的语言总结文档内容回答用户问题；修改正文、替换关键词、批量改写段落；并转为纯文本/Markdown 供进一步处理。
  当用户提供 .docx/.doc 文件、问文档内容、要求提取或编辑 Word 文档时触发。
license: MIT
compatibility: python3 + python-docx; .doc 旧格式需 libreoffice/antiword 转换
---

# Word 文档理解与处理

angles-cli 读 Word 并理解/编辑的核心链路。**理解靠提取纯文本喂模型，编辑靠 python-docx 精确操作。**

## When to use

- 用户给 .docx 问"写了什么"、"帮我总结"、"重点是什么"
- 需要抽取标题层级 / 表格 / 高亮
- 需要改正文、替换词、插入章节
- 拿到的是旧 .doc，需要先转换

## 依赖检查

```bash
python3 -c "import docx" 2>/dev/null && echo "python-docx OK" || echo "need: pip install python-docx"
```

## 提正文 + 结构（理解基础）

```python
from docx import Document
doc = Document("文档.docx")

# 段落（含样式=标题层级线索）
for p in doc.paragraphs:
    style = p.style.name if p.style else ""
    txt = p.text.strip()
    if txt:
        print(f"[{style}] {txt}")
```

### 只提主要段落（跳过空/纯格式）

```python
for p in doc.paragraphs:
    if p.text.strip():
        print(p.text.strip())
```

### 表格

```python
for t in doc.tables:
    for row in t.rows:
        print(" | ".join(c.text.strip() for c in row.cells))
    print("---")
```

### 文档属性 / 标题概览

```python
from docx import Document
doc = Document("文档.docx")
core = doc.core_properties
print("标题:", core.title, "| 作者:", core.author, "| 主题:", core.subject)

# 标题层级一览（只需给模型看骨架）
for p in doc.paragraphs:
    if p.style and "Heading" in p.style.name:
        print(f"{p.style.name}: {p.text}")
```

## 编辑正文

```python
from docx import Document
doc = Document("文档.docx")

# 替换关键词（注意要处理 run 分裂——同一词可能跨多个 run）
def replace_all(doc, old, new):
    for p in doc.paragraphs:
        for run in p.runs:
            if old in run.text:
                run.text = run.text.replace(old, new)

replace_all(doc, "旧公司名", "新公司名")
doc.save("修改后.docx")
```

### 追加一段 / 插入标题

```python
doc = Document("文档.docx")
doc.add_paragraph("这是新增的结尾段落。")
doc.add_heading("新章节标题", level=1)
doc.save("追加后.docx")
```

## 旧 .doc 转换（docx 之前格式）

docx 库打不开 .doc，先转：

```bash
# 有 libreoffice：
libreoffice --headless --convert-to docx 旧文件.doc --outdir ./out/
# 或仅提取文本：
antiword 旧文件.doc > 旧文件.txt
```

## 交给模型回答

- 总结/正文理解：把段落文字（去样式标记，最多前几页正文）给模型
- 结构问答：给标题骨架 + 相关章节正文
- 数据表：把表格行原样拼成 Markdown 表格给模型

**铁律**：答案基于提取的真实内容。文档没有的别编；提取不到就贴原文段落让模型读。

## 常见坑

- 词跨多个 run = 无法简单 `run.text.replace`，用上面 `replace_all` 跨段落兜底或接受分段替换
- 空段落很多，先 `.strip()` 过滤
- 只读/加密 docx 无法编辑——先检查文件锁
- 大文档（几十页）先给模型目录，再按需取正文
