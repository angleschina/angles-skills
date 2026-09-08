# angles-skills

[`angles-cli`](https://github.com/ZSJ305/angles-cli) 的技能库（Skills）——一组开箱即用的文档/办公自动化与开发能力，让 angles 的 agent 能理解 PDF/Word/Excel/图片，生成 PPT 与 Markdown 报告，在文档格式间自由转换，也能辅助生成 iOS(WinUI/UWP)、Windows(WinUI 3 / WSL) 与 静态/动态网站 的源码与工程骨架。

## 技能列表

| Skill | 用途 | 触发词示例 |
|---|---|---|
| [pdf-understand](./skills/pdf-understand/SKILL.md) | 理解 PDF 内容、提取文本/表格、扫描件 OCR、合并拆分加水印 | "这个 PDF 讲了什么" / "把两个 PDF 合并" |
| [word-understand](./skills/word-understand/SKILL.md) | 理解 Word 文档、提取正文/标题/表格、编辑替换 | "读一下这份 docx" / "把文档里的公司名换掉" |
| [pptx-generate](./skills/pptx-generate/SKILL.md) | 从零生成 PPT，支持标题/bullets/表格/matplotlib 图表 | "做个汇报 PPT" / "按这个提纲生成幻灯片" |
| [excel-work](./skills/excel-work/SKILL.md) | 读写 Excel/CSV、查询统计、多表合并、生成报表 | "这表里销售额合计多少" / "把 CSV 导成 Excel" |
| [image-understand](./skills/image-understand/SKILL.md) | 看图理解、OCR 提字、二维码解码、图片裁剪压缩比对 | "看这张图" / "图里的字是什么" |
| [md-report](./skills/md-report/SKILL.md) | 生成结构化 Markdown 报告，可导出 Word/PDF/HTML | "写份周报" / "整理成 Markdown 文档" |
| [doc-convert](./skills/doc-convert/SKILL.md) | 文档格式互转：PDF↔Word、MD↔Word/PDF、图片合成 PDF 等 | "转成 PDF" / "图片合成一个 PDF" |
| [web-content-extractor](./skills/web-content-extractor/SKILL.md) | 从 URL 提取网页正文、去噪音、带元信息，长文转 Markdown | "读这个链接" / "总结这个网页" |
| [mail-operations](./skills/mail-operations/SKILL.md) | 邮件收发、筛选、读正文/附件、回复/自动回复、汇总 | "看看邮箱" / "回复这封邮件" |
| [json-processing](./skills/json-processing/SKILL.md) | JSON 查询/过滤/合并/排序/扁平化/格式化，转 Markdown | "从这段 JSON 取字段" / "合并两个 JSON" |
| [log-analyze](./skills/log-analyze/SKILL.md) | 日志过滤、错误频次统计、堆栈提取、时间窗定位、问题归纳 | "看看日志为什么报错" / "哪个错最多" |
| [ios-app-dev](./skills/ios-app-dev/SKILL.md) | 开发 iOS/iPadOS 原生应用：SwiftUI 源码、界面、数据持久化、工程骨架 | "做个 iOS 应用" / "写 SwiftUI 界面" |
| [winui-dev](./skills/winui-dev/SKILL.md) | 开发 Windows 原生应用：WinUI 3 (Windows App SDK) C#/XAML、MVVM、MSIX 打包 | "做个 Windows 应用" / "写 WinUI" |
| [wsl-usage](./skills/wsl-usage/SKILL.md) | 使用 Windows 上的 WSL(WSL2)：装/管理发行版、Linux↔Windows 文件与命令互访 | "在 WSL 里装什么/跑什么" / "怎么访问 Windows 文件" |
| [static-webgen](./skills/static-webgen/SKILL.md) | 生成静态网页(HTML/CSS/JS，无需后端)：落地页/简历/小站/工具页 | "做个网页/落地页/简历页" / "写个纯前端页" |
| [dynamic-webgen](./skills/dynamic-webgen/SKILL.md) | 生成带后端的动态网站：路由、数据读写、表单存储、登录鉴权(Flask/Express+SQLite) | "做个能存数据的网站" / "需要登录的网页小程序" |

## Skill 结构

每个 skill 是一个目录，核心文件 `SKILL.md`（YAML front-matter + Markdown 正文）：

```yaml
---
name: <skill 名>
description: >        # 命中该 skill 的触发条件和能力描述
  ...
license: MIT
compatibility: ...   # 所需运行时 / 依赖
---
# <技能正文，含依赖检查、代码示例、常见坑、使用铁律>
```

## 使用方式

skill 由 angles-cli 的 agent 在需要时按 `name: description` 匹配加载，按 SKILL.md 内的指引执行（通常调用 python3/pandoc/libreoffice 等工具完成实际工作），并用模型能力解读结果、给用户可用的答复。

```bash
# 依赖缺什么，skill 内都有依赖检查命令，按需安装即可
pip install pypdf pymupdf python-docx python-pptx openpyxl pandas pillow
brew install tesseract zbar   # macOS OCR / 二维码
brew install pandoc libreoffice  # 文档转换
```

## 贡献

- 新增 skill：在 `skills/<name>/` 下建 `SKILL.md`，头部 YAML 写清 `name` / `description`（触发词） / `license`，正文遵守"结论前置 + 可复现示例 + 常见坑"。
- 保持每个 skill 自包含：能独立安装依赖、直接跑通。
- 内容要求：以真实可查证据或可用代码为准，不给空话。

## License

MIT
