# DOCA File Compression

> **分類**: 🔄 資料傳輸應用
> **測試狀態**: ✅ **成功（軟體降級）** - E-Series 自動降級到 zlib 軟體壓縮
> **DOCA 組件**: Comch · DMA · Compress · Buffer

**測試摘要**: 730 bytes 檔案傳輸成功，MD5 驗證通過 (`025d58144a49056290923b3bba355891`)

---

## 📚 文檔導航

### 核心文檔

1. **[背景知識](Background.md)** - 壓縮原理、DOCA Compress 架構、程式碼分析
2. **[環境準備](Setup.md)** - 系統需求、Hugepages 配置、mlnx_snap 修復
3. **[編譯與執行](Build_and_Run.md)** - 編譯步驟、Server/Client 啟動、參數說明
4. **[測試驗證](Testing.md)** - 實際測試結果、E-Series 軟體降級行為分析
5. **[故障排除](Troubleshooting.md)** - 常見問題、診斷腳本、性能優化

### 相關資源

- [DMA Copy](../dma_copy/README.md) - Comch 基礎應用（建議先閱讀）
- [Secure Channel](../secure_channel/README.md) - 雙向通訊應用
- [Project README](../../README.md) - 專案總覽

---

## 🚀 快速開始

### 一行測試命令

**DPU Server**:
```bash
sudo /opt/mellanox/doca/applications/build/file_compression/doca_file_compression -p 03:00.0 -r 07:00.0 -f /tmp/received.txt
```

**Host Client**:
```bash
sudo /path/to/doca_file_compression -p 07:00.0 -f /tmp/test_file.txt
```

### 預期結果

```
[DOCA][INF] Server connection established
[DOCA][WRN] compress_deflate is not supported by the device  ← E-Series 正常警告
[DOCA][INF] Failed to find device for compress task, running SW compress with zlib
[DOCA][INF] SUCCESS: file was received and decompressed successfully
```

✅ 軟體降級是**正常行為**，不是錯誤！

---

## 📊 測試狀態

| 項目 | 狀態 | 詳情 |
|------|------|------|
| **編譯** | ✅ 成功 | 154KB executable |
| **執行** | ✅ 成功 | 自動降級到 zlib |
| **E2E 測試** | ✅ 成功 | 730 bytes 傳輸 |
| **MD5 驗證** | ✅ 通過 | `025d58144a49056290923b3bba355891` |
| **硬體壓縮** | ⚠️ E-Series 不支援 | 軟體壓縮可用 |

---

## 🔑 關鍵發現

1. **E-Series 無硬體壓縮**:
   - BlueField-3 E-Series 沒有壓縮引擎
   - DOCA 自動降級到 zlib 軟體壓縮
   - 功能完全正常，僅性能較低

2. **優雅降級機制**:
   - 應用層程式碼完全不需修改
   - 自動偵測硬體能力並選擇實作
   - 這是良好的軟體設計範例

3. **性能特徵**:
   - 硬體壓縮: ~10 GB/s
   - 軟體壓縮: ~500 MB/s
   - 小檔案場景差異不明顯

詳細分析請見 [Testing.md](Testing.md)

---

**文檔版本**: 1.0  
**最後更新**: 2025-10-18  
**測試環境**: BlueField-3 B3210E E-Series, DOCA 3.1.0105  
**維護者**: DOCA Application Documenter Agent
