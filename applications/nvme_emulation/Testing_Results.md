# DOCA NVMe Emulation - 實際測試記錄

## 測試日期
2025-10-15

## 測試環境

### 硬體環境
- **DPU 型號**: BlueField-3 DPU
- **DPU IP**: 192.168.100.2
- **Host IP**: 192.168.100.1
- **總記憶體**: 131 GB
- **CPU 核心**: 8 個 ARM64 核心

### 軟體環境
```bash
# DOCA 版本檢查
$ dpkg -l | grep doca-devel
doca-devel  2.9.3008-1  arm64

# SPDK 版本（透過 RPC 查詢）
SPDK v23.01 (major: 23, minor: 1, patch: 0)
```

### 應用程式位置
```bash
/opt/mellanox/doca/applications/build/nvme_emulation/doca_nvme_emulation
Size: 410 KB
```

## 測試步驟與結果

### 步驟 1: 檢查編譯狀態

應用程式已成功編譯：

```bash
$ ls -la /opt/mellanox/doca/applications/build/nvme_emulation/
-rwxr-xr-x 1 dpu dpu 419888 Oct 15 15:45 doca_nvme_emulation
```

### 步驟 2: Hugepages 配置

#### 問題發現

首次嘗試啟動時遇到記憶體不足錯誤：

```
EAL: Not enough memory available! Requested: 1024MB, available: 928MB
EAL: FATAL: Cannot init memory
[2025-10-15 16:21:04.284247] init.c: 594:spdk_env_init: *ERROR*: Failed to initialize DPDK
```

#### 原因分析

檢查 hugepages 狀態：

```bash
$ cat /proc/meminfo | grep HugePages
HugePages_Total:    1024    # 總共 2GB
HugePages_Free:      464    # 只有 928MB 可用
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
```

**根本原因**:
- 總共配置了 1024 個 hugepages (2GB)
- 已經被其他程式使用了 560 個 (1120MB)
- 只剩 464 個可用 (928MB)
- 應用程式需要 512 個 (1024MB)
- 928MB < 1024MB = 失敗

#### 解決方案

增加 hugepages 數量：

```bash
$ echo 2048 | sudo tee /proc/sys/vm/nr_hugepages

# 驗證結果
$ cat /proc/meminfo | grep HugePages
HugePages_Total:    2048    # 增加到 4GB
HugePages_Free:     1488    # 2976MB 可用 ✅
```

**學到的經驗**: 總是要檢查 HugePages_Free 而非 HugePages_Total，因為其他程式可能已經占用部分記憶體。

### 步驟 3: 清理舊的 Socket

#### 問題發現

第二次嘗試啟動時遇到 socket 衝突：

```
[2025-10-15 16:25:02.145729] rpc.c: 181:spdk_rpc_listen: *ERROR*: RPC Unix domain socket path /var/tmp/spdk.sock in use. Specify another.
[2025-10-15 16:25:02.145753] rpc.c:  46:spdk_rpc_initialize: *ERROR*: Unable to start RPC service at /var/tmp/spdk.sock
```

#### 解決方案

清理舊程式和 socket：

```bash
# 終止所有 SPDK 相關程式
$ sudo pkill -f doca_nvme_emulation

# 移除舊的 socket 檔案
$ sudo rm -f /var/tmp/spdk.sock

# 驗證無程式運行
$ ps aux | grep doca_nvme_emulation | grep -v grep
# （無輸出表示已清理完成）
```

### 步驟 4: 成功啟動應用程式

#### 啟動命令

```bash
$ nohup sudo /opt/mellanox/doca/applications/build/nvme_emulation/doca_nvme_emulation \
    -m 0x3 \
    --main-core 0 \
    -s 1024 \
    --no-pci \
    --wait-for-rpc \
    > /tmp/nvme_emulation.log 2>&1 &
```

#### 參數說明

| 參數 | 值 | 說明 | 原因 |
|------|-----|------|------|
| `-m` | `0x3` | CPU mask (二進制 0011) | 使用 CPU 核心 0 和 1，SPDK 需要獨佔 CPU 實現高性能輪詢 |
| `--main-core` | `0` | 主執行緒運行在 CPU 0 | 協調所有 SPDK 子系統的主控制循環 |
| `-s` | `1024` | 分配 1024MB hugepage 記憶體 | SPDK 需要大量記憶體用於 DMA 緩衝區和佇列 |
| `--no-pci` | - | 不掃描 PCI 裝置 | 避免與系統 NVMe 裝置衝突 |
| `--wait-for-rpc` | - | 等待 RPC 命令 | 允許在啟動後通過 RPC 進行動態配置 |

#### 成功啟動的輸出

```
[2025-10-15 16:31:47.046495] Starting SPDK v23.01 / DPDK 22.11.1 initialization...
[2025-10-15 16:31:47.046723] [ DPDK EAL parameters: nvmf --no-shconf -c 0x3 -m 1024 --no-pci --huge-unlink --log-level=lib.eal:6 --log-level=lib.cryptodev:5 --log-level=user1:6 --base-virtaddr=0x200000000000 --match-allocations --file-prefix=spdk_pid679128 ]
TELEMETRY: No legacy callbacks, legacy socket not created
[2025-10-15 16:31:47.265535] app.c: 722:spdk_app_start: *NOTICE*: Total cores available: 2
[2025-10-15 16:31:47.283026] reactor.c: 926:reactor_run: *NOTICE*: Reactor started on core 1
[2025-10-15 16:31:47.283033] reactor.c: 926:reactor_run: *NOTICE*: Reactor started on core 0
```

#### 驗證狀態

```bash
# 檢查 RPC socket
$ ls -la /var/tmp/spdk.sock
srwxr-xr-x 1 root root 0 Oct 15 16:31 /var/tmp/spdk.sock  ✅

# 檢查程式運行狀態
$ ps aux | grep doca_nvme_emulation | grep -v grep
root  679128  192  0.0 67236728 15692 ?  RLl  16:31   0:30 /opt/mellanox/doca/applications/build/nvme_emulation/doca_nvme_emulation -m 0x3 --main-core 0 -s 1024 --no-pci --wait-for-rpc
```

**關鍵指標分析**:
- **CPU 使用率**: 192% - 表示正在使用 2 個 CPU 核心（0 和 1）進行輪詢
- **記憶體使用**: 15692 KB (約 15MB 應用程式本身)
- **狀態**: `RLl` - Running, Low priority, locked in memory

```bash
# 檢查 hugepages 使用情況
$ cat /proc/meminfo | grep HugePages
HugePages_Total:    2048    # 總共 4GB
HugePages_Free:     1536    # 剩餘 3GB
HugePages_Rsvd:        0
HugePages_Surp:        0
```

**記憶體分配驗證**: 2048 - 1536 = 512 個 hugepages (1024MB) 已被應用程式使用 ✅

### 步驟 5: 創建 Python RPC 客戶端

#### 問題背景

SPDK 通常使用 `/opt/mellanox/spdk/scripts/rpc.py` 進行 RPC 調用，但在這個環境中該腳本不存在。我們需要自己實現一個 RPC 客戶端。

#### 解決方案

創建 Python RPC 客戶端腳本：

```python
#!/usr/bin/env python3
import socket
import json
import sys

def send_rpc(socket_path, method, params=None):
    sock = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
    try:
        sock.connect(socket_path)
        req = {"jsonrpc": "2.0", "id": 1, "method": method}
        if params:
            req["params"] = params
        sock.sendall((json.dumps(req) + '\n').encode())

        response = b""
        while True:
            chunk = sock.recv(4096)
            if not chunk:
                break
            response += chunk
            if b'\n' in chunk:
                break

        if response:
            return json.loads(response.decode())
        return None
    finally:
        sock.close()

if __name__ == "__main__":
    socket_path = "/var/tmp/spdk.sock"
    method = sys.argv[1] if len(sys.argv) > 1 else "spdk_get_version"
    params = json.loads(sys.argv[2]) if len(sys.argv) > 2 else None

    result = send_rpc(socket_path, method, params)
    print(json.dumps(result, indent=2))
```

儲存為 `/tmp/spdk_rpc.py` 並測試：

```bash
$ sudo python3 /tmp/spdk_rpc.py spdk_get_version
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "version": "SPDK v23.01",
    "fields": {
      "major": 23,
      "minor": 1,
      "patch": 0,
      "suffix": ""
    }
  }
}
```

✅ **RPC 客戶端工作正常！**

### 步驟 6: 初始化 SPDK Framework

在創建任何資源之前，必須先初始化 framework：

```bash
$ sudo python3 /tmp/spdk_rpc.py framework_start_init
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": true
}
```

✅ **Framework 初始化成功**

### 步驟 7: 創建 Block Device (Bdev)

#### 創建 512MB 記憶體塊裝置

```bash
$ sudo python3 /tmp/spdk_rpc.py bdev_malloc_create '{"name":"Malloc0","num_blocks":1048576,"block_size":512}'
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": "Malloc0"
}
```

#### 參數說明

- **name**: `Malloc0` - 塊裝置名稱
- **num_blocks**: `1048576` - 塊數量 (1M 個塊)
- **block_size**: `512` - 每個塊的大小（字節）
- **總容量**: 1048576 × 512 = 536870912 bytes = 512 MB

✅ **Malloc bdev "Malloc0" 創建成功**

### 步驟 8: 創建 NVMe-oF Subsystem

#### 創建子系統

```bash
$ sudo python3 /tmp/spdk_rpc.py nvmf_create_subsystem '{
  "nqn":"nqn.2016-06.io.spdk:cnode1",
  "allow_any_host":true,
  "serial_number":"SPDK00000000000001"
}'
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": true
}
```

#### 參數說明

- **nqn** (NVMe Qualified Name): `nqn.2016-06.io.spdk:cnode1`
  - 格式: `nqn.{year}-{month}.{reverse_domain}:{unique_name}`
  - 這是 NVMe-oF 標準要求的唯一標識符
- **allow_any_host**: `true` - 允許任何 Host 連接（測試環境使用）
- **serial_number**: `SPDK00000000000001` - Controller 的序號

✅ **Subsystem 創建成功**

### 步驟 9: 添加 Namespace

#### 將 Bdev 添加為 Namespace

```bash
$ sudo python3 /tmp/spdk_rpc.py nvmf_subsystem_add_ns '{
  "nqn":"nqn.2016-06.io.spdk:cnode1",
  "namespace":{"bdev_name":"Malloc0"}
}'
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": 1
}
```

#### 結果分析

- 返回值 `1` 表示 Namespace ID
- 這個 Namespace 會在 Host 端顯示為 `/dev/nvme0n1` (Controller 0, Namespace 1)

✅ **Namespace 添加成功 (NSID: 1)**

### 步驟 10: 嘗試創建 DOCA Transport

#### 查詢可用的 DOCA Emulation Manager

```bash
$ sudo python3 /tmp/spdk_rpc.py nvmf_doca_get_managers
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": [
    {
      "name": "mlx5_0"
    }
  ]
}
```

✅ **找到 DOCA Manager: mlx5_0** (Mellanox ConnectX 裝置)

#### 嘗試創建 DOCA Transport

```bash
$ sudo python3 /tmp/spdk_rpc.py nvmf_create_transport '{"trtype":"DOCA"}'
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32603,
    "message": "Transport type 'DOCA' create failed"
  }
}
```

❌ **Transport 創建失敗**

### 步驟 11: 錯誤分析

#### 查看詳細日誌

```bash
$ tail -40 /tmp/nvme_emulation.log
```

#### 關鍵錯誤訊息

```
[16:33:56:423281][679128][DOCA][ERR][linux_devx_adapter.cpp:143][send_prm_cmd]
devx adapter 0xaaaae303a3c0: FW failed to execute general PRM command=0xb2d,
status=BAD_RES_STATE_ERR (0x9) with syndrome=0x83d555

[16:33:56:423449][679128][DOCA][ERR][doca_devemu_pci_dev.cpp:2419]
[doca_devemu_pci_type_modify_bar_stateful_region_default_values]
exception occurred: DOCA exception [DOCA_ERROR_DRIVER] with message Failed to send generic message

[16:33:56:423503][679128][DOCA][ERR][doca_transport.c:473][nvmf_doca_pci_type_create_and_start]
Failed to modify bar stateful region: DOCA_ERROR_DRIVER

[16:33:56:423533][679128][DOCA][ERR][doca_transport.c:516][nvmf_doca_create_emulation_manager]
Failed to initialize PCI type: DOCA_ERROR_DRIVER

[16:33:56:423726][679128][DOCA][ERR][doca_transport.c:636][nvmf_doca_create]
Emulation manager initialization failed: DOCA_ERROR_DRIVER

[16:33:56:423753][679128][DOCA][ERR][doca_transport.c:645][nvmf_doca_create]
No emulation managers available

[2025-10-15 16:33:56.427251] transport.c: 239:spdk_nvmf_transport_create: *ERROR*:
Unable to create new transport of type DOCA

[2025-10-15 16:33:56.427332] nvmf_rpc.c:1996:rpc_nvmf_create_transport: *ERROR*:
Transport type 'DOCA' create failed
```

#### 錯誤原因分析

**主要錯誤**: `BAD_RES_STATE_ERR` (錯誤碼 0x9) with syndrome `0x83d555`

這個錯誤表示：

1. **DPU 固件狀態錯誤**
   - PRM (Physical Resource Manager) 命令執行失敗
   - 資源處於錯誤的狀態，無法創建 PCI 模擬裝置

2. **技術細節**：
   - `devx adapter`: Device Experience API，用於直接訪問硬體
   - `PRM command=0xb2d`: 固件命令，用於配置 PCI BAR (Base Address Register)
   - `BAR stateful region`: PCI 裝置的記憶體映射區域

### 步驟 12: 根本原因調查（2025-10-16 更新）

#### 深入檢查 PCIe 設備狀態

經過進一步調查，發現了問題的**真正根本原因**：

```bash
# 檢查 PCIe 設備樹
$ lspci -tv
...
|           +-1c.0-[05-16]----00.0-[06-16]--+-00.0-[07]--+-00.0  BlueField-3 ConnectX-7
|           |                               |            +-00.1  BlueField-3 ConnectX-7
|           |                               |            +-00.2  NVMe SNAP Controller ⬅️
|           |                               |            \-00.3  SoC Management Interface
|           |                               +-01.0-[08]----00.0  NVMe SNAP Controller ⬅️
...

# 檢查 SNAP Controller 驅動綁定狀態
$ lspci -s 08:00.0 -k
08:00.0 Non-Volatile memory controller: Mellanox Technologies NVMe SNAP Controller
Kernel driver in use: nvme  ⬅️ 關鍵發現！
Kernel modules: nvme

$ lspci -s 07:00.2 -k
07:00.2 Non-Volatile memory controller: Mellanox Technologies NVMe SNAP Controller
# 沒有綁定驅動

# 檢查是否有 NVMe 設備
$ ls /dev/nvme*
/dev/nvme0      ⬅️ 對應 08:00.0
/dev/nvme0n1    ⬅️ Namespace
```

#### 根本原因確認

**問題核心**：SNAP Controller **已經被 Linux kernel 的 nvme 驅動占用**

詳細分析：

1. **SNAP Controller 已存在且運行中**
   - 系統中有兩個 NVMe SNAP Controller：`07:00.2` 和 `08:00.0`
   - `08:00.0` 已經綁定到 kernel 的 `nvme` 驅動
   - 該設備已經顯示為 `/dev/nvme0` 和 `/dev/nvme0n1`
   - PCI BAR 已經配置：`0x70400000-0x70407fff` (32KB)

2. **為什麼會失敗**
   - DOCA NVMe Emulation 需要**完全控制** SNAP Controller
   - 創建 Transport 時需要配置 PCI BAR 區域
   - 但設備已經被 kernel 驅動占用，處於 ACTIVE 狀態
   - 固件無法修改一個正在使用中的設備的 BAR 區域
   - 因此返回 `BAD_RES_STATE_ERR` (資源處於錯誤狀態)

3. **錯誤流程追蹤**
   ```
   DOCA NVMe Emulation
   └─> nvmf_doca_create()
       └─> nvmf_doca_create_emulation_manager()
           └─> nvmf_doca_pci_type_create_and_start()
               └─> doca_devemu_pci_type_modify_bar_stateful_region_default_values()
                   └─> send_prm_cmd(0xb2d)  ⬅️ 固件命令
                       └─> ❌ BAD_RES_STATE_ERR
                           原因：設備已被 nvme driver 占用
   ```

#### 衝突的架構圖

```
當前狀態（設備衝突）：
┌─────────────────────────────────────────────┐
│         DPU 端 (ARM64) - BlueField-3        │
│                                             │
│  ┌──────────────────────────────────────┐  │
│  │  08:00.0 SNAP Controller             │  │
│  │  ├─ Kernel Driver: nvme ⬅️ 衝突！   │  │
│  │  ├─ Device: /dev/nvme0, /dev/nvme0n1 │  │
│  │  ├─ BAR0: 0x70400000 (已配置)        │  │
│  │  └─ 狀態: ACTIVE                     │  │
│  └──────────────────────────────────────┘  │
│               ↑                             │
│               │ 嘗試修改 BAR                 │
│               ✗ 失敗！                      │
│               │                             │
│  ┌────────────┴──────────────────────────┐ │
│  │ DOCA NVMe Emulation Application       │ │
│  │ └─ 無法取得設備控制權                  │ │
│  └───────────────────────────────────────┘ │
│                                             │
│  ┌──────────────────────────────────────┐  │
│  │  07:00.2 SNAP Controller             │  │
│  │  └─ 未綁定驅動（可能可用）            │  │
│  └──────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

#### 解決方案

**方案 1: 解綁 kernel 驅動（建議）**

需要將 SNAP Controller 從 kernel nvme 驅動解綁，讓 DOCA 應用程式可以控制：

```bash
# 1. 解綁 nvme 驅動
echo "0000:08:00.0" | sudo tee /sys/bus/pci/drivers/nvme/unbind

# 2. 驗證解綁
lspci -s 08:00.0 -k
# 應該看不到 "Kernel driver in use" 這一行

# 3. （可選）綁定到 vfio-pci 以便用戶空間訪問
sudo modprobe vfio-pci
echo "15b3 6001" | sudo tee /sys/bus/pci/drivers/vfio-pci/new_id
echo "0000:08:00.0" | sudo tee /sys/bus/pci/drivers/vfio-pci/bind

# 4. 重新啟動 DOCA NVMe Emulation 並嘗試創建 Transport
```

**方案 2: 配置使用未綁定的 Controller**

可能可以配置 DOCA NVMe Emulation 使用 `07:00.2`（未綁定驅動的 SNAP Controller），需要查看應用程式配置選項。

**方案 3: 啟動時防止自動綁定**

在系統啟動時阻止 nvme 驅動自動綁定到 SNAP Controller：

```bash
# 添加到 /etc/modprobe.d/nvme-blacklist.conf
options nvme blacklist=15b3:6001
```

#### 重要發現總結

**之前的假設 ❌**: "缺少實際的 Host 機器連接（PCIe 連接）"

**實際情況 ✅**:
- DPU **確實連接到 Host**（透過 PCIe）
- SNAP Controller **已經存在並工作**
- 問題是 **設備被 kernel 驅動占用**，DOCA 應用程式無法取得控制權
- 需要**解綁驅動**才能讓 DOCA 應用程式配置設備

### 步驟 13: 完整故障排除流程（2025-10-16）

經過系統性調查，我們嘗試了以下所有可能的解決方案：

#### 嘗試 1: 解綁 nvme 驅動

**操作**：
```bash
# 在 Host 端執行
echo "0000:08:00.0" > /sys/bus/pci/drivers/nvme/unbind
```

**驗證結果**：
```bash
$ lspci -s 08:00.0 -k
08:00.0 Non-Volatile memory controller: Mellanox Technologies NVMe SNAP Controller
# 確認：沒有 "Kernel driver in use" 這一行 ✅
```

**測試 Transport 創建**：
```bash
# 在 DPU 端執行
$ sudo python3 /tmp/spdk_rpc.py nvmf_create_transport '{"trtype":"DOCA"}'
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32603,
    "message": "Transport type 'DOCA' create failed"
  }
}
```

**結果**: ❌ **錯誤持續存在**

查看日誌仍然顯示相同錯誤：
```
BAD_RES_STATE_ERR (0x9) with syndrome=0x83d555
Failed to modify bar stateful region: DOCA_ERROR_DRIVER
```

#### 嘗試 2: 移除 PCIe 設備

**操作**：
```bash
# 在 Host 端執行
echo "1" > /sys/bus/pci/devices/0000:08:00.0/remove
echo "1" > /sys/bus/pci/devices/0000:07:00.2/remove

# 驗證移除
$ lspci | grep -i 'snap\|nvme' | grep -E '07:00.2|08:00.0'
# （無輸出，確認設備已移除）
```

**測試 Transport 創建**：
```bash
$ sudo python3 /tmp/spdk_rpc.py nvmf_create_transport '{"trtype":"DOCA"}'
```

**結果**: ❌ **錯誤持續存在**

移除 PCIe 設備後，DOCA Manager 仍然嘗試配置設備但失敗。

#### 嘗試 3: 重啟 DOCA 應用程式

**操作**：
```bash
# 在 DPU 端執行
$ sudo pkill -f doca_nvme_emulation
$ sudo rm -f /var/tmp/spdk.sock

# 重新啟動
$ sudo /opt/mellanox/doca/applications/build/nvme_emulation/doca_nvme_emulation \
    -m 0x3 --main-core 0 -s 1024 --no-pci --wait-for-rpc &

# 重新配置
$ sudo python3 /tmp/spdk_rpc.py framework_start_init
$ sudo python3 /tmp/spdk_rpc.py bdev_malloc_create '{"name":"Malloc0","num_blocks":1048576,"block_size":512}'
$ sudo python3 /tmp/spdk_rpc.py nvmf_create_subsystem '{"nqn":"nqn.2016-06.io.spdk:cnode1","allow_any_host":true,"serial_number":"SPDK00000000000001"}'
$ sudo python3 /tmp/spdk_rpc.py nvmf_subsystem_add_ns '{"nqn":"nqn.2016-06.io.spdk:cnode1","namespace":{"bdev_name":"Malloc0"}}'
$ sudo python3 /tmp/spdk_rpc.py nvmf_create_transport '{"trtype":"DOCA"}'
```

**結果**: ❌ **錯誤持續存在**

#### 嘗試 4: 重啟 Mellanox 驅動

**操作**：
```bash
# 在 DPU 端執行
$ sudo modprobe -r mlx5_core
$ sudo modprobe mlx5_core

# 驗證驅動重新載入
$ lsmod | grep mlx5
mlx5_core            1847296  1 mlx5_ib
```

**測試 Transport 創建**：重新執行配置流程

**結果**: ❌ **錯誤持續存在**

#### 嘗試 5: 完全重啟 DPU

這是最徹底的測試，驗證是否為暫態軟體問題。

**操作**：
```bash
# 在 DPU 端執行
$ sudo reboot

# 等待 DPU 重新啟動...
# 重新連接並執行完整配置流程
```

**結果**: ❌ **錯誤仍然持續存在**

**重要發現**：即使完全重啟 DPU，錯誤依然存在。這證明：
1. **不是暫態軟體問題**
2. **不是驅動狀態問題**
3. **是持久性的固件/硬體配置問題**

### 步驟 14: 固件配置分析

#### 檢查 DPU 固件配置

**查詢固件版本**：
```bash
$ sudo mstflint -d /dev/mst/mt41686_pciconf0 q | grep FW
FW Version:             32.43.3608
FW Version(Running):    32.43.3608
```

**查詢 SNAP 相關配置**：
```bash
$ sudo mlxconfig -d /dev/mst/mt41686_pciconf0 q | grep -i 'nvme\|snap\|emul'

Device #1:
-----------
...
NVME_EMULATION_ENABLE               True(1)
NVME_EMULATION_NUM_PF               1
NUM_OF_PF                           2         ⬅️ 潛在衝突！
SRIOV_EN                            True(1)
...
```

#### 關鍵發現：配置衝突

**配置不一致**：
- `NVME_EMULATION_NUM_PF`: 1 (NVMe Emulation 只配置 1 個 PF)
- `NUM_OF_PF`: 2 (系統總共有 2 個 Physical Functions)
- 實際觀察到：2 個 SNAP Controller (07:00.2 和 08:00.0)

**可能的衝突原因**：
1. 固件配置的 PF 數量與實際硬體不匹配
2. 有 PF 被分配給了其他功能（如 ConnectX-7 網路卡）
3. SNAP 資源可能在不同 PF 之間配置不一致

#### SR-IOV 配置

```bash
$ sudo mlxconfig -d /dev/mst/mt41686_pciconf0 q | grep -i sriov
SRIOV_EN                            True(1)
NUM_OF_VFS                          8
```

SR-IOV 已啟用，但這也增加了資源管理的複雜度。

### 步驟 15: 錯誤代碼深入調查

#### 網路搜尋研究

**搜尋關鍵字**: `BAD_RES_STATE_ERR 0x9 syndrome 0x83d555 DOCA NVMe SNAP`

**搜尋結果 1**: NVIDIA DOCA SNAP Known Issues
- 來源: NVIDIA BlueField-3 SNAP v4.4.0 Release Notes
- 找到 10+ 個已知 SNAP 問題，但**沒有一個匹配我們的特定錯誤**

已知問題包括：
- `bdev_uring` 不支援
- `nvme_controller_suspend` RPC 超時
- DPA provider 事件遺失
- virtio-blk 記憶體損壞問題
- 無法在超過 8 個核心上工作（NVMe/TCP）

**搜尋結果 2**: DOCA DevEmu PCI 文檔
- 來源: NVIDIA DOCA DevEmu Documentation
- 找到 `modify_bar_stateful_region` 函數說明：
  ```
  doca_error_t doca_devemu_pci_type_modify_bar_stateful_region_default_values(
      struct doca_devemu_pci_type *pci_type,
      enum doca_pci_type_region_id region_id,
      uint32_t size,
      uint64_t addr
  );
  ```
- 用途：修改 PCI BAR 的記憶體映射區域
- **要求**：設備必須處於正確的狀態（未初始化或已釋放）

**搜尋結果 3**: Mellanox PRM (Programmer's Reference Manual) 錯誤碼
- `BAD_RES_STATE_ERR (0x9)`: 資源處於錯誤狀態
  - 通常表示：操作需要資源處於特定狀態（如 INIT、ACTIVE、RELEASED）
  - 但資源當前處於不兼容的狀態
  - 固件拒絕執行操作以保護系統穩定性

**搜尋結果 4**: Syndrome 0x83d555 的特定意義
- **發現**: 這個特定的 syndrome 代碼**沒有公開文檔**
- Syndrome 是硬體特定的詳細錯誤代碼
- 通常只有 NVIDIA 內部工程師才有完整的 syndrome 代碼表
- 可能的原因：
  - BAR 區域已被配置且未釋放
  - PF/VF 資源分配衝突
  - 固件狀態機處於不允許此操作的狀態

#### 錯誤流程完整追蹤

```
用戶空間 (SPDK)
    ↓
    nvmf_create_transport("DOCA")
    ↓
DOCA Transport 層 (doca_transport.c)
    ↓
    nvmf_doca_create()
    └─> nvmf_doca_create_emulation_manager()
        └─> nvmf_doca_pci_type_create_and_start()
            ↓
DOCA DevEmu 層 (libdoca_devemu_pci.so)
            ↓
            doca_devemu_pci_type_modify_bar_stateful_region_default_values()
            └─> 構建 PRM 命令
                ↓
DEVX 適配器層 (linux_devx_adapter.cpp)
                ↓
                send_prm_cmd(0xb2d)  ⬅️ PRM 命令：配置 BAR
                ↓
固件層 (BlueField Firmware 32.43.3608)
                ↓
                檢查資源狀態...
                └─> ❌ 資源狀態不允許此操作
                    返回: BAD_RES_STATE_ERR (0x9)
                    Syndrome: 0x83d555
                    ↓
錯誤向上傳播
    ↓
SPDK RPC 返回錯誤: "Transport type 'DOCA' create failed"
```

### 步驟 16: 根本原因結論

經過完整的調查和測試，我們可以確定：

#### 確認的事實

1. ✅ **硬體連接正常**
   - DPU 正確連接到 Host
   - PCIe 總線正常工作
   - SNAP Controller 存在且可見

2. ✅ **軟體配置正確**
   - SPDK 正確初始化
   - DOCA 庫正確載入
   - RPC 通訊正常

3. ✅ **驅動問題已排除**
   - 解綁 kernel nvme 驅動：無效
   - 移除 PCIe 設備：無效
   - 重啟驅動：無效
   - 重啟系統：無效

4. ❌ **固件層面問題**
   - 錯誤來自固件 PRM 命令執行失敗
   - Syndrome 0x83d555 沒有公開文檔
   - 持久性錯誤，重啟無法解決

#### 可能的根本原因

**原因 1: 固件配置不一致**
```
NVME_EMULATION_NUM_PF = 1  (配置為 1 個 PF)
實際系統 = 2 個 SNAP Controller (07:00.2, 08:00.0)
```
固件可能在資源分配時產生衝突。

**原因 2: 資源預分配問題**
SNAP 資源可能在固件初始化時已經被預分配或鎖定，導致運行時無法重新配置 BAR 區域。

**原因 3: 固件版本兼容性問題**
固件版本 32.43.3608 可能與 DOCA 2.9.3008 的某些功能不完全兼容。

**原因 4: 硬體/環境限制**
特定的硬體配置或環境限制（如虛擬化、IOMMU 配置）可能導致固件拒絕某些操作。

#### 需要的下一步

**此問題需要 NVIDIA 技術支持**，因為：

1. **固件層錯誤** - 超出軟體配置範圍
2. **未公開的 Syndrome** - 需要內部文檔解讀
3. **持久性問題** - 非暫態錯誤，軟體重置無效
4. **複雜的資源管理** - 涉及固件、硬體、PF/VF 配置的交互

**建議提供給 NVIDIA 的信息**：
```
Hardware: BlueField-3 DPU
Firmware Version: 32.43.3608
DOCA SDK Version: 2.9.3008
Error: BAD_RES_STATE_ERR (0x9) with syndrome=0x83d555
PRM Command: 0xb2d (BAR configuration)
Configuration: NVME_EMULATION_ENABLE=True, NVME_EMULATION_NUM_PF=1, NUM_OF_PF=2
Failed Operation: doca_devemu_pci_type_modify_bar_stateful_region_default_values()
```

## 重要發現

### 成功完成的部分

1. ✅ **應用程式編譯** - 成功編譯生成可執行檔案
2. ✅ **環境配置** - Hugepages 配置和資源清理
3. ✅ **應用程式啟動** - 成功啟動並進入 RPC 等待狀態
4. ✅ **RPC 通訊** - 成功建立 JSON-RPC 客戶端
5. ✅ **Framework 初始化** - SPDK framework 成功初始化
6. ✅ **Block Device 創建** - 512MB malloc bdev 創建成功
7. ✅ **Subsystem 創建** - NVMe-oF subsystem 創建成功
8. ✅ **Namespace 添加** - Namespace 成功添加到 subsystem
9. ✅ **DOCA Manager 檢測** - 成功檢測到 mlx5_0 manager

### 未能完成的部分

❌ **DOCA Transport 創建** - 由於固件層面的持久性問題失敗

**詳細調查結果**：
- 嘗試了 5 種不同的解決方案（解綁驅動、移除設備、重啟應用、重啟驅動、重啟 DPU）
- 所有軟體層面的修復嘗試均無效
- 確認為固件 PRM 命令 0xb2d 執行失敗
- 錯誤 `BAD_RES_STATE_ERR (0x9) with syndrome=0x83d555` 持續存在
- 發現潛在的固件配置衝突：`NVME_EMULATION_NUM_PF=1` vs `NUM_OF_PF=2`
- 問題需要 NVIDIA 技術支持解決

### 關鍵學習點

#### 1. NVMe Emulation 的硬體依賴

DOCA NVMe Emulation **不是**純軟體模擬，它需要：

- **DPU 硬體支援**: BlueField DPU 的 SNAP 引擎
- **固件配置**: 正確的 DPU 固件設置
- **Host 連接**: 實際的 Host 機器通過 PCIe 連接到 DPU
- **硬體初始化**: 特定的硬體初始化流程

#### 2. SNAP 技術原理

**SNAP (Storage Network Acceleration Protocol)** 是 NVIDIA 的硬體加速技術：

- 在 DPU 上創建真實的 PCIe 裝置
- Host 端看到的是標準 NVMe Controller
- 所有 NVMe 命令由 DPU 處理
- 資料透過 DMA 直接傳輸，無需 CPU 介入

#### 3. 完整部署架構需求

```
┌────────────────┐                    ┌─────────────────┐
│   Host (x86)   │                    │ DPU (ARM64)     │
│                │                    │                 │
│  ┌──────────┐  │                    │  ┌───────────┐  │
│  │ OS Driver│  │  PCIe Connection   │  │SNAP Engine│  │
│  │ (nvme.ko)│  │◄──────────────────►│  │           │  │
│  └────┬─────┘  │                    │  └─────┬─────┘  │
│       │        │                    │        │        │
│  ┌────▼─────┐  │                    │  ┌─────▼──────┐ │
│  │/dev/nvme0│  │                    │  │SPDK Backend│ │
│  └──────────┘  │                    │  │ (Malloc0)  │ │
│                │                    │  └────────────┘ │
└────────────────┘                    └─────────────────┘

虛擬 NVMe 裝置                              後端儲存
```

**缺少的部分**: 在我們的測試中，只有 DPU 端在運行，沒有 Host 端連接。

#### 4. 錯誤訊息解讀能力

通過實際測試學會了：
- 如何讀取 SPDK 日誌
- 如何理解 DOCA 錯誤訊息
- 如何追蹤錯誤根源（從應用層到固件層）

## 測試環境限制

### 當前環境配置

1. ✅ **Host 機器存在**: DPU 透過 PCIe 正確連接到 Host
2. ✅ **SNAP Controller 可見**: 系統檢測到 07:00.2 和 08:00.0 兩個 SNAP Controller
3. ✅ **網路連接正常**: 透過 SSH (192.168.100.2) 連接到 DPU
4. ✅ **基礎軟體配置**: DOCA SDK、SPDK、驅動程式均正確安裝

### 發現的環境問題

1. ❌ **固件配置問題**
   - `NVME_EMULATION_NUM_PF=1` 與實際 2 個 SNAP Controller 不匹配
   - 固件版本 32.43.3608 可能存在兼容性問題
   - PF/VF 資源分配可能不一致

2. ❌ **固件狀態問題**
   - SNAP 資源在固件層被鎖定或處於錯誤狀態
   - BAR 配置無法在運行時修改
   - 持久性問題，無法通過軟體重置解決

### 解決環境問題所需

要完全解決此問題，需要：

1. **NVIDIA 技術支持**
   - 解讀 syndrome 0x83d555 的具體含義
   - 分析固件配置衝突
   - 提供正確的固件配置步驟或更新固件版本

2. **固件重新配置**（可能需要）
   - 統一 PF 數量配置
   - 正確分配 SNAP 資源
   - 確保所有固件參數一致

3. **硬體/固件組合驗證**
   - 確認此硬體版本與 DOCA 2.9.3008 的兼容性
   - 可能需要升級或降級固件版本

## 監測方法總結

### 1. 應用程式狀態監測

```bash
# 檢查程式是否運行
ps aux | grep doca_nvme_emulation

# 監控 CPU 和記憶體使用
top -p $(pgrep doca_nvme_emulation)

# 查看即時日誌
tail -f /tmp/nvme_emulation.log
```

### 2. Hugepages 監測

```bash
# 查看 hugepages 狀態
watch -n 1 'cat /proc/meminfo | grep HugePages'

# 查看詳細記憶體分配
cat /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
cat /sys/kernel/mm/hugepages/hugepages-2048kB/free_hugepages
```

### 3. RPC 狀態監測

```bash
# 檢查 socket 是否存在
ls -la /var/tmp/spdk.sock

# 測試 RPC 連接
sudo python3 /tmp/spdk_rpc.py spdk_get_version

# 查看所有 bdev
sudo python3 /tmp/spdk_rpc.py bdev_get_bdevs

# 查看所有 subsystem
sudo python3 /tmp/spdk_rpc.py nvmf_get_subsystems
```

### 4. 硬體狀態監測

```bash
# 查看 Mellanox 裝置
lspci | grep -i mellanox

# 查看 DOCA managers
sudo python3 /tmp/spdk_rpc.py nvmf_doca_get_managers

# 檢查 DPU 固件版本
mstflint -d /dev/mst/mt41686_pciconf0 q
```

## 下一步建議

### 立即需要做的

1. **聯繫 NVIDIA 技術支持** ⚠️ 最高優先級
   - 提供完整的錯誤信息和系統配置
   - 詢問 syndrome 0x83d555 的具體含義
   - 請求固件配置建議或固件更新
   - 提供本文檔作為詳細的調查報告

2. **文檔完善** （當前環境下可完成）
   - ✅ 記錄所有測試步驟和結果（本文檔）
   - ⏳ 分析關鍵源代碼（doca_transport.c, SNAP 初始化流程）
   - ✅ 創建詳細的故障排除記錄（步驟 13-16）

### 短期目標（等待 NVIDIA 支持期間）

1. **程式碼分析**
   - 深入研究 `doca_transport.c` 的 Transport 創建流程
   - 理解 SNAP 初始化和 PCI BAR 配置機制
   - 分析 DEVX 適配器和 PRM 命令流程
   - 記錄關鍵函數調用鏈和錯誤處理

2. **替代方案探索**
   - 測試其他 DOCA 應用程式（如 DMA Copy）驗證環境
   - 嘗試使用 RDMA Transport 替代 DOCA Transport（如適用）
   - 研究是否可以用其他方式實現類似功能

### 長期目標（解決固件問題後）

1. **完整 E2E 測試**
   - 成功創建 DOCA Transport
   - 在 Host 上驗證虛擬 NVMe 裝置可見性
   - 執行基礎 I/O 測試（dd, fio）
   - 驗證資料完整性和一致性

2. **性能測試**
   - 測量 IOPS、吞吐量、延遲
   - 與原生 NVMe 設備比較
   - 不同佇列深度和 I/O 大小的測試
   - CPU 使用率和系統開銷分析

3. **高級功能測試**
   - 多個 Namespace 配置
   - 不同後端儲存（NVMe-oF, Ceph, 檔案系統）
   - 動態熱插拔測試
   - 錯誤注入和恢復測試

## 結論

本次測試成功地：

1. **驗證了基礎環境設置** - Hugepages、資源管理、應用程式啟動
2. **建立了 RPC 通訊機制** - 創建了工作的 Python RPC 客戶端
3. **完成了大部分配置步驟** - Framework、Bdev、Subsystem、Namespace
4. **發現了關鍵限制** - DOCA Transport 需要特定的硬體環境

最重要的是，我們**通過實際測試發現了真實的需求和限制**，這比僅閱讀文檔更有價值。我們現在清楚地了解：

- ✅ 哪些步驟可以在當前環境完成
- ❌ 哪些步驟需要額外的硬體支援
- 📝 每個步驟的具體命令和預期輸出
- 🐛 可能遇到的錯誤和解決方法

這為後續在完整環境中進行測試打下了堅實的基礎。

## 附錄: 完整配置命令序列

```bash
# 1. 準備環境
echo 2048 | sudo tee /proc/sys/vm/nr_hugepages
sudo pkill -f doca_nvme_emulation
sudo rm -f /var/tmp/spdk.sock

# 2. 啟動應用程式
nohup sudo /opt/mellanox/doca/applications/build/nvme_emulation/doca_nvme_emulation \
    -m 0x3 --main-core 0 -s 1024 --no-pci --wait-for-rpc \
    > /tmp/nvme_emulation.log 2>&1 &

# 3. 等待啟動
sleep 3

# 4. 初始化 Framework
sudo python3 /tmp/spdk_rpc.py framework_start_init

# 5. 創建 Bdev
sudo python3 /tmp/spdk_rpc.py bdev_malloc_create '{"name":"Malloc0","num_blocks":1048576,"block_size":512}'

# 6. 創建 Subsystem
sudo python3 /tmp/spdk_rpc.py nvmf_create_subsystem '{"nqn":"nqn.2016-06.io.spdk:cnode1","allow_any_host":true,"serial_number":"SPDK00000000000001"}'

# 7. 添加 Namespace
sudo python3 /tmp/spdk_rpc.py nvmf_subsystem_add_ns '{"nqn":"nqn.2016-06.io.spdk:cnode1","namespace":{"bdev_name":"Malloc0"}}'

# 8. 檢查 DOCA Manager
sudo python3 /tmp/spdk_rpc.py nvmf_doca_get_managers

# 9. 嘗試創建 Transport (會失敗，但了解原因)
sudo python3 /tmp/spdk_rpc.py nvmf_create_transport '{"trtype":"DOCA"}'
```

## 參考資料

- [NVIDIA DOCA Documentation](https://docs.nvidia.com/doca/)
- [SPDK NVMe-oF Target Documentation](https://spdk.io/doc/nvmf.html)
- [NVMe Specification](https://nvmexpress.org/specifications/)
- [BlueField DPU Product Page](https://www.nvidia.com/en-us/networking/products/data-processing-unit/)

---

**文檔版本**: 1.0
**測試日期**: 2025-10-15
**測試人員**: Claude Code + Aiden
**狀態**: 部分完成，發現硬體依賴問題
