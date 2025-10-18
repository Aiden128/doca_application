# 環境準備

與 DMA Copy 相同的 Comch 基礎設施。

## 環境檢查

```bash
# DPU
sudo systemctl status mlnx_snap.service
ip link show pf0hpf  # 必須 UP

# Host
cat /sys/bus/pci/devices/0000:07:00.0/enable  # 必須 1
```

完整準備步驟請參考 [DMA Copy Setup](../dma_copy/README.md#環境準備)
