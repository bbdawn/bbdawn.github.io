---
title: "[Troubleshooting] Loadbalancer 생성 실패 원인(1) : Anti-affinity instance group policy was violated"
date: 2026-06-30 16:00:00 +0900
categories: [Troubleshooting, Openstack]
subcategory: Troubleshooting
tags: [openstack, nova, anti-affinity, server-group, scheduler, amphora, octavia, troubleshooting]
---

## 오류 메시지

```
No valid host was found.
There are not enough hosts available.
Anti-affinity instance group policy was violated.
```

Octavia Amphora 인스턴스 또는 일반 VM을 생성할 때 nova-scheduler가 위 메시지와 함께 인스턴스 생성을 거부합니다.

---

## 원인

### Anti-affinity 정책이란

Nova **Server Group**의 `anti-affinity` 정책은 같은 그룹 안의 VM들이 **반드시 서로 다른 Compute 호스트에 배치**되도록 강제합니다.

```
Server Group (policy: anti-affinity)
├── VM-A  →  compute-node-01
├── VM-B  →  compute-node-02
└── VM-C  →  compute-node-03  ← 반드시 다른 호스트
```

### 실패하는 경우

그룹 내 VM 수가 사용 가능한 Compute 호스트 수를 초과하면 nova-scheduler가 정책을 만족할 수 없어 스케줄링을 거부합니다.

```
Server Group VM 수  >  사용 가능한 Compute 호스트 수
       3개          >          2개
                   → 스케줄링 실패
```

### Octavia에서 자주 발생하는 패턴

Octavia는 Loadbalancer의 Active-Standby를 위해 두 Amphora 인스턴스를 `anti-affinity` 서버 그룹으로 묶습니다. Compute 호스트가 1대뿐인 환경에서는 Standby Amphora를 배치할 다른 호스트가 없어 반복적으로 실패합니다.

```
[정상 환경]
Amphora-Active   → compute-node-01
Amphora-Standby  → compute-node-02  ← 다른 호스트

[단일 호스트 환경]
Amphora-Active   → compute-node-01
Amphora-Standby  → compute-node-01  ← 불가 (anti-affinity 위반)
```

---

## 확인 방법

### 1. 서버 그룹 목록 및 정책 확인

```bash
openstack server group list
openstack server group show <group-id>
```

```
+----------+------------------+-----------------+
| Id       | Name             | Policies        |
+----------+------------------+-----------------+
| xxxx-... | loadbalancer-xxx | anti_affinity   |
+----------+------------------+-----------------+
```

### 2. 그룹 내 VM 수 및 배치 호스트 확인

```bash
# 그룹에 속한 인스턴스 목록
openstack server group show <group-id> -f json | jq '.members'

# 각 인스턴스의 배치 호스트 확인
openstack server show <instance-id> -c OS-EXT-SRV-ATTR:host
```

### 3. 사용 가능한 Compute 호스트 수 확인

```bash
openstack compute service list --service nova-compute | grep enabled
```

그룹 내 VM 수가 `enabled` 상태의 Compute 호스트 수보다 많으면 배치 실패합니다.

---

## 해결 방법

### 방법 1. Compute 호스트 추가 (근본 해결)

가장 올바른 해결 방법입니다. Compute 노드를 추가하면 nova-scheduler가 정책을 만족할 수 있습니다.

```bash
# 새 Compute 노드 추가 후 서비스 확인
openstack compute service list --service nova-compute
```

### 방법 2. Octavia — anti-affinity 정책 완화 또는 비활성화

단일 호스트 환경에서 Octavia를 운영해야 하는 경우 `/etc/octavia/octavia.conf`를 수정합니다.

**옵션 A — soft-anti-affinity (권장)**: 가능하면 다른 호스트에 배치하되, 불가능해도 스케줄링 자체는 실패하지 않습니다.

```ini
[nova]
anti_affinity_policy = soft-anti-affinity
```

**옵션 B — anti-affinity 완전 비활성화**: Active/Standby 모두 동일한 호스트에 올라가는 것을 허용합니다.

```ini
[nova]
enable_anti_affinity = False
```

```bash
# 설정 적용 후 Octavia 재시작
sudo systemctl restart octavia-worker octavia-health-manager octavia-housekeeping
```

> 단일 호스트 환경에서는 Active/Standby가 같은 노드에 올라가므로 호스트 자체 장애에 대한 HA는 보장되지 않습니다. 운영 환경에서는 Compute 노드를 2개 이상 구성하는 것을 권장합니다.

### 방법 3. 기존 서버 그룹의 VM 정리

그룹 내에 더 이상 필요하지 않은 VM이 있다면 삭제하여 슬롯을 확보합니다.

```bash
# 그룹 내 인스턴스 확인
openstack server group show <group-id>

# 불필요한 인스턴스 삭제
openstack server delete <instance-id>
```

### 방법 4. anti-affinity 없이 서버 그룹 재생성

HA가 불필요한 환경이라면 정책 없이 서버 그룹을 새로 만들거나 그룹 자체를 사용하지 않습니다.

```bash
# affinity 정책으로 새 그룹 생성 (같은 호스트 허용)
openstack server group create --policy affinity <group-name>
```

---

## 케이스 요약

| 상황 | 원인 | 해결 |
|------|------|------|
| Compute 호스트 부족 | VM 수 > 호스트 수 | 호스트 추가 |
| 단일 호스트 Octavia | anti-affinity 강제 | soft-anti-affinity 또는 비활성화 |
| 불필요한 VM 점유 | 그룹 슬롯 소진 | 기존 VM 정리 |
| HA 불필요 환경 | 정책 자체가 불필요 | affinity 또는 그룹 미사용 |
