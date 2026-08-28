---
name: mail-operations
description: >
  收发并处理电子邮件。可列出收件箱、读取某封邮件正文/附件、按主题/发件人筛选、回复/转发邮件、对未读或指定邮件做自动回复，以及把关键信息摘要出来交给你判断。适合"帮我看看邮件""回复这封""自动回复询价/咨询""整理邮件简报"等需求。
  当用户提到邮件、收件箱、发件人、自动回复、转发邮件时触发。
license: MIT
compatibility: python3 + imaplib(收) / smtplib(发)，邮箱需开启 IMAP/SMTP 并配置授权码，或走组织已有邮件 API
---

# 邮件收发与处理

基于 IMAP（收） + SMTP（发）的轻量邮件处理。**理解正文靠模型，收发/筛选靠库，别手动拼协议。**

## When to use

- 用户要"看邮件""有没有新邮件""这封说了什么"
- 需要按发件人/主题/时间筛选、提取关键信息
- 需要回复、转发，或对特定邮件自动回复
- 定期汇总邮件成简报

## 前置：邮箱配置

需要 IMAP/SMTP 服务器 + 授权码（不是登录密码）。环境变量建议：

```bash
MAIL_HOST=imap.qq.com         # 收件服务器（QQ邮箱示例，企业邮箱各自不同）
MAIL_USER=you@example.com
MAIL_PASS=授权码              # IMAP/SMTP 专用授权码
SMTP_HOST=smtp.qq.com
```

**安全**：授权码、密码等凭据不要写进脚本/对话明文，用环境变量传入。

## 收邮件：列出/读正文（理解基础）

```python
import os, imaplib, email, email.header
from email.utils import parsedate_to_datetime

host, user, pwd = os.environ["MAIL_HOST"], os.environ["MAIL_USER"], os.environ["MAIL_PASS"]
imap = imaplib.IMAP4_SSL(host)
imap.login(user, pwd)
imap.select("INBOX")

# 取最近 N 封
typ, data = imap.search(None, "ALL")
ids = data[0].split()[-10:]   # 最近10封
for mid in reversed(ids):
    typ, msg_data = imap.fetch(mid, "(RFC822)")
    msg = email.message_from_bytes(msg_data[0][1])
    subj = str(email.header.make_header(email.header.decode_header(msg["Subject"])))
    from_ = msg["From"]
    date = parsedate_to_datetime(msg["Date"]).strftime("%Y-%m-%d %H:%M")
    print(f"[{date}] {from_} ｜ {subj}")

# 读某一封正文（含纯文本/HTML回退）
typ, msg_data = imap.fetch(ids[-1], "(RFC822)")
msg = email.message_from_bytes(msg_data[0][1])
body = ""
if msg.is_multipart():
    for part in msg.walk():
        if part.get_content_type() == "text/plain":
            body = part.get_payload(decode=True).decode(part.get_content_charset() or "utf-8", "ignore")
            break
else:
    body = msg.get_payload(decode=True).decode(msg.get_content_charset() or "utf-8", "ignore")
print(body[:4000])
```

## 筛选未读 / 特定发件人

```python
# 未读
typ, data = imap.search(None, 'UNSEEN')
# 某发件人
typ, data = imap.search(None, 'FROM "sales@example.com"')
# 主题含关键词
typ, data = imap.search(None, 'SUBJECT "报价"')
# 组合
typ, data = imap.search(None, '(UNSEEN FROM "a@b.com")')
```

## 发邮件：回复 / 新建

```python
import os, smtplib
from email.mime.text import MIMEText
from email.utils import formataddr

def send(subject, to_addr, body, reply_to=None):
    msg = MIMEText(body, "plain", "utf-8")
    msg["Subject"] = subject
    msg["From"] = formataddr(("angles", os.environ["MAIL_USER"]))
    msg["To"] = to_addr
    if reply_to:  # 回复时带 In-Reply-To/References 让对方Threading
        msg["In-Reply-To"] = reply_to
        msg["References"] = reply_to
    s = smtplib.SMTP_SSL(os.environ["SMTP_HOST"], 465)
    s.login(os.environ["MAIL_USER"], os.environ["MAIL_PASS"])
    s.send_message(msg)
    s.quit()

send("Re: 关于报价", "customer@example.com", "您好，报价答复见正文……")
```

## 附件处理

```python
for part in msg.walk():
    if part.get_content_disposition() == "attachment" and part.get_filename():
        fn = email.header.make_header(email.header.decode_header(part.get_filename()))
        open(f"/tmp/{fn}", "wb").write(part.get_payload(decode=True))
        print("已存附件:", fn)   # 交给对应 skill 处理(如 doc-convert/pdf-understand)
```

## 交给模型回答

- "讲了什么"：把 `[date] from｜subject` + 正文首 2000-3000 字喂给模型
- 筛选/分类：把多封的 `subject+摘要` 列表给模型分类/汇总
- 回复/自动回复：模型生成正文 → `send()` 发出前**先给用户看一眼**（重要邮件别盲目自动发出）
- 附件拿到后要解读，切到对应的 pdf/word/excel skill

## 安全与克制（重要）

- **带凭据**：所有邮箱操作走环境变量，杜绝明文密码入脚本/对话
- **自动回复要谨慎**：只有用户明确授权才自动 `send()`；对外邮件默认先给草稿确认
- **范围**：只读用户明确要求看的邮箱/文件夹，别枚举狂拉全部
- 邮件含隐私，回传模型时只带处理所需字段，不整包外泄

## 常见坑

- 正文编码不对 → 用 `part.get_content_charset()` 而非硬编码 utf-8
- 主题/发件人 `=?utf-8?B?...?=` 编码 → 一定用 `email.header.make_header(decode_header(...))`
- 附件中文名乱码 → 同上 header 解码
- IMAP 连不上 → 确认是否开启该邮箱的 IMAP/SMTP 权限（QQ/163 需在设置开启）
- 企业邮箱用固定 SMTP/IMAP 地址和端口，与公网邮箱不同，先确认
