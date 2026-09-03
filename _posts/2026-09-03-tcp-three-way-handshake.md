---
title: "[Study] 네트워크 기초 (4) — TCP 3-Way Handshake"
date: 2026-09-03 10:00:00 +0900
categories: [Study, Network]
subcategory: Study
tags: [network, tcp, tcp-ip, handshake, syn, ack, fin, rst, connection, tcpdump]
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

## TCP Connection

3-way handshake가 끝나면 연결은 **ESTABLISHED** 상태가 되고, 이때부터 클라이언트와 서버는 동시에 양방향으로 데이터를 주고받을 수 있습니다(full-duplex).

하나의 연결은 다음 4가지 값의 조합(4-tuple)으로 식별됩니다.

```
(src IP, src Port, dst IP, dst Port)
```

같은 서버·포트로 여러 클라이언트가 접속해도, 클라이언트의 IP/Port가 다르면 서로 다른 연결로 구분됩니다. 서버 입장에서 하나의 리스닝 소켓이 여러 개의 ESTABLISHED 연결을 동시에 처리할 수 있는 이유입니다.

```bash
# 현재 ESTABLISHED 연결 목록 확인 (로컬/원격 주소·포트 조합)
ss -tn state established
```

---

## 연결 종료 — FIN

연결 수립은 3-way지만, 종료는 보통 4-way(FIN → ACK → FIN → ACK)입니다. TCP는 양방향 통신이라 각 방향을 독립적으로 닫아야 하기 때문입니다(half-close).

```
Client                          Server
  | ---------- FIN ----------> |   (1) 클라이언트: 더 보낼 데이터 없음
  | <---------- ACK ---------- |   (2) 서버: FIN 확인
  | <---------- FIN ---------- |   (3) 서버: 더 보낼 데이터 없음
  | ---------- ACK ----------> |   (4) 클라이언트: FIN 확인 → 종료
```

### 상태 전이

```
[클라이언트]                         [서버]
ESTABLISHED                        ESTABLISHED
  └─ FIN 전송 → FIN_WAIT_1
      └─ ACK 수신 → FIN_WAIT_2         └─ FIN 수신 → CLOSE_WAIT (ACK 전송)
          └─ FIN 수신 → TIME_WAIT           └─ FIN 전송 → LAST_ACK
              (ACK 전송)                        └─ ACK 수신 → CLOSED
              └─ 일정 시간 후 → CLOSED
```

서버가 FIN을 받아도 아직 보낼 데이터가 남아있을 수 있어 ACK와 FIN을 즉시 합치지 못하는 경우가 많습니다(그래서 `CLOSE_WAIT` 상태가 따로 존재). 반대로 수립 단계에서는 서버가 SYN 수신 즉시 자신의 SYN을 함께 보낼 수 있어 3번으로 끝납니다.

`TIME_WAIT`은 마지막 ACK가 유실되어 상대방이 FIN을 재전송하는 경우에 대비해, 연결을 닫은 쪽(대개 능동적으로 종료를 시작한 쪽)이 일정 시간(기본 2×MSL) 더 유지하는 상태입니다.

```bash
# TIME_WAIT 연결 수 확인
ss -n state time-wait | wc -l
```

---

## 비정상 종료 — RST

RST(Reset)는 FIN과 달리 **정상적인 종료 절차 없이 연결을 즉시 끊을 때** 사용됩니다. 핸드셰이크 없이 한 번의 패킷으로 연결이 바로 닫힙니다.

### RST가 발생하는 대표적인 상황

| 상황 | 설명 |
|------|------|
| 닫힌 포트로 SYN 전송 | 리스닝 중인 프로세스가 없으면 SYN에 대해 `RST, ACK` 응답 |
| 존재하지 않는 연결로 패킷 수신 | 서버가 이미 종료한 연결로 데이터가 오면 RST로 응답 |
| 애플리케이션의 강제 종료 | `SO_LINGER` 옵션 등으로 버퍼를 비우지 않고 즉시 종료 |
| 방화벽/보안 장비의 연결 차단 | 정책 위반 시 세션을 강제로 끊기 위해 RST 주입 |

```bash
# 닫힌 포트로 접속 시도 → SYN 보내자마자 RST,ACK 수신
sudo tcpdump -i any 'tcp port 8888' -n

14:40:11.001 IP 10.0.0.5.51200 > 10.0.0.9.8888: Flags [S], seq 2000
14:40:11.003 IP 10.0.0.9.8888 > 10.0.0.5.51200: Flags [R.], seq 0, ack 2001
```

### FIN vs RST

| 구분 | FIN | RST |
|------|-----|-----|
| 종료 방식 | 정상 종료 (4-way) | 비정상/강제 종료 (1-way) |
| 버퍼 데이터 | 전송 완료 후 종료 | 즉시 폐기 가능 |
| TIME_WAIT | 발생 | 발생하지 않음 |
| 상대방 인지 | "더 보낼 데이터 없음" | "연결 상태가 유효하지 않음" |

RST는 시퀀스 번호나 연결 상태가 예상과 다를 때 커널이 즉시 정리하는 용도이기 때문에, 애플리케이션 레벨에서 남은 데이터가 있어도 그대로 버려질 수 있습니다.

---

지금까지는 L3(IP)/L4(TCP) 계층이었다면, 다음 글에서는 한 단계 아래인 L2(스위치) 계층에서 네트워크를 논리적으로 분리하는 **VLAN**을 다룹니다.
