# 故障排除

## 常見問題

### 1. Server 無法啟動

**症狀**: `Failed to create DOCA device representor`  
**解決**: 檢查 mlnx_snap 服務和 Host PF 啟用狀態

### 2. 壓縮失敗

**症狀**: `Failed to submit compress task`  
**解決**: 檢查 hugepages 配置 (需 >= 2GB)

### 3. Client 連接超時

**症狀**: `Failed to connect to server`  
**解決**: 確認 DPU Server 正在運行，檢查 pf0hpf UP 狀態

## 診斷腳本

```bash
sudo /path/to/doca_env_check.sh dpu   # DPU 端
sudo /path/to/doca_env_check.sh host  # Host 端
```

## 性能優化

### E-Series 環境建議

- 小檔案批次處理
- 調整壓縮等級 (速度 vs 壓縮率)
- 大檔案分塊傳輸

### 升級路徑

需要硬體壓縮？升級到 BlueField-3 **B3210 (Standard)** 或 **B3210C (Crypto)**
