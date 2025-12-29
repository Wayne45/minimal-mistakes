---
title: "為什麼我在裝置回應追蹤中選擇 Redis"
excerpt: "比起同步等待，Redis 更適合用於裝置回應的狀態彙總與統計。本文說明成功/失敗/Offline 統計的設計方式。"
categories:
  - Backend
  - IoT
tags:
  - Redis
  - IoT
  - Real-time
  - Statistics
toc: true
---

## 前言

比起同步等待（await），Redis 更適合用於裝置回應的狀態彙總與統計。本文說明成功 / 失敗 / Offline 統計的設計方式。

## 為什麼選擇 Redis

### 同步等待的問題

- 裝置回應時間不可預測
- 大量裝置時 Connection 資源耗盡
- 無法即時查看進度

### Redis 的優勢

- 原子操作支援計數統計
- 支援 TTL 自動過期
- Pub/Sub 即時通知
- 高效能低延遲

## 設計架構

```
Device Response → Redis Hash → Statistics API
                      ↓
                 Redis Counter
                      ↓
                 Real-time Dashboard
```

## 實作範例

### 統計結構設計

```java
@Service
public class DeviceResponseTracker {

    private final RedisTemplate<String, String> redisTemplate;

    public void recordResponse(String batchId, String deviceId, ResponseStatus status) {
        String hashKey = "batch:" + batchId + ":responses";
        String counterKey = "batch:" + batchId + ":stats";

        // 記錄個別裝置狀態
        redisTemplate.opsForHash().put(hashKey, deviceId, status.name());

        // 更新統計計數
        redisTemplate.opsForHash().increment(counterKey, status.name(), 1);

        // 設定過期時間
        redisTemplate.expire(hashKey, Duration.ofHours(24));
        redisTemplate.expire(counterKey, Duration.ofHours(24));
    }

    public BatchStatistics getStatistics(String batchId) {
        String counterKey = "batch:" + batchId + ":stats";
        Map<Object, Object> stats = redisTemplate.opsForHash().entries(counterKey);

        return BatchStatistics.builder()
            .success(toLong(stats.get("SUCCESS")))
            .failed(toLong(stats.get("FAILED")))
            .offline(toLong(stats.get("OFFLINE")))
            .pending(toLong(stats.get("PENDING")))
            .build();
    }
}
```

### 即時進度查詢

```java
@RestController
@RequestMapping("/api/batch")
public class BatchController {

    @GetMapping("/{batchId}/progress")
    public ResponseEntity<BatchProgress> getProgress(@PathVariable String batchId) {
        BatchStatistics stats = tracker.getStatistics(batchId);
        int total = batchService.getTotalDevices(batchId);
        int completed = stats.getSuccess() + stats.getFailed() + stats.getOffline();

        return ResponseEntity.ok(BatchProgress.builder()
            .total(total)
            .completed(completed)
            .percentage((completed * 100) / total)
            .statistics(stats)
            .build());
    }
}
```

## 結論

Redis 的原子操作和高效能特性，使其成為裝置回應追蹤的理想選擇。
