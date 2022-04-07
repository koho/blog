---
title: "Golang 在 Windows 下防止程序多开"
date: 2021-10-11T11:02:41+08:00
archives: 
    - 2021
tags:
    - Golang
    - Windows
image: /images/multiple-instance-image.jpg
draft: false
---

当用户运行多个程序实例，如果操作的是同一资源，可能会造成数据不一致。所以有一个防止多开的需求，原理上是利用 [CreateMutex](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-createmutexa) 这个函数创建互斥对象。
```go
func checkSingleton() (windows.Handle, error) {
	path, err := os.Executable()
	if err != nil {
		return 0, err
	}
	hashName := md5.Sum([]byte(path))
	name, err := syscall.UTF16PtrFromString("Local\\" + hex.EncodeToString(hashName[:]))
	if err != nil {
		return 0, err
	}
	return windows.CreateMutex(nil, false, name)
}
```
检查`error`，如果是`syscall.ERROR_ALREADY_EXISTS`，则说明有一个程序已经在运行。

上面创建的互斥对象是局部的(有`Local`前缀)，多个用户登录还是可以同时多开程序的。如果需要全局限制，则替换为`Global`前缀。如果你的程序以服务的形式运行，则可能需要设置互斥对象的权限。
```go
func checkSingleton() (windows.Handle, error) {
	path, err := os.Executable()
	if err != nil {
		panic(err)
	}
	hashName := md5.Sum([]byte(path))
	name, err := syscall.UTF16PtrFromString("Global\\" + hex.EncodeToString(hashName[:]))
	if err != nil {
		panic(err)
	}
	sa := windows.SecurityAttributes{}
	sd, err := windows.SecurityDescriptorFromString("D:(A;;GA;;;WD)")
	if err != nil {
		panic(err)
	}
	sa.Length = uint32(unsafe.Sizeof(sa))
	sa.SecurityDescriptor = sd
	return windows.CreateMutex(&sa, false, name)
}
```

若要用户体验更好的话：如果程序已经在运行，则显示已在运行的程序的主窗口，然后退出本次程序的启动。我的思路是先查找本程序的窗口，然后对比窗口程序和本程序的文件路径，如果一致则显示该窗口。
```go
func showMainWindow() error {
	var windowToShow win.HWND
	path, err := os.Executable()
	if err != nil {
		return err
	}
	execFileInfo, err := os.Stat(path)
	if err != nil {
		return err
	}
	syscall.MustLoadDLL("user32.dll").MustFindProc("EnumWindows").Call(
		syscall.NewCallback(func(hwnd syscall.Handle, lparam uintptr) uintptr {
			className := make([]uint16, windows.MAX_PATH)
			if _, err = win.GetClassName(win.HWND(hwnd), &className[0], len(className)); err != nil {
				return 1
			}
			if windows.UTF16ToString(className) == "\\o/ Walk_MainWindow_Class \\o/" {
				var pid uint32
				var imageName string
				var imageFileInfo fs.FileInfo
				if _, err = windows.GetWindowThreadProcessId(windows.HWND(hwnd), &pid); err != nil {
					return 1
				}
				imageName, err = getImageName(pid)
				if err != nil {
					return 1
				}
				imageFileInfo, err = os.Stat(imageName)
				if err != nil {
					return 1
				}
				if os.SameFile(execFileInfo, imageFileInfo) {
					windowToShow = win.HWND(hwnd)
					return 0
				}
			}
			return 1
		}), 0)
	if windowToShow != 0 {
		if win.IsIconic(windowToShow) {
			win.ShowWindow(windowToShow, win.SW_RESTORE)
		} else {
			win.SetForegroundWindow(windowToShow)
		}
	}
	return nil
}

func getImageName(pid uint32) (string, error) {
	proc, err := windows.OpenProcess(windows.PROCESS_QUERY_LIMITED_INFORMATION, false, pid)
	if err != nil {
		return "", err
	}
	defer windows.CloseHandle(proc)
	var exeNameBuf [261]uint16
	exeNameLen := uint32(len(exeNameBuf) - 1)
	err = windows.QueryFullProcessImageName(proc, 0, &exeNameBuf[0], &exeNameLen)
	if err != nil {
		return "", err
	}
	return windows.UTF16ToString(exeNameBuf[:exeNameLen]), nil
}
```
