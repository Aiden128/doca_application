# 背景知識

## 官方定義

> "DOCA Secure Channel 利用 DOCA Comm Channel API 建立安全、網路獨立的 Host 與 DPU 之間通訊通道。" — NVIDIA DOCA Documentation

## Producer-Consumer 模型（程式碼佐證）

### Producer 端核心程式碼

```c
// 引用自: secure_channel_core.c:454-466

// 1. 創建 Producer
result = doca_comch_producer_create(ctx->comch_connection, &producer);

// 2. 配置 Send Task callbacks
result = doca_comch_producer_task_send_set_conf(producer,
                                                send_task_completed_callback,  // 成功
                                                send_task_fail_callback,        // 失敗
                                                total_tasks);
```

**關鍵設計**: 使用非阻塞 I/O + Progress Engine 輪詢硬體事件。

### Consumer 端核心程式碼

```c
// 引用自: secure_channel_core.c:790-796

// 預先 post recv tasks (接收 buffer 準備好)
for (i = 0; i < total_tasks; i++) {
    result = doca_comch_consumer_task_post_recv_alloc_init(consumer, doca_buf[i], &task[i]);
    result = doca_task_submit(doca_comch_consumer_task_post_recv_as_task(task[i]));
}
```

**關鍵設計**: RDMA 風格，預先告訴硬體「這些 buffer 可接收資料」。

## 雙向通訊架構

```
Main Thread (Control Plane)
    ↓
pthread 1: run_producer() → 發送訊息
pthread 2: run_consumer() → 接收訊息
```

**參考**: [NVIDIA DOCA Secure Channel Guide](https://docs.nvidia.com/doca/archive/2-5-2/nvidia+doca+secure+channel+application+guide/index.html)
