---
title: "给 exe 添加文件信息"
date: 2022-02-08T15:36:31+08:00
archives: 
    - 2022
tags:
    - Windows
image: /images/exeinfo.gif
draft: false
---

## 创建清单文件
文件名为`manifest.xml`。如果不需要管理器权限，就把`trustInfo`删掉。
```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<assembly xmlns="urn:schemas-microsoft-com:asm.v1" manifestVersion="1.0">
    <assemblyIdentity version="1.0.0.0" processorArchitecture="*" name="app" type="win32"/>
    <trustInfo xmlns="urn:schemas-microsoft-com:asm.v2">
        <security>
          <requestedPrivileges xmlns="urn:schemas-microsoft-com:asm.v3">
            <requestedExecutionLevel level="requireAdministrator" uiAccess="false" />
          </requestedPrivileges>
        </security>
    </trustInfo>
    <dependency>
        <dependentAssembly>
            <assemblyIdentity type="win32" name="Microsoft.Windows.Common-Controls" version="6.0.0.0" processorArchitecture="*" publicKeyToken="6595b64144ccf1df" language="*"/>
        </dependentAssembly>
    </dependency>
    <application xmlns="urn:schemas-microsoft-com:asm.v3">
        <windowsSettings>
            <dpiAwareness xmlns="http://schemas.microsoft.com/SMI/2016/WindowsSettings">PerMonitorV2, PerMonitor</dpiAwareness>
            <dpiAware xmlns="http://schemas.microsoft.com/SMI/2005/WindowsSettings">True</dpiAware>
        </windowsSettings>
    </application>
</assembly>
```

## 创建资源文件
文件名为`resources.rc`，如果没有清单文件，可将对应的行删掉，图标文件类似。
```c
#include <windows.h>

#pragma code_page(65001) // UTF-8

#define STRINGIZE(x) #x
#define EXPAND(x) STRINGIZE(x)

LANGUAGE LANG_NEUTRAL, SUBLANG_NEUTRAL
CREATEPROCESS_MANIFEST_RESOURCE_ID RT_MANIFEST manifest.xml
10        ICON           EXPAND(APP_ICO)

VS_VERSION_INFO VERSIONINFO
FILEVERSION    VERSION_ARRAY
PRODUCTVERSION VERSION_ARRAY
FILEFLAGSMASK  VS_FFI_FILEFLAGSMASK
FILEFLAGS      0x0
FILEOS         VOS__WINDOWS32
FILETYPE       VFT_APP
FILESUBTYPE    VFT2_UNKNOWN
BEGIN
  BLOCK "StringFileInfo"
  BEGIN
    BLOCK "080404B0"
    BEGIN
      VALUE "CompanyName", "Your Company"
      VALUE "FileDescription", "Example"
      VALUE "FileVersion", EXPAND(VERSION_STR)
      VALUE "InternalName", "Example"
      VALUE "LegalCopyright", "© example"
      VALUE "OriginalFilename", "example.exe"
      VALUE "ProductName", "EXAMPLE"
      VALUE "ProductVersion", EXPAND(PRODUCT_VERSION_STR)
      VALUE "Comments", "https://example.com"
    END
  END
  BLOCK "VarFileInfo"
  BEGIN
    VALUE "Translation", 0x0804, 1200
  END
END
```

## 编译资源
```shell
windres -DVERSION_ARRAY=1,0,1,0 -DVERSION_STR=1.0.1.0 -DPRODUCT_VERSION_STR=1.0.1.0 -DAPP_ICO=app.ico -i resources.rc -o rsrc.o -O coff -c 65001
```

## 参考

[1] [VERSIONINFO Resource](https://docs.microsoft.com/en-us/windows/win32/menurc/versioninfo-resource)
