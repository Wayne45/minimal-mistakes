---
title: "從 iPerf3 得到的啟發：用 Go 打造 UDP Speed Test"
excerpt: "UDP 測速與延遲測量在網路品質評估中非常關鍵。本文分享如何用 Go 實作 sender/receiver，並模仿 iPerf3 的設計思路。"
categories:
  - Networking
  - Go
tags:
  - Go
  - UDP
  - Network Testing
  - iPerf
toc: true
---

## 前言

UDP 測速與延遲測量在網路品質評估中非常關鍵。本文分享如何用 Go 實作 sender / receiver，並模仿 iPerf3 的設計思路。

## 設計目標

- 測量 UDP 吞吐量
- 計算封包遺失率
- 測量 Jitter（延遲抖動）
- 支援雙向測試

## 封包結構設計

```go
type UDPPacket struct {
    SequenceNum uint64    // 序號
    Timestamp   int64     // 發送時間戳
    PayloadSize uint32    // 負載大小
    Payload     []byte    // 填充資料
}

func (p *UDPPacket) Marshal() []byte {
    buf := make([]byte, 20+len(p.Payload))
    binary.BigEndian.PutUint64(buf[0:8], p.SequenceNum)
    binary.BigEndian.PutInt64(buf[8:16], p.Timestamp)
    binary.BigEndian.PutUint32(buf[16:20], p.PayloadSize)
    copy(buf[20:], p.Payload)
    return buf
}
```

## Sender 實作

```go
type Sender struct {
    conn       *net.UDPConn
    targetAddr *net.UDPAddr
    config     SenderConfig
}

type SenderConfig struct {
    PacketSize   int
    Duration     time.Duration
    Bandwidth    int64 // bits per second, 0 = unlimited
}

func (s *Sender) Run(ctx context.Context) (*SenderResult, error) {
    result := &SenderResult{}
    payload := make([]byte, s.config.PacketSize-20)

    ticker := s.createTicker()
    defer ticker.Stop()

    var seq uint64
    startTime := time.Now()

    for {
        select {
        case <-ctx.Done():
            return result, nil
        case <-ticker.C:
            if time.Since(startTime) > s.config.Duration {
                return result, nil
            }

            packet := &UDPPacket{
                SequenceNum: seq,
                Timestamp:   time.Now().UnixNano(),
                PayloadSize: uint32(len(payload)),
                Payload:     payload,
            }

            _, err := s.conn.WriteToUDP(packet.Marshal(), s.targetAddr)
            if err != nil {
                result.Errors++
                continue
            }

            result.PacketsSent++
            result.BytesSent += int64(s.config.PacketSize)
            seq++
        }
    }
}
```

## Receiver 實作

```go
type Receiver struct {
    conn   *net.UDPConn
    config ReceiverConfig
}

func (r *Receiver) Run(ctx context.Context) (*ReceiverResult, error) {
    result := &ReceiverResult{
        ReceivedSeqs: make(map[uint64]bool),
    }

    buf := make([]byte, 65536)
    var lastSeq uint64
    var lastTimestamp int64

    for {
        select {
        case <-ctx.Done():
            return r.calculateResult(result), nil
        default:
            r.conn.SetReadDeadline(time.Now().Add(100 * time.Millisecond))
            n, _, err := r.conn.ReadFromUDP(buf)
            if err != nil {
                if netErr, ok := err.(net.Error); ok && netErr.Timeout() {
                    continue
                }
                return nil, err
            }

            packet := UnmarshalPacket(buf[:n])
            now := time.Now().UnixNano()

            result.PacketsReceived++
            result.BytesReceived += int64(n)
            result.ReceivedSeqs[packet.SequenceNum] = true

            // 計算 Jitter (RFC 3550)
            if lastTimestamp > 0 {
                d := abs(int64(now-lastTimestamp) - int64(packet.Timestamp-lastTimestamp))
                result.Jitter += (float64(d) - result.Jitter) / 16.0
            }

            lastSeq = packet.SequenceNum
            lastTimestamp = packet.Timestamp
        }
    }
}
```

## 結果計算

```go
type TestResult struct {
    Duration        time.Duration
    BytesSent       int64
    BytesReceived   int64
    PacketsSent     uint64
    PacketsReceived uint64
    PacketLoss      float64
    Throughput      float64 // Mbps
    Jitter          float64 // ms
}

func CalculateResult(sender *SenderResult, receiver *ReceiverResult) *TestResult {
    return &TestResult{
        Duration:        sender.Duration,
        BytesSent:       sender.BytesSent,
        BytesReceived:   receiver.BytesReceived,
        PacketsSent:     sender.PacketsSent,
        PacketsReceived: receiver.PacketsReceived,
        PacketLoss:      float64(sender.PacketsSent-receiver.PacketsReceived) / float64(sender.PacketsSent) * 100,
        Throughput:      float64(receiver.BytesReceived*8) / sender.Duration.Seconds() / 1e6,
        Jitter:          receiver.Jitter / 1e6, // ns to ms
    }
}
```

## 使用範例

```bash
# Server 端
$ udptest -s -p 5201

# Client 端
$ udptest -c 192.168.1.100 -p 5201 -t 10 -b 100M

[ ID]  Interval        Transfer    Bandwidth       Jitter    Lost/Total
[  1]  0.0-10.0 sec   119 MBytes   100 Mbits/sec   0.123 ms  12/85234 (0.014%)
```

## 結論

透過理解 iPerf3 的設計，我們可以用 Go 實作自己的網路測速工具，更好地理解 UDP 傳輸特性。
