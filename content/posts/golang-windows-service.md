---
title: "Golang 程序以 Windows 服务运行"
date: 2021-11-12T13:15:06+08:00
archives: 
    - 2021
tags:
    - Golang
    - Windows
image: /images/windowsservices-serviceswindow.png
draft: false
---

Windows 服务是运行后台程序的一个很好的选择，可以支持开机自动启动，程序异常退出后自动重启这些实用功能。

Windows 服务程序有一套自己的机制。如果你不想在你的程序添加任何代码的话，有一些工具可以直接把可执行程序作为 Windows 服务运行，比如 [winsw](https://github.com/winsw/winsw)。

本文介绍的是在 Go 中制作自己的服务程序，主要使用的是 [`golang.org/x/sys/windows/svc`](https://pkg.go.dev/golang.org/x/sys/windows/svc) 这个包。正如文档所说，实现`Handler`接口即可，当服务启动时，会调用`Execute`方法。
```go
type demoService struct {

}

func (service *demoService) Execute(args []string, r <-chan svc.ChangeRequest, changes chan<- svc.Status) (svcSpecificEC bool, exitCode uint32) {
	changes <- svc.Status{State: svc.StartPending}

	defer func() {
		changes <- svc.Status{State: svc.StopPending}
		log.Println("Shutting down")
	}()

    // 要执行的主程序代码
	go yourMainFunc()

	changes <- svc.Status{State: svc.Running, Accepts: svc.AcceptStop | svc.AcceptShutdown}
	log.Println("Startup complete")

	for {
		select {
		case c := <-r:
			switch c.Cmd {
			case svc.Stop, svc.Shutdown:
				return
			case svc.Interrogate:
				changes <- c.CurrentStatus
			default:
				log.Printf("Unexpected services control request #%d\n", c)
			}
		}
	}
}
```
实现了该接口后，在主函数里运行服务：
```go
const svcName = "demoService"
svc.Run(svcName, &demoService{})
```
最后这个程序就变成了服务程序了，如果直接在命令行中运行是会报错的，因为它需要运行在服务模式下。当然你可以选择性地判断是否需要以服务形式运行，这样既可以作为服务程序，也可以作为控制台程序。
```go
inService, err := svc.IsWindowsService()
if err != nil {
	fatal(err)
}
if inService {
    svc.Run(svcName, &demoService{})
} else {
    // Running in console mode
}
```

## 管理服务
服务程序需要进行安装，卸载操作，有时还需要查询服务的状态，使用服务管理器可以实现这些操作。
```go
var cachedServiceManager *mgr.Mgr

func serviceManager() (*mgr.Mgr, error) {
	if cachedServiceManager != nil {
		return cachedServiceManager, nil
	}
	m, err := mgr.Connect()
	if err != nil {
		return nil, err
	}
	cachedServiceManager = m
	return cachedServiceManager, nil
}
```

### 安装服务
本文演示的是，如果系统已经安装了服务，则删除后重新安装。
```go
func InstallService() error {
	m, err := serviceManager()
	if err != nil {
		return err
	}
	path, err := os.Executable()
	if err != nil {
		return err
	}
	service, err := m.OpenService(svcName)
	if err == nil {
		status, err := service.Query()
		if err != nil && err != windows.ERROR_SERVICE_MARKED_FOR_DELETE {
			service.Close()
			return err
		}
		if status.State != svc.Stopped && err != windows.ERROR_SERVICE_MARKED_FOR_DELETE {
			service.Close()
			return errors.New("service already installed and running")
		}
		err = service.Delete()
		service.Close()
		if err != nil && err != windows.ERROR_SERVICE_MARKED_FOR_DELETE {
			return err
		}
		for {
			service, err = m.OpenService(svcName)
			if err != nil && err != windows.ERROR_SERVICE_MARKED_FOR_DELETE {
				break
			}
			if service != nil {
				service.Close()
			}
			time.Sleep(time.Second / 3)
		}
	}
	conf := mgr.Config{
		ServiceType:  windows.SERVICE_WIN32_OWN_PROCESS,
		StartType:    mgr.StartAutomatic,
		ErrorControl: mgr.ErrorNormal,
		DisplayName:  "Demo Service",
		Description:  "This is a demo service",
		SidType:      windows.SERVICE_SID_TYPE_UNRESTRICTED,
	}
	service, err = m.CreateService(svcName, path, conf)
	if err != nil {
		return err
	}
	err = service.Start()
	service.Close()
	return err
}
```

### 卸载服务
停止并卸载服务，参数`wait`指定是否等待服务停止。
```go
func UninstallService(wait bool) error {
	m, err := serviceManager()
	if err != nil {
		return err
	}
	service, err := m.OpenService(svcName)
	if err != nil {
		return err
	}
	service.Control(svc.Stop)
	if wait {
		try := 0
		for {
			time.Sleep(time.Second / 3)
			try++
			status, err := service.Query()
			if err != nil {
				return err
			}
			if status.ProcessId == 0 || try >= 3 {
				break
			}
		}
	}
	err = service.Delete()
	err2 := service.Close()
	if err != nil && err != windows.ERROR_SERVICE_MARKED_FOR_DELETE {
		return err
	}
	return err2
}
```

### 查询服务
查询服务是否正在运行，从返回的`error`可以判断服务是否存在。
```go
func QueryService() (bool, error) {
	m, err := serviceManager()
	if err != nil {
		return false, err
	}
	service, err := m.OpenService(svcName)
	if err != nil {
		return false, err
	}
	defer service.Close()
	status, err := service.Query()
	if err != nil && err != windows.ERROR_SERVICE_MARKED_FOR_DELETE {
		return false, err
	}
	return status.State != svc.Stopped && err != windows.ERROR_SERVICE_MARKED_FOR_DELETE, nil
}
```
