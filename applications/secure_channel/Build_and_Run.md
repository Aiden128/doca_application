# 編譯與執行

## 編譯

**DPU/Host**:
```bash
cd /opt/mellanox/doca/applications/build
meson .. && ninja -C . secure_channel/doca_secure_channel
```

## 執行

### Server (DPU)

```bash
sudo doca_secure_channel -p 03:00.0 -r 07:00.0 -s 1024 -n 10
```

### Client (Host)

```bash
sudo doca_secure_channel -p 07:00.0 -s 1024 -n 10
```

## 參數

- `-s`: 訊息大小 (bytes, 最大 65535)
- `-n`: 訊息數量
