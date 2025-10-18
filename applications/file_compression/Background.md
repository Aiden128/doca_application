# 背景知識

## 壓縮計算本質

DEFLATE 壓縮需要在 32KB 滑動視窗中尋找重複模式，複雜度 O(n × 32KB)。

### 硬體 vs 軟體壓縮

| 特性 | 硬體 (Standard+) | 軟體 (E-Series) |
|------|-----------------|----------------|
| Throughput | ~10 GB/s | ~500 MB/s |
| 延遲 (1MB) | ~0.1 ms | ~2 ms |
| CPU 使用 | < 5% | 100% (單核) |

## DOCA Compress 三層架構

```
Application Layer (doca_file_compression)
    ↓
DOCA Compress Library (自動降級邏輯) ← 關鍵！
    ↓
Hardware Path (FPGA/ASIC) | Software Path (zlib)
```

## E-Series 軟體降級機制（程式碼佐證）

實際測試日誌：
```
[DOCA][WRN] compress_deflate is not supported by the device
[DOCA][INF] Failed to find device for compress task, running SW compress with zlib
```

這展示了 DOCA 的優雅降級：
1. 嘗試硬體壓縮 → 失敗
2. 自動切換 zlib → 成功
3. 應用層完全透明

**相關資源**: [DOCA Compress Library Documentation](https://docs.nvidia.com/doca/sdk/doca-compress/index.html)
