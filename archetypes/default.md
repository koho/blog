---
title: "{{ replace .Name "-" " " | title }}"
date: {{ .Date }}
archives: 
    - {{ now.Format "2006" }}
tags:
    - 其他
image: /images/placeholder.png
leading: false
draft: true
---
