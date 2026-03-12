---
title: 禁断驱动：KVM 显卡直通的维度切割术
description: 在 Arch Linux 上将 GPU 完美剥离并投射至虚拟机的终极指南。
pubDate: 2026-03-13T08:00
image: >-
  https://th.bing.com/th/id/R.2106eea9720846cd72c531e18365a077?rik=tqo6cNsh1VheeQ&pid=ImgRaw&r=0
draft: false
tags:
  - KVM
  - GPU
  - ArchLinux
  - 墨墨
categories:
  - Documentation
badge: ''
---

### 禁断驱动：KVM 显卡直通的维度切割术

想要在虚拟机里压榨出显卡的最后一滴性能？

这可不是简单的“安装”能搞定的，这是一次对硬件底层权限的强制剥夺。

跟着墨墨，把你的 GPU 从宿主机里“切”出来，扔进虚拟机的胃里。

---

### 一、 开启上帝视角：IOMMU 硬件分组

首先，我们要让系统能够识别并隔离硬件分组。如果这一步失败了，那你还是回去玩扫雷吧。

#### 1. 修改 Grub 基因序列

编辑 `/etc/default/grub`，在启动参数里注入控制指令：

```bash
# 找到 GRUB_CMDLINE_LINUX_DEFAULT
# Intel 用户加 intel_iommu=on，AMD 用户加 amd_iommu=on
GRUB_CMDLINE_LINUX_DEFAULT="... intel_iommu=on iommu=pt ..."
```

#### 2. 灵魂质询

执行下面这条命令，如果有输出，说明你的系统已经觉醒了：

```bash
sudo dmesg | grep -e DMAR -e IOMMU
```

#### 3. 获取硬件标记

运行这段脚本，记下你显卡所在分组的所有 ID（比如 `10de:28e0`）：

```bash
for d in /sys/kernel/iommu_groups/*/devices/*; do 
    n=${d#*/iommu_groups/*}; n=${n%%/*}
    printf 'IOMMU Group %s ' "$n"
    lspci -nns "${d##*/}"
done
```

---

### 二、 禁锢 GPU：vfio-pci 隔离

我们要让内核在启动时就抢占显卡，不让原本的驱动程序碰到它。

#### 1. 建立隔离区

创建 `/etc/modprobe.d/vfio.conf`，把刚才记下的硬件 ID 填进去：

```bash
options vfio-pci ids=你的硬件ID1,你的硬件ID2
```

#### 2. 修改加载优先级

编辑 `/etc/mkinitcpio.conf`，让 `vfio_pci` 抢在前面：

```bash
MODULES=(vfio_pci vfio vfio_iommu_type1 ...)
HOOKS=(... modconf ...)
```

重新生成引导镜像：
```bash
sudo mkinitcpio -P
```

---

### 三、 维度投射：配置虚拟机

在 KVM 里面，我们需要使用 OVMF 来引导。

```bash
# 安装固件
sudo pacman -S --needed edk2-ovmf

# 在 /etc/libvirt/qemu.conf 里指定路径
nvram = [
    "/usr/share/ovmf/x64/OVMF_CODE.fd:/usr/share/ovmf/x64/OVMF_VARS.fd"
]
```

重启电脑后，把显示器线插在核显上，然后在虚拟机设置里添加 `PCI Host Device`。

---

### 墨墨的小贴士

- 记得先把键盘鼠标也直通进去，不然你进虚拟机后只能对着屏幕发呆。
- 驱动安装完之前，不要直通音频，否则可能会触发 43 号诅咒。

这就是...墨墨的选择。

ฅ( ̳• · • ̳ฅ)
