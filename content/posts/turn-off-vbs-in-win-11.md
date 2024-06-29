+++
title = "如何保持 Hyper-V 开启，同时临时关闭 VBS"
date = 2021-10-23 15:54:00
lastmod = 2021-10-23 15:54:00
description = "教你如何在 Windows 11 关闭 VBS 的同时不影响 Hyper-V 服务，保持 WSL 和 WSA 正常工作 —— 没有 WSL 怎么种庄稼？"
tags = [ "windows11" ]
+++

此前IT之家的文章 [Win11 VBS 安全功能导致性能下降，教你如何检测并关闭](https://www.ithome.com/0/579/260.htm) 中介绍使用 `PowerShell` 命令

```PowerShell
bcdedit /set hypervisorlaunchtype off
```

来关闭 VBS 功能，以避免性能损失。

经过实测，这虽然可以关闭 VBS 功能，但同时也会关闭 Hyper-V 服务，进而影响 WSL、WSA 等功能，出现下图所示的报错:

![WSA报错图片](https://i.loli.net/2021/10/23/4rbJl7dLNsZviAM.png)

参照 MSDN 上[这篇问答](https://docs.microsoft.com/en-us/answers/questions/245071/disable-virtualization-based-security-without-disb.html)，可以遵循以下步骤关闭 VBS 同时保持 WSL 和 WSA 正常工作。

需要注意的是，这个办法并不是永久的，重启后就会失效，VBS 会自动重新开启。目前我还没有找到永久关闭 VBS 而不影响 WSL 的办法。

## 0. 修复 Hyper-V /虚拟机平台

如果此前使用 `Powershell` 命令关闭了 Hyper-V 服务导致 WSL 等功能出现问题，首先以管理员权限在系统 `Powershell` 中运行以下脚本来恢复 Hyper-V 服务。

```PowerShell
bcdedit /set hypervisorlaunchtype auto
```

之后可以在 Windows 可选功能中取消勾选 Hyper-V 及虚拟机平台，重启生效后再次启用。

此时在 Powershell 中运行下列脚本，可以查看当前的 VBS 状态。

```PowerShell
Get-CimInstance -ClassName Win32_DeviceGuard -Namespace root\Microsoft\Windows\DeviceGuard
```

运行后应该显示出如下信息。

![VBS状态检查结果](https://i.loli.net/2021/10/23/1viqtUBDFcG68dA.png)

其中 `VirtualizationBasedSecurityStatus` (图片中高亮) 代表了当前 VBS 状态 (`0` 代表未启用，`1` 代表启用但未运行，`2` 代表正在运行)。

也可以使用 `msinfo32` 进行查看，与IT之家[这篇文章](https://www.ithome.com/0/579/260.htm)中方法相同。

## 1. 下载工具

此项操作需要使用到微软官方提供的 [HCVI and Windows Defender Credential Guard hardware readiness tool](https://www.microsoft.com/en-us/download/details.aspx?id=53337) 工具。在页面中点击 Download 进行下载。

![工具下载页面截图](https://i.loli.net/2021/10/23/vxGl8QeMKWIdcaC.png)

## 2. 运行脚本

下载后解压到任意一个文件夹中，使用带有管理员权限的 `PowerShell` 进入到文件夹中，运行如下脚本。

```PowerShell
./DG_Readiness_Tool_v3.6 -Disable
```

此时 Powershell 中会打印出下列字符串，其中 `Disabling Hyper-V and IOMMU` 会持续运行一段时间，要耐心等待 :)

```text

Directory: C:\


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----        10/23/2021   2:57 PM                DGLogs
###########################################################################
Readiness Tool Version 3.4 Release.
Tool to check if your device is capable to run Device Guard and Credential Guard.
###########################################################################
Disabling Device Guard and Credential Guard
Deleting RegKeys to disable DG/CG
ERROR: The system was unable to find the specified registry key or value.
ERROR: The system was unable to find the specified registry key or value.
ERROR: The system was unable to find the specified registry key or value.
ERROR: The system was unable to find the specified registry key or value.
ERROR: The system was unable to find the specified registry key or value.
ERROR: The system was unable to find the specified registry key or value.
del : Cannot find path 'C:\Windows\System32\CodeIntegrity\SIPolicy.p7b' because it does not exist.
At line:1 char:1
 + del  "$env:windir\System32\CodeIntegrity\SIPolicy.p7b"
 + ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : ObjectNotFound: (C:\Windows\Syst...ty\SIPolicy.p7b:String) [Remove-Item], ItemNotFoundEx
   ception
    + FullyQualifiedErrorId : PathNotFound,Microsoft.PowerShell.Commands.RemoveItemCommand

Disabling Hyper-V and IOMMU
Disabling Hyper-V and IOMMU successful

Please reboot the machine, for settings to be applied.

```

我们能看到一些报错，但是不需要担心，按照指示重启电脑。

## 3. 重启电脑

重启电脑过程中，在进入系统前应该能看到如下字段

```text
                         Credential Guard Opt-out Tool
Do you want to disable Credential Guard?
Disabling this functionality can allow malware to read the password and other
credentials of all users signing on to Windows. For the correct action in your
organization, contact your administrator before disabling protection.





Press the Windows key or F3 to disable Credential Guard. ESC to skip this step
```

遵照指示按下 `Windows` 键或者 `F3`，电脑会重新启动，正常进入系统。

## 4. 确认效果

重新运行步骤 0 中的脚本，或者打开 `msinfo32` 就可以看到现在 VBS 已经被禁用了。

还是要再次提醒，这个办法的关闭效果并不是永久的，重启后就会失效，但是至少能保证 WSL 和 WSA 正常工作，比较适合电脑性能比较吃紧 (比如我有一台 5200U 的笔记本)，也比较能接受电脑日常不关机只睡眠的用户。
