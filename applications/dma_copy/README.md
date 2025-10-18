# DOCA DMA Copy 應用程式

> **分類**: 🔄 [資料傳輸應用](../../README.md#資料傳輸應用)
> **相關應用**: [File Integrity](../file_integrity/README.md) · File Compression
> **DOCA 組件**: Comch · DMA · Buffer

## 📖 概述

DOCA DMA Copy 示範如何使用 NVIDIA BlueField DPU 的 DMA (Direct Memory Access) 引擎，在 Host 和 DPU 記憶體之間進行高速資料傳輸，無需 CPU 干預。

## 🎯 學習目標

完成本指南後，你將了解：
- DMA 的基本原理和優勢
- DOCA Comch（通訊通道）的使用
- Host-DPU 間的記憶體映射和資料傳輸
- PCIe BAR 配置和設備識別

## 📚 文檔結構

本目錄包含以下文檔：

### 📋 [測試記錄 (Testing_Results.md)](./Testing_Results.md)
**2025-10-18 實際測試完整記錄** - 必讀！

包含編譯驗證、PCI 設備識別、測試檔案準備的完整過程，以及發現的**關鍵硬體限制**：
- ❌ **Host PF (Physical Function) 必須啟用** 才能運行 DMA Copy
- 與 NVMe Emulation 類似的 firmware 層級問題
- 無法透過軟體解決，需要 firmware 配置或 NVIDIA 技術支援

### ⭐ [完整指南 (Complete_Guide.md)](./Complete_Guide.md)
一份詳盡的端到端指南，涵蓋所有內容：

#### 📌 主要章節

1. **[背景知識](./Complete_Guide.md#背景知識)**
   - DMA 原理介紹
   - 為什麼 DPU 需要 DMA Copy
   - DOCA DMA Copy 的工作原理
   - 核心技術元件

2. **[系統架構](./Complete_Guide.md#系統架構)**
   - 硬體拓撲圖
   - 軟體堆疊說明
   - Host-DPU 通訊流程

3. **[環境要求](./Complete_Guide.md#環境要求)**
   - 硬體需求
   - 軟體需求
   - DOCA 版本檢查

4. **[配置步驟](./Complete_Guide.md#配置步驟)**
   - 識別 PCI 設備地址
   - 編譯應用程式
   - 準備測試文件
   - 配置 IOMMU 和 Huge Pages

5. **[執行測試](./Complete_Guide.md#執行測試)**
   - 在 DPU 上啟動 Server
   - 在 Host 上啟動 Client
   - 驗證傳輸結果

6. **[監測方法](./Complete_Guide.md#監測方法)**
   - DOCA 日誌級別
   - PCIe 頻寬監控
   - 性能指標收集

7. **[故障排除](./Complete_Guide.md#故障排除)**
   - 常見問題與解決方案
   - 除錯技巧
   - 系統診斷

8. **[性能優化](./Complete_Guide.md#性能優化)**
   - DMA Buffer 大小調整
   - Huge Pages 配置
   - PCIe 配置優化
   - CPU 親和性設置

## 🚀 快速開始

### 最小化測試步驟

如果你已經熟悉 DOCA 和 DMA 原理，可以跳轉到這些快速步驟：

1. **識別 PCI 地址** → [配置步驟 - 步驟 1](./Complete_Guide.md#步驟-1-識別-pci-設備地址)
2. **編譯應用程式** → [配置步驟 - 步驟 2](./Complete_Guide.md#步驟-2-編譯應用程式)
3. **執行測試** → [執行測試](./Complete_Guide.md#執行測試)

### 關鍵 PCI 地址 (根據你的環境調整)

```
Host 端:  07:00.3 (BlueField-3 SoC Management Interface)
DPU 端:   03:00.0 (ConnectX-7 Ethernet Controller)
Representor: 07:00.3
```

### 一鍵啟動命令

**DPU 端** (先啟動):
```bash
/opt/mellanox/doca/applications/build/dma_copy/doca_dma_copy \
    -p 03:00.0 \
    -r 07:00.3 \
    -f received_file.txt
```

**Host 端**:
```bash
/path/to/doca_dma_copy \
    -p 07:00.3 \
    -f test_input.txt
```

## 📊 測試狀態

| 項目 | 狀態 | 備註 |
|------|------|------|
| 文檔完整性 | ✅ 完成 | 涵蓋從背景到優化的所有主題 |
| 編譯驗證 | ✅ 完成 | Host (121KB) 和 DPU (119KB) 都成功編譯 |
| PCI 設備識別 | ✅ 完成 | Host: 07:00.0, DPU: 03:00.0 |
| 測試檔案準備 | ✅ 完成 | 1MB 隨機資料，MD5: 1a5c75c6d2ed860b72c9a7496a3d1595 |
| mlnx_snap 服務修復 | ✅ **完成** | 修改 hugepages 檢查腳本 |
| Host PF 啟用 | ✅ **完成** | 啟用 PCI 設備並綁定驅動 |
| Server 啟動測試 | ✅ **成功** | DPU Server 正常運行 |
| 實際傳輸測試 | ✅ **成功** | Host → DPU 傳輸完成 |
| 檔案完整性驗證 | ✅ **通過** | MD5 checksum 完全匹配 |

### 🎉 測試成功！

**DMA Copy 已成功運行並完成端到端測試！**

**測試成果** (2025-10-18):
- ✅ 成功傳輸 1MB 檔案 (Host → DPU)
- ✅ 檔案完整性驗證通過 (MD5: `1a5c75c6d2ed860b72c9a7496a3d1595`)
- ✅ DMA 傳輸確認成功："Final status message was successfully received"

**關鍵修復步驟**:
1. ✅ 修復 mlnx_snap.service (DOCA 3.1 hugepages 問題)
2. ✅ 啟用 Host PF PCI 設備 (enable=1, BusMaster+)
3. ✅ 綁定 mlx5_core 驅動
4. ✅ 正確配置 PCI 地址 (Server 使用 `-r 07:00.0`)

**重要發現**:
- mlnx_snap 服務對 DMA Copy 至關重要
- Host PF 必須完全啟用（enable + BusMaster + driver）
- `-r` 參數應使用 `07:00.0` (Ethernet PF)，而非 `07:00.3` (Management Interface)

詳細測試記錄和修復步驟請見 [Testing_Results.md](./Testing_Results.md)

## 🔗 相關資源

### 官方文檔
- [DOCA DMA Copy Application Guide](https://docs.nvidia.com/doca/sdk/doca+dma+copy+application+guide/index.html)
- [DOCA DMA Programming Guide](https://docs.nvidia.com/doca/sdk/doca+dma/index.html)
- [DOCA Comch API Reference](https://docs.nvidia.com/doca/sdk/doca+comch/index.html)

### 相關應用
- **Storage Zero Copy**: 使用類似的 Comch 和 RDMA 技術
- **File Integrity**: 在 DMA 傳輸基礎上加入 SHA 驗證
- **File Compression**: 結合 DMA 和硬體壓縮

## 💡 學習建議

### 適合初學者
如果你是第一次接觸 DOCA DMA：
1. 先閱讀 [背景知識](./Complete_Guide.md#背景知識) 章節
2. 理解 [系統架構](./Complete_Guide.md#系統架構)
3. 按照 [配置步驟](./Complete_Guide.md#配置步驟) 逐步操作

### 適合進階用戶
如果你已經了解 DMA 原理：
1. 直接查看 [執行測試](./Complete_Guide.md#執行測試) 章節
2. 參考 [性能優化](./Complete_Guide.md#性能優化) 進行調優
3. 修改源碼實現自定義功能

## 🤝 貢獻

發現問題或有改進建議？
- 更新文檔
- 提交 Pull Request
- 分享你的測試結果

---

**文檔版本**: 1.0
**最後更新**: 2025-10-18
**測試環境**: BlueField-3 DPU, DOCA 2.9.3008
