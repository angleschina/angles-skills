---
name: image-understand
description: >
  理解并分析图片（.png/.jpg/.jpeg/.gif/.webp/.bmp）。可做 OCR 提取图中文字、识别物体/场景、读取二维码、判断图片相似度。当图片是"视觉内容"时，优先交给带视觉能力的模型直接看图；本 skill 处理需要本地工具完成的部分（OCR、二维码、裁剪、相似度）。
  当用户提供图片、要"看这张图"、提取图中文字、识别二维码 或 比较两张图时触发。
license: MIT
compatibility: 视觉模型(可直读图) / python3 + pytesseract(opencv) / zbar 或 pyzbar 二维码 / 视觉框架(apple-vision)
---

# 图片理解与处理

关键判断：**你如果是多模态模型（能直接看图），图片直接用视觉输入喂给你，无需本地工具。** 只有这几类才走本地工具：
- OCR/读图中文字（模型文字识别不稳定时）
- 读二维码/条形码（一串编码，模型瞎猜不如解码）
- 裁剪/缩放/格式转换（改图）
- 判断两张图是否相同/重复（特征比对）

## 什么时候用视觉模型直接看

当图片由你（agent）携带进多模态请求时——直接把图片以 image/data URL 发给模型，让它描述。这是"看一张图"的正解，OCR 只影响纯文字提取需求。

## 本地工具：依赖检查

```bash
which tesseract && echo "tesseract OK" || echo "need: brew install tesseract"
python3 -c "import PIL" 2>/dev/null && echo "PIL OK" || echo "need: pip install pillow 或 apk add py3-pillow"
python3 -c "import pyzbar" 2>/dev/null && echo "pyzbar OK" || echo "need: pip install pyzbar (依赖 zbar)"
```

## OCR 提取图中文字

```python
import subprocess
# tesseract 命令行最省事
subprocess.run(["tesseract", "图.png", "/tmp/out", "-l", "chi_sim+eng", "--psm", "6"])
text = open("/tmp/out.txt", encoding="utf-8").read()
print(text)
```

psm 参数（排版复杂度）：
- `--psm 6`：整行文本（截图/网页）——最常用
- `--psm 3`：默认自动
- `--psm 11`：稀疏文本
- `--psm 7`：单行

若识别差，先提高分辨率再 OCR：
```python
from PIL import Image
im = Image.open("图.png")
im = im.resize((im.width*2, im.height*2), Image.LANCZOS)
im.save("/tmp/big.png")
```

## 二维码/条形码解码

```python
from pyzbar.pyzbar import decode
from PIL import Image
for code in decode(Image.open("qr.png")):
    print("类型:", code.type, "| 内容:", code.data.decode())
```

## 图像信息 + 裁剪/压缩/转换

```python
from PIL import Image
im = Image.open("照片.jpg")
print("尺寸:", im.size, "模式:", im.mode)

# 裁剪
im.crop((左, 上, 右, 下)).save("裁剪.jpg")
# 缩放到宽度 800
w, h = im.size
im.thumbnail((800, 800), Image.LANCZOS)
im.convert("RGB").save("小图.jpg", quality=85)
# 格式转换
im.convert("RGB").save("图.webp")
im.save("图.png")
```

## 两张图相似 / 是否同一张

若纯视觉模型能直判，直接看。需要可靠去重时：

```python
# 感知哈希（简单去重）
import hashlib
from PIL import Image
def phash(path, size=8):
    im = Image.open(path).convert("L").resize((size, size), Image.LANCZOS)
    px = list(im.getdata())
    avg = sum(px) / len(px)
    return "".join("1" if v >= avg else "0" for v in px)

h1, h2 = phash("a.jpg"), phash("b.jpg")
diff = sum(a != b for a, b in zip(h1, h2))
print("不同位:", diff, "（<10 视为相似）")
```

## 交给模型回答

- "看这张图描述一下"：直接把图片喂给视觉模型
- "图里这些字是什么"：OCR 提取后喂文本
- "这两个码是什么"：解码后把内容给模型
- 大图超限：先压缩到合适宽度再喂/再处理

## 常见坑

- 中文 OCR 必须 `-l chi_sim+eng`，只带 eng 中文全乱
- 图片方向不对 OCR 差——先 `rotate(90/180/270)` 试
- 不要靠模型"读二维码"，解码最稳
- GIF 有多帧，PIL 只给首帧，要点动图需逐帧处理
- 视觉模型直读时，分辨率极高的大图要先缩，control token 有限
