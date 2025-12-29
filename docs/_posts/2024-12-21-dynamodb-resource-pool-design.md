---
title: "使用 DynamoDB 作為資源池的安全設計"
excerpt: "將 DynamoDB 當作資源池時，必須考量高併發存取。本文介紹 ConditionExpression、Version Control 與狀態流轉的實作方式。"
categories:
  - AWS
  - Database
tags:
  - DynamoDB
  - Resource Pool
  - Concurrency
toc: true
---

## 前言

將 DynamoDB 當作資源池（如憑證、Token）時，必須考量高併發存取。本文介紹 ConditionExpression、Version Control 與狀態流轉的實作方式。

## 資源池設計模式

### Table Schema 設計

```json
{
  "TableName": "ResourcePool",
  "KeySchema": [
    { "AttributeName": "resourceId", "KeyType": "HASH" }
  ],
  "AttributeDefinitions": [
    { "AttributeName": "resourceId", "AttributeType": "S" },
    { "AttributeName": "status", "AttributeType": "S" }
  ],
  "GlobalSecondaryIndexes": [
    {
      "IndexName": "status-index",
      "KeySchema": [
        { "AttributeName": "status", "KeyType": "HASH" }
      ]
    }
  ]
}
```

### 安全取得資源

```java
public Optional<Resource> acquireResource() {
    // 1. 查詢可用資源
    QueryRequest query = QueryRequest.builder()
        .tableName("ResourcePool")
        .indexName("status-index")
        .keyConditionExpression("#status = :available")
        .expressionAttributeNames(Map.of("#status", "status"))
        .expressionAttributeValues(Map.of(":available", AttributeValue.builder().s("AVAILABLE").build()))
        .limit(1)
        .build();

    // 2. 使用條件更新取得資源
    UpdateItemRequest update = UpdateItemRequest.builder()
        .tableName("ResourcePool")
        .key(Map.of("resourceId", resource.resourceId()))
        .updateExpression("SET #status = :inUse, #version = #version + :one")
        .conditionExpression("#status = :available")
        .build();

    return Optional.of(resource);
}
```

## 狀態流轉設計

```
AVAILABLE → RESERVED → IN_USE → RELEASING → AVAILABLE
                ↓
            EXPIRED
```

## 結論

透過 ConditionExpression 與版本控制，可以在高併發環境下安全地管理資源池。
