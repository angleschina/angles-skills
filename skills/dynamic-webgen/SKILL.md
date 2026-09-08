---
name: dynamic-webgen
description: |
  生成带后端的动态网站：有路由、数据读写、表单能提交存储、登录鉴权、模板渲染。覆盖 Node（Express）与 Python（Flask / FastAPI）的最小可用 Web 服务加前端模板，配合 SQLite / JSON 持久化。适合需要提交保存数据、登录、按用户数据展示、对用户动态响应的小型 Web 应用。当用户要能存数据的网站、带后端的页面、提交表单能保存、需要登录的网页小程序、做个小 Web 应用时触发。
license: MIT
compatibility: 交付前端模板 + 服务端源码 + 工程结构；本机可跑 Node / Python 验证基本路由（iPhone 上 iSH 自带 python node）。真实部署由有端服务器完成。
---

# 动态网站生成 (带后端, 最小可用)

为需要 **路由 + 数据持久化 + 交互(存/查)** 的动态 Web 应用生成最小可用实现。默认不引重框架，单文件后端起步，SQLite/JSON 存数据。

## When to use（先分清这是不是动态）

用户提出这些 → **是**（转到这里）：
- "能提交保存的数据 / 留言 / 报名 / 收藏"
- "需要登录,不同人有不同内容"
- "点按钮更新/后台/每用户状态"
- "对接我打接口 / fetch 后端"

纯展示、不写死数据就交互后即变 → 如果只是展示不变 → 用 static-webgen。

## 三选一技术栈（默认按运行环境推荐）

| 场景 | 推荐 | 服务端写法要点 |
|---|---|---|
| 通用、Node 熟 | **Node + Express + SQLite(better-sqlite3)** | 大生态 |
| Python 熟 / AI agent 环境 | **Python Flask 或 FastAPI** | 直接 pip; Flask 更轻, FastAPI 自带文档 |
| 极力最小、单 exe/脚本 | Python http.server + 手写 | 仅演示 |
| 已经有前端框架比如 react | 那给 API+前端的拆分也 OK | 让它明白我们做 API + fetch |

给用户最能跑的那套，别问太多技术选择，按用户环境定一个，其余一句话说"也可用 X".

## 最小可用：Flask + SQLite（推给 Python 环境）

工程：
```
app/
├── app.py            # Flask 后端
├── database.db       # 运行时生成
├── templates/
│   └── index.html    # 页面
└── static/           # CSS/JS(可选)
```

**app.py**：增删查 + 一个表单页,一条数据:

```python
from flask import Flask, render_template, request, redirect
import sqlite3, os

app = Flask(__name__)
DB = "database.db"

def init():
    db = sqlite3.connect(DB)
    db.execute("CREATE TABLE IF NOT EXISTS notes(id INTEGER PRIMARY KEY, body TEXT, created TEXT DEFAULT CURRENT_TIMESTAMP)")
    db.commit(); db.close()

@app.route("/")
def home():
    db = sqlite3.connect(DB)
    rows = db.execute("SELECT id, body FROM notes ORDER BY id DESC").fetchall()
    db.close()
    return render_template("index.html", notes=rows)

@app.route("/add", methods=["POST"])
def add():
    body = request.form.get("text", "").strip()
    if body:
        db = sqlite3.connect(DB)
        db.execute("INSERT INTO notes(body) VALUES(?)", (body,))
        db.commit(); db.close()
    return redirect("/")

init()
app.run(host="0.0.0.0", port=8080, debug=True)
```

**templates/index.html**（模板渲染,值要转义防 XSS）:
```html
<!DOCTYPE html><html lang="zh"><head><meta charset="utf-8">
<meta name="viewport"..." > <title>便签</title></head><body>
<h1>我的便签</h1>
<form method="post" action="/add">
  <input name="text" placeholder="写点什么" required>
  <button>保存</button>
</form>
<ul>{% for n in notes %}<li>{{ n[1] }}</li>{% else %}<li>暂无</li>{% endfor %}</ul>
</body></html>
```

跑法:`python3 -m venv venv && source venv/bin/activate && pip install flask && python app.py`,浏览器开 http://localhost:8080。SQLite 数据自动持久化，重启不丢——这就是"动态/会保存"要的效果。

## Node + Express 等价最小版

```js
// app.js
const express=require('express');
const Database=require('better-sqlite3');
const app=express();
const db=new Database('app.db');
db.exec('CREATE TABLE IF NOT EXISTS notes(id INTEGER PRIMARY KEY AUTOINCREMENT, body TEXT)');
app.use(express.urlencoded({extended:true}));
app.set('view engine','ejs');            // npm i express better-sqlite3 ejs
app.get('/',(req,res)=>{
  const rows=db.prepare('SELECT * FROM notes ORDER BY id DESC').all();
  res.render('index',{rows});
});
app.post('/add',(req,res)=>{
  db.prepare('INSERT INTO notes(body) VALUES(?)').run(req.body.text);
  res.redirect('/');
});
app.listen(8080);
```
`views/index.ejs` 一行渲染:`<ul><% rows.forEach(r=>{%><li><%= r.body %></li><%})%></ul>`。记得 `res.redirect` 用 POST/Redirect/GET 防重复提交。

## 登录 / 鉴权（最小可跑）

别自己写复杂 session。给最小 cookie/session 实现而不是教密码学：
- 用框架内置: Flask `session` 或 Express `express-session`
- 只演示 "登录一个固定账号后存 user_id",**密码 hash 必用 bcrypt / werkzeug**,别存明文:

```python
from flask import session
from werkzeug.security import check_password_hash
# 登录验证通过: session['uid']=row['id']
# 访问保护: 若 uid 不在 session 则 redirect('/login')
```

提供用户做 "简单鉴权" 的完整代码而不丢安全细节(存 hash, 用框架 session)。

## 交付时一并给的"怎么跑 + 坑"

- `requirements.txt` / `package.json` 都给，克隆就能装
- 提醒 **host 0.0.0.0 才是别人能访问**，本地 localhost 只有自己
- SQLite 写在本地文件在服务器要持久盘(/data/xxx.db),别放 /tmp、别随容器重置灭数据
- 数据库字段全用参数化(上面的 `?`)，**永不 f-string 拼 SQL** → 防注入,这个必须写进代码与示例
- API 返回前端直接 fetch:JSON、CORS 若前后端不同源则加 `flask-cors`/设置允许(仅开发)

## 常见坑

- 表单提示 405/404 : 路由 method 没写 POST / action 错; 检查 form + route 一致
- 刷新重复提交 : 一定要 POST → redirect(GET) (代码里的 redirect("/"))
- XSS : 模板默认转义(EJS/Jinja 自动),别拿 `{{ n[1]|safe }}` 裸信任
- 数据库文件写入往只读目录 / 容器重启丢数据 → 用固定持久路径
- "能 run" 用 localhost(绑 127.0.0.1),要能被局域网/公网访问则绑 0.0.0.0 + 防火墙放端口
- 加了框架却记不住就跑 `--help`/骨架太久 → 先跑通单路由再加

## 原则

**目标是让用户在一条"能增删改查 + 存活的数据库 + 有基本登录"的路径上拿到可部署的最小代码库——而不是丢几十框架文件。** 路由/模板/DB写入都给全可跑版本，安全(参数化/哈希/转义)当默认铁律写进去。
