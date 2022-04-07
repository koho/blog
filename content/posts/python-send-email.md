---
title: "Python 发送 Email 通知"
date: 2021-04-20T01:36:21+08:00
archives: 
    - 2021
tags:
    - Python
image: /images/email.webp
draft: false
---

感觉没什么好说的，代码看一眼就明了，当备忘吧。

```python
import smtplib
from email.header import Header
from email.mime.text import MIMEText
from email.utils import formataddr

def send_email(subject, text, receiver):
    message = MIMEText(text, 'plain', 'utf-8')
    message['From'] = formataddr(("发送人名字", '发送人账号'))
    message['To'] = formataddr(("", receiver))

    message['Subject'] = Header(subject, 'utf-8')

    smtp = smtplib.SMTP_SSL("SMTP 服务器", 465)
    smtp.login("发送人账号", "客户端密码")
    smtp.sendmail('发送人账号', receiver, message.as_string())
    smtp.quit()
```
