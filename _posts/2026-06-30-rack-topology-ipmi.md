---
title: "[Study] IPMI (Intelligent Platform Management Interface) 개념 정리"
date: 2026-06-30 14:00:00 +0900
categories: [Study, Infra]
subcategory: Study
tags: [rack, topology, ipmi, bmc, out-of-band, hardware-monitoring]
---

[데이터센터 랙 토폴로지 시각화 프로젝트]({% post_url 2026-06-30-rack-topology %})에서 서버의 전원·하드웨어 상태를 수집하는 데 사용한 IPMI를 정리합니다.

## 개요

IPMI(Intelligent Platform Management Interface)는 서버 OS의 동작 여부와 상관없이 하드웨어 상태를 관리·모니터링할 수 있는 표준 인터페이스입니다.

## 핵심 개념

- **BMC (Baseboard Management Controller)**: 메인보드에 내장된 관리 전용 칩. OS와 독립적으로 동작
- **Out-of-Band 관리**: 서버가 꺼져 있거나 OS가 응답하지 않아도 네트워크로 원격 접근 가능
- **IPMI over LAN**: BMC의 전용 네트워크 인터페이스를 통해 원격에서 질의/제어

```
[관리 PC] --IPMI over LAN--> [BMC] --내부 버스--> [센서 / 전원 / 팬 등 하드웨어]
```

## 주요 수집 정보

| 항목 | 설명 |
|------|------|
| 전원 상태 | On / Off / 재부팅 여부 |
| 센서 | 온도, 팬 속도, 전압 |
| SEL (System Event Log) | 하드웨어 오류/이벤트 이력 |
| FRU 정보 | 제조사, 모델명, 시리얼 번호 |

## 명령어 예시

```bash
# 전원 상태 확인
ipmitool -I lanplus -H <bmc-ip> -U <user> -P <pass> power status

# 센서 목록 조회
ipmitool -I lanplus -H <bmc-ip> -U <user> -P <pass> sensor list

# FRU(하드웨어 정보) 조회
ipmitool -I lanplus -H <bmc-ip> -U <user> -P <pass> fru print
```

## 랙 토폴로지에서의 활용

서버를 랙에 등록할 때 IPMI(BMC) 접속 정보를 함께 저장하고, 주기적으로 조회해 전원 상태와 하드웨어 상태를 랙 구성도에 반영합니다. 서버가 물리적으로 꺼져 있어도 상태를 확인할 수 있다는 점이 SNMP만으로는 얻기 힘든 IPMI의 강점입니다.

## 기술 스택

| 역할 | 기술 |
|------|------|
| 프로토콜 | IPMI 2.0 (IPMI over LAN) |
| 조회 도구 | ipmitool |
| 연동 대상 | 서버 BMC |
