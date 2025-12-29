---
title: "從實務看 AWS Lambda 並發處理的設計陷阱"
excerpt: "當 Lambda 同時被大量觸發時，若沒有妥善處理共享資源，容易產生碰撞與重複使用問題。"
categories:
  - AWS
  - Cloud
tags:
  - Lambda
  - Concurrency
  - DynamoDB
  - Optimistic Locking
toc: true
---

## 前言

當 Lambda 同時被大量觸發時，若沒有妥善處理共享資源（如 DynamoDB、S3、Certificate Pool），容易產生碰撞與重複使用問題。本文分享實務上如何透過 Optimistic Locking 與狀態欄位避免風險。

## 問題場景

在高併發環境下，多個 Lambda 實例可能同時：
- 讀取相同的資源
- 嘗試更新相同的記錄
- 競爭有限的資源池

## 解決方案

### 1. Optimistic Locking

使用版本號控制，確保更新時資料未被其他程序修改：

```python
response = table.update_item(
    Key={'id': resource_id},
    UpdateExpression='SET #status = :new_status, #version = :new_version',
    ConditionExpression='#version = :current_version',
    ExpressionAttributeNames={
        '#status': 'status',
        '#version': 'version'
    },
    ExpressionAttributeValues={
        ':new_status': 'IN_USE',
        ':new_version': current_version + 1,
        ':current_version': current_version
    }
)
```

### 2. 狀態欄位設計

定義明確的狀態流轉：
- `AVAILABLE` → `IN_USE` → `RELEASED`
- 每次狀態變更都需要條件檢查

## 結論

在設計 Lambda 並發處理時，務必考慮資源競爭問題，善用 DynamoDB 的條件表達式可以有效避免資料不一致的情況。
