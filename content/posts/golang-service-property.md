---
title: "Golang 打开 Windows 服务属性窗口"
date: 2021-08-01T11:35:17+08:00
archives: 
    - 2021
tags:
    - Golang
    - Windows
image: /images/windows-service-properties.webp
draft: false
---

本文介绍如何在 Golang 程序里面打开某个 Windows 服务的属性窗口，这里需要用到 COM 接口，详细文档请参考 MMC 2.0 的[官方文档](https://docs.microsoft.com/en-us/previous-versions/windows/desktop/mmc/mmc-programmer-s-guide-overview)。

既然是 COM 的话，程序里也可以使用`cmd`或者`powershell`来调用 COM 打开服务窗口。下面介绍直接调用 COM 组件的方法：

使用 [github.com/go-ole/go-ole](https://github.com/go-ole/go-ole) 这个库调用 COM：
```go
import (
	"github.com/go-ole/go-ole"
	"github.com/go-ole/go-ole/oleutil"
	"time"
)

// 服务的显示名称
displayName := "Windows Update"

ole.CoInitialize(0)
unknown, _ := oleutil.CreateObject("MMC20.Application")
mmc, _ = unknown.QueryInterface(ole.IID_IDispatch)
// 读取服务
oleutil.MustCallMethod(mmc, "Load", "services.msc")
document := oleutil.MustGetProperty(mmc, "Document").ToIDispatch()
view := oleutil.MustGetProperty(document, "ActiveView").ToIDispatch()
list := oleutil.MustGetProperty(view, "ListItems").ToIDispatch()
count := int(oleutil.MustGetProperty(list, "Count").Val)
// 注意索引从1开始
for i := 1; i <= count; i++ {
    item := oleutil.MustCallMethod(list, "Item", i).ToIDispatch()
	name := oleutil.MustGetProperty(item, "Name").ToString()
	if name == displayName {
	    oleutil.MustCallMethod(view, "Select", item)
	    // 显示属性窗口
		oleutil.MustCallMethod(view, "DisplaySelectionPropertySheet")
		item.Release()
		break
	}
	item.Release()
}
time.Sleep(60 * time.Second)
// 关闭窗口
oleutil.MustCallMethod(document, "Close", false)
list.Release()
view.Release()
document.Release()
// 等待窗口关闭，不然会弹出提示框
time.Sleep(2 * time.Second)
// 退出mmc
oleutil.CallMethod(mmc, "Quit")
mmc.Release()
ole.CoUninitialize()
```
