---
title: "[Study] 네트워크 기초 (5) — VLAN"
date: 2026-09-03 11:00:00 +0900
categories: [Study, Network]
subcategory: Study
tags: [network, vlan, vlan-id, access-port, trunk-port, tagged, untagged, 802.1q]
---

## 개요

VLAN(Virtual LAN)은 물리적으로 하나의 스위치를 **논리적으로 여러 개의 LAN처럼 분리**하는 기술입니다. 물리 배선을 바꾸지 않고도 소프트웨어 설정만으로 네트워크를 나눌 수 있습니다.

```
[물리적으로는 스위치 1대]

포트 1,2   ── VLAN 10 (개발팀)
포트 3,4   ── VLAN 20 (운영팀)
포트 5,6   ── VLAN 30 (게스트)

→ 같은 스위치에 꽂혀 있어도 VLAN이 다르면 서로 통신 불가 (브로드캐스트도 격리)
```

---

## VLAN이 왜 필요한가

| 문제 | VLAN 없이 | VLAN 사용 시 |
|------|-----------|--------------|
| 브로드캐스트 도메인 | 스위치 전체가 하나의 도메인 → ARP 등 브로드캐스트가 전 구간에 전파 | VLAN 단위로 도메인 분리 → 트래픽 격리, 성능 향상 |
| 보안 | 같은 스위치에 있으면 기본적으로 서로 통신 가능 | 다른 VLAN 간 통신은 라우터/L3 스위치를 거쳐야만 가능 (접근 통제 지점 확보) |
| 물리적 제약 | 부서가 바뀔 때마다 배선/스위치 재배치 필요 | 포트 설정 변경만으로 논리적 재구성 가능 |
| 스위치 비용 | 그룹마다 별도 스위치 필요 | 스위치 1대로 여러 네트워크 운영 |

---

## VLAN ID

VLAN ID는 12비트 값으로 **1 ~ 4094**까지 사용 가능합니다(0, 4095는 예약).

| VLAN ID | 의미 |
|---------|------|
| 1 | 대부분의 장비에서 기본(default) VLAN |
| 2 ~ 1001 | 일반적으로 사용자가 자유롭게 할당 |
| 1002 ~ 1005 | 레거시(Token Ring/FDDI) 예약 대역 (일부 장비) |
| 4094 | 관례적으로 최대값 (실제 최대는 4094, 4095는 예약) |

같은 VLAN ID를 가진 포트끼리만 같은 브로드캐스트 도메인에 속하게 됩니다.

---

## Access / Trunk

스위치 포트는 용도에 따라 두 가지 모드로 설정합니다.

```
[Access Port]                       [Trunk Port]
서버/PC 등 엔드포인트 연결용             스위치-스위치, 스위치-라우터 간 연결용
하나의 VLAN에만 소속                    여러 VLAN의 트래픽을 동시에 전달
프레임에 VLAN 태그 없음                  프레임에 VLAN 태그를 붙여서 구분
```

| 구분 | Access | Trunk |
|------|--------|-------|
| 소속 VLAN | 단일 (Access VLAN 1개) | 다중 (Allowed VLAN 목록) |
| 연결 대상 | 서버, PC, 프린터 등 | 스위치, 라우터, 하이퍼바이저 |
| 프레임 형태 | Untagged | Tagged (Native VLAN은 예외적으로 Untagged 가능) |

```
[서버] --Access(VLAN 10)--> [스위치A] --Trunk(VLAN 10,20 허용)--> [스위치B] --Access(VLAN 10)--> [서버]
```

---

## Tagged / Untagged

- **Untagged 프레임**: VLAN 정보가 프레임에 없음. Access 포트로 들어온 프레임은 해당 포트의 VLAN으로 내부적으로만 처리되고, 프레임 자체에는 태그가 붙지 않습니다.
- **Tagged 프레임**: 프레임 헤더에 VLAN ID가 명시적으로 삽입됨. Trunk 링크로 여러 VLAN의 트래픽을 함께 보낼 때, 수신 측이 "이 프레임이 어느 VLAN 소속인지" 구분할 수 있도록 태그를 붙입니다.

```
[Access 포트로 들어온 프레임]        [Trunk로 나갈 때: VLAN 태그 삽입]         [Trunk로 들어온 프레임: 태그 제거 후 전달]
Dst MAC|Src MAC|Type|Data      →   Dst MAC|Src MAC|[VLAN Tag]|Type|Data   →   Dst MAC|Src MAC|Type|Data (Access로 나갈 때)
```

---

## 802.1Q

802.1Q는 이더넷 프레임에 VLAN 태그를 삽입하는 방식을 정의한 IEEE 표준입니다. 기존 이더넷 헤더의 Type 필드 앞에 4바이트짜리 태그가 추가됩니다.

```
[일반 이더넷 프레임]
Dst MAC(6) | Src MAC(6) | Type(2) | Payload | FCS

[802.1Q 태그 프레임]
Dst MAC(6) | Src MAC(6) | TPID(2)=0x8100 | TCI(2) | Type(2) | Payload | FCS
                          └──────── 4 bytes 추가 ────────┘
```

| 필드 | 크기 | 의미 |
|------|------|------|
| TPID | 2 byte | 태그 프레임임을 나타내는 고정값 `0x8100` |
| TCI | 2 byte | PCP(3bit, 우선순위) + DEI(1bit) + VLAN ID(12bit) |

```bash
# 리눅스에서 VLAN 서브인터페이스 생성 (802.1Q 태깅)
ip link add link eth0 name eth0.10 type vlan id 10
ip link set eth0.10 up
ip addr add 192.168.10.5/24 dev eth0.10

# 생성된 VLAN 인터페이스 확인
ip -d link show eth0.10
```

이 명령은 `eth0`으로 나가는 프레임에 VLAN ID 10 태그를 붙여, 물리 인터페이스 하나로 여러 VLAN에 동시에 참여할 수 있게 해줍니다(Trunk 포트에 연결되어 있어야 정상 동작).

---

여기까지 IP/MAC 기본기(1) → ARP(2) → Routing(3) → TCP 3-Way Handshake(4) → VLAN(5)으로 이어지는 네트워크 기초 시리즈였습니다.
