# 故障排除

請參考 [DMA Copy Troubleshooting](../dma_copy/README.md#故障排除)，相同的 Comch 基礎設施。

## 訊息限制

```c
#define MAX_MSG_SIZE 65535      // 最大 64KB
#define MAX_FASTPATH_TASKS 1024 // 最大並發
```

## 性能調整

```bash
# 小訊息高頻 (控制訊息)
sudo doca_secure_channel -p ... -s 256 -n 1000

# 大訊息低頻 (資料傳輸)
sudo doca_secure_channel -p ... -s 65535 -n 10
```

## 程式碼閱讀建議

**源碼位置**: `/opt/mellanox/doca/applications/secure_channel/`

**建議閱讀順序**:
1. `run_producer()` (行 397-592)
2. `run_consumer()` (行 673-859)
3. `send_task_completed_callback()` (行 330-365)
