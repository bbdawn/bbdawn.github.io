---
title: "[Study] LLDP (Link Layer Discovery Protocol) 개념 정리"
date: 2026-06-30 14:20:00 +0900
categories: [Study, Infra]
subcategory: Study
tags: [rack, topology, lldp, network-discovery, switch]
---

[데이터센터 랙 토폴로지 시각화 프로젝트]({% post_url 2026-06-30-rack-topology %})에서 서버 ↔ 스위치 포트 연결 관계를 자동으로 알아내는 데 사용한 LLDP를 정리합니다.

## 개요

LLDP(Link Layer Discovery Protocol)는 인접한 네트워크 장비가 서로 자신의 정보(장비명, 포트 번호 등)를 주고받아, 어떤 장비의 어떤 포트가 어떤 장비의 어떤 포트와 실제로 연결되어 있는지 자동으로 알아낼 수 있게 하는 프로토콜입니다.

## 핵심 개념

- **L2(데이터링크 계층) 프로토콜**: IP 주소 없이도 동작
- **LLDPDU (LLDP Data Unit)**: 각 장비가 주기적으로 인접 포트에 전송하는 정보 패킷
- 수신 장비는 상대방의 Chassis ID, Port ID, 장비명 등을 인접 테이블에 저장

```
[스위치 A - Port 1] --LLDPDU 전송--> [서버 NIC]
[서버 NIC]          --LLDPDU 전송--> [스위치 A - Port 1]
        ↓
양쪽 모두 상대방 정보를 인접 테이블(Neighbor Table)에 저장
```

## 주요 수집 정보

| 항목 | 설명 |
|------|------|
| Chassis ID | 상대 장비 식별자 (MAC 등) |
| Port ID | 상대 장비에서 연결된 포트 |
| System Name | 상대 장비 이름 |
| TTL | 정보 유효 시간 |

## 명령어 예시

```bash
# 스위치에서 LLDP 이웃 목록 확인 (Cisco 계열)
show lldp neighbors detail

# 리눅스 호스트에서 lldpd 사용
lldpctl
```

## 랙 토폴로지에서의 활용

[물리 네트워크 토폴로지]({% post_url 2026-06-30-rack-topology %}) 기능에서 "서버 ↔ 스위치 포트 연결 매핑"을 자동으로 채우는 데 LLDP를 사용합니다. IPMI/SNMP가 각 장비의 개별 상태를 조회하는 프로토콜이라면, LLDP는 장비 간 실제 물리 연결 관계 자체를 알려주기 때문에 포트 단위 배선도를 수작업 입력 없이 자동으로 구성할 수 있습니다.

## 세 프로토콜의 역할 비교

| 프로토콜 | 계층 | 주 용도 | 랙 토폴로지에서의 역할 |
|----------|------|---------|------------------------|
| [IPMI]({% post_url 2026-06-30-rack-topology-ipmi %}) | 하드웨어 (BMC) | 전원 / 센서 / 이벤트 관리 | 서버 전원·하드웨어 상태 |
| [SNMP]({% post_url 2026-06-30-rack-topology-snmp %}) | 네트워크 장비 상태 | 포트 상태 / 트래픽 조회 | 스위치 상태·트래픽 수집 |
| LLDP | L2 (링크) | 인접 장비 자동 탐지 | 포트 단위 연결 관계(배선도) 자동 구성 |

## 기술 스택

| 역할 | 기술 |
|------|------|
| 프로토콜 | LLDP |
| 조회 도구 | lldpctl, show lldp neighbors |
| 연동 대상 | 스위치, 서버 NIC |
