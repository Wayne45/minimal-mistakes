---
title: "IoT 憑證生命週期管理的完整流程設計"
excerpt: "從憑證啟用、綁定、停用到刪除，任何一步出錯都可能影響裝置連線。本文以實際流程說明安全且可回溯的設計方式。"
categories:
  - IoT
  - Security
tags:
  - IoT
  - Certificate
  - Security
  - Lifecycle Management
toc: true
---

## 前言

從憑證啟用、綁定、停用到刪除，任何一步出錯都可能影響裝置連線。本文以實際流程說明安全且可回溯的設計方式。

## 憑證生命週期狀態

```
CREATED → ACTIVATED → BOUND → IN_USE → SUSPENDED → REVOKED → DELETED
              ↓                            ↑
           EXPIRED ─────────────────────────
```

### 狀態說明

| 狀態 | 說明 |
|-----|------|
| CREATED | 憑證已產生，尚未啟用 |
| ACTIVATED | 憑證已啟用，可供綁定 |
| BOUND | 已綁定至特定裝置 |
| IN_USE | 裝置正在使用中 |
| SUSPENDED | 暫時停用 |
| EXPIRED | 已過期 |
| REVOKED | 已撤銷（不可恢復）|
| DELETED | 已刪除 |

## 資料模型設計

```java
@Entity
@Table(name = "certificates")
public class Certificate {

    @Id
    private String certificateId;

    @Enumerated(EnumType.STRING)
    private CertificateStatus status;

    private String deviceId;

    private LocalDateTime createdAt;
    private LocalDateTime activatedAt;
    private LocalDateTime boundAt;
    private LocalDateTime expiresAt;
    private LocalDateTime revokedAt;

    @Version
    private Long version;

    @OneToMany(mappedBy = "certificate", cascade = CascadeType.ALL)
    private List<CertificateAuditLog> auditLogs;
}

@Entity
@Table(name = "certificate_audit_logs")
public class CertificateAuditLog {

    @Id
    @GeneratedValue
    private Long id;

    @ManyToOne
    private Certificate certificate;

    private String action;
    private String previousStatus;
    private String newStatus;
    private String performedBy;
    private LocalDateTime performedAt;
    private String reason;
}
```

## 狀態轉換服務

```java
@Service
@Transactional
public class CertificateLifecycleService {

    private static final Map<CertificateStatus, Set<CertificateStatus>> VALID_TRANSITIONS = Map.of(
        CREATED, Set.of(ACTIVATED, DELETED),
        ACTIVATED, Set.of(BOUND, EXPIRED, REVOKED),
        BOUND, Set.of(IN_USE, SUSPENDED, REVOKED),
        IN_USE, Set.of(SUSPENDED, EXPIRED, REVOKED),
        SUSPENDED, Set.of(IN_USE, REVOKED),
        EXPIRED, Set.of(REVOKED, DELETED),
        REVOKED, Set.of(DELETED)
    );

    public Certificate transition(String certId, CertificateStatus newStatus, String reason) {
        Certificate cert = repository.findById(certId)
            .orElseThrow(() -> new CertificateNotFoundException(certId));

        validateTransition(cert.getStatus(), newStatus);

        CertificateAuditLog log = CertificateAuditLog.builder()
            .certificate(cert)
            .action("STATUS_CHANGE")
            .previousStatus(cert.getStatus().name())
            .newStatus(newStatus.name())
            .performedBy(SecurityContext.getCurrentUser())
            .performedAt(LocalDateTime.now())
            .reason(reason)
            .build();

        cert.setStatus(newStatus);
        cert.getAuditLogs().add(log);

        // 觸發相關事件
        eventPublisher.publish(new CertificateStatusChangedEvent(cert, newStatus));

        return repository.save(cert);
    }

    private void validateTransition(CertificateStatus from, CertificateStatus to) {
        Set<CertificateStatus> validTargets = VALID_TRANSITIONS.get(from);
        if (validTargets == null || !validTargets.contains(to)) {
            throw new InvalidStateTransitionException(from, to);
        }
    }
}
```

## 綁定流程

```java
@Service
public class CertificateBindingService {

    public BindingResult bind(String deviceId, String certificateId) {
        // 1. 驗證裝置
        Device device = deviceService.findById(deviceId)
            .orElseThrow(() -> new DeviceNotFoundException(deviceId));

        // 2. 檢查裝置是否已有憑證
        if (device.getCertificateId() != null) {
            throw new DeviceAlreadyBoundException(deviceId);
        }

        // 3. 取得並驗證憑證
        Certificate cert = certificateService.findById(certificateId)
            .orElseThrow(() -> new CertificateNotFoundException(certificateId));

        if (cert.getStatus() != CertificateStatus.ACTIVATED) {
            throw new CertificateNotAvailableException(certificateId);
        }

        // 4. 執行綁定
        cert.setDeviceId(deviceId);
        cert.setBoundAt(LocalDateTime.now());
        lifecycleService.transition(certificateId, BOUND, "Bound to device: " + deviceId);

        device.setCertificateId(certificateId);
        deviceService.save(device);

        return BindingResult.success(deviceId, certificateId);
    }
}
```

## 過期檢查排程

```java
@Component
public class CertificateExpirationChecker {

    @Scheduled(cron = "0 0 * * * ?") // 每小時執行
    public void checkExpiredCertificates() {
        List<Certificate> expiring = repository.findByStatusInAndExpiresAtBefore(
            List.of(ACTIVATED, BOUND, IN_USE),
            LocalDateTime.now()
        );

        for (Certificate cert : expiring) {
            try {
                lifecycleService.transition(cert.getId(), EXPIRED, "Certificate expired");

                // 通知相關人員
                notificationService.notifyCertificateExpired(cert);

            } catch (Exception e) {
                log.error("Failed to expire certificate: {}", cert.getId(), e);
            }
        }
    }
}
```

## API 設計

```java
@RestController
@RequestMapping("/api/certificates")
public class CertificateController {

    @PostMapping
    public ResponseEntity<Certificate> create(@RequestBody CreateCertificateRequest request) {
        return ResponseEntity.ok(certificateService.create(request));
    }

    @PostMapping("/{id}/activate")
    public ResponseEntity<Certificate> activate(@PathVariable String id) {
        return ResponseEntity.ok(lifecycleService.transition(id, ACTIVATED, "Manual activation"));
    }

    @PostMapping("/{id}/bind")
    public ResponseEntity<BindingResult> bind(
            @PathVariable String id,
            @RequestBody BindRequest request) {
        return ResponseEntity.ok(bindingService.bind(request.getDeviceId(), id));
    }

    @PostMapping("/{id}/suspend")
    public ResponseEntity<Certificate> suspend(
            @PathVariable String id,
            @RequestBody SuspendRequest request) {
        return ResponseEntity.ok(lifecycleService.transition(id, SUSPENDED, request.getReason()));
    }

    @PostMapping("/{id}/revoke")
    public ResponseEntity<Certificate> revoke(
            @PathVariable String id,
            @RequestBody RevokeRequest request) {
        return ResponseEntity.ok(lifecycleService.transition(id, REVOKED, request.getReason()));
    }

    @GetMapping("/{id}/audit-logs")
    public ResponseEntity<List<CertificateAuditLog>> getAuditLogs(@PathVariable String id) {
        return ResponseEntity.ok(auditService.getLogsForCertificate(id));
    }
}
```

## 結論

完善的憑證生命週期管理需要：

1. **明確的狀態定義**：每個狀態的意義清楚
2. **嚴格的轉換驗證**：防止非法狀態變更
3. **完整的稽核記錄**：所有變更可追溯
4. **自動化監控**：過期檢查、異常告警

這樣的設計可以確保 IoT 裝置連線的安全性與可靠性。
