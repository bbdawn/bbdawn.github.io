---
title: "[Study] 네트워크 기초 (2) — ARP"
date: 2026-09-03 08:00:00 +0900
categories: [Study, Network]
subcategory: Study
tags: [network, arp, ip-mac, ip-neigh, arp-cache, arp-spoofing]
---

## 개요

같은 네트워크(L2) 안에서 프레임을 전달하려면 상대방의 MAC 주소가 필요합니다. 하지만 애플리케이션은 IP 주소만 알고 있으므로, **IP 주소를 MAC 주소로 변환**하는 과정이 필요합니다. 이 역할을 하는 프로토콜이 ARP(Address Resolution Protocol)입니다.

```
"192.168.1.20의 MAC 주소가 뭐야?" ──ARP──> "00:1A:2B:3C:4D:5E 입니다"
```

---

## IP → MAC 변환이 필요한 이유

이더넷 프레임은 목적지 MAC 주소를 반드시 채워야 전송할 수 있습니다.

```
[이더넷 프레임]
Dst MAC | Src MAC | Type | ... IP 패킷 ... | FCS
  ↑
이 값이 없으면 프레임을 만들 수 없음
```

애플리케이션/OS는 통신 상대의 IP만 알고 있으므로, 전송 직전에 "이 IP의 MAC이 뭔지" 조회하는 과정이 ARP입니다.

---

## ARP Request / Reply

ARP Request는 **브로드캐스트**로, ARP Reply는 요청한 호스트에게 **유니캐스트**로 전달됩니다.

```
Host A (192.168.1.10)                    전체 네트워크 (브로드캐스트)
  | --- ARP Request ------------------> | "192.168.1.20 누구야? MAC 알려줘"
  |                                      (모든 호스트가 수신)

Host B (192.168.1.20, 해당 IP 소유자)
  | <--- ARP Reply (유니캐스트) -------- | "저예요. MAC은 00:1A:2B:3C:4D:5E"
```

- **Request**: 목적지 MAC을 모르므로 `FF:FF:FF:FF:FF:FF`(브로드캐스트)로 전체 세그먼트에 전송
- **Reply**: 이미 요청자의 MAC을 알고 있으므로 요청자에게만 직접(유니캐스트) 응답

```bash
# ARP 패킷 캡처
sudo tcpdump -i any arp -n

12:00:01 ARP, Request who-has 192.168.1.20 tell 192.168.1.10, length 28
12:00:01 ARP, Reply 192.168.1.20 is-at 00:1a:2b:3c:4d:5e, length 28
```

---

## ARP Cache

매번 통신할 때마다 ARP를 반복하면 비효율적이므로, 조회 결과를 로컬에 캐싱합니다. 이 캐시를 리눅스에서는 **neighbor table**이라고 부릅니다.

```bash
# ARP 캐시(neighbor table) 조회
ip neigh show

# 출력 예시
192.168.1.20 dev eth0 lladdr 00:1a:2b:3c:4d:5e REACHABLE
192.168.1.1  dev eth0 lladdr aa:bb:cc:dd:ee:ff STALE
```

### neighbor table 상태

| 상태 | 의미 |
|------|------|
| `REACHABLE` | 최근 확인된 유효한 항목 |
| `STALE` | 유효기간은 지났지만 아직 사용 가능 (다음 통신 시 재확인) |
| `DELAY` / `PROBE` | 유효성 재확인 진행 중 |
| `FAILED` | 확인 실패 (해당 MAC으로 통신 불가) |
| `PERMANENT` | 수동으로 등록해 만료되지 않는 항목 |

```bash
# 특정 IP의 ARP 캐시 항목 직접 등록/삭제
ip neigh add 192.168.1.20 lladdr 00:1a:2b:3c:4d:5e dev eth0
ip neigh del 192.168.1.20 dev eth0

# 전체 캐시 비우기
ip neigh flush all
```

---

## ip neigh 명령어

`ip neigh`는 과거 `arp -n` 명령을 대체하는 최신 도구입니다.

```bash
ip neigh show               # 전체 캐시 조회
ip neigh show dev eth0       # 특정 인터페이스만
ip neigh show 192.168.1.20   # 특정 IP만
```

---

## 참고: ARP Spoofing

ARP는 인증 절차가 없어 누구나 "내가 이 IP의 소유자다"라고 응답할 수 있습니다. 이를 악용해 공격자가 위조된 ARP Reply로 다른 호스트의 트래픽을 가로채는 공격이 **ARP Spoofing(ARP Poisoning)**입니다. 정적 ARP 등록이나 스위치의 Dynamic ARP Inspection으로 방어할 수 있습니다.

---

ARP로 상대방의 MAC을 알아냈다면, 이제 실제로 패킷이 어떤 경로로 전달되는지는 **라우팅**이 결정합니다. 다음 글에서 이어집니다.
