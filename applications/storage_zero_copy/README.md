# DOCA Storage Zero Copy 完整指南

## 概述

DOCA Storage Zero Copy 是一個展示如何使用 DOCA Comch、DOCA Core 和 DOCA RDMA 函式庫來實現零拷貝儲存解決方案的參考應用程式集。這個應用程式集由三個獨立但協同工作的程式組成，共同實現了一個高性能的儲存系統，能夠在不進行不必要的資料拷貝的情況下高效地儲存和檢索資料。

## 為什麼需要零拷貝儲存？

在傳統的儲存架構中，資料在從發起端（Initiator）傳輸到儲存目標（Target）的過程中，通常需要經過多次拷貝：

1. **應用程式到核心緩衝區**：資料從用戶空間拷貝到核心空間
2. **核心緩衝區到網路卡**：資料從核心記憶體拷貝到網路卡的 DMA 緩衝區
3. **接收端網路卡到核心**：在目標端，資料從網路卡拷貝回核心
4. **核心到儲存裝置**：最後拷貝到實際的儲存裝置

每次拷貝都會：
- 消耗 CPU 週期
- 增加延遲
- 限制吞吐量
- 增加記憶體頻寬的壓力

DOCA Storage Zero Copy 透過使用 **RDMA（Remote Direct Memory Access）** 技術來消除這些中間拷貝。RDMA 允許一台機器直接讀寫另一台機器的記憶體，而不需要任何一方的 CPU 介入。結合 DPU 的硬體加速能力，整個資料傳輸路徑可以完全在硬體層面完成，實現真正的零拷貝。

## 三元架構

Storage Zero Copy 採用三元架構設計，由以下三個應用程式組成：

### 1. Initiator Comch（發起者）

**角色**：作為儲存系統的客戶端，發起讀寫請求。

**使用的技術**：
- **DOCA Comch**：Communication Channel，用於與 DPU 上的中介層通訊
- **DOCA Core**：基礎記憶體管理和裝置操作
- **DOCA RDMA**：為資料傳輸建立 RDMA 連接

**主要功能**：
- 配置資料緩衝區（透過 DOCA mmap 管理）
- 發送讀寫請求到中介層
- 提供資料給 Target 直接透過 RDMA 讀取（寫入操作）
- 允許 Target 直接透過 RDMA 寫入資料（讀取操作）
- 執行效能基準測試

**運行位置**：Host 端（x86_64）

### 2. Comch to RDMA（中介層）

**角色**：作為 Host 和 DPU 之間的橋樑，將 Comch 訊息轉換為 RDMA 操作。

**使用的技術**：
- **DOCA Comch Server**：接收來自 Initiator 的控制訊息
- **DOCA RDMA**：建立和管理多個 RDMA 連接
- **訊息路由邏輯**：將請求轉發到正確的目標

**主要功能**：
- 接受來自 Initiator 的 Comch 連接
- 為 Initiator 和 Target 之間創建 RDMA 連接（包括 control 和 data 連接）
- 轉發控制訊息（配置、啟動、停止等）
- 建立 I/O 訊息的傳遞路徑
- 管理連接生命週期

**運行位置**：DPU 端（ARM64）

### 3. Target RDMA（儲存目標）

**角色**：模擬儲存裝置，實際執行資料的讀寫操作。

**使用的技術**：
- **DOCA RDMA**：執行遠端記憶體讀寫
- **記憶體塊作為儲存後端**：使用 malloc 分配的記憶體模擬磁碟

**主要功能**：
- 接收來自中介層的 RDMA 連接
- 處理讀寫請求
- 使用 RDMA Read 從 Initiator 記憶體讀取要寫入的資料
- 使用 RDMA Write 將讀取的資料寫入 Initiator 記憶體
- 發送操作結果回 Initiator

**運行位置**：DPU 端（ARM64）或獨立的伺服器

## 資料流程圖

```
┌─────────────────┐
│   Initiator     │  (Host x86_64)
│   (Comch)       │
└────────┬────────┘
         │ Comch 控制訊息
         │ ┌────────────────────────────┐
         ↓ │                            │
┌─────────────────┐              RDMA 資料通道
│ Comch to RDMA   │  (DPU ARM64)       │
│   (中介層)       │                    │
└────────┬────────┘                    │
         │ RDMA 控制訊息               │
         │                            │
         ↓                            ↓
┌─────────────────┐              ┌────────┐
│  Target RDMA    │  (DPU ARM64) │ Data   │
│  (儲存目標)      │◄────────────►│ Memory │
└─────────────────┘              └────────┘
```

### 完整的讀取操作流程

1. **Initiator** 發送讀取請求給 **Comch to RDMA**（透過 Comch）
   - 請求包含：要讀取的地址、大小、相關 ID
2. **Comch to RDMA** 將請求轉發給 **Target RDMA**（透過 RDMA control 連接）
3. **Target RDMA** 從模擬儲存讀取資料
4. **Target RDMA** 使用 **RDMA Write** 直接將資料寫入 **Initiator** 的記憶體緩衝區
5. **Target RDMA** 發送完成訊息給 **Initiator**（透過 RDMA control 連接）
6. **Initiator** 收到通知，資料已在其記憶體中就緒

關鍵點：資料傳輸（步驟 4）完全繞過 Comch to RDMA，直接在 Target 和 Initiator 之間進行，實現真正的零拷貝。

### 完整的寫入操作流程

1. **Initiator** 準備要寫入的資料在其記憶體緩衝區中
2. **Initiator** 發送寫入請求給 **Comch to RDMA**（透過 Comch）
   - 請求包含：資料在 Initiator 記憶體中的地址、大小
3. **Comch to RDMA** 將請求轉發給 **Target RDMA**
4. **Target RDMA** 使用 **RDMA Read** 直接從 **Initiator** 的記憶體讀取資料
5. **Target RDMA** 將資料寫入模擬儲存
6. **Target RDMA** 發送完成訊息給 **Initiator**

資料傳輸（步驟 4）同樣完全繞過中介層，實現零拷貝。

## 文檔結構

本文檔包含以下章節，詳細介紹 Storage Zero Copy 的各個方面：

### [完整技術指南](Complete_Guide.md)

包含以下內容：

1. **背景知識**
   - RDMA 基礎概念
   - DOCA Comch 通訊機制
   - 記憶體映射和 mmap export
   - 零拷貝的技術原理

2. **系統架構詳解**
   - 三個應用程式的內部實現
   - 控制訊息和 I/O 訊息的格式
   - RDMA 連接的建立過程
   - 資料路徑的優化設計

3. **編譯與建置**
   - 環境準備
   - Meson 編譯配置
   - 編譯步驟
   - 常見編譯問題

4. **配置與執行**
   - 命令列參數詳解
   - 啟動順序和依賴關係
   - PCI 地址和裝置識別
   - 網路配置（如果使用獨立伺服器）

5. **實際測試**
   - 讀取操作測試
   - 寫入操作測試
   - 效能基準測試
   - 資料驗證

6. **監測與除錯**
   - 日誌級別配置
   - RDMA 連接狀態監測
   - Comch 訊息追蹤
   - 效能指標收集

7. **常見問題與故障排除**
   - Comch 連接失敗
   - RDMA 連接建立失敗
   - 資料傳輸錯誤
   - 性能問題診斷

8. **進階主題**
   - 與真實儲存後端整合
   - 多客戶端支援
   - 高可用性架構
   - 性能調優技巧

## 快速開始

如果您想快速上手，建議按照以下步驟：

### 最小化測試流程

1. **編譯所有三個應用程式**（在 DPU 上）
   ```bash
   cd /opt/mellanox/doca/applications
   mkdir -p build && cd build
   meson setup .. -Denable_storage_zero_copy=true
   ninja
   ```

2. **啟動 Target**（在 DPU 上）
   ```bash
   sudo ./storage/zero_copy/doca_storage_zero_copy_target_rdma -d <device_id>
   ```

3. **啟動 Comch to RDMA**（在 DPU 上）
   ```bash
   sudo ./storage/zero_copy/doca_storage_zero_copy_comch_to_rdma -d <device_id>
   ```

4. **啟動 Initiator**（在 Host 上）
   ```bash
   sudo ./doca_storage_zero_copy_initiator_comch -d <device_id> --operation write --run-limit-operation-count 1000
   ```

詳細的參數說明和配置方法請參閱 [完整技術指南](Complete_Guide.md)。

## 系統需求

### 硬體需求
- **DPU**: NVIDIA BlueField-2 或 BlueField-3
- **Host**: x86_64 架構伺服器，透過 PCIe 連接到 DPU
- **網路**: （可選）如果 Target 運行在獨立伺服器上

### 軟體需求
- **作業系統**:
  - DPU: Ubuntu 22.04 LTS (ARM64)
  - Host: Ubuntu 22.04 LTS (x86_64)
- **DOCA SDK**: 2.5.0 或更新版本（本文檔基於 2.9.3008 測試）
- **編譯器**: GCC 11+ 或 Clang 14+（支援 C++17）

## 關鍵特性

- **零拷貝設計**：資料直接在 Initiator 和 Target 之間透過 RDMA 傳輸，沒有中間拷貝
- **硬體加速**：利用 DPU 的 RDMA 引擎進行高速資料傳輸
- **低延遲**：典型的讀寫延遲在 10-20 微秒範圍內
- **高吞吐量**：可以達到接近網路卡的理論頻寬
- **模組化設計**：三個獨立的應用程式可以靈活部署
- **可擴展性**：容易擴展以支援多個 Initiator 或 Target

## 應用場景

- **高性能運算（HPC）**：科學計算中的大規模資料存取
- **資料庫加速**：資料庫的遠端記憶體擴展
- **分散式儲存**：作為分散式儲存系統的資料傳輸層
- **虛擬化儲存**：為虛擬機器提供高性能的遠端儲存
- **開發和測試**：RDMA 應用程式的開發和效能測試平台

## 技術支援

- **NVIDIA DOCA 官方文檔**: https://docs.nvidia.com/doca/
- **DOCA Storage Zero Copy 文檔**: https://docs.nvidia.com/doca/sdk/nvidia+doca+storage+zero+copy/
- **BlueField DPU 產品頁面**: https://www.nvidia.com/en-us/networking/products/data-processing-unit/
- **NVIDIA 開發者論壇**: https://forums.developer.nvidia.com/c/data-center/doca/

## 版本資訊

- **文檔版本**: 1.0
- **測試環境**: BlueField-3 DPU + DOCA 2.9.3008
- **最後更新**: 2025-10-15

---

**下一步**：建議閱讀 [完整技術指南](Complete_Guide.md) 以深入了解 Storage Zero Copy 的技術細節和使用方法。
