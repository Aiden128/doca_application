# DOCA DMA Copy 故障排除指南

> **本指南適用於**: DMA Copy, File Integrity, File Compression 等所有基於 Comch 的 Host-DPU 通訊應用

---

## 📋 目錄

1. [環境檢查流程](#環境檢查流程)
2. [常見錯誤與解決方案](#常見錯誤與解決方案)
3. [系統性除錯方法](#系統性除錯方法)
4. [快速檢查清單](#快速檢查清單)

---

## 環境檢查流程

### 🔍 第一步：確認你在哪個端 (Host or DPU)

**如何判斷**:
```bash
hostname
# Host 端通常顯示: dpu-host 或其他主機名
# DPU 端通常顯示: localhost 或 DPU 特定名稱

# 更可靠的方法：檢查網路接口
ip addr show | grep tmfifo_net0
# 如果找到 tmfifo_net0，就是 DPU 端
# 如果沒有，就是 Host 端
```

**為什麼重要**: Host 和 DPU 的檢查項目不同，搞混會浪費時間

---

### 🔍 第二步：DPU 端檢查 (在 192.168.100.2 執行)

#### 檢查 2.1: Hugepages 配置

**為什麼需要**: mlnx_snap 服務需要至少 2GB hugepages 才能啟動

**如何檢查**:
```bash
grep Huge /proc/meminfo
```

**正常輸出應該看到**:
```
HugePages_Total:    1024
Hugepagesize:       2048 kB
Hugetlb:         2097152 kB  # 這個要 >= 2097152 (2GB)
```

**如果 Hugetlb 小於 2GB**:
```bash
# 臨時配置（重啟後失效）
echo 1024 | sudo tee /proc/sys/vm/nr_hugepages

# 永久配置
echo "vm.nr_hugepages=1024" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

---

#### 檢查 2.2: mlnx_snap 服務狀態

**為什麼需要**: DMA Copy 完全依賴這個服務，它負責 SNAP emulation

**如何檢查**:
```bash
sudo systemctl status mlnx_snap.service
```

**✅ 正常輸出**:
```
● mlnx_snap.service - SNAP Emulation Daemon
     Active: active (running) since ...
```

**❌ 異常輸出 1: 服務 failed**
```
● mlnx_snap.service - SNAP Emulation Daemon
     Active: failed (Result: exit-code)
```

**如何修復**:
```bash
# 1. 查看詳細錯誤
sudo journalctl -u mlnx_snap.service -n 50

# 2. 如果看到 "doca-hugepages command not found"
#    這是 DOCA 3.1 的已知問題

# 3. 修復方法：修改檢查腳本
sudo cp /usr/sbin/mlnx_snap_hugepages.sh /usr/sbin/mlnx_snap_hugepages.sh.backup

sudo nano /usr/sbin/mlnx_snap_hugepages.sh
# 內容改為：
```

```bash
#!/bin/bash -eE

MIN_HUGEMEM_GB=2

hugepage_size_kb=$(grep Hugepagesize /proc/meminfo | awk '{print $2}')
hugetlb_kb=$(grep "^Hugetlb:" /proc/meminfo | awk '{print $2}')

required_kb=$((MIN_HUGEMEM_GB * 1024 * 1024))

if [ $hugetlb_kb -ge $required_kb ]; then
  echo "Hugepages OK: ${hugetlb_kb}KB >= ${required_kb}KB"
  exit 0
else
  echo "ERROR: Insufficient hugepages"
  exit 1
fi
```

```bash
# 4. 重啟服務
sudo systemctl restart mlnx_snap.service

# 5. 確認成功
sudo systemctl status mlnx_snap.service
```

---

#### 檢查 2.3: Host PF Representor 狀態

**為什麼需要**: Representor 是 DPU 用來與 Host PF 通訊的虛擬接口

**如何檢查**:
```bash
ip link show pf0hpf
```

**✅ 正常輸出**:
```
9: pf0hpf: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
    state UP mode DEFAULT ...
```

關鍵字: **UP, LOWER_UP, state UP**

**❌ 如果找不到 pf0hpf**:
```
Device "pf0hpf" does not exist.
```

**這表示**:
1. mlnx_snap 服務可能未運行
2. Host PF 可能未正確初始化
3. 需要先檢查 Host 端狀態（見下一節）

**如何深入檢查**:
```bash
# 查看所有 representor
ip link show | grep pf0

# 查看 devlink port 映射
sudo devlink port show | grep pf0hpf
# 應該看到: controller 1 pfnum 0 external true
# controller 1 = Host 端
```

---

### 🔍 第三步：Host 端檢查 (在 192.168.100.1 執行)

#### 檢查 3.1: PCI 設備識別

**如何檢查**:
```bash
lspci | grep Mellanox
```

**正常輸出範例**:
```
07:00.0 Ethernet controller: Mellanox Technologies MT43244 BlueField-3 integrated ConnectX-7
07:00.1 Ethernet controller: Mellanox Technologies MT43244 BlueField-3 integrated ConnectX-7
07:00.3 DMA controller: Mellanox Technologies MT43244 BlueField-3 SoC Management Interface
```

**重點**: 記住第一個 Ethernet controller 的地址（例如 `07:00.0`）

---

#### 檢查 3.2: PCI 設備 Enable 狀態

**為什麼重要**: 這是最常見的問題！設備必須 enable 才能使用

**如何檢查**:
```bash
# 假設你的設備是 07:00.0
cat /sys/bus/pci/devices/0000:07:00.0/enable
```

**✅ 正常輸出**: `1` (已啟用)
**❌ 異常輸出**: `0` (未啟用)

**如何修復**:
```bash
echo 1 | sudo tee /sys/bus/pci/devices/0000:07:00.0/enable

# 確認
cat /sys/bus/pci/devices/0000:07:00.0/enable
# 應該顯示: 1
```

---

#### 檢查 3.3: BusMaster 狀態

**為什麼重要**: BusMaster 控制設備是否能主動發起 DMA 傳輸

**如何檢查**:
```bash
lspci -vvv -s 07:00.0 | grep "Control:"
```

**✅ 正常輸出**:
```
Control: ... BusMaster+ ...
```

關鍵字: **BusMaster+** (有 `+` 號)

**❌ 異常輸出**:
```
Control: ... BusMaster- ...
```

**如何修復**:
通常在執行 driver binding 後會自動啟用（見下一步）

---

#### 檢查 3.4: mlx5_core 驅動狀態

**為什麼重要**: 沒有驅動，設備無法工作

**如何檢查**:
```bash
lspci -vvv -s 07:00.0 | grep "Kernel driver"
```

**✅ 正常輸出**:
```
Kernel driver in use: mlx5_core
```

**❌ 異常輸出**: 沒有任何輸出，或顯示其他驅動

**如何修復**:
```bash
# 1. 確認驅動模組已載入
lsmod | grep mlx5_core
# 應該看到: mlx5_core ...

# 2. 如果沒有，載入模組
sudo modprobe mlx5_core

# 3. 綁定驅動到設備
echo "0000:07:00.0" | sudo tee /sys/bus/pci/drivers/mlx5_core/bind

# 4. 等待初始化
sleep 3

# 5. 確認
lspci -vvv -s 07:00.0 | grep "Kernel driver"
# 應該顯示: Kernel driver in use: mlx5_core

# 6. 確認 BusMaster 也啟用了
lspci -vvv -s 07:00.0 | grep "BusMaster"
# 應該顯示: BusMaster+
```

**✅ 成功標誌**: 會看到新的網路接口出現
```bash
ip link show | grep enp7
# 例如: enp7s0f0np0, enp7s0f1np1
```

---

### 🔍 第四步：PCI 地址配置驗證

#### 關鍵概念

**DPU Server 需要兩個 PCI 地址**:
- `-p`: 本地 DPU 的 PCI 設備 (例如 `03:00.0`)
- `-r`: Host PF 的 Representor PCI 地址 (例如 `07:00.0`)

**Host Client 只需要一個**:
- `-p`: 本地 Host 的 PCI 設備 (例如 `07:00.0`)

#### 如何確認正確的地址

**在 DPU 端查找 `-p` 地址**:
```bash
lspci | grep "ConnectX" | head -1
# 輸出: 03:00.0 Ethernet controller: ... ConnectX-7
# 使用: -p 03:00.0
```

**在 DPU 端查找 `-r` 地址**:
```bash
sudo devlink port show | grep pf0hpf
# 輸出: pci/0000:03:00.0/262144: ... controller 1 pfnum 0 external true
```

這告訴我們 Host 的 controller 1, pfnum 0 對應的是什麼。

然後在 Host 端確認:
```bash
lspci | grep "Ethernet.*ConnectX" | head -1
# 輸出: 07:00.0 Ethernet controller: ... ConnectX-7
# 使用: -r 07:00.0  (不是 07:00.3!)
```

**❌ 常見錯誤**: 使用 `07:00.3` (Management Interface) 而非 `07:00.0` (Ethernet PF)

---

## 常見錯誤與解決方案

### 錯誤 1: "Host PF is disabled"

**完整錯誤訊息**:
```
[DOCA][WRN] Host PF is disabled
[DOCA][ERR] Failed to create representor list
[DOCA][ERR] Failed to open Comm Channel DOCA device representor
```

**原因分析**:
這個錯誤有兩種可能:
1. **DPU 端**: mlnx_snap 服務未運行
2. **Host 端**: PCI 設備未啟用或驅動未綁定

**診斷流程**:
```bash
# Step 1: 在 DPU 端檢查
ssh ubuntu@192.168.100.2 "sudo systemctl status mlnx_snap.service"

# 如果 failed，先修復 mlnx_snap（見檢查 2.2）
# 如果 running，繼續下一步

# Step 2: 在 Host 端檢查
cat /sys/bus/pci/devices/0000:07:00.0/enable
# 如果是 0，執行: echo 1 | sudo tee /sys/bus/pci/devices/0000:07:00.0/enable

# Step 3: 檢查驅動
lspci -vvv -s 07:00.0 | grep "Kernel driver"
# 如果沒有 mlx5_core，綁定它（見檢查 3.4）
```

---

### 錯誤 2: "Matching device not found"

**完整錯誤訊息**:
```
[DOCA][WRN] Matching device not found
[DOCA][ERR] Failed to open Comm Channel DOCA device representor based on PCI address
```

**原因**: `-r` 參數的 PCI 地址不正確

**解決方案**:
```bash
# 在 DPU 端，不要用 07:00.3，要用 07:00.0
# ❌ 錯誤:
sudo doca_dma_copy -p 03:00.0 -r 07:00.3 -f file.txt

# ✅ 正確:
sudo doca_dma_copy -p 03:00.0 -r 07:00.0 -f file.txt
```

---

### 錯誤 3: "Connection timeout"

**症狀**: Server 啟動成功，但 Client 無法連接

**可能原因**:
1. 網路不通 (tmfifo_net0)
2. 防火牆阻擋
3. Server 和 Client 的 PCI 地址不匹配

**診斷**:
```bash
# 1. 測試網路連通性
# 在 Host 端:
ping 192.168.100.2

# 2. 確認 tmfifo_net0 狀態
# 在兩端都執行:
ip addr show tmfifo_net0

# 3. 檢查 PCI 地址是否一致
# Host Client 的 -p 應該對應 DPU Server 的 -r
```

---

## 系統性除錯方法

當遇到問題時，按照這個順序診斷:

### 診斷流程圖

```
開始
  ↓
錯誤發生在哪一端？
  ├─ DPU 端 →
  │   ↓
  │   mlnx_snap 服務 running?
  │   ├─ No → 修復 mlnx_snap（檢查 hugepages, 腳本）
  │   └─ Yes →
  │       ↓
  │       pf0hpf representor UP?
  │       ├─ No → 檢查 Host 端 PCI 設備
  │       └─ Yes → 檢查 PCI 地址配置
  │
  └─ Host 端 →
      ↓
      PCI 設備 enable=1?
      ├─ No → echo 1 > enable
      └─ Yes →
          ↓
          mlx5_core 驅動已綁定?
          ├─ No → 綁定驅動
          └─ Yes →
              ↓
              BusMaster+?
              ├─ No → 重新綁定驅動
              └─ Yes → 檢查 PCI 地址配置
```

---

## 快速檢查清單

### 🚀 啟動前檢查 (2分鐘)

**DPU 端** (192.168.100.2):
```bash
# ✓ Hugepages >= 2GB
grep Hugetlb /proc/meminfo | awk '{print $2}'  # >= 2097152

# ✓ mlnx_snap running
sudo systemctl is-active mlnx_snap.service  # active

# ✓ pf0hpf UP
ip link show pf0hpf | grep "state UP"  # 應該看到 UP
```

**Host 端** (192.168.100.1):
```bash
# ✓ PCI enable=1
cat /sys/bus/pci/devices/0000:07:00.0/enable  # 1

# ✓ BusMaster+
lspci -vvv -s 07:00.0 | grep "BusMaster+"  # 應該看到

# ✓ mlx5_core loaded
lspci -vvv -s 07:00.0 | grep "mlx5_core"  # 應該看到
```

**如果以上全部通過 → 可以開始測試！**

---

### 📋 測試執行清單

**1. DPU 端啟動 Server**:
```bash
# Terminal 1 (DPU)
sudo /opt/mellanox/doca/applications/build/dma_copy/doca_dma_copy \
    -p 03:00.0 \
    -r 07:00.0 \
    -f received_file.txt

# 等待看到:
# [DOCA][INF] DOCA version ...
# Server waiting for connection...
```

**2. Host 端啟動 Client**:
```bash
# Terminal 2 (Host)
sudo /path/to/doca_dma_copy \
    -p 07:00.0 \
    -f /path/to/test_file.txt

# 成功標誌:
# [DOCA][INF] File was found locally, it will be DMA copied to the DPU
# [DOCA][INF] Final status message was successfully received
```

**3. 驗證完整性**:
```bash
# DPU 端
md5sum received_file.txt

# Host 端
md5sum test_file.txt

# 兩個 checksum 應該完全一樣
```

---

## 💡 故障排除技巧

### 技巧 1: 分層診斷

```
Layer 4: 應用層 (DMA Copy)
    ↓ 錯誤在應用，檢查 PCI 地址配置
Layer 3: 通訊層 (Comch)
    ↓ 錯誤在 Comch，檢查 representor
Layer 2: 驅動層 (mlx5_core)
    ↓ 錯誤在驅動，檢查 driver binding
Layer 1: 硬體層 (PCI 設備)
    ↓ 錯誤在硬體，檢查 enable + BusMaster
Layer 0: 服務層 (mlnx_snap)
    ↓ 錯誤在服務，檢查 hugepages
```

從下往上檢查，不要跳過！

---

### 技巧 2: 使用日誌級別

```bash
# 啟用詳細日誌
export DOCA_LOG_LEVEL=20  # INFO
export DOCA_LOG_LEVEL=10  # DEBUG (非常詳細)

# 然後執行應用
sudo DOCA_LOG_LEVEL=10 ./doca_dma_copy ...
```

---

### 技巧 3: 隔離變數

**測試時一次只改一個東西**:
- ❌ 同時修改 PCI 地址和重啟服務
- ✅ 先確認服務 OK，再測試 PCI 地址

---

## 🔧 環境恢復腳本

如果完全搞亂了，用這個恢復:

```bash
#!/bin/bash
# 在 Host 端執行

# 1. Reset PCI 設備
PCI_ADDR="07:00.0"

# Unbind driver
echo "0000:$PCI_ADDR" | sudo tee /sys/bus/pci/drivers/mlx5_core/unbind 2>/dev/null || true

# Disable device
echo 0 | sudo tee /sys/bus/pci/devices/0000:$PCI_ADDR/enable

# Wait
sleep 2

# Enable device
echo 1 | sudo tee /sys/bus/pci/devices/0000:$PCI_ADDR/enable

# Bind driver
echo "0000:$PCI_ADDR" | sudo tee /sys/bus/pci/drivers/mlx5_core/bind

# Verify
sleep 3
lspci -vvv -s $PCI_ADDR | grep "Kernel driver"
lspci -vvv -s $PCI_ADDR | grep "BusMaster"
```

---

## 📚 參考資料

1. **Testing_Results.md** - 實際測試記錄和問題修復過程
2. **Complete_Troubleshooting_Journey.md** - 完整除錯旅程
3. **Complete_Guide.md** - DMA Copy 完整指南
4. **DOCA Documentation** - https://docs.nvidia.com/doca/

---

**最後更新**: 2025-10-18
**作者**: DOCA Application Documenter Agent
**狀態**: Production Ready ✅
