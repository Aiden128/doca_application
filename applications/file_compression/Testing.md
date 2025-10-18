# 測試驗證

## 實際測試結果 (2025-10-18)

**環境**: BlueField-3 E-Series, DOCA 3.1.0105  
**檔案**: 730 bytes  
**MD5**: `025d58144a49056290923b3bba355891`

## 測試步驟

1. 創建測試檔案：
```bash
echo "Test data for compression" > /tmp/test_file.txt
md5sum /tmp/test_file.txt
```

2. 啟動 Server (DPU)：
```bash
sudo doca_file_compression -p 03:00.0 -r 07:00.0 -f /tmp/received.txt
```

3. 啟動 Client (Host)：
```bash
sudo doca_file_compression -p 07:00.0 -f /tmp/test_file.txt
```

4. 驗證結果 (DPU)：
```bash
md5sum /tmp/received.txt
# 025d58144a49056290923b3bba355891 ✅ 匹配！
```

## E-Series 軟體降級行為

**預期日誌**:
```
[DOCA][WRN] compress_deflate is not supported by the device
[DOCA][INF] running SW compress with zlib
[DOCA][INF] SUCCESS: file was received and decompressed successfully
```

✅ Warning 是正常的，表示自動降級成功
