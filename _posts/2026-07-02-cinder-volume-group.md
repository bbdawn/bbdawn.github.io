---
title: "[Feature] Cinder 볼륨 그룹(Volume Group) 관리 기능 개발"
date: 2026-07-02 10:00:00 +0900
categories: [Feature, Openstack]
subcategory: Feature
tags: [openstack, cinder, volume-group, storage, java, spring]
---

## 개요

Cinder의 볼륨 그룹(Volume Group) 기능을 콘트라베이스에서 생성·관리할 수 있도록 CRUD API 연동을 개발했습니다.

## 왜 볼륨 그룹인가

볼륨을 하나씩 개별로 다루면, 여러 볼륨이 묶여야 의미가 있는 작업(그룹 단위 스냅샷, 그룹 단위 백업 등)에서 볼륨 간 정합성을 보장하기 어렵습니다. Cinder의 Volume Group은 여러 볼륨을 하나의 그룹으로 묶어 **일관된 시점(consistency point)** 기준으로 다룰 수 있게 해줍니다.

```
Volume Group
├── Volume A
├── Volume B
└── Volume C
     ↓
그룹 단위로 스냅샷/백업 시 세 볼륨이 같은 시점 기준으로 처리됨
```

---

## 개발 범위

Cinder API를 통해 아래 CRUD를 콘트라베이스 UI에서 제공하도록 구현했습니다.

| 기능 | 설명 |
|------|------|
| 그룹 생성 | Volume Group Type 지정 후 그룹 생성 |
| 볼륨 추가/제거 | 기존 볼륨을 그룹에 편입하거나 그룹에서 제외 |
| 그룹 조회 | 그룹에 속한 볼륨 목록 및 상태 조회 |
| 그룹 삭제 | 그룹 및 그룹 내 볼륨 처리 옵션 포함 삭제 |

```java
// Volume Group 생성 예시
VolumeGroup group = cinderClient.createVolumeGroup(
    VolumeGroupCreateRequest.builder()
        .name(groupName)
        .groupType(groupTypeId)
        .volumeTypes(volumeTypeIds)
        .build()
);

// 기존 볼륨을 그룹에 추가
cinderClient.updateVolumeGroup(group.getId(),
    VolumeGroupUpdateRequest.builder()
        .addVolumes(List.of(volumeId))
        .build()
);
```

---

## 이후 활용: 인스턴스 복제 기능의 기반

이 볼륨 그룹 기능은 이후 개발된 **인스턴스 복제(Instance Clone)** 기능에서 기반 자원으로 사용되었습니다. 인스턴스에 연결된 여러 볼륨을 그룹 단위로 묶어 일관된 시점에 복제해야 했기 때문입니다.

> 인스턴스 복제 기능 자체는 제가 기술 검토(설계 방향 및 Volume Group 활용 방안 검토)까지 참여했고, 실제 구현은 다른 팀원이 진행했습니다.

---

## 기술 스택

| 역할 | 기술 |
|------|------|
| 백엔드 | Java, Spring Boot |
| 인프라 연동 | OpenStack Cinder API |
