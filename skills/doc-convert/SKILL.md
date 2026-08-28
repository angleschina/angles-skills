---
name: doc-convert
description: >
  在文档格式之间相互转换：道/PDF↔Word/Word↔PDF/Markdown↔Word/Markdown↔PDF/图片合并成PDF/HTML↔PDF/CSV↔Excel 等。适合把文件转成用户需要交付或消费的格式。
  当用户要求"转成 PDF"、"把 Word 转成 PDF"、"把 Markdown 转成 Word"、"图片合成一个 PDF"、"把表格导成 Excel"等格式转换时触发。
license: MIT
compatibility: pandoc (markdown/html↔word/pdf) + libreoffice(headless, office格式互相转) + python(图片合成pdf)
---

# 文档格式转换

用现成工具（pandoc / libreoffice / python）做格式互转。**先确认输入格式，再选对工具**，别硬套错命令。

## When to use

- 用户给任意文档要求"转成 XX 格式"
- Markdown → Word/PDF/HTML
- Word ↔ PDF
- 图片列表 → 单个 PDF
- CSV → Excel / 表格 → 图片

## 依赖检查

```bash
which pandoc && echo "pandoc OK" || echo "need pandoc"
which libreoffice soffice 2>/dev/null && echo "libreoffice OK" || echo "need libreoffice"
python3 -c "import fitz" 2>/dev/null && echo "pymupdf OK" || echo "need pip install pymupdf"
```

## 转换速查表（本 skill 主推命令）

| 从 → 到 | 命令 |
|---|---|
| Markdown → Word | `pandoc 文件.md -o 文件.docx` |
| Markdown → PDF | `pandoc 文件.md -o 文件.pdf --pdf-engine=weasyprint`（中文友好）|
| Markdown → HTML | `pandoc 文件.md -o 文件.html --standalone --metadata title="标题"` |
| Word → PDF | `libreoffice --headless --convert-to pdf 文件.docx --outdir ./` |
| Word → Markdown | `pandoc 文件.docx -t gfm -o 文件.md` |
| HTML → PDF | `weasyprint 页.html 文件.pdf`（或 `libreoffice --headless --convert-to pdf`）|
| CSV → Excel | `python3 -c "import pandas as pd; pd.read_csv('数据.csv',encoding='utf-8-sig').to_excel('数据.xlsx',index=False)"` |
| 多张图 → PDF | 见下方 Python 段 |
| PPTX → PDF | `libreoffice --headless --convert-to pdf 演示.pptx --outdir ./` |

## 中文 PDF 注意（pandoc）

直接 `--pdf-engine=xelatex` 中文易乱，**weasyprint 对中文最省心**：

```bash
pandoc 报告.md -o 报告.pdf --pdf-engine=weasyprint
# 或手动指定中文字体
pandoc 报告.md -o 报告.pdf --pdf-engine=weasyprint \
  -V mainfont="PingFang SC" -V fontsize=11pt
```

无 weasyprint 可依赖系统已有 xelatex：
```bash
pandoc 报告.md -o 报告.pdf --pdf-engine=xelatex \
  -V CJKmainfont="PingFang SC"
```

## 图片 → 单 PDF（pymupdf，最稳）

```python
import fitz
imgs = ["1.png", "2.jpg", "3.png"]          # 保持顺序
doc = fitz.open()
for fn in imgs:
    rect = fitz.paper_rect("a4")            # 或传入图片自身尺寸
    page = doc.new_page(width=rect.width, height=rect.height)
    page.insert_image(rect, filename=fn)
doc.save("图集.pdf")
print("已生成 图集.pdf，共", len(imgs), "页")
```

## 批量转换（for 循环）

```bash
for f in *.md; do pandoc "$f" -o "${f%.md}.docx"; done
# 或全部 .docx → .pdf
for f in *.docx; do libreoffice --headless --convert-to pdf "$f" --outdir ./; done
```

## 常见坑

- **找不到 `libreoffice` 命令**：尝试 `soffice`，或完整路径 `/Applications/LibreOffice.app/Contents/MacOS/soffice`（macOS）
- pandoc 中文 PDF：永远优先 weasyprint，xelatex 需要中文字体套件，别裸跑
- Word → Markdown 带 `-t gfm` 保持表格语法；默认 md 表格可能变 HTML
- 图片合成 PDF 用 pymupdf 按顺序插页，别用 ImageMagick（慢且丢清晰度）
- 转换先确认输入文件存在、编码正确（CSV 用 utf-8-sig）
- 输出同名时加 `--outdir` 分隔，避免覆盖源文件

## 交付

转换后确认目标文件已生成（`ls -la 输出文件`），并把产出路径告知用户，方便打开。
