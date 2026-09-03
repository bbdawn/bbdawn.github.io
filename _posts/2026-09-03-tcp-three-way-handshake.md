---
title: "[Study] TCP 3-Way Handshake — 연결 수립 과정 정리"
date: 2026-09-03 10:00:00 +0900
categories: [Study, Network]
subcategory: Study
tags: [network, tcp, tcp-ip, handshake, syn, ack, tcpdump]
---

## 개요

TCP는 연결 지향(connection-oriented) 프로토콜로, 데이터를 주고받기 전에 클라이언트와 서버가 먼저 논리적인 연결을 맺습니다. 이 연결 수립 과정에서 세 개의 패킷을 주고받는다고 해서 **3-way handshake**라고 부릅니다.

```
Client                          Server
  | ---------- SYN ----------> |   (1) 연결 요청 + 초기 시퀀스 번호(ISN) 전달
  | <------- SYN, ACK -------- |   (2) 요청 수락 + 자신의 ISN 전달
  | ---------- ACK ----------> |   (3) 응답 확인 → 연결 수립 완료
```

---

## 단계별 동작

### 1) SYN

클라이언트가 서버에 연결을 요청하며, 자신이 사용할 초기 시퀀스 번호(ISN, Initial Sequence Number)를 함께 보냅니다.

```
Flags: SYN
Seq: client_isn (예: 1000)
```

서버는 이 패킷을 받으면 연결 요청을 큐(SYN queue)에 넣고 다음 단계로 넘어갑니다.

### 2) SYN-ACK

서버는 클라이언트의 SYN을 확인(ACK)하는 동시에, 자신의 초기 시퀀스 번호를 담아 SYN을 함께 보냅니다.

```
Flags: SYN, ACK
Seq: server_isn (예: 5000)
Ack: client_isn + 1 (1001)
```

### 3) ACK

클라이언트는 서버의 SYN을 확인하는 ACK를 보냅니다. 이 패킷이 도달하면 양쪽 모두 연결이 수립된 상태(ESTABLISHED)가 됩니다.

```
Flags: ACK
Seq: client_isn + 1 (1001)
Ack: server_isn + 1 (5001)
```

이후 시퀀스/확인 번호는 각자 보낸 데이터의 바이트 수만큼 증가하며 데이터 전송에 이용됩니다.

---

## 왜 3번인가

| 단계 | 목적 |
|------|------|
| 2-way로는 부족 | 서버의 SYN이 클라이언트에게 도달했는지 서버가 확인할 방법이 없음 |
| 3-way면 충분 | 마지막 ACK로 서버도 "클라이언트가 내 SYN을 받았다"는 것을 확인 가능 |
| 4-way는 불필요 | 서버의 SYN과 ACK를 하나의 패킷(SYN-ACK)에 합쳐 왕복 횟수를 줄임 |

핵심은 **양방향 시퀀스 번호 동기화**입니다. 클라이언트와 서버 각자가 상대방의 ISN을 알고, 상대방이 자신의 ISN을 받았다는 것까지 확인해야 하므로 최소 3번의 메시지가 필요합니다.

---

## 서버 측 상태 변화

```
LISTEN
  └─ SYN 수신 → SYN_RECEIVED (SYN queue에 등록, SYN-ACK 전송)
      └─ ACK 수신 → ESTABLISHED (Accept queue로 이동, accept() 반환)
```

`SYN_RECEIVED` 상태에서 마지막 ACK가 오지 않으면 연결은 **half-open** 상태로 남으며, 이를 대량으로 유발하는 공격이 SYN Flood입니다.

```bash
# 현재 커널의 SYN backlog 크기 확인
sysctl net.ipv4.tcp_max_syn_backlog

# SYN_RECV 상태 연결 수 확인
ss -n state syn-recv
```

---

## tcpdump로 직접 확인하기

```bash
# 로컬에서 특정 포트로 나가는 패킷 캡처
sudo tcpdump -i any 'tcp port 443' -n

# 예시 출력 (SYN → SYN,ACK → ACK 순서)
14:32:01.001 IP 10.0.0.5.51000 > 93.184.216.34.443: Flags [S], seq 1000
14:32:01.032 IP 93.184.216.34.443 > 10.0.0.5.51000: Flags [S.], seq 5000, ack 1001
14:32:01.032 IP 10.0.0.5.51000 > 93.184.216.34.443: Flags [.], ack 5001
```

`Flags` 값은 `S`(SYN), `S.`(SYN+ACK), `.`(ACK)로 표시되며, 세 줄만 보면 handshake 여부를 바로 확인할 수 있습니다.

---

## 연결 종료와의 차이

연결 수립은 3-way지만, 종료는 보통 4-way(FIN → ACK → FIN → ACK)입니다. 서버가 FIN을 받아도 아직 보낼 데이터가 남아있을 수 있어 ACK와 FIN을 즉시 합치지 못하는 경우가 많기 때문입니다. 반대로 수립 단계에서는 서버가 SYN 수신 즉시 자신의 SYN을 함께 보낼 수 있어 3번으로 끝납니다.
