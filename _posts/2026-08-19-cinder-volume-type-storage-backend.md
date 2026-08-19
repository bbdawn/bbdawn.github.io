---
title: "[Study] Volume Type과 스토리지 백엔드의 연관관계 — Cinder는 Backend를 어떻게 고르는가"
date: 2026-08-19 09:00:00 +0900
categories: [Study, Openstack]
subcategory: Study
tags: [openstack, cinder, volume-type, storage-backend, scheduler]
---

Cinder에 스토리지 백엔드를 여러 개(Ceph, PowerFlex 등) 붙여서 운영하다 보면, "사용자가 볼륨을 생성할 때 어떤 백엔드로 가는지"를 결정하는 기준이 궁금해집니다. 이걸 정리하다 보니 핵심은 결국 **Volume Type**이었습니다.

## Volume Type이 백엔드를 결정하는 원리

Volume Type은 어떤 스토리지 백엔드에서 볼륨을 생성할지 결정하는 기준입니다. 사용자가 직접 백엔드를 지정하는 게 아니라, Volume Type에 붙은 `extra_specs`를 Cinder Scheduler가 보고 백엔드를 골라줍니다.

```bash
사용자
  │
  ▼
Volume Type
  │
  │ extra_specs
  ▼
Cinder Scheduler
  │
  │ Backend 선택
  ▼
Storage Backend
  │
  ├── Ceph
  ├── LVM
  ├── Dell PowerFlex
  ├── NetApp
  └── 기타
       │
       ▼
     Volume
```

## 예시: 백엔드 2개를 Volume Type으로 매핑하기

스토리지 백엔드가 2개 있다고 가정해보겠습니다.

```bash
Cinder
 ├── Backend A → Ceph
 └── Backend B → PowerFlex
```

여기에 Volume Type을 두 개 만듭니다.

```text
volume-type-a
volume-type-b
```

그리고 각 Volume Type의 `extra_specs`에 아래처럼 백엔드를 매핑합니다.

```text
volume-type-a
 └── storage_backend=powerflex

volume-type-b
 └── storage_backend=ceph
```

이렇게 구성하면 사용자가 `volume-type-a`로 볼륨을 생성하면 **PowerFlex 백엔드**에, `volume-type-b`로 생성하면 **Ceph 백엔드**에 볼륨이 생성됩니다. 즉 사용자 입장에서는 스토리지 종류를 몰라도, Volume Type만 골라 쓰면 되는 구조입니다.

---

## `--default--` Volume Type

`--default--`는 Cinder에서 특별하게 취급되는 기본 Volume Type입니다. 사용자가 Volume Type을 명시하지 않고 볼륨을 생성하면 이 타입이 적용됩니다.

### Cinder 내부 동작 흐름

```bash
API
 │
 │ create volume
 ▼
cinder-api
 │
 ▼
cinder-scheduler
 │
 │ Volume Type의 extra_specs 확인
 │
 │ Backend 상태/용량/Capabilities 확인
 ▼
Backend 선택
 │
 ▼
cinder-volume
 │
 ▼
Storage Driver
 │
 ▼
실제 Storage
```

`cinder.conf`에서 `enabled_backends`와 각 백엔드 섹션의 `volume_backend_name`을 확인하면, 어떤 백엔드들이 후보로 등록돼 있는지 알 수 있습니다.

```ini
[DEFAULT]
enabled_backends = ceph, powerflex

[ceph]
volume_backend_name = ceph
volume_driver = cinder.volume.drivers.rbd.RBDDriver

[powerflex]
volume_backend_name = powerflex
volume_driver = cinder.volume.drivers.dell_emc.powerflex.driver.PowerFlexDriver
```

### 중요한 포인트

`--default--` 타입에 아래처럼 조건이 걸려 있으면:

```text
volume_backend_name = powerflex
```

Backend 후보가 그 하나로 **제한**됩니다.

```text
--default--
     │
     ▼
powerflex 조건
     │
     ▼
PowerFlex Backend
```

반대로 아무 조건이 없다면, Scheduler가 등록된 백엔드들을 상태·용량·Capabilities 기준으로 직접 필터링해서 선택합니다.

```text
--default--
     │
     ▼
특정 Backend 제한 없음
     │
     ▼
Scheduler가 Backend들을 Filter
     │
     ├── Ceph      ❌ 용량 부족
     ├── PowerFlex ✅
     └── LVM       ❌ 조건 불충족
              ↓
          PowerFlex 선택
```

> 여기서 최종적으로 백엔드를 **"결정"하는 주체는 `cinder-volume`이 아니라 `cinder-scheduler`**라는 점을 기억해두면 헷갈리지 않습니다. `cinder-volume`은 이미 결정된 백엔드에서 실제 프로비저닝만 수행합니다.

---

## 정리

| 개념 | 한 줄 설명 |
|------|-----------|
| Volume Type | 어떤 백엔드에서 볼륨을 만들지 정하는 기준 |
| extra_specs | Volume Type에 붙는 조건 (예: storage_backend=ceph) |
| cinder-scheduler | extra_specs와 백엔드 상태를 보고 실제로 백엔드를 결정하는 주체 |
| cinder-volume | 결정된 백엔드에서 볼륨을 실제로 생성(프로비저닝)하는 컴포넌트 |
| `--default--` | Volume Type을 지정하지 않았을 때 적용되는 기본 타입 |

## 기술 스택

| 역할 | 기술 |
|------|------|
| 스토리지 오케스트레이션 | Cinder (cinder-api, cinder-scheduler, cinder-volume) |
| 스토리지 백엔드 | Ceph, Dell PowerFlex, LVM, NetApp 등 |
| 플랫폼 | OpenStack |
