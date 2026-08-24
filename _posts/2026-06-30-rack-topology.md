---
title: "[Feature] 데이터센터 랙 토폴로지 시각화 프로젝트"
date: 2026-06-30 03:00:00 +0900
categories: [Project, Infra]
subcategory: Feature
tags: [rack, topology, visualization, openstack, snmp, java, spring]
---

## 개요

데이터센터의 물리/가상 인프라 구조를 시각화하는 프로젝트입니다.

---

## 주요 기능

### 랙 구성도

랙에서 호스트, 스위치, 스토리지의 위치 및 서버 상태 정보를 표시합니다.

- Server, Storage, Switch 등 물리 자원 등록 및 관리
- 랙 단위 자원 배치 시각화

![랙 구성도 화면 - 랙별 장비 배치 및 서버 상세 정보](/assets/img/posts/rack-topology-rack-view.png)
_랙 목록과 개별 장비의 상태·온도·팬 속도 등 상세 정보를 확인하는 랙 구성도 화면_

### 물리 네트워크 토폴로지

랙 구성도에 등록한 정보를 기반으로 호스트 및 스위치 연결정보를 표시합니다.

서버와 스위치 간 포트 단위 연결을 시각화합니다.

```
서버 ↔ 스위치 포트 연결 매핑
```

IPMI/SNMP 수집 데이터를 활용해 실제 연결 상태를 반영합니다.

![물리 네트워크 구성도 화면 - 스위치/장비 포트 연결 및 스위치 상세 정보](/assets/img/posts/rack-topology-physical-network-view.png)
_스위치와 서버 간 포트 연결을 시각화하고, 스위치 클릭 시 제조사·SNMP·APIC 등 상세 정보를 확인하는 화면_

### 가상 네트워크 구성도

물리 호스트에 설치된 Contrabass Engine에 구성된 가상 네트워크(VLAN) 기반 외부 네트워크에 존재하는 VM 정보를 시각적으로 표시합니다.

OpenStack 기반으로 VLAN별 연결된 인스턴스를 매핑·시각화합니다.

![가상 네트워크 구성도 화면 - 네트워크/프로젝트별 인스턴스 매핑 및 상세 정보](/assets/img/posts/rack-topology-virtual-network-view.png)
_네트워크 디렉토리 트리와 프로젝트별 인스턴스 현황을 함께 보여주고, 인스턴스 클릭 시 상태·고정 IP 등 상세 정보를 확인하는 화면_

### 멀티 공급자 자원 공유

계층 구조 내 자원 중복 등록 문제를 해결했습니다.

- 스위치 단일 등록으로 여러 공급자에서 재사용 가능
- 공급자 활성화/비활성화 시 하위 자원 상태 일괄 관리

---

## 기술 스택

| 역할 | 기술 |
|------|------|
| 백엔드 | Java, Spring Boot, JPA |
| 데이터베이스 | MySQL |
| 인프라 연동 | OpenStack, IPMI, SNMP |
