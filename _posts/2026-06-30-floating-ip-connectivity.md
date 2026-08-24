---
title: "[Feature] Floating IP 연동 가능 여부 표출"
date: 2026-06-30 14:00:00 +0900
categories: [Features, Openstack]
subcategory: Feature
tags: [openstack, neutron, floating-ip, network, router, java, spring]
---

## 문제 상황

OpenStack Neutron에서 Floating IP를 포트에 연동할 때, **라우터가 연결되지 않은 네트워크의 포트**와 연동을 시도하면 실패합니다.

기존 UI에서는 포트 목록을 그대로 보여줬기 때문에, 사용자가 연동 불가능한 포트를 선택해 오류를 만나는 상황이 반복됐습니다.

![유동 IP 자원 연동 화면 - 연동 가능 여부 표출](/assets/img/posts/floating-ip-connectivity-associate-dialog.png)
_유동 IP 자원 연동 화면에서 고정 IP별 연동 가능 여부를 표시한 예시_

---

## 연동 가능 조건 분석

Floating IP 연동이 성공하려면 아래 경로가 모두 연결되어 있어야 합니다.

```
[External Network]
       ↕  (External Gateway)
   [Router]
       ↕  (Router Interface)
   [Network]
       ↕
    [Port]  ← FloatingIP 연동 대상
```

### 조건 체크리스트

| 조건 | 확인 방법 |
|------|----------|
| 포트가 속한 네트워크 확인 | `port.network_id` |
| 해당 네트워크에 라우터 인터페이스 존재 여부 | `router interface` 조회 |
| 라우터에 External Gateway 설정 여부 | `router.external_gateway_info` |

세 조건을 모두 만족하는 포트만 Floating IP 연동이 가능합니다.

---

## 구현 방법

### 1. 라우팅 가능한 네트워크 목록 추출

Neutron API를 통해 External Gateway가 설정된 라우터 목록을 조회하고, 그 라우터들의 인터페이스(`network:router_interface`) 포트가 속한 네트워크 ID를 모아 "라우팅 가능한 네트워크" 집합을 만듭니다.

### 2. 포트 목록 조회 시 연동 가능 여부 표시

포트 목록을 반환할 때, 각 포트의 네트워크 ID가 앞서 구한 라우팅 가능 네트워크 집합에 포함되는지를 `floatingIpAssociatable` 필드로 함께 내려줍니다.

### 3. 응답 구조

포트 응답에는 id, name, networkId, fixedIps와 함께 `floatingIpAssociatable`(연동 가능 여부, boolean) 필드가 포함됩니다.

---

## 연동 불가 케이스 정리

| 케이스 | 원인 |
|--------|------|
| 네트워크에 라우터 인터페이스 없음 | 라우터와 연결된 서브넷이 없어 외부 통신 불가 |
| 라우터에 External Gateway 미설정 | 라우터가 외부 네트워크로 나가는 경로 없음 |
| 포트가 External Network에 직접 연결됨 | Floating IP는 Internal Network 포트에만 연동 가능 |

---


## 효과

- 연동 불가 포트를 선택했을 때 발생하던 오류 사전 차단
- 사용자가 UI에서 연동 가능한 포트만 선택할 수 있도록 안내
- Neutron API 호출 실패로 인한 불필요한 에러 로그 감소


