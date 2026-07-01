---
title: "[Study] SNMP (Simple Network Management Protocol) 개념 정리"
date: 2026-06-30 14:10:00 +0900
categories: [Study, Infra]
subcategory: Study
tags: [rack, topology, snmp, network-monitoring, switch]
---

[데이터센터 랙 토폴로지 시각화 프로젝트]({% post_url 2026-06-30-rack-topology %})에서 스위치 상태와 트래픽을 수집하는 데 사용한 SNMP를 정리합니다.

## 개요

SNMP(Simple Network Management Protocol)는 네트워크 장비(스위치, 라우터 등)의 상태 정보를 표준화된 방식으로 조회하는 프로토콜입니다.

## 핵심 개념

- **MIB (Management Information Base)**: 장비가 제공하는 값들을 트리 구조로 정의한 문서/스펙
- **OID (Object Identifier)**: MIB 내 특정 값을 가리키는 고유 주소 (예: `1.3.6.1.2.1.1.1.0`)
- **SNMP 버전**: v1/v2c(커뮤니티 스트링 기반 인증), v3(사용자 인증 + 암호화 지원)

```
[관리 서버] --SNMP GET / WALK (OID)--> [스위치 SNMP Agent] --조회--> [MIB 값 반환]
```

## 주요 수집 정보

| 항목 | OID 예시 |
|------|---------|
| 시스템 정보 | `1.3.6.1.2.1.1` (sysDescr 등) |
| 인터페이스 상태 | `1.3.6.1.2.1.2.2` (ifTable) |
| 포트별 트래픽 | ifInOctets / ifOutOctets |
| 연결 상태 (Bridge MIB) | dot1dTpFdbTable 등 |

## 명령어 예시

```bash
# 스위치 시스템 정보 조회
snmpget -v2c -c public <switch-ip> 1.3.6.1.2.1.1.1.0

# 인터페이스 목록 전체 조회
snmpwalk -v2c -c public <switch-ip> 1.3.6.1.2.1.2.2.1.2

# 포트별 트래픽 조회
snmpwalk -v2c -c public <switch-ip> 1.3.6.1.2.1.2.2.1.10
```

## 랙 토폴로지에서의 활용

스위치를 랙에 등록하면 주기적으로 SNMP로 포트 상태와 트래픽을 수집해 화면에 반영합니다. [멀티 공급자 자원 공유]({% post_url 2026-06-30-rack-topology-multi-provider %}) 글에서 다뤘듯, 동일 스위치가 여러 공급자에 중복 등록되면 SNMP 요청도 그만큼 중복 발생하기 때문에 스위치를 한 번만 등록하고 참조하는 방식으로 SNMP 수집 부하를 줄였습니다.

## 기술 스택

| 역할 | 기술 |
|------|------|
| 프로토콜 | SNMP v2c |
| 조회 도구 | snmpget, snmpwalk |
| 연동 대상 | 스위치 |
