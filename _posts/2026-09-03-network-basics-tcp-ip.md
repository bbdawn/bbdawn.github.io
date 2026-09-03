---
title: "[Study] 네트워크 기초 (1) — TCP/IP 기본"
date: 2026-09-03 07:00:00 +0900
categories: [Study, Network]
subcategory: Study
tags: [network, tcp-ip, ip-address, subnet, cidr, gateway, mac, port, icmp]
---

## 개요

TCP/IP는 인터넷에서 사용하는 프로토콜 스택으로, 계층별로 서로 다른 주소 체계와 역할을 가집니다.

```
Application  : HTTP, DNS, ...
Transport    : TCP, UDP        → Port
Internet     : IP, ICMP, ARP   → IP 주소
Link         : Ethernet        → MAC 주소
```

이 글에서는 IP 주소, Subnet/CIDR, Gateway, MAC 주소, Port, TCP/UDP, ICMP를 하나씩 정리합니다.

---

## IP 주소

IPv4 주소는 32비트를 8비트씩 4묶음(옥텟)으로 나눠 점으로 구분해 표기합니다.

```
192.168.1.10
└┬─┘ └┬─┘ └┬┘ └┬┘
 8bit 8bit 8bit 8bit  = 32bit
```

| 대역 | 용도 |
|------|------|
| `10.0.0.0/8` | 사설 IP (Private) |
| `172.16.0.0/12` | 사설 IP (Private) |
| `192.168.0.0/16` | 사설 IP (Private) |
| `127.0.0.0/8` | 루프백 (Loopback) |
| 그 외 | 공인 IP (Public, 인터넷 라우팅 가능) |

```bash
# 현재 인터페이스의 IP 주소 확인
ip addr show
```

---

## Subnet / CIDR

서브넷은 하나의 네트워크 대역을 나눈 부분망입니다. **서브넷 마스크**로 IP 주소 중 네트워크 부분과 호스트 부분을 구분합니다.

```
IP 주소     : 192.168.1.10
서브넷 마스크 : 255.255.255.0   (또는 /24)

네트워크 부분 : 192.168.1.0   (앞 24bit)
호스트 부분   : .10           (뒤 8bit)
```

CIDR(Classless Inter-Domain Routing) 표기법은 서브넷 마스크의 1비트 개수를 슬래시 뒤에 붙여 표현합니다.

| CIDR | 서브넷 마스크 | 사용 가능 호스트 수 |
|------|---------------|----------------------|
| `/24` | 255.255.255.0 | 254 (256 - 네트워크주소 - 브로드캐스트주소) |
| `/25` | 255.255.255.128 | 126 |
| `/28` | 255.255.255.240 | 14 |
| `/30` | 255.255.255.252 | 2 (point-to-point 링크용) |

```bash
# 네트워크 대역 계산
ipcalc 192.168.1.10/24
```

---

## Gateway

게이트웨이(정확히는 **기본 게이트웨이**)는 자신의 서브넷을 벗어난 목적지로 패킷을 보낼 때 거쳐가는 라우터의 IP입니다. 같은 서브넷 안의 통신에는 게이트웨이가 필요 없습니다.

```bash
# 기본 게이트웨이 확인
ip route show default
# default via 192.168.1.1 dev eth0
```

---

## MAC 주소

MAC 주소는 네트워크 인터페이스에 물리적으로 부여된 48비트(6바이트) 식별자입니다.

```
00:1A:2B:3C:4D:5E
└──┬──┘ └──┬──┘
  OUI    장치 고유값
 (제조사 식별)
```

IP 주소가 "어디로 보낼지"를 정한다면, MAC 주소는 실제로 같은 네트워크(L2) 안에서 "누구에게 전달할지"를 결정합니다.

```bash
# 인터페이스 MAC 주소 확인
ip link show eth0
```

---

## Port

포트는 하나의 IP(호스트) 안에서 여러 애플리케이션을 구분하기 위한 16비트 번호(0~65535)입니다.

| 범위 | 용도 |
|------|------|
| 0 ~ 1023 | Well-known ports (예: 22-SSH, 80-HTTP, 443-HTTPS) |
| 1024 ~ 49151 | Registered ports |
| 49152 ~ 65535 | Dynamic/Private ports (클라이언트 임시 포트) |

```bash
# 현재 열려 있는 포트/연결 확인
ss -tnlp
```

---

## TCP / UDP

| 구분 | TCP | UDP |
|------|-----|-----|
| 연결 방식 | 연결 지향 (3-way handshake) | 비연결 지향 |
| 순서 보장 | 보장 | 보장 안 함 |
| 재전송 | 있음 (신뢰성 보장) | 없음 |
| 오버헤드 | 큼 | 작음 |
| 대표 사용처 | HTTP, SSH, DB 연결 | DNS 쿼리, 스트리밍, VoIP |

---

## ICMP

ICMP(Internet Control Message Protocol)는 데이터 전송이 아니라 **네트워크 상태를 확인·보고**하기 위한 프로토콜입니다. `ping`, `traceroute`가 대표적으로 ICMP를 사용합니다.

| Type | 의미 |
|------|------|
| 0 | Echo Reply (ping 응답) |
| 8 | Echo Request (ping 요청) |
| 3 | Destination Unreachable |
| 11 | Time Exceeded (TTL 만료, traceroute에서 사용) |

```bash
ping -c 3 8.8.8.8
```

다음 글에서는 IP 주소를 실제로 MAC 주소로 변환하는 **ARP**를 다룹니다.
