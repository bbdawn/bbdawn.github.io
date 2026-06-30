---
title: "[Feature] 데이터센터 랙 토폴로지 시각화 프로젝트"
date: 2026-06-30 03:00:00 +0900
categories: [Project, Infra]
subcategory: Features
tags: [rack, topology, visualization, openstack, snmp, java, spring]
---

## 개요

데이터센터의 물리/가상 인프라 구조를 시각화하는 프로젝트입니다.

---

## 주요 기능

### 랙 구성도

- Server, Storage, Switch 등 물리 자원 등록 및 관리
- 랙 단위 자원 배치 시각화

### 물리 네트워크 토폴로지

서버와 스위치 간 포트 단위 연결을 시각화합니다.

```
서버 ↔ 스위치 포트 연결 매핑
```

IPMI/SNMP 수집 데이터를 활용해 실제 연결 상태를 반영합니다.

### 가상 네트워크 구성도

OpenStack 기반으로 VLAN별 연결된 인스턴스를 매핑·시각화합니다.

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
