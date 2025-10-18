# DOCA Secure Channel

> **分類**: 🔄 資料傳輸應用  
> **測試狀態**: ✅ **完全成功** - 雙向訊息傳輸，微秒級延遲  
> **DOCA 組件**: Comch · Producer · Consumer

**測試摘要**: 10 則訊息雙向傳輸，總延遲 < 0.14 ms，展示 Host-DPU 控制平面通訊能力。

---

## 📚 文檔導航

1. **[背景知識](Background.md)** - Producer-Consumer 模型、程式碼分析、官方定義
2. **[環境準備](Setup.md)** - 系統需求、環境檢查
3. **[編譯與執行](Build_and_Run.md)** - 編譯步驟、雙向通訊啟動
4. **[測試驗證](Testing.md)** - 實際測試結果、延遲分析
5. **[故障排除](Troubleshooting.md)** - 常見問題、性能優化

### 相關資源

- [DMA Copy](../dma_copy/README.md) - Comch 基礎
- [File Compression](../file_compression/README.md) - 壓縮卸載
- [Project README](../../README.md) - 專案總覽

---

## 🚀 快速開始

**DPU Server**:
```bash
sudo /opt/mellanox/doca/applications/build/secure_channel/doca_secure_channel \
    -p 03:00.0 -r 07:00.0 -s 1024 -n 10
```

**Host Client**:
```bash
sudo /path/to/doca_secure_channel -p 07:00.0 -s 1024 -n 10
```

**預期結果**:
```
[DOCA][INF] Producer sent 10 messages in approximately 0.1372 milliseconds
[DOCA][INF] Consumer received 10 messages in approximately 0.0049 milliseconds
```

---

## 📊 測試狀態

| 項目 | 結果 |
|------|------|
| **雙向通訊** | ✅ 成功 |
| **發送延遲** | 0.1372 ms (10 訊息) |
| **接收延遲** | 0.0049 ms (10 訊息) |
| **Crypto 需求** | ❌ 不需要 |

---

**文檔版本**: 1.0  
**最後更新**: 2025-10-18  
**維護者**: DOCA Application Documenter Agent
