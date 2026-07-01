---
title: "[Troubleshooting] GPU Instance 생성 실패 원인"
date: 2026-06-30 06:00:00 +0900
categories: [Troubleshooting, GPU]
subcategory: Troubleshooting
tags: [openstack, gpu, nova, placement, mig, pci, flavor, troubleshooting]
---

## 개요

OpenStack에서 GPU 인스턴스 생성 시 `No valid host was found` 또는 BUILD 상태에서 멈추는 오류가 반복되었습니다. 원인은 단일하지 않고 여러 계층에 걸쳐 있었으며, 아래 5가지 케이스로 분류할 수 있었습니다.

---

## 1. GPU 자원 부족

### 원인

GPU 호스트의 물리 슬롯이 모두 사용 중이거나, 다른 인스턴스에 이미 할당된 경우입니다.

```
nova-scheduler → No valid host found
  └── 모든 Compute 노드의 GPU 리소스 할당량 초과
```

### 확인 방법

```bash
# Placement에서 GPU 리소스 잔여량 확인
openstack resource provider list
openstack resource provider inventory list <provider-uuid>

# 현재 GPU 사용 현황 (DCGM)
dcgmi group -l
nvidia-smi
```

### 해결

- 기존 인스턴스 삭제 또는 GPU 반납 후 재시도
- GPU 호스트 추가 증설 후 nova-compute 재시작

---

## 2. Flavor 설정 오류

### 원인

Flavor의 `extra_specs`에 `pci_passthrough:alias` 항목이 없거나 잘못된 값으로 설정된 경우입니다. nova-scheduler가 GPU를 요구하는 Flavor임을 인식하지 못해 일반 호스트에 배치를 시도합니다.

```
nova-scheduler → GPU 없는 호스트에 배치 시도 → 실패
```

### 확인 방법

```bash
openstack flavor show <flavor-id> --format json | grep extra_specs
```

올바른 extra_specs 예시:

```
pci_passthrough:alias = gpu-a100:1
```

### 해결

```bash
openstack flavor set <flavor-id> \
  --property pci_passthrough:alias="gpu-a100:1"
```

alias 이름은 `nova.conf`의 `[pci] alias` 설정과 정확히 일치해야 합니다.

---

## 3. PCI Alias 불일치

### 원인

Flavor의 `pci_passthrough:alias` 값과 Compute 노드 `nova.conf`의 alias 이름이 다른 경우입니다.

```
Flavor extra_specs:  pci_passthrough:alias = "gpu-a100:1"
nova.conf alias:     name = "gpu_a100"          ← 이름 불일치
```

nova-scheduler는 alias 이름으로 호스트를 탐색하기 때문에, 불일치 시 유효한 호스트를 찾지 못합니다.

### 확인 방법

```bash
# Compute 노드에서 확인
grep -A5 "\[pci\]" /etc/nova/nova.conf
```

```ini
[pci]
alias = {"name": "gpu-a100", "product_id": "20b0", "vendor_id": "10de", "device_type": "type-PCI"}
```

### 해결

Flavor의 alias 값과 `nova.conf`의 `name` 필드를 동일하게 맞춥니다.

```bash
# nova.conf 수정 후 nova-compute 재시작
sudo systemctl restart nova-compute
```

---

## 4. Placement 정보 미등록

### 원인

Compute 노드가 GPU를 보유하고 있어도, Placement 서비스에 해당 리소스 클래스가 등록되지 않은 경우입니다. nova-scheduler는 Placement에서 할당 가능한 리소스를 조회하기 때문에, 등록이 누락되면 후보 호스트 목록에서 제외됩니다.

```
nova-scheduler
  → Placement API 조회
    → GPU 리소스 클래스 없음
      → 해당 Compute 노드 후보 제외
```

### 확인 방법

```bash
# Resource Provider 목록 확인
openstack resource provider list

# 특정 노드의 인벤토리 확인
openstack resource provider inventory list <provider-uuid>
```

GPU가 있는 노드에 `CUSTOM_GPU_*` 또는 `PGPU` 클래스가 없으면 미등록 상태입니다.

### 해결

```bash
# 리소스 클래스 수동 등록
openstack resource provider inventory set <provider-uuid> \
  --resource PGPU:total=4 \
  --resource PGPU:reserved=0 \
  --resource PGPU:min_unit=1 \
  --resource PGPU:max_unit=1 \
  --resource PGPU:step_size=1 \
  --resource PGPU:allocation_ratio=1.0
```

nova-compute를 재시작하면 자동으로 Placement를 갱신하는 경우도 있습니다.

```bash
sudo systemctl restart nova-compute
```

---

## 5. MIG Profile 부족

### 원인

MIG(Multi-Instance GPU) 환경에서 요청한 MIG 프로파일(예: `1g.5gb`)이 GPU에 설정되어 있지 않거나, 이미 다른 인스턴스가 해당 슬라이스를 점유한 경우입니다.

```
요청: 1g.5gb MIG 슬라이스 1개
GPU 상태:
  └── 설정된 프로파일 없음 (또는 전부 사용 중)
      → No valid host found
```

### 확인 방법

```bash
# GPU에 설정된 MIG 프로파일 확인
nvidia-smi mig -lgip

# 현재 MIG 인스턴스 목록
nvidia-smi mig -lgi

# MIG 인스턴스 용량 잔여 확인
nvidia-smi
```

### 해결

**MIG 프로파일 신규 생성:**

```bash
# MIG 활성화 확인
nvidia-smi -i 0 --query-gpu=mig.mode.current --format=csv,noheader

# 프로파일 생성 (예: A100에서 1g.5gb 7개)
nvidia-smi mig -cgi 19,19,19,19,19,19,19 -C
```

**이미 생성된 슬라이스가 점유 중인 경우:**

OpenStack Placement에서 해당 리소스가 반납됐는지 확인합니다. mdev orphan 상태로 남아있는 경우 별도 정리가 필요합니다.

```bash
# orphan mdev 확인
ls /sys/bus/mdev/devices/

# Placement에서 사용량 확인
openstack resource provider usage show <provider-uuid>
```

---

## 케이스 요약

| 원인 | 주요 증상 | 확인 명령어 |
|------|----------|-------------|
| GPU 자원 부족 | No valid host | `openstack resource provider inventory list` |
| Flavor 설정 오류 | 일반 호스트에 배치 시도 | `openstack flavor show` |
| PCI Alias 불일치 | 스케줄러 호스트 탐색 실패 | `nova.conf` alias 확인 |
| Placement 미등록 | 후보 호스트 없음 | `openstack resource provider list` |
| MIG Profile 부족 | MIG 슬라이스 없음 | `nvidia-smi mig -lgip` |
