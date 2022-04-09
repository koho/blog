---
title: "Windows 创建临时文件"
date: 2021-07-19T09:36:38+08:00
archives: 
    - 2021
tags:
    - Windows
image: /images/temp-file.jpg
draft: false
---

在 Windows 上希望创建一个临时文件，该文件在程序退出时自动删除，即使是被强制停止的。这时需要使用 [CreateFile](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-createfilea) 这个函数：
```c
HANDLE CreateFileA(
  [in]           LPCSTR                lpFileName, // 指向文件名的指针
  [in]           DWORD                 dwDesiredAccess, // 访问模式，写/读
  [in]           DWORD                 dwShareMode, // 共享模式
  [in, optional] LPSECURITY_ATTRIBUTES lpSecurityAttributes, // 指向安全属性的指针
  [in]           DWORD                 dwCreationDisposition, // 如何创建
  [in]           DWORD                 dwFlagsAndAttributes, // 文件属性
  [in, optional] HANDLE                hTemplateFile // 用于复制文件句柄
);
```
重点关注`dwShareMode`和`dwFlagsAndAttributes`这两个参数。

共享模式`dwShareMode`有四种选项，分别为
- 0：禁止其他进程进行删除、读取和写入
- FILE_SHARE_DELETE：允许删除
- FILE_SHARE_READ：允许读取
- FILE_SHARE_WRITE：允许写入

文件属性`dwFlagsAndAttributes`需要设定两项：
- FILE_ATTRIBUTE_TEMPORARY：该文件为临时文件，尽可能使用内存缓存，避免写入磁盘。
- FILE_FLAG_DELETE_ON_CLOSE：当所有句柄被关闭时，立刻删除文件。

实验：
```go
h, _ := windows.CreateFile(
    windows.StringToUTF16Ptr("tmp"), windows.GENERIC_WRITE, windows.FILE_SHARE_READ, nil,
    windows.OPEN_ALWAYS, windows.FILE_ATTRIBUTE_TEMPORARY|windows.FILE_FLAG_DELETE_ON_CLOSE,
	0,
)
var done uint32
windows.WriteFile(h, []byte("hello world"), &done, nil)
time.Sleep(60 * time.Second)
windows.CloseHandle(h)
```
发现`tmp`文件被成功创建，但当用记事本打开时，提示另一个程序正在使用此文件，进程无法访问。我们在创建时已经指定了`FILE_SHARE_READ`了，应该可以被其他进程读取才对。
很多文本编辑器都无法正常打开这个文件，但有例外的，比如 Sublime Text，Git Bash 的`cat`命令(和 MinGW 有关)等，都能正常读取文件内容。

调查发现，当文件指定了`FILE_FLAG_DELETE_ON_CLOSE`时，其他进程必须用`FILE_SHARE_DELETE`模式才能正常打开文件。目前很多程序或编程语言都没有默认使用这个模式打开文件的，因此大部分程序都打不开。
Golang 曾有这方面的[讨论](https://github.com/golang/go/issues/32088)，但最终没有被采用。
如果读取的程序是自己开发的，那可以手动用这个 API 函数读取文件；如果是其他第三方程序，那可能需要考虑其他方法了。
