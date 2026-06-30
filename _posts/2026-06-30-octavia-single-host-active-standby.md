---
title: "[Troubleshooting] 단일 호스트 환경에서 Amphora active-standby 설정 문제"
date: 2026-06-30 12:00:00 +0900
categories: [Study, Openstack]
subcategory: Troubleshooting
tags: [openstack, octavia, loadbalancer, amphora, active-standby, troubleshooting]
---

## 문제 상황

Octavia 로드밸런서를 `ACTIVE_STANDBY` 토폴로지로 생성했을 때, Amphora 인스턴스가 `PENDING_CREATE` 상태에서 멈추거나 생성에 실패하는 문제가 발생했다.

환경은 컴퓨트 노드가 **단일 호스트** 하나뿐인 상태였다.

---

## 원인 분석

Octavia의 `ACTIVE_STANDBY` 토폴로지는 기본적으로 **anti-affinity** 정책을 적용한다.

Active Amphora와 Standby Amphora를 **서로 다른 컴퓨트 노드**에 배치해 고가용성을 보장하는 방식인데, 컴퓨트 노드가 1개뿐이면 두 번째 Amphora의 스케줄링이 실패한다.

```
[Nova Scheduler]
  Active Amphora  → 호스트 A ✅
  Standby Amphora → 호스트 A 배치 시도 → anti-affinity 위반 → 스케줄링 실패 ❌
```

Octavia는 스케줄링 실패 시 재시도하다가 결국 로드밸런서를 `ERROR` 상태로 전환한다.

---

## 해결 방법

`octavia.conf`에서 anti-affinity 옵션을 비활성화한다.

```ini
# /etc/octavia/octavia.conf

[nova]
enable_anti_affinity = False
```

설정 변경 후 Octavia 서비스를 재시작한다.

```bash
systemctl restart octavia-worker
systemctl restart octavia-housekeeping
systemctl restart octavia-health-manager
```

---

## 적용 결과

anti-affinity를 비활성화하면 Active/Standby Amphora 모두 동일한 컴퓨트 노드에 배치되며 로드밸런서가 정상 생성된다.

```
[Nova Scheduler]
  Active Amphora  → 호스트 A ✅
  Standby Amphora → 호스트 A ✅ (anti-affinity 비활성화로 허용)
```

---

## 참고

단일 호스트 환경에서는 Active/Standby가 같은 노드에 올라가므로, 호스트 자체 장애에 대한 HA는 보장되지 않는다. 운영 환경에서는 컴퓨트 노드를 2개 이상 구성하는 것을 권장한다.
