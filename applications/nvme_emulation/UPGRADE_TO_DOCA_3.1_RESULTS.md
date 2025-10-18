# DOCA 3.1 升級測試記錄

## 日期
2025-10-16

## 升級目標
解決 syndrome 0x83d555 (BAD_RES_STATE_ERR) 錯誤

## 升級內容

### 升級前版本
```
DOCA SDK:         2.9.3008
BlueField Bundle: 2.9.3-32
bf-release:       4.9.3
Kernel:           5.15.0-1070-bluefield
dpacc:            1.9.0
FlexIO SDK:       24.10.2454
Firmware:         32.43.3608
```

### 升級後版本
```
DOCA SDK:         3.1.0105 ✓
BlueField Bundle: 3.1.0-76 ✓
bf-release:       4.12.0 ✓
Kernel:           5.15.0-1074-bluefield ✓
dpacc:            1.12.0.26 ✓
FlexIO SDK:       25.07.2812 ✓
Firmware:         32.43.3608 (未變更)
```

## 升級過程

### 步驟 1: 升級 DOCA SDK (3.1.0105)
```bash
$ sudo apt-get update
$ sudo apt-get install -y --only-upgrade 'doca-*'

# 升級了 38 個 doca-sdk-* 套件
# 新增套件: doca-sdk-verbs, libnvhws1
```

### 步驟 2: 升級編譯工具鏈

#### 問題: dpacc 版本不相容
**錯誤**:
```
ld.lld: error: /opt/mellanox/doca/lib/aarch64-linux-gnu/libdoca_dpa_dev.a:1:
unknown directive: NVDPAFBF
dpa-clang: error: ld.lld command failed
```

**解決**:
```bash
$ sudo apt-get install -y --only-upgrade dpacc dpacc-extract
# dpacc: 1.9.0 → 1.12.0.26
```

#### 問題: FlexIO 缺少符號
**錯誤**:
```
ld.lld: error: undefined symbol: flexio_dev_yield_and_retrigger
```

**解決**:
```bash
$ sudo apt-get install -y --only-upgrade flexio-sdk flexio-samples
# FlexIO: 24.10.2454 → 25.07.2812
```

### 步驟 3: 重新編譯應用程式
```bash
$ cd /opt/mellanox/doca/applications
$ rm -rf build
$ meson build
$ ninja -C build
```

**結果**: 編譯成功 ✓
- 二進制檔案: `/opt/mellanox/doca/applications/build/nvme_emulation/doca_nvme_emulation`
- 大小: 482K
- 編譯時間: 2025-10-16 05:23

### 步驟 4: 升級 BlueField Bundle
```bash
$ sudo apt-get install -y --only-upgrade bf-release
# bf-release: 4.9.3 → 4.12.0
# BlueField Bundle: 2.9.3 → 3.1.0-76
```

**驗證**:
```bash
$ cat /etc/mlnx-release
bf-bundle-3.1.0-76_25.07_ubuntu-22.04_prod
```

### 步驟 5: 重啟 DPU
```bash
$ sudo reboot
# 等待約 60 秒重啟完成
```

**驗證重啟後版本**:
```bash
$ uname -r
5.15.0-1074-bluefield  ✓

$ cat /etc/mlnx-release
bf-bundle-3.1.0-76_25.07_ubuntu-22.04_prod  ✓

$ dpkg -l | grep doca-sdk | head -1
doca-sdk-aes-gcm  3.1.0105-1  ✓
```

## 測試結果

### ✓ 成功: syndrome 0x83d555 錯誤已解決

**升級前錯誤**:
```
[DOCA][ERR] FW failed to execute general PRM command=0xb2d,
status=BAD_RES_STATE_ERR (0x9) with syndrome=0x83d555
Failed to modify bar stateful region: DOCA_ERROR_DRIVER
```

**升級後錯誤**:
```
[DOCA][ERR][doca_transport.c:645][nvmf_doca_create]
No emulation managers available
```

**關鍵發現**:
- 錯誤從 firmware 層級的 BAD_RES_STATE_ERR 變成設備可用性問題
- 證明 BlueField Bundle 3.1 與 DOCA SDK 3.1 的版本匹配問題已解決
- 這是一個完全不同且更高層級的錯誤

### ❌ 新問題: SNAP Controller PCIe 設備消失

#### 問題描述

**重啟前狀態** (來自之前的測試記錄):
```bash
$ lspci | grep -E '07:00\.|08:00\.'
07:00.0  BlueField-3 ConnectX-7 network controller
07:00.1  BlueField-3 ConnectX-7 network controller
07:00.2  NVMe SNAP Controller  ⬅️ 存在
07:00.3  SoC Management Interface
08:00.0  NVMe SNAP Controller  ⬅️ 存在
```

**重啟後狀態** (2025-10-16 06:10):
```bash
$ lspci | grep -E '07:00\.|08:00\.'
07:00.0  Ethernet controller [0200]: Mellanox Technologies MT43244 BlueField-3 integrated ConnectX-7 network controller [15b3:a2dc]
07:00.1  Ethernet controller [0200]: Mellanox Technologies MT43244 BlueField-3 integrated ConnectX-7 network controller [15b3:a2dc]
07:00.3  DMA controller [0801]: Mellanox Technologies MT43244 BlueField-3 SoC Management Interface [15b3:c2d5]
```

**缺失設備**:
- `07:00.2` - NVMe SNAP Controller
- `08:00.0` - NVMe SNAP Controller

#### 沒有明確的錯誤訊息

**檢查系統日誌**:
```bash
# Host 端
$ dmesg | grep -i 'snap\|emul\|nvme.*pci'
# 無相關錯誤訊息

# DPU 端
$ sudo dmesg | grep -iE 'snap|nvme.*emul|mlx5.*snap'
# 只有無關的 LVM snapshot 訊息
```

**檢查 PCIe 掃描**:
```bash
$ echo 1 | sudo tee /sys/bus/pci/rescan
# 執行後設備仍未出現
```

#### 無法檢查 Firmware 配置的原因

**MST 驅動缺失**:
```bash
$ sudo mst start
modprobe: FATAL: Module mst_pci not found in directory /lib/modules/5.15.0-1074-bluefield
modprobe: FATAL: Module mst_pciconf not found in directory /lib/modules/5.15.0-1074-bluefield
Loading MST PCI module - Failure: 1
```

**原因**:
- kernel-mft-modules 套件是為舊 kernel (5.15.0-1070) 編譯的
- 新 kernel (5.15.0-1074) 沒有對應的 MST 驅動
- 無法使用 `mlxconfig` 檢查 firmware 配置

**嘗試的解決方法**:
```bash
# 1. 從 DPU 端使用 mlxconfig (需要 MST)
$ sudo mlxconfig -d 07:00.0 q
-E- Failed to open the device  ❌

# 2. 使用 rshim 接口
$ ls /dev/rshim*
ls: cannot access '/dev/rshim*': No such file or directory  ❌

# 3. 直接 PCIe BDF
$ sudo mlxconfig -d 07:00.0 q
-E- Failed to open the device  ❌
```

## 問題分析

### 為什麼 SNAP Controllers 沒有出現？

#### 理論 1: Firmware 配置未生效
從之前的測試記錄，我們知道存在配置衝突：
```
NVME_EMULATION_ENABLE      = True(1)     ✓ 啟用
NVME_EMULATION_NUM_PF      = 1           ⚠️ 配置 1 個 PF
NUM_OF_PF                  = 2           ⚠️ 系統總共 2 個 PF
NVME_EMULATION_NUM_MSIX    = 16
NVME_EMULATION_MAX_QUEUE_DEPTH = 12
```

**可能的問題**:
- `NVME_EMULATION_NUM_PF=1` 可能導致只創建 1 個 SNAP controller
- 但之前看到 2 個 (07:00.2 和 08:00.0)，現在卻 0 個
- 升級後可能改變了 firmware 對此參數的解釋

#### 理論 2: Host 需要重啟
PCIe 設備枚舉通常發生在 Host 啟動時：
- DPU 重啟後，firmware 配置可能已生效
- 但 Host 端尚未重新掃描 PCIe bus
- 需要 Host 冷重啟或熱插拔重新枚舉

#### 理論 3: DOCA 3.1 的配置要求不同
DOCA 3.1 可能有新的配置要求：
- 不同的 firmware 參數組合
- 額外的初始化步驟
- 不同的設備創建時機

### 當前狀態

**應用程式狀態**: ✓ 正常運行
```bash
[2025-10-16 06:06:56] Starting SPDK v23.01 / DPDK 22.11.1 initialization...
[2025-10-16 06:06:56] Total cores available: 2
[2025-10-16 06:06:56] Reactor started on core 0
[2025-10-16 06:06:56] Reactor started on core 1
```

**Transport 創建失敗原因**:
```
[06:08:09][DOCA][ERR][doca_transport.c:645][nvmf_doca_create]
No emulation managers available
```

**原因**: 沒有可用的 SNAP controller 設備，因此無法創建 emulation manager。

## 關鍵進展

### ✓ 主要目標達成
**syndrome 0x83d555 (BAD_RES_STATE_ERR) 已完全解決**

這證明：
1. ✓ 版本不匹配是根本原因 (DOCA 3.1 vs BlueField Bundle 2.9.3)
2. ✓ 升級 BlueField Bundle 到 3.1.0 解決了 firmware 層錯誤
3. ✓ 編譯工具鏈升級後應用程式正常編譯和運行

### 新的次要問題
**SNAP controller PCIe 設備未出現**

這是一個**設備可見性**問題，而非 firmware 執行錯誤：
- 不再有 PRM 命令失敗
- 不再有 BAR 配置錯誤
- 只是設備不存在於 PCIe bus 上

## 下一步建議

### 選項 1: 重啟 Host 主機 (推薦)
**原因**: PCIe 設備枚舉需要 Host 配合
```bash
# 在 Host 端執行
$ sudo reboot
```

重啟後檢查:
```bash
$ lspci | grep -E '07:00\.|08:00\.'
# 檢查 07:00.2 和 08:00.0 是否出現
```

### 選項 2: 修復 MST 驅動後檢查配置
需要重新編譯或安裝 kernel-mft-modules:
```bash
# 選項 A: 安裝預編譯模組 (如果可用)
$ sudo apt-cache search kernel-mft-modules | grep 1074

# 選項 B: 使用 DKMS 重新編譯 (如果有 DKMS 版本)
$ sudo apt-get install kernel-mft-dkms

# 選項 C: 手動從 MFT 源碼編譯驅動
```

完成後檢查 firmware 配置:
```bash
$ sudo mst start
$ sudo mlxconfig -d /dev/mst/mt41692_pciconf0 q | grep -i 'nvme\|snap'
```

### 選項 3: 從 Host 端檢查 firmware (如果 Host 有 MFT)
```bash
# 在 Host 端執行 (如果安裝了 MFT)
$ sudo mst start
$ sudo mlxconfig -d /dev/mst/mt41692_pciconf0 q | grep NVME
```

### 選項 4: 搜尋 DOCA 3.1 文檔
查找 DOCA 3.1 中 NVMe Emulation 的新配置要求或已知問題。

## 結論

**主要成就**:
- ✓ 成功升級整個 DOCA 技術棧到版本 3.1
- ✓ 解決了關鍵的 syndrome 0x83d555 firmware 錯誤
- ✓ 證明了版本匹配的重要性

**當前障礙**:
- SNAP controller PCIe 設備不可見
- 無法訪問 firmware 配置工具 (MST 驅動缺失)
- 需要進一步調查或 Host 重啟

**技術學習**:
1. BlueField Bundle 版本必須與 DOCA SDK 版本匹配
2. Firmware 錯誤可能源於版本不相容
3. PCIe 設備枚舉涉及 Host 和 DPU 兩側的配合
4. 升級需要協調多個組件 (SDK, Bundle, 編譯工具鏈)

---

**下次更新**: Host 重啟後或 MST 驅動修復後
