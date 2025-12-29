---
title: "用 AWS SQS 解耦清理型任務是否真的必要？"
excerpt: "不是所有背景任務都需要 SQS。本文以「憑證清理流程」為例，分析何時直接 DB + 狀態欄位就足夠。"
categories:
  - AWS
  - Architecture
tags:
  - SQS
  - Architecture
  - Design Patterns
toc: true
---

## 前言

不是所有背景任務都需要 SQS。本文以「憑證清理流程」為例，分析何時直接 DB + 狀態欄位就足夠，何時才需要 Queue。

## 案例：憑證清理流程

### 需求

- 定期清理過期憑證
- 清理前需要通知相關服務
- 需要記錄清理歷史

### 方案一：SQS 解耦

```
Scheduler → SQS → Lambda → Cleanup Service
                      ↓
                   DynamoDB
```

**優點：**
- 解耦各元件
- 自動重試機制
- 可處理大量訊息

**缺點：**
- 架構複雜度增加
- 需要處理訊息重複
- 成本較高

### 方案二：DB + 狀態欄位

```
Scheduler → Cleanup Service → DB (with status)
```

**優點：**
- 架構簡單
- 狀態可查詢
- 成本低

**缺點：**
- 需要自行實作重試
- 擴展性有限

## 如何選擇

| 考量因素 | 選 SQS | 選 DB + 狀態 |
|---------|--------|-------------|
| 任務量 | 大量、突發 | 固定、可預測 |
| 即時性 | 可延遲 | 需要即時 |
| 可靠性要求 | 極高 | 一般 |
| 跨服務協作 | 多服務 | 單一服務 |
| 團隊熟悉度 | 熟悉 AWS | 偏好簡單 |

## 憑證清理的選擇

對於憑證清理這類場景：
- 任務量固定（每日一次）
- 單一服務處理
- 需要即時查詢狀態

**結論：DB + 狀態欄位就足夠**

```java
@Scheduled(cron = "0 0 2 * * ?")
public void cleanupExpiredCertificates() {
    List<Certificate> expired = certRepo.findByStatusAndExpiredBefore(
        CertStatus.ACTIVE,
        LocalDateTime.now()
    );

    for (Certificate cert : expired) {
        cert.setStatus(CertStatus.CLEANING);
        certRepo.save(cert);

        try {
            cleanupService.cleanup(cert);
            cert.setStatus(CertStatus.CLEANED);
        } catch (Exception e) {
            cert.setStatus(CertStatus.CLEANUP_FAILED);
            cert.setErrorMessage(e.getMessage());
        }
        certRepo.save(cert);
    }
}
```

## 結論

選擇技術方案時，應該根據實際需求而非技術潮流。簡單有效的方案往往是最好的選擇。
