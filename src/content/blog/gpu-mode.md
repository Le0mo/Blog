---
title: 显卡直通
description: ''
pubDate: 2026-03-02T23:21
draft: false
tags: []
categories: []
---
#显卡直通

PCI passthrough via OVMF - ArchWiki

需要有两个显卡。

    确认iommu是否开启，有输出说明开启

    sudo dmesg | grep -e DMAR -e IOMMU

    现代设备通常都支持IOMMU且默认开启，BIOS里的选项通常为Intel VT-d、AMD-V或者IOMMU。如果没有的话搜索一下自己的cpu和主板型号看看是否支持。

    获取显卡的硬件id，显卡所在group的所有设备的id都记下

    for d in /sys/kernel/iommu_groups/*/devices/*; do 
        n=${d#*/iommu_groups/*}; n=${n%%/*}
        printf 'IOMMU Group %s ' "$n"
        lspci -nns "${d##*/}"
    done

    隔离GPU

    sudo vim /etc/modprobe.d/vfio.conf

    写入

    options vfio-pci ids=10de:28e0,10de:22be （硬件id与硬件id之间用英文逗号隔开）

    编辑内核参数让vfio-pci抢先加载

        sudo vim /etc/mkinitcpio.conf

        MODULES=（）里面写入vfio_pci vfio vfio_iommu_type1

        MODULES=(... vfio_pci vfio vfio_iommu_type1  ...)

        HOOKS=()里面确认有 modconf

        HOOKS=(... modconf ...)

    重新生成initramfs

    sudo mkinitcpio -P

    安装和配置ovmf

    sudo pacman -S --needed edk2-ovmf

    编辑配置文件

    sudo vim /etc/libvirt/qemu.conf

    搜索nvram，在合适的地方写入：

    nvram = [
    	"/usr/share/ovmf/x64/OVMF_CODE.fd:/usr/share/ovmf/x64/OVMF_VARS.fd"
    ]
