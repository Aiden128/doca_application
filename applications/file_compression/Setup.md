# 環境準備

## 系統需求

**DPU**: BlueField-2/3, Hugepages >= 2GB  
**Host**: x86_64, PCIe 連接  
**軟體**: DOCA >= 3.1.0 (DPU), >= 2.9.3 (Host)

## Hugepages 配置

```bash
echo 1024 | sudo tee /proc/sys/vm/nr_hugepages
echo "vm.nr_hugepages=1024" | sudo tee -a /etc/sysctl.conf
```

## mlnx_snap 服務修復（DOCA 3.1 重要）

修改 `/usr/sbin/mlnx_snap_hugepages.sh`:
```bash
#!/bin/bash -eE
MIN_HUGEMEM_GB=2
hugepage_size_kb=$(grep Hugepagesize /proc/meminfo | awk '{print $2}')
hugetlb_kb=$(grep "^Hugetlb:" /proc/meminfo | awk '{print $2}')
required_kb=$((MIN_HUGEMEM_GB * 1024 * 1024))
[ $hugetlb_kb -ge $required_kb ] && exit 0 || exit 1
```

重啟服務：
```bash
sudo systemctl restart mlnx_snap.service
```

## Host PF 啟用

```bash
echo 1 | sudo tee /sys/bus/pci/devices/0000:07:00.0/enable
echo "0000:07:00.0" | sudo tee /sys/bus/pci/drivers/mlx5_core/bind
```

完整步驟請參考 [DMA Copy Setup](../dma_copy/README.md#環境準備)
