---
title: "Spring Batch 在 IoT 大量指令派送的應用經驗"
excerpt: "當需要對上萬台 IoT 裝置發送指令時，單純 for-loop 並不可行。本文分享如何使用 Spring Batch 搭配 MQTT Worker 進行批次派送。"
categories:
  - IoT
  - Backend
tags:
  - Spring Batch
  - MQTT
  - IoT
  - Batch Processing
toc: true
---

## 前言

當需要對上萬台 IoT 裝置發送指令時，單純 for-loop 並不可行。本文分享如何使用 Spring Batch 搭配 MQTT Worker 進行可控、可追蹤的批次派送。

## 為什麼需要 Spring Batch

### 挑戰

- 數萬台裝置需要同時接收指令
- 需要追蹤每個裝置的執行狀態
- 失敗時需要重試機制
- 需要限制併發數量避免系統過載

### Spring Batch 優勢

- 內建 Chunk 處理模式
- 完整的 Job 狀態追蹤
- 支援 Restart 與 Skip 策略
- 可配置的執行緒池

## 架構設計

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Job Launcher │ → │  Batch Job   │ → │ MQTT Worker  │
└─────────────┘     └─────────────┘     └─────────────┘
                           ↓
                    ┌─────────────┐
                    │  Device DB   │
                    └─────────────┘
```

## 實作範例

```java
@Configuration
public class DeviceCommandJobConfig {

    @Bean
    public Job deviceCommandJob(JobRepository jobRepository, Step sendCommandStep) {
        return new JobBuilder("deviceCommandJob", jobRepository)
            .start(sendCommandStep)
            .build();
    }

    @Bean
    public Step sendCommandStep(JobRepository jobRepository,
                                 PlatformTransactionManager transactionManager,
                                 ItemReader<Device> deviceReader,
                                 ItemProcessor<Device, CommandResult> commandProcessor,
                                 ItemWriter<CommandResult> resultWriter) {
        return new StepBuilder("sendCommandStep", jobRepository)
            .<Device, CommandResult>chunk(100, transactionManager)
            .reader(deviceReader)
            .processor(commandProcessor)
            .writer(resultWriter)
            .taskExecutor(taskExecutor())
            .throttleLimit(10)
            .build();
    }

    @Bean
    public TaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        return executor;
    }
}
```

## 結論

Spring Batch 提供了完整的批次處理框架，非常適合 IoT 大量指令派送的場景。
