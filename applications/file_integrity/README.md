# DOCA File Integrity 應用程式

> **分類**: 🔄 [資料傳輸應用](../../README.md#資料傳輸應用)
> **相關應用**: [DMA Copy](../dma_copy/README.md) · File Compression
> **DOCA 組件**: Comch · SHA · Buffer

## 📖 概述

DOCA File Integrity 展示如何使用 NVIDIA BlueField DPU 的硬體加速 SHA 引擎，在 Host 和 DPU 之間安全地傳輸檔案並驗證完整性。應用程式結合了 **DOCA Comch**（通訊通道）和 **DOCA SHA**（安全哈希演算法）庫，確保傳輸的檔案未被篡改。

## 🎯 學習目標

完成本指南後，你將了解：
- 如何使用 DOCA SHA 庫進行硬體加速的哈希計算
- DOCA Comch 在 Host-DPU 通訊中的應用
- Client-Server 架構下的安全檔案傳輸
- 檔案完整性驗證的最佳實踐
- 如何檢測傳輸過程中的資料損壞或篡改

## 📚 文檔結構

本目錄包含以下文檔：

### ⭐ [完整指南 (Complete_Guide.md)](./Complete_Guide.md)
詳盡的技術指南，涵蓋所有內容：

#### 📌 主要章節

1. **[背景知識](./Complete_Guide.md#背景知識)**
   - 什麼是檔案完整性驗證
   - SHA (Secure Hash Algorithm) 原理
   - 為什麼需要硬體加速的 SHA
   - DOCA File Integrity 的工作流程

2. **[系統架構](./Complete_Guide.md#系統架構)**
   - Client-Server 架構
   - DOCA 組件交互
   - 資料流程圖
   - SHA 計算流程

3. **[環境要求](./Complete_Guide.md#環境要求)**
   - 硬體需求
   - 軟體需求
   - DOCA SHA 支援檢查

4. **[編譯步驟](./Complete_Guide.md#編譯步驟)**
   - 相依性檢查
   - Meson 配置
   - 編譯指令

5. **[執行測試](./Complete_Guide.md#執行測試)**
   - Server 端設置
   - Client 端設置
   - 完整性驗證

6. **[監測方法](./Complete_Guide.md#監測方法)**
   - SHA 計算監控
   - 傳輸進度追蹤
   - 完整性驗證結果

7. **[故障排除](./Complete_Guide.md#故障排除)**
   - 常見問題
   - SHA 計算錯誤
   - 完整性驗證失敗

8. **[進階主題](./Complete_Guide.md#進階主題)**
   - 部分 SHA 計算
   - 大檔案處理
   - 性能優化

## 🚀 快速開始

### 核心概念

**File Integrity 工作原理**：

```
Client 端:                    Server 端:
┌─────────────┐              ┌─────────────┐
│ 讀取檔案    │              │ 等待連接    │
└─────┬───────┘              └─────┬───────┘
      │                            │
┌─────▼───────┐              ┌─────▼───────┐
│ 計算 SHA    │              │ 接收檔案    │
│ (硬體加速)  │              │ + SHA digest│
└─────┬───────┘              └─────┬───────┘
      │                            │
┌─────▼───────┐              ┌─────▼───────┐
│ 發送檔案    │──────────►   │ 重新計算SHA │
│ + SHA       │              │ (硬體加速)  │
└─────────────┘              └─────┬───────┘
                                   │
                             ┌─────▼───────┐
                             │ 比對 SHA    │
                             │ ✓ 相同 = OK │
                             │ ✗ 不同 = 篡改│
                             └─────────────┘
```

### 關鍵 PCI 地址

```
Host 端:  07:00.3 (BlueField-3 SoC Management Interface)
DPU 端:   03:00.0 (ConnectX-7 Ethernet Controller)
Representor: 07:00.3
```

### 快速測試命令

**Server 端 (DPU)** - 先啟動:
```bash
./doca_file_integrity \
    -p 03:00.0 \
    -r 07:00.3 \
    -f received_file.txt
```

**Client 端 (Host)**:
```bash
./doca_file_integrity \
    -p 07:00.3 \
    -f test_file.txt
```

### 參數說明

| 參數 | 說明 | 必須 |
|------|------|------|
| `-p, --pci-addr` | DOCA Comch 設備 PCI 地址 | ✓ |
| `-r, --rep-pci` | Representor PCI 地址 (僅 Server) | Server 端必須 |
| `-f, --file` | 要發送/接收的檔案路徑 | ✓ |
| `-t, --timeout` | 超時時間 (秒)，預設 5 | ✗ |

## 📊 測試狀態

| 項目 | 狀態 | 備註 |
|------|------|------|
| 文檔完整性 | ✅ 完成 | README + Complete Guide |
| 編譯驗證 | ✅ 完成 | Host 端成功編譯 (121KB) |
| PCI 設備識別 | ✅ 完成 | Host: 07:00.3, DPU: 03:00.0 |
| 實際傳輸測試 | ⏳ 待完成 | 需要 Host-DPU 連接 |
| SHA 驗證測試 | ⏳ 待完成 | 需完成傳輸後驗證 |
| 篡改檢測測試 | ⏳ 待完成 | 測試修改檔案後的檢測 |

## 🔗 相關資源

### 官方文檔
- [DOCA File Integrity Application Guide](https://docs.nvidia.com/doca/sdk/nvidia+doca+file+integrity+application+guide/)
- [DOCA SHA Programming Guide](https://docs.nvidia.com/doca/sdk/doca+sha/)
- [DOCA Comch API Reference](https://docs.nvidia.com/doca/sdk/doca+comch/)

### 相關應用
- **DMA Copy**: File Integrity 基於類似的 Comch 通訊機制
- **File Compression**: 同樣使用 Comch 傳輸，但加入硬體壓縮
- **Storage Zero Copy**: 展示 RDMA 零拷貝傳輸

### 相關技術
- **SHA-256**: 加密哈希演算法標準
- **Hardware Acceleration**: 利用 DPU 硬體加速計算
- **Message Authentication Code (MAC)**: 訊息認證碼

## 💡 使用場景

### 適合的使用案例

1. **安全檔案傳輸**
   - 確保檔案在傳輸過程中未被修改
   - 適用於敏感資料傳輸

2. **資料同步驗證**
   - 驗證備份檔案的完整性
   - 檢測儲存媒體損壞

3. **合規性要求**
   - 符合資料完整性規範（如 HIPAA, PCI-DSS）
   - 可審計的檔案傳輸記錄

### 進階應用

- 結合加密傳輸（TLS）實現機密性 + 完整性
- 整合到企業檔案傳輸系統
- 建立自動化的完整性檢查流程

## 🔍 與 DMA Copy 的差異

| 特性 | DMA Copy | File Integrity |
|------|----------|----------------|
| 主要目的 | 高速資料傳輸 | 安全檔案傳輸 |
| 完整性檢查 | ✗ 無 | ✓ SHA 驗證 |
| 硬體加速 | DMA 引擎 | DMA + SHA 引擎 |
| 性能 | 更快 | 稍慢（需計算 SHA）|
| 安全性 | 基本 | 高（可檢測篡改）|
| 適用場景 | 內部資料移動 | 跨信任邊界傳輸 |

## 📝 學習路徑

### 初學者路徑
1. 先理解 [DMA Copy](../dma_copy/README.md) 的基本概念
2. 閱讀 [背景知識](./Complete_Guide.md#背景知識) 了解 SHA
3. 按照 [執行測試](./Complete_Guide.md#執行測試) 進行實作

### 進階路徑
1. 研究 DOCA SHA API 的進階功能
2. 實作部分 SHA 計算（適合大檔案）
3. 整合到現有的檔案傳輸系統

## 🤝 貢獻

發現問題或有改進建議？
- 更新文檔
- 分享測試結果
- 提交優化方案

---

**文檔版本**: 1.0
**最後更新**: 2025-10-18
**測試環境**: BlueField-3 DPU, DOCA 2.9.3008
**編譯狀態**: ✅ 成功
