---
title: "[Study] 네트워크 기초 (3) — Routing"
date: 2026-09-03 09:00:00 +0900
categories: [Study, Network]
subcategory: Study
tags: [network, routing, routing-table, default-gateway, static-route, ip-route, subnet]
---

## 개요

라우팅은 패킷이 목적지까지 도달하기 위해 **어느 인터페이스로, 어디를 거쳐** 나갈지 결정하는 과정입니다. 리눅스 호스트도 서버·라우터 구분 없이 각자 라우팅 테이블을 가지고 이 결정을 내립니다.

---

## Routing Table

라우팅 테이블은 "이 목적지 대역으로 가려면 이 경로로 나가라"는 규칙의 목록입니다.

```bash
ip route show

# 출력 예시
default via 192.168.1.1 dev eth0
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.10
10.0.0.0/24 via 192.168.1.254 dev eth0
```

| 항목 | 의미 |
|------|------|
| 목적지 대역 (`192.168.1.0/24`) | 이 규칙이 적용되는 대상 네트워크 |
| `via` | 다음 홉(gateway)의 IP |
| `dev` | 나가는 인터페이스 |
| `scope link` | 게이트웨이 없이 직접 연결된 네트워크 |

패킷은 라우팅 테이블에서 **가장 구체적으로 일치하는(longest prefix match)** 규칙을 따릅니다. 어떤 규칙에도 일치하지 않으면 `default` 경로로 나갑니다.

---

## Default Gateway

`default` 경로는 다른 규칙에 일치하지 않는 모든 목적지를 처리하는 catch-all 규칙입니다.

```bash
ip route show default
# default via 192.168.1.1 dev eth0
```

기본 게이트웨이가 없거나 잘못 설정되면, 같은 서브넷 밖으로 나가는 통신이 모두 실패합니다.

---

## Static Route

특정 대역만 별도의 경로로 보내고 싶을 때 정적 라우팅을 등록합니다. 예를 들어 `10.0.0.0/24`는 기본 게이트웨이가 아니라 사내 라우터(`192.168.1.254`)를 거쳐야 하는 경우입니다.

```bash
# 정적 라우트 추가
ip route add 10.0.0.0/24 via 192.168.1.254 dev eth0

# 정적 라우트 삭제
ip route del 10.0.0.0/24

# 재부팅 후에도 유지하려면 배포판별 네트워크 설정 파일에 등록 필요
# (예: Ubuntu netplan, RHEL /etc/sysconfig/network-scripts/route-eth0)
```

---

## ip route 명령어 모음

```bash
ip route show                          # 전체 라우팅 테이블
ip route get 8.8.8.8                   # 특정 목적지로 나갈 때 사용할 경로 확인
ip route add 10.0.0.0/24 via 192.168.1.254 dev eth0
ip route del 10.0.0.0/24
ip route replace default via 192.168.1.1 dev eth0   # 기본 경로 교체
```

---

## 같은 subnet / 다른 subnet 통신

### 같은 subnet (L2 통신)

목적지가 같은 네트워크 대역이면, 게이트웨이 없이 **ARP로 직접 MAC을 조회**해 바로 프레임을 전달합니다.

```
Host A (192.168.1.10)  ──ARP로 MAC 확인──>  Host B (192.168.1.20)
        └──────────── 직접 프레임 전달 ────────────┘
```

### 다른 subnet (L3 통신)

목적지가 다른 대역이면, 라우팅 테이블에 일치하는 규칙이 없으므로 **기본 게이트웨이**를 거칩니다. 이때 이더넷 프레임의 목적지 MAC은 최종 목적지가 아니라 **게이트웨이의 MAC**입니다.

```
Host A (192.168.1.10)                Gateway (192.168.1.1)              Host C (10.0.0.20)
  | -- IP dst: 10.0.0.20 -->    |
  |    MAC dst: Gateway MAC     |
  |                              | -- 라우팅 후 재전송 --> |
  |                              |    IP dst: 10.0.0.20     |
  |                              |    MAC dst: Host C MAC   |
```

즉, IP 목적지 주소는 출발지부터 끝까지 바뀌지 않지만, **MAC 목적지 주소는 홉을 지날 때마다 바뀝니다**. 이것이 L2(MAC)와 L3(IP) 통신의 핵심 차이입니다.

---

지금까지의 IP/MAC/라우팅 지식을 바탕으로, 다음 글에서는 실제 연결이 어떻게 수립되는지 **TCP 3-Way Handshake**로 이어집니다.
