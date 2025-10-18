# 編譯與執行

## 編譯步驟

**DPU**:
```bash
cd /opt/mellanox/doca/applications/build
meson .. && ninja -C . file_compression/doca_file_compression
```

**Host**:
```bash
cd /path/to/doca/applications/build
meson .. && ninja -C . file_compression/doca_file_compression
```

## 執行方法

### Server (DPU)

```bash
sudo /opt/mellanox/doca/applications/build/file_compression/doca_file_compression \
    -p 03:00.0 -r 07:00.0 -f /tmp/received.txt
```

### Client (Host)

```bash
sudo /path/to/doca_file_compression -p 07:00.0 -f /tmp/test_file.txt
```

## 參數說明

| 參數 | 說明 | 必須 |
|------|------|------|
| `-p` | PCI 設備地址 | ✓ |
| `-r` | Representor (僅 Server) | ✓ |
| `-f` | 檔案路徑 | ✓ |
