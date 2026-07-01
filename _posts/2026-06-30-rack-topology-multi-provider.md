---
title: "[Feature] 멀티 공급자(Multi-Provider) 자원 공유"
date: 2026-06-30 13:00:00 +0900
categories: [Project, Infra]
subcategory: Feature
tags: [rack, topology, multi-provider, snmp, ipmi, openstack, java, spring]
---

## 개요

데이터센터 랙 토폴로지 시스템에서 **공급자(Provider) 간 자원 중복 등록 문제**를 해결하는 기능입니다.

---

## 계층 구조

자원은 아래와 같은 4단계 계층으로 구성됩니다.

```
Provider (공급자)
└── Datacenter (데이터센터)
    └── Rack (랙)
        └── Resource (서버 / 스위치 / 스토리지)
```

---

## 문제 상황

기존에는 Switch가 **여러 공급자에 중복 등록**될 수 있었습니다.

동일한 물리 스위치가 여러 공급자에 각각 등록되면:

- 같은 스위치에 SNMP 요청이 공급자 수만큼 반복 발생
- 불필요한 네트워크 부하 및 스위치 응답 지연
- 데이터 일관성 문제 (공급자마다 다른 수집 결과)

---

## 해결 방법: 자원 공유 설계

Switch를 **한 번만 등록**하고, 다른 공급자에서는 **불러오기(import)** 방식으로 재사용합니다.

```
[공급자 A]              [공급자 B]
  Rack A                 Rack B
  └── Switch X  ←─────── Switch X (불러오기)
       (원본)              (참조)
```

- Switch는 원본 공급자에 1회만 등록
- 다른 공급자는 동일 Switch를 참조로 가져와 사용
- SNMP 수집은 원본 등록 기준으로 1회만 수행

---

## 공급자 상태 관리

공급자 단위로 **활성화 / 비활성화 / 삭제** 상태를 관리합니다.

| 상태 | 동작 |
|------|------|
| 활성화 | SNMP/IPMI 수집 대상 포함, 하위 자원 정상 동작 |
| 비활성화 | 수집 대상에서 제외, 하위 자원 상태 일괄 비활성화 |
| 삭제 | 공급자 및 하위 자원 전체 제거 |

### 비활성화 → 활성화 복구

공급자를 비활성화했다가 다시 활성화할 때, 하위 계층 전체를 일괄 복구합니다.

```
Provider 활성화
└── Datacenter 상태 복구
    └── Rack 상태 복구
        └── Resource 상태 복구
```

재귀적으로 하위 자원을 순회하며 상태를 갱신하므로, 수동으로 각 자원을 복구할 필요가 없습니다.

---

## DB 설계 포인트

비활성화된 공급자의 자원은 SNMP/IPMI 수집 쿼리에서 **자동으로 제외**됩니다.

```sql
-- 수집 대상 조회 시 공급자 활성 상태 조인
SELECT r.*
FROM resource r
JOIN rack rk ON r.rack_id = rk.id
JOIN datacenter dc ON rk.datacenter_id = dc.id
JOIN provider p ON dc.provider_id = p.id
WHERE p.status = 'ACTIVE'
```

공급자 상태 컬럼 하나로 하위 전체 자원의 수집 포함 여부를 제어할 수 있습니다.

---

## 기술 스택

| 역할 | 기술 |
|------|------|
| 백엔드 | Java, Spring Boot, JPA |
| 데이터베이스 | MySQL |
| 인프라 연동 | SNMP, IPMI |
