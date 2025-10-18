# 測試驗證

## 實際測試結果 (2025-10-18)

**參數**: 1024 bytes × 10 messages  
**環境**: BlueField-3, DOCA 3.1.0105

### 延遲統計

```
Producer sent 10 messages in 0.1372 ms → 每則 ~13.72 μs
Consumer received 10 messages in 0.0049 ms → 每則 ~0.49 μs
```

### 性能分析

- **發送**: 包含 task submission + 硬體處理
- **接收**: Buffer 預先 post，僅觸發 callback
- **微秒級延遲**: PCIe 直接通訊，無 TCP/IP 開銷

## 與其他應用對比

| 應用 | 方向 | 延遲 | 適合場景 |
|------|------|------|---------|
| Secure Channel | 雙向 | ~0.01 ms/訊息 | 控制訊息 |
| DMA Copy | 單向 | < 1 s (1MB) | 大檔案傳輸 |
