---
name: pdf-understand
description: >
  理解 PDF 文件内容并执行常见处理。可用于提取 PDF 文本、表格、图片和元数据；将扫描件做 OCR 成可搜索文本；合并/拆分 PDF；旋转页面；添加水印；以及用自己的语言总结 PDF 内容回答用户问题。
  当用户提供 .pdf 文件、询问 PDF 讲了什么、要求从 PDF 提取信息或对 PDF 做处理时触发。
license: MIT
compatibility: python3 + pypdf / pymupdf (fitz); 扫描件 OCR 需 pytesseract + tesseract 二进制
---

# PDF 理解与处理

针对 angles-cli 提供的"读 PDF 并理解、回答内容问题"的能力。核心思路：**先提取纯文本喂给模型，让模型基于文本回答**，不要自己凭空总结。

## When to use

- 用户给了 PDF 问"这讲了什么"、"帮我总结"、"关键数据是多少"
- 需要从 PDF 抽取表格、清单、条款
- 需要把多页 PDF 合并、把一页拆开、旋转、加/删水印
- 扫描版的 PDF（图片页）需要 OCR 才能读

## 依赖检查

首次使用先确认工具链：

```bash
python3 -c "import pypdf" 2>/dev/null && echo "pypdf OK" || echo "need: pip install pypdf"
python3 -c "import fitz" 2>/dev/null && echo "pymupdf OK" || echo "need: pip install pymupdf"
which tesseract >/dev/null 2>&1 && echo "tesseract OK" || echo "OCR unavailable"
```

缺失时安装（environ 里没有就提示用户）：
```bash
pip install pypdf pymupdf
# OCR（可选）：
# macOS: brew install tesseract tesseract-lang
# Debian/Ubuntu: sudo apt install tesseract-ocr tesseract-ocr-chi-sim
```

## 提文本（理解的基础）

### 首选 pymupdf（更快更准）

```python
import fitz
doc = fitz.open("文件.pdf")
print(f"页数: {doc.page_count}")
text = "\n".join(page.get_text() for page in doc)
print(text[:4000])  # 头4000字喂给模型
```

### 分页提取（避免超长截断）

```python
import fitz
doc = fitz.open("文件.pdf")
for i, page in enumerate(doc):
    t = page.get_text()
    if t.strip():
        print(f"--- 第{i+1}页 ---\n{t[:1500]}")
```

### 元数据

```python
import fitz
doc = fitz.open("文件.pdf")
print({k: v for k, v in doc.metadata.items() if v})
```

## 提取表格

pymupdf 自带表识别（v1.23+）：

```python
import fitz
doc = fitz.open("表格.pdf")
for page in doc:
    tables = page.find_tables()
    for tb in tables:
        for row in tb.extract():
            print("\t".join(c or "" for c in row))
```

## OCR 扫描件

页面没有文本层（`get_text()` 为空）时走 OCR：

```bash
# 单页 → 高分辨率图片再识别
python3 - <<'EOF'
import fitz
doc = fitz.open("scan.pdf")
for i, page in enumerate(doc[:5]):
    pix = page.get_pixmap(dpi=300)
    pix.save(f"/tmp/page_{i+1}.png")
EOF
for f in /tmp/page_*.png; do tesseract "$f" "${f%.png}" -l chi_sim+eng 2>/dev/null; cat "${f%.png}.txt"; done
```

## 合并 / 拆分 / 旋转 / 水印

```python
import fitz

# 合并
merged = fitz.open()
for f in ["a.pdf", "b.pdf"]:
    src = fitz.open(f)
    merged.insert_pdf(src)
merged.save("合并.pdf")

# 拆分（每页一个文件）
doc = fitz.open("大.pdf")
for i, page in enumerate(doc):
    out = fitz.open()
    out.insert_pdf(doc, from_page=i, to_page=i)
    out.save(f"页{i+1}.pdf")

# 旋转
doc = fitz.open("横.pdf")
for page in doc:
    page.set_rotation(90)
doc.save("竖.pdf")

# 加水印
doc = fitz.open("原.pdf")
for page in doc:
    page.insert_text((72, 72), "内部资料", fontsize=28, rotate=0,
                     color=(0.8, 0.8, 0.8), overlay=True)
doc.save("带水印.pdf")
```

## 交给模型回答

提取完文本后，把**文本原文（必要时截断到上下文限制）**放进模型输入，让模型回答：
- 总结：给前 3000-5000 字符的主体文本即可
- 具体问答：先定位页（搜关键词）再针对性地给对应段落
- 数据抽取：把含数据的表格文本原样给出

**铁律**：回答必须以提取出的真实内容为依据，PDF 没写的不要编。提取不到就把原文段落贴给模型，让模型自己读。

## 常见坑

- 空 `get_text()` = 扫描件，必须 OCR，别拿着空文本硬回答
- 中文 PDF 用 `fitz` 通常直接正常；乱码时检查字体嵌入
- 超大 PDF（>100页）先 `get_toc()` 给模型目录再按需提正文
- 加密 PDF 用 `doc.authenticate("密码")`，无密码则提示用户
