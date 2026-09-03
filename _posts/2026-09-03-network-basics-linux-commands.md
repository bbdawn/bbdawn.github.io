---
title: "[Study] 네트워크 기초 (6) — Linux 네트워크 명령어"
date: 2026-09-03 12:00:00 +0900
categories: [Study, Network]
subcategory: Study
tags: [network, linux, ip-addr, ip-link, ip-route, ip-neigh, ping, traceroute, ss]
---

## 개요

지금까지 다룬 IP/MAC/라우팅/ARP/TCP 개념은 대부분 리눅스 명령어 하나로 직접 확인할 수 있습니다. 이 글은 앞서 나온 개념들을 실제로 조회·검증할 때 쓰는 핵심 명령어 7가지를 정리합니다.

| 명령어 | 무엇을 확인하는가 | 관련 개념 |
|--------|-------------------|-----------|
| `ip addr` | 인터페이스에 할당된 IP 주소 | [1] IP 주소, Subnet |
| `ip link` | 인터페이스 상태(L2) — UP/DOWN, MAC, MTU | [1] MAC 주소 |
| `ip route` | 라우팅 테이블, 패킷이 나갈 경로 | [3] Routing |
| `ip neigh` | ARP 캐시(IP-MAC 매핑) | [2] ARP |
| `ping` | 목적지까지 왕복 가능 여부, 지연시간 | [1] ICMP |
| `traceroute` | 목적지까지 거쳐가는 경로(홉) | [3] Routing |
| `ss` | 현재 소켓/연결 상태 | [4] TCP Connection |

---

## ip addr — IP 주소 확인

인터페이스에 어떤 IP가 붙어 있는지, 서브넷은 어떻게 되는지 확인합니다.

```bash
ip addr show
# 또는 축약형
ip a
```

```
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
    inet 192.168.1.10/24 brd 192.168.1.255 scope global eth0
    inet6 fe80::21a:2bff:fe3c:4d5e/64 scope link
```

| 확인 항목 | 위치 |
|-----------|------|
| IP 주소 / 서브넷 | `inet 192.168.1.10/24` |
| 브로드캐스트 주소 | `brd 192.168.1.255` |
| 인터페이스 상태 | `<...,UP,LOWER_UP>` (UP=관리자가 켬, LOWER_UP=실제 링크 연결됨) |

```bash
# 특정 인터페이스만
ip addr show eth0
```

---

## ip link — 인터페이스(L2) 상태 확인

`ip addr`가 IP(L3) 관점이라면, `ip link`는 그 아래 계층인 MAC/인터페이스 자체의 상태를 봅니다.

```bash
ip link show eth0
```

```
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc ...
    link/ether 00:1a:2b:3c:4d:5e brd ff:ff:ff:ff:ff:ff
```

| 확인 항목 | 위치 |
|-----------|------|
| MAC 주소 | `link/ether 00:1a:2b:3c:4d:5e` |
| MTU | `mtu 1500` |
| 인터페이스 up/down | `UP` 플래그 유무 |

```bash
# 인터페이스 강제로 내리기/올리기
ip link set eth0 down
ip link set eth0 up
```

---

## ip route — 라우팅 테이블 확인

패킷이 어느 경로로 나갈지, 기본 게이트웨이는 무엇인지 확인합니다.

```bash
ip route show
```

```
default via 192.168.1.1 dev eth0
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.10
```

```bash
# 특정 목적지로 나갈 때 실제로 어떤 경로/소스 IP를 쓰는지 확인
ip route get 8.8.8.8
# 8.8.8.8 via 192.168.1.1 dev eth0 src 192.168.1.10
```

| 확인 항목 | 용도 |
|-----------|------|
| `default via ...` | 기본 게이트웨이 |
| 대역별 라우트 | 특정 목적지가 어느 인터페이스/게이트웨이로 나가는지 |
| `ip route get` | "이 목적지로 지금 패킷을 보내면 어떻게 되는지" 시뮬레이션 |

---

## ip neigh — ARP 캐시 확인

같은 서브넷 안의 IP-MAC 매핑(neighbor table)을 확인합니다.

```bash
ip neigh show
```

```
192.168.1.1  dev eth0 lladdr aa:bb:cc:dd:ee:ff REACHABLE
192.168.1.20 dev eth0 lladdr 00:1a:2b:3c:4d:5e STALE
```

| 확인 항목 | 용도 |
|-----------|------|
| IP-MAC 매핑 | 통신 상대의 실제 MAC이 정상적으로 조회됐는지 |
| 상태(REACHABLE/STALE/FAILED) | ARP 결과가 유효한지, 통신 불가 여부 |

같은 서브넷 내 통신이 안 될 때 `ip neigh`가 `FAILED`로 나오면, ARP 단계(L2)에서 문제가 있다는 신호입니다.

---

## ping — 도달 가능 여부 / 왕복 지연 확인

ICMP Echo Request를 보내고 Echo Reply를 받아, 목적지까지 통신이 되는지와 걸리는 시간을 확인합니다.

```bash
ping -c 3 192.168.1.20
```

```
64 bytes from 192.168.1.20: icmp_seq=1 ttl=64 time=0.412 ms
64 bytes from 192.168.1.20: icmp_seq=2 ttl=64 time=0.389 ms
64 bytes from 192.168.1.20: icmp_seq=3 ttl=64 time=0.401 ms
--- 192.168.1.20 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss
```

| 확인 항목 | 위치 |
|-----------|------|
| 도달 가능 여부 | 응답 수신 여부 (`Destination Host Unreachable`이면 라우팅 문제) |
| 왕복 시간(RTT) | `time=0.412 ms` |
| 패킷 손실률 | `0% packet loss` |
| 홉 수(대략) | `ttl=64` (출발 시 TTL 64에서 몇 감소했는지로 역산 가능) |

응답이 없다고 무조건 "네트워크 문제"는 아닙니다 — 방화벽이 ICMP만 막아둔 경우도 흔합니다.

---

## traceroute — 경로(홉) 확인

목적지까지 패킷이 **어떤 라우터들을 거쳐** 가는지 순서대로 보여줍니다. TTL을 1부터 하나씩 늘려가며, 각 홉에서 TTL이 0이 될 때 돌아오는 ICMP Time Exceeded 응답을 이용합니다.

```bash
traceroute 8.8.8.8
```

```
 1  192.168.1.1 (192.168.1.1)  0.523 ms
 2  10.10.0.1 (10.10.0.1)  1.204 ms
 3  * * *
 4  8.8.8.8 (8.8.8.8)  12.331 ms
```

| 확인 항목 | 용도 |
|-----------|------|
| 각 홉의 IP/응답 시간 | 어느 구간에서 지연이 발생하는지 |
| `* * *` | 해당 홉이 ICMP 응답을 안 주거나 차단 (반드시 장애는 아님) |
| 마지막 줄 | 목적지까지 도달했는지 여부 |

`ping`이 "도달 여부"만 알려준다면, `traceroute`는 "어느 구간에서 문제인지" 구간별로 좁혀줍니다.

---

## ss — 소켓/연결 상태 확인

현재 호스트에서 어떤 포트가 열려 있고(LISTEN), 어떤 연결이 맺어져 있는지(ESTABLISHED) 확인합니다. 과거의 `netstat`을 대체하는 최신 도구입니다.

```bash
ss -tnlp     # TCP, 숫자로 표시, LISTEN 상태만, 프로세스 정보 포함
```

```
State   Recv-Q  Send-Q  Local Address:Port   Peer Address:Port  Process
LISTEN  0       128     0.0.0.0:22           0.0.0.0:*          users:(("sshd",pid=1023))
LISTEN  0       128     0.0.0.0:443          0.0.0.0:*          users:(("nginx",pid=2044))
```

```bash
ss -tn state established   # 현재 맺어진 연결만
ss -tn state syn-recv       # 핸드셰이크 도중(half-open) 연결만
ss -tn state time-wait      # 종료 중(TIME_WAIT) 연결만
```

| 확인 항목 | 용도 |
|-----------|------|
| `LISTEN` 목록 | 어떤 서비스가 어느 포트를 열어두고 있는지 |
| `ESTABLISHED` 목록 | 실제 맺어진 연결(4-tuple)과 상대방 |
| `SYN-RECV` / `TIME-WAIT` 수 | 핸드셰이크 지연, 연결 과다 여부 (SYN Flood 등 이상 징후 확인) |

---

## 정리

같은 서브넷 통신이 안 될 때는 `ip addr`(내 IP/서브넷이 맞는지) → `ip neigh`(ARP가 되는지) 순으로 확인하고, 다른 서브넷 통신이 안 될 때는 `ip route`(경로가 있는지) → `ping`/`traceroute`(어디까지 도달하는지) → `ss`(실제로 연결이 맺어졌는지) 순으로 좁혀가면 됩니다.

여기까지 TCP/IP 기본(1) → ARP(2) → Routing(3) → TCP 3-Way Handshake(4) → VLAN(5) → Linux 네트워크 명령어(6)로 이어지는 네트워크 기초 시리즈였습니다.
