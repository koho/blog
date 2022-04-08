---
title: "Git 备忘"
date: 2022-01-16T15:22:17+08:00
archives: 
    - 2022
tags:
    - 开发工具
image: /images/git.jpg
draft: false
---

### 设置用户信息
```shell
git config --global --list
git config --global user.name "your-name"
git config --global user.email user@example.com
```

### 克隆仓库
```shell
git clone https://github.com/xxx/x.git
```
有子模块的：
```shell
git clone --recursive https://github.com/xxx/x.git
```

### 标签
#### 删除本地标签
```shell
git tag --delete tagname
```
#### 删除远程标签
```shell
git push --delete origin tagname
```
