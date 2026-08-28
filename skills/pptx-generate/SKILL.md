---
name: pptx-generate
description: >
  从零生成 PowerPoint 演示文稿（.pptx），或基于 Markdown/JSON 大纲批量生成结构化幻灯片。支持标题页、bullets、分栏、表格、图表(柱状/折线/饼图)和图片插入。适合汇报、教程、产品介绍等场景。
  当用户要求"做一个 PPT"、"生成幻灯片"、"把这份提纲做成演示文稿"、提供 .md 大纲或要求批量建幻灯片时触发。
license: MIT
compatibility: python3 + python-pptx; 图表可选 pillow/matplotlib
---

# PPTX 生成

用 python-pptx 从零搭幻灯片。**工作流：先由模型（你）生成结构化大纲 → 脚本渲染成 .pptx → 打开/保存。**

## When to use

- 用户要"做 PPT / 幻灯片 / 演示文稿"
- 给一段 Markdown 或文字提纲，要求排成多页
- 要自动生成带图表、表格、图片的汇报 deck

## 依赖检查

```bash
python3 -c "import pptx" 2>/dev/null && echo "python-pptx OK" || echo "need: pip install python-pptx"
# 图表绘图（可选）：
pip install matplotlib  # 生成图表 PNG 再贴
```

## 最小可运行生成脚本骨架

先让文件能打开，再谈美观。**每页 = 一个 add_slide**：

```python
from pptx import Presentation
from pptx.util import Inches, Pt
from pptx.dml.color import RGBColor

prs = Presentation()

# 1. 标题页（使用"标题和内容"布局之一）
slide = prs.slides.add_slide(prs.slide_layouts[0])
slide.shapes.title.text = "项目启动汇报"
slide.placeholders[1].text = "张三 ｜ 2026-08-28"

# 2. 内容页
def content_slide(title, bullets):
    s = prs.slides.add_slide(prs.slide_layouts[1])
    s.shapes.title.text = title
    body = s.placeholders[1].text_frame
    body.text = bullets[0]
    for b in bullets[1:]:
        p = body.add_paragraph()
        p.text = b
        p.level = 1 if b.startswith("-") else 0
    return s

content_slide("核心要点", ["第一点", "- 子点1", "- 子点2", "第二点"])

prs.save("汇报.pptx")
print("已保存 汇报.pptx，共", len(prs.slides.__iter__.__self__._sldIdLst), "页")
```

## 结构化大纲 → 多页（推荐）

你（agent）把用户需求整理成 JSON 大纲，再喂脚本批量渲染：

```bash
python3 - <<'EOF' 大纲.json  # 或把 JSON 直接嵌进脚本
EOF
import json, sys
from pptx import Presentation
from pptx.util import Inches

with open(sys.argv[1]) as f:
    deck = json.load(f)   # {"title":..., "pages":[{"title":..,"bullets":[...]}]}

prs = Presentation()
# 封面
s = prs.slides.add_slide(prs.slide_layouts[0])
s.shapes.title.text = deck["title"]
# 内容页
for pg in deck["pages"]:
    s = prs.slides.add_slide(prs.slide_layouts[1])
    s.shapes.title.text = pg["title"]
    tf = s.placeholders[1].text_frame
    tf.text = pg["bullets"][0] if pg.get("bullets") else ""
    for b in pg["bullets"][1:]:
        tf.add_paragraph().text = b

prs.save(deck.get("outfile", "输出.pptx"))
print("已生成", len(deck["pages"]) + 1, "页")
EOF
```

## 添加表格

```python
from pptx.util import Inches
from pptx import Presentation

prs = Presentation()
s = prs.slides.add_slide(prs.slide_layouts[5])  # 空白布局
table = s.shapes.add_table(rows=3, cols=3, left=Inches(1), top=Inches(1.5),
                           width=Inches(8), height=Inches(2)).table
data = [["季度","收入","增长"],["Q1","120万","8%"],["Q2","145万","21%"]]
for r, row in enumerate(data):
    for c, val in enumerate(row):
        table.cell(r, c).text = val
prs.save("含表格.pptx")
```

## 添加图表（matplotlib 出 PNG 再贴）

```python
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt

plt.plot([1,2,3,4],[120,145,168,190], marker="o")
plt.title("年度收入(万元)")
plt.savefig("/tmp/chart.png", dpi=150, bbox_inches="tight")

from pptx import Presentation
from pptx.util import Inches
prs = Presentation()
s = prs.slides.add_slide(prs.slide_layouts[5])
s.shapes.add_picture("/tmp/chart.png", Inches(1.5), Inches(1), width=Inches(6))
prs.save("含图表.pptx")
```

## 常见坑

- `python-pptx` 默认 4:3；要宽屏改 `prs.slide_width`/`height`（Emu：`Inches(13.333)` x `Inches(7.5)`）
- 占位符索引不固定 —— 布局 0=标题页, 1=标题+内容, 5=空白，先确认再写
- bullets 缩进：`paragraph.level = 1` 有子点更专业
- 中文标题懒加载字体没问题，但输出到没有中文字体的机器会显示方框——建议内置字体或用系统字体
- 生成后如果可能，在本地预览确认页面数和布局再交付

## 交给模型（你）做的部分

**别让脚本生成内容，让脚本渲染。** 内容（标题措辞、要点取舍、数据）由你按用户需求组织成大纲 JSON/文字；脚本只负责排版落地。这才是"用角度草生成 PPT"的正确姿势。
