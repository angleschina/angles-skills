---
name: static-webgen
description: >
  生成静态网页（HTML/CSS/JS 直接可用，无需后端）。适合落地页、个人站点/简历、单页文档、展示页、工具小站；可产出纯 HTML、或多页 + 简单路由(锚点/切换)，内联或外链样式脚本，中文适配移动端。一键产出能在任意静态托管上线的完整文件。
  当用户说"做个网页/落地页/小站/简历页/展示页""HTML 页面""写个纯前端页面""不跑后端的站点" 时触发。
license: MIT
compatibility: 纯 静态 HTMl/CSS/JS，无后端无依赖；本机可直接生成用文本工具(nano/cat/python 写文件)，
---

# 静态网页生成

生成 **纯前端无需后端** 的网页文件，交付即可在浏览器打开 / 任意静态托管(GitHub Pages / Vercel / 云服务器 nginx)上线。

## When to use

- 落地页 / 产品介绍 / 个人主页 / 简历 / 活动页
- 单页工具(计算器/查表把玩/图表展示)，数据前端写死或从接口 fetch
- 多"页"(比如 tab / 单屏切换)站，不需求登录不涉及后端
- 用户明确"不要后端，能打开就行"

## 核心：先问清 3 件事

1. **用途 & 受众**（产品页？简历？小工具？）→ 决定排版/文案调性
2. **单页 or 多区块/多“页”** → 决定结构：单 <main> vs tab 切换
3. **有无真数据/会不会变** → 纯静态固定内容，还是需动态拼（动态需求转到 dynamic-webgen）

## 交付形态（最低->丰富）

**A. 一个自包含 HTML**（最适合落地页/简历，复制即用）：
```html
<!DOCTYPE html>
<html lang="zh">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>我的简历</title>
<style>/* 少量内联样式，移动优先 */</style>
</head>
<body>
  <header><h1>张三 · 全栈工程师</h1><p>坐标杭州</p></header>
  <main>
    <section><h2>简介</h2><p>...</p></section>
    <section><h2>项目</h2><ul>...</ul></section>
  </main>
  <footer>© 2026</footer>
</body>
</html>
```

**B. 多文件工程**（多页/样式分离）：
```
site/
├── index.html
├── about.html
├── style.css
├── main.js
└── assets/ favicon.png
```

**C. 生成式页用脚本**：内容列表/卡片多时，用 python 脚本循环生成片段再组装，避免手打重复结构。

## 静态页该有（页面骨架别忘了）

```html
<meta name="viewport" content="width=device-width, initial-scale=1">   <!-- 移动必备 -->
<html lang="zh">                                                       <!-- SEO/读屏 -->
<title>…</title> + <meta name="description" content="…">               <!-- 搜到 -->
<link 图标/> <style>：reset(可简) + flex/grid 布局 + 字体/间距变量
```

若用户未给方向，给一份移动优先 + 响应断点的默认 CSS：
```css
:root{ --bg:#fff; --ink:#111; --brand:#0af; }
*{ box-sizing:border-box; margin:0; }
body{ font-family: system-ui, "PingFang SC", "Microsoft YaHei", sans-serif;
      color:var(--ink); background:var(--bg); line-height:1.6; padding:1rem; max-width:760px; margin:auto;}
img{ max-width:100%; height:auto; }
@media (min-width:640px){ .cards{ display:grid; grid-template-columns:repeat(auto-fit,minmax(220px,1fr)); gap:1rem;} }
```

## 多区块导航 / 高亮

落地页常用锚点 + 顶部固定栏：
```html
<nav style="position:sticky;top:0;background:#fff;display:flex;gap:1rem;justify-content:center;padding:.6rem;">
  <a href="#intro">简介</a><a href="#offer">产品</a><a href="#contact">联系</a>
</nav>
<main>
  <section id="intro">…</section><section id="offer">…</section><section id="contact">…</section>
</main>
```
实现平滑滚动可有 `html{scroll-behavior:smooth}` + `scroll-margin-top` 防被导航盖住。

## 常用纯前端小组件（内嵌即可、无依赖）

```js
// Tab / 手风琴切换：一点 JS 监听隐藏
document.querySelectorAll('[data-tab]').forEach(btn=>{
  btn.onclick=()=>document.querySelectorAll('.pane').forEach(p=>p.hidden=p.id!==btn.dataset.tab);
});
```

```js
// 从公开 API 拉数据显示（静态照样可用 fetch JSON —— 数据不变就把结果嵌进 HTML 更好）
fetch('https://api.example.com/data').then(r=>r.json()).then(render);
```

```html
<!-- 嵌入视频/地图(得到 embed 地址) -->
<iframe src="https://www.youtube.com/embed/..."></iframe>
<iframe src="https://map.baidu.com/…"></iframe>
```

**优先静态内容**：能用固定数据就别 defer fetch —— 快、不怕离线、不被跨域/密钥卡。

## 交付 & 预览

- 产出到文件后用能打开静态的方式预览（本地 `python3 -m http.server 8000`，或让用户在浏览器直接打开单 html 也是静态就够）
- 若文件夹多文件，注意相对引用 `./style.css`、`../assets/logo.png`；页面能被 <img src> 相对路径读到
- 结尾告诉用户：可直接双击 html 看效果；上线走 GitHub Pages/Vercel 拖拽/或 nginx 放目录即可

## 常见坑

- **http.server 跨站**：若页面用 fetch 远程接口有 CORS 限制，静态托管同域安全不需要特殊处理，本地起 http 也行
- 中文乱码：一定 `<meta charset="utf-8">`
- 移动白边：忘 viewport meta —— 最大而常见问题
- 图标/emoji 找不到：检查相对路径、assets 是否与 html 同级于对
- 用户要"会登录/表单能存/数据后端" → 这是动态需求，转 `dynamic-webgen`
- 过度 JS 框架：落地页/简历纯 CSS+一行 JS 足矣，别上 build 链

## 原则

**目标是"给一段能直接跑、移动友好、结构清晰的静态页面"，用最简 CSS+零/少 JS 完成。** 数据和交互不够静态覆盖才升级到 dynamic skill。
