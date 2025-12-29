---
title: "macOS 工程師常見的 launchctl 與自動化問題"
excerpt: "login item 重複、launchctl load 失敗、plist 權限錯誤，是 macOS 工程師常見困擾。本文整理實戰排查流程。"
categories:
  - macOS
  - DevOps
tags:
  - launchctl
  - macOS
  - Automation
toc: true
---

## 前言

login item 重複、launchctl load 失敗、plist 權限錯誤，是 macOS 工程師常見困擾。本文整理實戰排查流程。

## 常見問題與解法

### 1. launchctl load 失敗

**症狀：**
```bash
$ launchctl load ~/Library/LaunchAgents/com.example.myapp.plist
Load failed: 5: Input/output error
```

**排查步驟：**

```bash
# 檢查 plist 語法
plutil -lint ~/Library/LaunchAgents/com.example.myapp.plist

# 檢查權限
ls -la ~/Library/LaunchAgents/com.example.myapp.plist
# 應該是 -rw-r--r-- (644)

# 修正權限
chmod 644 ~/Library/LaunchAgents/com.example.myapp.plist
```

### 2. Login Item 重複

**症狀：**
應用程式在登入時啟動多次

**解法：**
```bash
# 列出所有 launch agents
launchctl list | grep com.example

# 移除重複項目
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/com.example.myapp.plist

# 清理 Login Items
osascript -e 'tell application "System Events" to delete login item "MyApp"'
```

### 3. plist 範例

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.example.myapp</string>

    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/myapp</string>
        <string>--daemon</string>
    </array>

    <key>RunAtLoad</key>
    <true/>

    <key>KeepAlive</key>
    <true/>

    <key>StandardOutPath</key>
    <string>/tmp/myapp.log</string>

    <key>StandardErrorPath</key>
    <string>/tmp/myapp.error.log</string>
</dict>
</plist>
```

### 4. 新版 macOS 的變化

macOS 10.10+ 使用新的 `launchctl` 語法：

```bash
# 舊語法（已棄用）
launchctl load ~/Library/LaunchAgents/com.example.myapp.plist
launchctl unload ~/Library/LaunchAgents/com.example.myapp.plist

# 新語法
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.example.myapp.plist
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/com.example.myapp.plist

# 啟動/停止服務
launchctl kickstart gui/$(id -u)/com.example.myapp
launchctl kill SIGTERM gui/$(id -u)/com.example.myapp
```

### 5. 除錯技巧

```bash
# 查看服務狀態
launchctl print gui/$(id -u)/com.example.myapp

# 查看系統日誌
log show --predicate 'subsystem == "com.apple.launchd"' --last 1h

# 手動執行測試
/usr/local/bin/myapp --daemon
```

## 常見錯誤碼

| 錯誤碼 | 說明 |
|-------|------|
| 5 | Input/output error（權限或路徑問題）|
| 37 | Operation already in progress |
| 113 | Could not find specified service |

## 結論

launchctl 問題通常來自權限設定或 plist 格式錯誤，善用 `plutil` 和 `log show` 可以快速定位問題。
