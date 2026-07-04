---
title: KVM虚拟化检测
description: ''
pubDate: 2026-07-05T03:11
draft: false
tags: []
categories:
  - Documentation
---
# [JDWA-KVM 虚拟机去虚拟化技术详解](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#jdwa-kvm-%E8%99%9A%E6%8B%9F%E6%9C%BA%E5%8E%BB%E8%99%9A%E6%8B%9F%E5%8C%96%E6%8A%80%E6%9C%AF%E8%AF%A6%E8%A7%A3)

> 作者：记得晚安(JDWA)
>
> GitHub: <https://github.com/AAASS554>
>
> Email: 1412800823@qq.com

## [目录](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#%E7%9B%AE%E5%BD%95)

* [简介](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#%E7%AE%80%E4%BB%8B)
* [问题背景](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#%E9%97%AE%E9%A2%98%E8%83%8C%E6%99%AF)
* [基本去虚拟化方法](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#%E5%9F%BA%E6%9C%AC%E5%8E%BB%E8%99%9A%E6%8B%9F%E5%8C%96%E6%96%B9%E6%B3%95)
* [内核级去虚拟化方法](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#%E5%86%85%E6%A0%B8%E7%BA%A7%E5%8E%BB%E8%99%9A%E6%8B%9F%E5%8C%96%E6%96%B9%E6%B3%95)
* [技术路线对比](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#%E6%8A%80%E6%9C%AF%E8%B7%AF%E7%BA%BF%E5%AF%B9%E6%AF%94)
* [内核驱动开发实例](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#%E5%86%85%E6%A0%B8%E9%A9%B1%E5%8A%A8%E5%BC%80%E5%8F%91%E5%AE%9E%E4%BE%8B)
* [将防检测钩子写入Windows ISO镜像内存的方法](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#%E5%B0%86%E9%98%B2%E6%A3%80%E6%B5%8B%E9%92%A9%E5%AD%90%E5%86%99%E5%85%A5windows-iso%E9%95%9C%E5%83%8F%E5%86%85%E5%AD%98%E7%9A%84%E6%96%B9%E6%B3%95)
* [实际操作步骤](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#%E5%AE%9E%E9%99%85%E6%93%8D%E4%BD%9C%E6%AD%A5%E9%AA%A4)
* [注意事项](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#%E6%B3%A8%E6%84%8F%E4%BA%8B%E9%A1%B9)
* [参考资源](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#%E5%8F%82%E8%80%83%E8%B5%84%E6%BA%90)

## [简介](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#%E7%AE%80%E4%BB%8B)

本文档详细介绍如何在Ubuntu系统中的QEMU-KVM虚拟机上实现Windows 10的内核去虚拟化，使Windows系统无法检测到自己运行在虚拟环境中。文档包含从基础配置到高级内核驱动开发的多种方法，以满足不同技术水平用户的需求。

## [问题背景](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#%E9%97%AE%E9%A2%98%E8%83%8C%E6%99%AF)

当Windows系统运行在虚拟机环境中时，会通过多种方式检测到自己是在虚拟环境中运行的。这些检测机制主要包括：

1. **CPUID指令检测**：程序通过执行CPUID指令获取处理器信息，虚拟机中会返回特定标志
2. **系统固件表检测**：通过调用`SystemFirmwareTable`API获取BIOS/SMBIOS信息，查找"VMware"、"Virtual"等特征字符串
3. **注册表检测**：查找特定的虚拟机相关注册表项
4. **设备检测**：检查虚拟机特有的设备驱动或硬件ID

虚拟机检测可能会影响某些软件的使用，或者在特殊场景下不希望被检测到是在虚拟环境中运行。

## [基本去虚拟化方法](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#%E5%9F%BA%E6%9C%AC%E5%8E%BB%E8%99%9A%E6%8B%9F%E5%8C%96%E6%96%B9%E6%B3%95)

### [1. 修改QEMU/KVM的CPU配置](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#_1-%E4%BF%AE%E6%94%B9qemu-kvm%E7%9A%84cpu%E9%85%8D%E7%BD%AE)

最基本的方法是修改虚拟机的XML配置文件，隐藏hypervisor特征，同时保留虚拟化支持。

#### [步骤一：启用XML编辑](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#%E6%AD%A5%E9%AA%A4%E4%B8%80-%E5%90%AF%E7%94%A8xml%E7%BC%96%E8%BE%91)

在virt-manager（如果您使用图形界面）中启用XML编辑功能：

1. 打开virt-manager
2. 点击"编辑" → "首选项" → "启用XML编辑"

#### [步骤二：编辑CPU配置](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#%E6%AD%A5%E9%AA%A4%E4%BA%8C-%E7%BC%96%E8%BE%91cpu%E9%85%8D%E7%BD%AE)

1. 打开您的Windows 10虚拟机设置
2. 点击"CPU"选项
3. 切换到XML视图，查找`<cpu>`元素
4. 修改为以下配置：

```
<cpu mode="custom" match="exact" check="partial">
  <model fallback="allow">IvyBridge</model>
  <feature policy="disable" name="hypervisor"/>
  <feature policy="require" name="vmx"/>
</cpu>
```

这个配置的关键点是：

* 禁用`hypervisor`特征，使Windows检测不到虚拟环境
* 启用`vmx`特征，保留虚拟化能力

### [2. 通过命令行方式配置](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#_2-%E9%80%9A%E8%BF%87%E5%91%BD%E4%BB%A4%E8%A1%8C%E6%96%B9%E5%BC%8F%E9%85%8D%E7%BD%AE)

如果您使用命令行配置QEMU，可以在启动参数中添加：

```
-cpu host,kvm=off,hv_vendor_id=null,-hypervisor,+vmx
```

### [3. 添加额外的防检测选项](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#_3-%E6%B7%BB%E5%8A%A0%E9%A2%9D%E5%A4%96%E7%9A%84%E9%98%B2%E6%A3%80%E6%B5%8B%E9%80%89%E9%A1%B9)

除了基本的CPU配置外，还可以添加以下设置来进一步隐藏虚拟机特征：

#### [修改时钟设置](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#%E4%BF%AE%E6%94%B9%E6%97%B6%E9%92%9F%E8%AE%BE%E7%BD%AE)

```
<clock offset="localtime">
  <timer name="rtc" tickpolicy="catchup"/>
  <timer name="pit" tickpolicy="delay"/>
  <timer name="hpet" present="no"/>
  <timer name="hypervclock" present="no"/>
</clock>
```

#### [随机化SMBIOS和DMI信息](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#%E9%9A%8F%E6%9C%BA%E5%8C%96smbios%E5%92%8Cdmi%E4%BF%A1%E6%81%AF)

```
<sysinfo type="smbios">
  <bios>
    <entry name="vendor">自定义厂商名</entry>
  </bios>
  <system>
    <entry name="manufacturer">自定义制造商</entry>
    <entry name="product">自定义产品名</entry>
    <entry name="version">自定义版本</entry>
  </system>
</sysinfo>
```

#### [随机化设备序列号](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#%E9%9A%8F%E6%9C%BA%E5%8C%96%E8%AE%BE%E5%A4%87%E5%BA%8F%E5%88%97%E5%8F%B7)

在QEMU命令行中添加：

```
-device ide-hd,bus=ide.0,drive=drive-sata0-0-0,id=sata0-0-0,serial=自定义随机序列号
```

### [4. 使用专门的去虚拟化工具](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#_4-%E4%BD%BF%E7%94%A8%E4%B8%93%E9%97%A8%E7%9A%84%E5%8E%BB%E8%99%9A%E6%8B%9F%E5%8C%96%E5%B7%A5%E5%85%B7)

一些开源项目如HiddenVM提供了完整的解决方案来隐藏虚拟机特征。对于基本需求，上述配置已经足够有效。

## [内核级去虚拟化方法](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#%E5%86%85%E6%A0%B8%E7%BA%A7%E5%8E%BB%E8%99%9A%E6%8B%9F%E5%8C%96%E6%96%B9%E6%B3%95)

如果基本的配置修改不足以满足需求，可以考虑更深层次的内核级去虚拟化方法。

### [1. QEMU/KVM高级配置](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#_1-qemu-kvm%E9%AB%98%E7%BA%A7%E9%85%8D%E7%BD%AE)

```
<cpu mode="host-passthrough" check="none">
  <topology sockets="1" cores="4" threads="2"/>
  <feature policy="disable" name="hypervisor"/>
  <feature policy="require" name="vmx"/>
  <feature policy="require" name="kvm"/>
  <feature policy="disable" name="kvm-hint-dedicated"/>
  <hidden state="on"/>
</cpu>
```

关键点：

* 使用`host-passthrough`模式直接传递主机CPU特性
* 设置`<hidden state="on"/>`彻底隐藏虚拟机状态
* 禁用所有可能暴露虚拟化的特性

### [2. 修改KVM模块参数](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#_2-%E4%BF%AE%E6%94%B9kvm%E6%A8%A1%E5%9D%97%E5%8F%82%E6%95%B0)

在Ubuntu主机系统中，修改KVM模块参数以隐藏更多虚拟化痕迹：

```
echo "options kvm ignore_msrs=1 report_ignored_msrs=0" > /etc/modprobe.d/kvm.conf
echo "options kvm-intel nested=1 enable_shadow_vmcs=1 enable_apicv=1 ept=1" >> /etc/modprobe.d/kvm.conf
```

然后重新加载KVM模块：

```
sudo modprobe -r kvm_intel
sudo modprobe -r kvm
sudo modprobe kvm
sudo modprobe kvm_intel
```

### [3. Windows内核层面的修改](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#_3-windows%E5%86%85%E6%A0%B8%E5%B1%82%E9%9D%A2%E7%9A%84%E4%BF%AE%E6%94%B9)

#### [a. 使用驱动程序拦截虚拟化检测](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#a-%E4%BD%BF%E7%94%A8%E9%A9%B1%E5%8A%A8%E7%A8%8B%E5%BA%8F%E6%8B%A6%E6%88%AA%E8%99%9A%E6%8B%9F%E5%8C%96%E6%A3%80%E6%B5%8B)

开发一个内核级驱动程序（.sys文件），用于：

* 拦截CPUID指令执行
* 修改返回的CPU信息，删除虚拟化标志
* 拦截MSR（Model Specific Registers）读取操作
* 拦截SIDT/SGDT等指令（通常用于检测虚拟机）

#### [b. 修改Windows注册表](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#b-%E4%BF%AE%E6%94%B9windows%E6%B3%A8%E5%86%8C%E8%A1%A8)

通过修改特定注册表项隐藏虚拟化痕迹：

```
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Class\{4D36E968-E325-11CE-BFC1-08002BE10318}\0000
```

#### [c. 安装内核级防检测软件](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#c-%E5%AE%89%E8%A3%85%E5%86%85%E6%A0%B8%E7%BA%A7%E9%98%B2%E6%A3%80%E6%B5%8B%E8%BD%AF%E4%BB%B6)

可以使用专门的内核级防检测工具：

1. **KDU（Kernel Driver Utility）** - 用于加载未签名驱动，修改内核行为
2. **Hypervisor Injection Framework** - 用于注入自定义hypervisor代码

### [4. MSR寄存器修改](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#_4-msr%E5%AF%84%E5%AD%98%E5%99%A8%E4%BF%AE%E6%94%B9)

在QEMU命令行中添加：

```
-cpu host,kvm=off,hv_vendor_id=null,-hypervisor,+vmx,+invtsc
```

同时通过对MSR的模拟来隐藏特征：

```
-device isa-applesmc,osk="..."
-device vmware-svga
-global driver=cfi.pflash01,property=secure,value=on
```

### [5. 内存特征修改](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#_5-%E5%86%85%E5%AD%98%E7%89%B9%E5%BE%81%E4%BF%AE%E6%94%B9)

阻止Windows检测内存中的虚拟化特征模式：

```
<memoryBacking>
  <nosharepages/>
  <locked/>
</memoryBacking>
```

### [6. 时钟系统隐藏](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#_6-%E6%97%B6%E9%92%9F%E7%B3%BB%E7%BB%9F%E9%9A%90%E8%97%8F)

```
<clock offset="localtime">
  <timer name="rtc" tickpolicy="catchup" track="guest"/>
  <timer name="pit" tickpolicy="delay"/>
  <timer name="hpet" present="no"/>
  <timer name="hypervclock" present="no"/>
  <timer name="tsc" present="yes" mode="native"/>
</clock>
```

### [7. 硬件路径随机化](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#_7-%E7%A1%AC%E4%BB%B6%E8%B7%AF%E5%BE%84%E9%9A%8F%E6%9C%BA%E5%8C%96)

```
<os>
  <smbios mode="sysinfo"/>
</os>
<sysinfo type="smbios">
  <system>
    <entry name="manufacturer">自定义制造商</entry>
    <entry name="product">自定义产品名</entry>
    <entry name="version">自定义版本号</entry>
    <entry name="serial">随机序列号</entry>
    <entry name="uuid">随机UUID</entry>
  </system>
</sysinfo>
```

## [技术路线对比](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#%E6%8A%80%E6%9C%AF%E8%B7%AF%E7%BA%BF%E5%AF%B9%E6%AF%94)

实现虚拟机防检测有两种主要的技术路线，各有优缺点：

### [1. 内核级去虚拟化方法](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#_1-%E5%86%85%E6%A0%B8%E7%BA%A7%E5%8E%BB%E8%99%9A%E6%8B%9F%E5%8C%96%E6%96%B9%E6%B3%95)

主要是修改QEMU/KVM的配置参数，隐藏虚拟机特征，并可能结合Windows内核驱动拦截检测方法。

### [2. ISO修改与内存逆向方法](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#_2-iso%E4%BF%AE%E6%94%B9%E4%B8%8E%E5%86%85%E5%AD%98%E9%80%86%E5%90%91%E6%96%B9%E6%B3%95)

直接对Windows系统文件进行修改，包括ISO修改和内存逆向分析。

| 特点 | 内核级去虚拟化 | ISO修改与内存逆向 |
| --- | --- | --- |
| 实现难度 | 中等，主要是配置修改 | 高，需要汇编级逆向能力 |
| 持久性 | 需随虚拟机配置更新 | 可以内置在系统镜像中 |
| 稳定性 | 相对稳定，不修改系统 | 可能影响系统稳定性 |
| 对系统更新的适应性 | 适应性好，不改动系统 | 系统更新后可能失效 |
| 检测绕过能力 | 主要针对虚拟化特征 | 可针对特定应用的检测逻辑 |

## [内核驱动开发实例](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#%E5%86%85%E6%A0%B8%E9%A9%B1%E5%8A%A8%E5%BC%80%E5%8F%91%E5%AE%9E%E4%BE%8B)

以下是一个内核级驱动程序示例，用于拦截和修改常见的虚拟机检测方法：

### [驱动程序代码结构](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#%E9%A9%B1%E5%8A%A8%E7%A8%8B%E5%BA%8F%E4%BB%A3%E7%A0%81%E7%BB%93%E6%9E%84)

```
#include <ntddk.h>
#include <wdm.h>

// SSDT相关的定义
typedef struct _SYSTEM_SERVICE_TABLE {
    PVOID ServiceTableBase;
    PVOID ServiceCounterTableBase;
    ULONG_PTR NumberOfServices;
    PVOID ParamTableBase;
} SYSTEM_SERVICE_TABLE, *PSYSTEM_SERVICE_TABLE;

// 原始的SystemFirmwareTable函数指针
typedef NTSTATUS(*PFN_NT_QUERY_SYSTEM_INFORMATION)(
    IN ULONG SystemInformationClass,
    OUT PVOID SystemInformation,
    IN ULONG SystemInformationLength,
    OUT PULONG ReturnLength OPTIONAL);

// 保存原始函数地址
PFN_NT_QUERY_SYSTEM_INFORMATION g_OriginalQuerySystemInformation = NULL;

// 我们的钩子函数
NTSTATUS HookedNtQuerySystemInformation(
    IN ULONG SystemInformationClass,
    OUT PVOID SystemInformation,
    IN ULONG SystemInformationLength,
    OUT PULONG ReturnLength OPTIONAL)
{
    // 调用原始函数获取数据
    NTSTATUS status = g_OriginalQuerySystemInformation(
        SystemInformationClass,
        SystemInformation,
        SystemInformationLength,
        ReturnLength);

    // 如果是系统固件表信息请求
    if (NT_SUCCESS(status) && SystemInformationClass == 76) { // SystemFirmwareTableInformation
        // 获取固件表信息结构
        PSYSTEM_FIRMWARE_TABLE_INFORMATION firmwareTableInfo = (PSYSTEM_FIRMWARE_TABLE_INFORMATION)SystemInformation;
        
        // 处理SMBIOS表
        if (firmwareTableInfo->ProviderSignature == 'RSMB') { // SMBIOS
            // 获取表数据指针
            PUCHAR tableData = (PUCHAR)firmwareTableInfo->TableBuffer;
            ULONG tableLength = firmwareTableInfo->TableBufferLength;
            
            // 在表中搜索并替换虚拟机特征字符串
            for (ULONG i = 0; i < tableLength - 7; i++) {
                // 搜索"VMware"字符串
                if (memcmp(tableData + i, "VMware", 6) == 0) {
                    // 替换为"PHYSIC"
                    memcpy(tableData + i, "PHYSIC", 6);
                }
                // 搜索"Virtual"字符串
                if (memcmp(tableData + i, "Virtual", 7) == 0) {
                    // 替换为"Reality"
                    memcpy(tableData + i, "Reality", 7);
                }
            }
        }
    }

    return status;
}

// 设置/移除钩子
BOOLEAN SetHook(BOOLEAN Install)
{
    // 获取SSDT表
    PSYSTEM_SERVICE_TABLE ServiceTable = GetSystemServiceTable();
    if (!ServiceTable) return FALSE;

    // 获取NtQuerySystemInformation的索引
    ULONG Index = GetServiceIndex("NtQuerySystemInformation");
    if (!Index) return FALSE;

    // 关闭写保护
    DisableWriteProtection();
    
    if (Install) {
        // 保存原始函数地址
        g_OriginalQuerySystemInformation = (PFN_NT_QUERY_SYSTEM_INFORMATION)ServiceTable->ServiceTableBase[Index];
        // 设置钩子
        ServiceTable->ServiceTableBase[Index] = (PVOID)HookedNtQuerySystemInformation;
    }
    else {
        // 恢复原始函数
        if (g_OriginalQuerySystemInformation)
            ServiceTable->ServiceTableBase[Index] = (PVOID)g_OriginalQuerySystemInformation;
    }
    
    // 恢复写保护
    EnableWriteProtection();
    
    return TRUE;
}

// 驱动入口
NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath)
{
    DriverObject->DriverUnload = DriverUnload;
    
    // 安装钩子
    SetHook(TRUE);
    
    return STATUS_SUCCESS;
}

// 驱动卸载
VOID DriverUnload(PDRIVER_OBJECT DriverObject)
{
    // 移除钩子
    SetHook(FALSE);
}
```

### [CPUID指令拦截](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#cpuid%E6%8C%87%E4%BB%A4%E6%8B%A6%E6%88%AA)

```
// CPUID指令拦截函数
VOID HandleCpuidVmexit(PGUEST_REGS GuestRegs)
{
    // 获取CPUID功能号
    INT32 function = (INT32)GuestRegs->rax;
    
    // 处理CPUID功能号1（处理器特性与功能位）
    if (function == 1) {
        // 修改ECX寄存器，清除虚拟化位(31位)
        GuestRegs->rcx &= ~(1 << 31);
    }
    // 处理供应商字符串检查(功能号0)
    else if (function == 0) {
        // 自定义的CPUID返回值
        GuestRegs->rbx = 0x756e6547; // "Genu"
        GuestRegs->rdx = 0x49656e69; // "ineI"
        GuestRegs->rcx = 0x6c65746e; // "ntel"
    }
    // 处理Hypervisor信息(功能号0x40000000)
    else if (function == 0x40000000) {
        // 返回无供应商信息
        GuestRegs->rax = 0;
        GuestRegs->rbx = 0;
        GuestRegs->rcx = 0;
        GuestRegs->rdx = 0;
    }
}
```

### [注册表修补方法](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#%E6%B3%A8%E5%86%8C%E8%A1%A8%E4%BF%AE%E8%A1%A5%E6%96%B9%E6%B3%95)

```
// 修改注册表项来隐藏虚拟机痕迹
NTSTATUS HideVmwareRegistry()
{
    UNICODE_STRING registryPath;
    OBJECT_ATTRIBUTES objAttr;
    HANDLE keyHandle;
    
    // 初始化目标注册表路径
    RtlInitUnicodeString(&registryPath, 
        L"\\Registry\\Machine\\HARDWARE\\DEVICEMAP\\Scsi\\Scsi Port 0\\Scsi Bus 0\\Target Id 0\\Logical Unit Id 0");
    
    InitializeObjectAttributes(&objAttr, 
                              &registryPath,
                              OBJ_CASE_INSENSITIVE | OBJ_KERNEL_HANDLE,
                              NULL, 
                              NULL);
    
    // 打开注册表键
    NTSTATUS status = ZwOpenKey(&keyHandle, KEY_ALL_ACCESS, &objAttr);
    if (!NT_SUCCESS(status)) {
        return status;
    }
    
    // 修改"Identifier"值
    UNICODE_STRING valueName;
    RtlInitUnicodeString(&valueName, L"Identifier");
    
    // 准备新的非虚拟机标识符
    WCHAR newIdentifier[] = L"SAMSUNG SSD PM9A1 NVMe 1TB";
    
    status = ZwSetValueKey(keyHandle,
                          &valueName,
                          0,
                          REG_SZ,
                          newIdentifier,
                          sizeof(newIdentifier));
    
    ZwClose(keyHandle);
    return status;
}
```

### [综合解决方案：代码最佳实践](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#%E7%BB%BC%E5%90%88%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%88-%E4%BB%A3%E7%A0%81%E6%9C%80%E4%BD%B3%E5%AE%9E%E8%B7%B5)

```
// 全局钩子管理结构
typedef struct _HOOK_ENTRY {
    PVOID OriginalFunction;
    PVOID HookedFunction;
    ULONG ServiceId;
    BOOLEAN Active;
} HOOK_ENTRY, *PHOOK_ENTRY;

// 初始化所有防检测钩子
NTSTATUS InitializeAllHooks()
{
    // 1. 设置SSDT钩子
    InitializeSsdtHooks();
    
    // 2. 设置硬件断点以拦截CPUID指令
    SetupCpuidInterception();
    
    // 3. 修补内存中的关键字符串
    PatchVirtualStringInMemory();
    
    // 4. 修改注册表
    HideVmwareRegistry();
    
    // 5. 修补MSR寄存器访问
    SetupMsrInterception();
    
    return STATUS_SUCCESS;
}
```

## [将防检测钩子写入Windows ISO镜像内存的方法](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#%E5%B0%86%E9%98%B2%E6%A3%80%E6%B5%8B%E9%92%A9%E5%AD%90%E5%86%99%E5%85%A5windows-iso%E9%95%9C%E5%83%8F%E5%86%85%E5%AD%98%E7%9A%84%E6%96%B9%E6%B3%95)

要将防检测钩子直接写入到Windows ISO镜像中，需要进行更低层次的系统修改。这种方法相比动态加载驱动更持久，但也更复杂。下面详细介绍实现步骤：

### [一、准备工作](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#%E4%B8%80%E3%80%81%E5%87%86%E5%A4%87%E5%B7%A5%E4%BD%9C)

#### [1. 所需工具](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#_1-%E6%89%80%E9%9C%80%E5%B7%A5%E5%85%B7)

* DISM (部署映像服务和管理)
* 7-Zip 或其他ISO解压/创建工具
* WinISO 或UltraISO等ISO编辑工具
* Windows ADK (评估和部署工具包)
* 十六进制编辑器 (如HxD)

#### [2. 环境要求](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#_2-%E7%8E%AF%E5%A2%83%E8%A6%81%E6%B1%82)

* Linux系统 (用于操作ISO镜像)
* 足够的磁盘空间 (至少50GB)
* Windows系统 (用于开发和测试钩子代码)

### [二、ISO镜像提取与修改流程](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#%E4%BA%8C%E3%80%81iso%E9%95%9C%E5%83%8F%E6%8F%90%E5%8F%96%E4%B8%8E%E4%BF%AE%E6%94%B9%E6%B5%81%E7%A8%8B)

#### [1. 提取ISO内容](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#_1-%E6%8F%90%E5%8F%96iso%E5%86%85%E5%AE%B9)

```
# 创建挂载点并挂载ISO
mkdir -p /mnt/windows_iso
mount -o loop Windows10.iso /mnt/windows_iso

# 复制ISO内容到工作目录
mkdir -p ~/windows_iso_edit
cp -r /mnt/windows_iso/* ~/windows_iso_edit/

# 卸载ISO
umount /mnt/windows_iso
```

#### [2. 提取并修改WIM/ESD映像](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#_2-%E6%8F%90%E5%8F%96%E5%B9%B6%E4%BF%AE%E6%94%B9wim-esd%E6%98%A0%E5%83%8F)

WIM (Windows映像文件) 或 ESD 文件包含着Windows的所有系统文件。

```
# 创建临时工作目录
mkdir -p ~/wim_mount
cd ~/windows_iso_edit/sources/

# 提取install.wim或install.esd (视ISO类型而定)
wimlib-imagex extract install.wim 1 ~/wim_mount
```

#### [3. 定位关键系统文件](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#_3-%E5%AE%9A%E4%BD%8D%E5%85%B3%E9%94%AE%E7%B3%BB%E7%BB%9F%E6%96%87%E4%BB%B6)

需要修改的主要文件：

* `ntoskrnl.exe` - Windows内核主文件
* `HAL.dll` - 硬件抽象层
* 虚拟机检测相关的驱动文件

### [三、钩子代码注入方法](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#%E4%B8%89%E3%80%81%E9%92%A9%E5%AD%90%E4%BB%A3%E7%A0%81%E6%B3%A8%E5%85%A5%E6%96%B9%E6%B3%95)

#### [1. 代码洞技术 (Code Cave)](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#_1-%E4%BB%A3%E7%A0%81%E6%B4%9E%E6%8A%80%E6%9C%AF-code-cave)

这是一种在现有系统文件中找到未使用空间并插入代码的技术：

```
// 钩子代码示例（需要汇编实现）
__declspec(naked) void CodeCaveHook() {
    __asm {
        // 保存寄存器状态
        push eax
        push ebx
        push ecx
        push edx
        
        // 检查CPUID调用
        mov eax, 1       // CPUID功能号1
        cpuid
        and ecx, 0x7FFFFFFF // 清除ECX的第31位(Hypervisor位)
        
        // 恢复寄存器并返回
        pop edx
        pop ecx
        pop ebx
        pop eax
        
        // 执行被覆盖的原始指令
        [原始指令]
        jmp [返回地址]
    }
}
```

#### [2. 在关键函数入口点设置钩子](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#_2-%E5%9C%A8%E5%85%B3%E9%94%AE%E5%87%BD%E6%95%B0%E5%85%A5%E5%8F%A3%E7%82%B9%E8%AE%BE%E7%BD%AE%E9%92%A9%E5%AD%90)

例如，修改`ntoskrnl.exe`中的`NtQuerySystemInformation`函数：

1. 用十六进制编辑器打开`ntoskrnl.exe`
2. 定位`NtQuerySystemInformation`函数入口点
3. 修改入口点代码为跳转指令，指向代码洞中的钩子代码
4. 在代码洞中实现检测拦截逻辑

##### [十六进制修改示例：](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#%E5%8D%81%E5%85%AD%E8%BF%9B%E5%88%B6%E4%BF%AE%E6%94%B9%E7%A4%BA%E4%BE%8B)

```
// 原始代码（示例）
55 8B EC 83 EC 18 ...

// 替换为跳转指令
E9 [4字节偏移量] ...
```

#### [3. 补丁文件方法](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#_3-%E8%A1%A5%E4%B8%81%E6%96%87%E4%BB%B6%E6%96%B9%E6%B3%95)

创建一个补丁文件，直接修改目标文件中的特定字节：

```
# 创建一个十六进制补丁文件example.hex
echo "00000A50: 90 90 90 90 90" > bypass_vm_detect.hex

# 使用xxd或其他工具应用补丁
xxd -r bypass_vm_detect.hex ntoskrnl.exe
```

### [四、SMBIOS/ACPI表修改](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#%E5%9B%9B%E3%80%81smbios-acpi%E8%A1%A8%E4%BF%AE%E6%94%B9)

#### [1. 修改ACPI表](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#_1-%E4%BF%AE%E6%94%B9acpi%E8%A1%A8)

ACPI表包含了系统硬件信息，可能被用于虚拟机检测：

```
# 定位并提取ACPI表相关文件
find ~/wim_mount -name "acpi.sys" -type f

# 使用十六进制编辑器修改acpi.sys中的虚拟机签名
# 搜索并替换"VMWARE"、"VIRTUAL"等字符串
```

#### [2. SMBIOS信息修改](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#_2-smbios%E4%BF%A1%E6%81%AF%E4%BF%AE%E6%94%B9)

```
# 定位SMBIOS处理相关文件
find ~/wim_mount -name "*smbios*" -type f
```

### [五、系统核心文件预加载钩子](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#%E4%BA%94%E3%80%81%E7%B3%BB%E7%BB%9F%E6%A0%B8%E5%BF%83%E6%96%87%E4%BB%B6%E9%A2%84%E5%8A%A0%E8%BD%BD%E9%92%A9%E5%AD%90)

#### [1. 修改bootmgr或winload.exe](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#_1-%E4%BF%AE%E6%94%B9bootmgr%E6%88%96winload-exe)

这些是启动过程中最早加载的组件之一，可以在此处设置钩子：

```
# 在wim_mount目录中定位bootmgr
find ~/wim_mount -name "bootmgr*" -type f

# 使用十六进制编辑器添加预加载钩子
```

#### [2. 创建自定义启动钩子DLL](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#_2-%E5%88%9B%E5%BB%BA%E8%87%AA%E5%AE%9A%E4%B9%89%E5%90%AF%E5%8A%A8%E9%92%A9%E5%AD%90dll)

编写一个自定义DLL，在系统启动早期就执行防检测代码：

```
// early_hook.c - 编译为DLL
#include <windows.h>

BOOL WINAPI DllMain(HINSTANCE hinstDLL, DWORD fdwReason, LPVOID lpvReserved) {
    if (fdwReason == DLL_PROCESS_ATTACH) {
        // 实现早期防检测代码
        PatchCpuidInstruction();
        HideSmbiosVirtualStrings();
        ModifyRegistryVirtualKeys();
    }
    return TRUE;
}
```

将编译好的DLL放入系统目录，并修改注册表确保早期加载。

### [六、重建ISO映像](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#%E5%85%AD%E3%80%81%E9%87%8D%E5%BB%BAiso%E6%98%A0%E5%83%8F)

完成所有修改后，需要重建WIM文件和ISO镜像：

```
# 重建WIM文件
cd ~/wim_mount
wimlib-imagex capture . ~/windows_iso_edit/sources/install.wim "Windows 10" --no-acls

# 重建ISO
cd ~/windows_iso_edit
mkisofs -b boot/etfsboot.com -no-emul-boot -boot-load-size 8 -iso-level 4 -J -l -D -N -joliet-long -relaxed-filenames -V "Windows 10" -o ~/Windows10_Modified.iso .
```

### [七、高级技术：替换关键系统DLL](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#%E4%B8%83%E3%80%81%E9%AB%98%E7%BA%A7%E6%8A%80%E6%9C%AF-%E6%9B%BF%E6%8D%A2%E5%85%B3%E9%94%AE%E7%B3%BB%E7%BB%9Fdll)

#### [1. 替换HAL.dll或内核组件](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#_1-%E6%9B%BF%E6%8D%A2hal-dll%E6%88%96%E5%86%85%E6%A0%B8%E7%BB%84%E4%BB%B6)

```
# 备份原始文件
cp ~/wim_mount/Windows/System32/hal.dll ~/hal.dll.backup

# 替换为修改后的文件
cp ~/modified_hal.dll ~/wim_mount/Windows/System32/hal.dll
```

#### [2. 关键DLL引用重定向](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#_2-%E5%85%B3%E9%94%AEdll%E5%BC%95%E7%94%A8%E9%87%8D%E5%AE%9A%E5%90%91)

修改导入表，使虚拟机检测函数指向我们的实现：

```
// 示例：修改PE文件导入表
void RedirectImportTable(const char* dllPath, const char* functionName, void* newFunction) {
    // 使用PE编辑库实现导入表重定向
    // 此处代码需要专业PE文件格式知识
}
```

### [八、实际操作注意事项](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#%E5%85%AB%E3%80%81%E5%AE%9E%E9%99%85%E6%93%8D%E4%BD%9C%E6%B3%A8%E6%84%8F%E4%BA%8B%E9%A1%B9)

1. **文件完整性**：修改系统文件后，需要重新计算签名或禁用签名检查，否则系统可能无法启动
2. **备份原始文件**：每次修改前务必备份原始文件
3. **测试验证**：在虚拟环境中测试修改后的ISO，确认系统可正常启动且防检测有效
4. **兼容性问题**：针对不同Windows版本，定位和修改的地址可能不同
5. **安全风险**：修改系统核心文件可能导致系统不稳定或安全漏洞，仅用于测试环境

### [九、实用工具和脚本](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#%E4%B9%9D%E3%80%81%E5%AE%9E%E7%94%A8%E5%B7%A5%E5%85%B7%E5%92%8C%E8%84%9A%E6%9C%AC)

#### [1. 自动定位虚拟化检测点的脚本](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#_1-%E8%87%AA%E5%8A%A8%E5%AE%9A%E4%BD%8D%E8%99%9A%E6%8B%9F%E5%8C%96%E6%A3%80%E6%B5%8B%E7%82%B9%E7%9A%84%E8%84%9A%E6%9C%AC)

```
#!/usr/bin/env python3
# vm_detect_finder.py - 定位虚拟机检测代码的脚本

import sys
import re

def find_vm_detection_patterns(binary_path):
    with open(binary_path, 'rb') as f:
        binary_data = f.read()
    
    # 搜索CPUID指令模式（0x0F 0xA2）
    cpuid_offsets = [m.start() for m in re.finditer(b'\x0F\xA2', binary_data)]
    
    # 搜索虚拟机字符串
    vm_strings = [b'VMware', b'VIRTUAL', b'VBOX', b'QEMU']
    string_matches = {}
    
    for vm_string in vm_strings:
        matches = [m.start() for m in re.finditer(vm_string, binary_data)]
        if matches:
            string_matches[vm_string] = matches
    
    return {
        'cpuid_offsets': cpuid_offsets,
        'vm_strings': string_matches
    }

if __name__ == "__main__":
    if len(sys.argv) != 2:
        print(f"Usage: {sys.argv[0]} <binary_file>")
        sys.exit(1)
    
    results = find_vm_detection_patterns(sys.argv[1])
    print(f"Found {len(results['cpuid_offsets'])} CPUID instructions")
    for vm_string, offsets in results['vm_strings'].items():
        print(f"Found {len(offsets)} instances of {vm_string}")
        for offset in offsets:
            print(f"  - Offset: 0x{offset:X}")
```

#### [2. 补丁生成器](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#_2-%E8%A1%A5%E4%B8%81%E7%94%9F%E6%88%90%E5%99%A8)

```
#!/usr/bin/env python3
# patch_generator.py - 生成虚拟机检测补丁

import sys
import struct

def generate_cpuid_patch(offset):
    """生成用于替换CPUID指令的补丁"""
    # 创建修改后的指令序列（例如：NOP序列）
    patch = b'\x90\x90'  # 两字节NOP替换CPUID
    
    patch_line = f"{offset:08X}: {' '.join([f'{b:02X}' for b in patch])}"
    return patch_line

if __name__ == "__main__":
    if len(sys.argv) < 3:
        print(f"Usage: {sys.argv[0]} <output_patch_file> <offset1> [offset2] ...")
        sys.exit(1)
    
    patch_file = sys.argv[1]
    offsets = [int(arg, 16) if arg.startswith("0x") else int(arg) for arg in sys.argv[2:]]
    
    with open(patch_file, 'w') as f:
        for offset in offsets:
            patch_line = generate_cpuid_patch(offset)
            f.write(patch_line + '\n')
    
    print(f"Generated patch file: {patch_file}")
```

### [十、专业工具推荐](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#%E5%8D%81%E3%80%81%E4%B8%93%E4%B8%9A%E5%B7%A5%E5%85%B7%E6%8E%A8%E8%8D%90)

1. **PE Explorer** - 分析和修改PE文件结构
2. **IDA Pro** - 反汇编和分析系统文件
3. **WinHex** - 高级十六进制编辑器，支持模板和结构分析
4. **oscdimg** - 微软官方ISO镜像创建工具
5. **DISM++** - 高级系统映像管理工具

通过以上方法，可以将防检测钩子代码直接嵌入到Windows ISO镜像中，实现更底层、更持久的虚拟机防检测效果。这种方法虽然技术复杂度高，但效果更彻底，对于严格的虚拟环境检测有更好的绕过能力。

## [实际操作步骤](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#%E5%AE%9E%E9%99%85%E6%93%8D%E4%BD%9C%E6%AD%A5%E9%AA%A4)

### [1. 创建项目](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#_1-%E5%88%9B%E5%BB%BA%E9%A1%B9%E7%9B%AE)

使用Visual Studio创建内核驱动项目：

* 安装Visual Studio 2019或更高版本
* 安装Windows Driver Kit (WDK) 10或更高版本
* 创建新的WDM驱动项目

### [2. 编译与签名](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#_2-%E7%BC%96%E8%AF%91%E4%B8%8E%E7%AD%BE%E5%90%8D)

1. 设置项目为Release x64配置
2. 构建解决方案
3. 对驱动签名（测试环境）：

```
makecert -r -pe -ss PrivateCertStore -n "CN=YourCertName" YourCert.cer
certmgr -add YourCert.cer -s -r localMachine root
SignTool sign /v /s PrivateCertStore /n YourCertName /t http://timestamp.digicert.com YourDriver.sys
```

### [3. 测试环境准备](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#_3-%E6%B5%8B%E8%AF%95%E7%8E%AF%E5%A2%83%E5%87%86%E5%A4%87)

1. 开启Windows测试模式：

```
bcdedit /set testsigning on
```

2. 重启系统

### [4. 部署驱动](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#_4-%E9%83%A8%E7%BD%B2%E9%A9%B1%E5%8A%A8)

1. 创建服务：

```
sc create AntiVMDetect binPath= C:\Path\To\YourDriver.sys type= kernel
```

2. 启动服务：

```
sc start AntiVMDetect
```

### [5. 验证效果](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#_5-%E9%AA%8C%E8%AF%81%E6%95%88%E6%9E%9C)

1. 运行检测工具（如Pafish）验证检测是否被绕过
2. 测试特定应用程序是否能检测到虚拟机环境

## [注意事项](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#%E6%B3%A8%E6%84%8F%E4%BA%8B%E9%A1%B9)

1. **合法用途**：确保您的用途是合法的，仅用于软件测试、安全研究等正当目的
2. **驱动签名**：生产环境中需要正式的驱动签名
3. **兼容性问题**：新版Windows更新可能会影响钩子的工作方式
4. **检测方法进化**：随着检测技术的发展，需要不断更新防检测方法
5. **安全风险**：内核驱动可能导致系统不稳定，建议在测试环境中使用
6. **专业知识**：内核驱动开发需要专业的Windows内核编程知识

## [参考资源](https://aaass554.github.io/tutorials/VirtualizationTechnology/jdwa-kvmandISO.html#%E5%8F%82%E8%80%83%E8%B5%84%E6%BA%90)

1. [VM-Undetected](https://github.com/nbviet300689/VM-Undetected) - 提供了绕过多种虚拟机检测方法的解决方案
2. [bypass\_vmp\_vm\_detect](https://github.com/KuNgia09/bypass_vmp_vm_detect) - 专门针对VMP保护的虚拟机检测机制的绕过方法
3. [HiddenVM](https://gitcode.com/gh_mirrors/hi/HiddenVM) - 隐藏式虚拟机解决方案
4. [VmwareHardenedLoader](https://github.com/hzqst/VmwareHardenedLoader) - VMware加固加载器
5. [Pafish](https://github.com/a0rtega/pafish) - 用于测试虚拟环境检测方法的工具

[Prev

JDWA虚拟化技术专题](https://aaass554.github.io/tutorials/VirtualizationTechnology/index.html)
