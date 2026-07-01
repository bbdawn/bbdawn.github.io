---
title: "[Troubleshooting] GPU Instance 생성 실패 원인(5) : MIG Profile 부족"
date: 2026-06-30 06:40:00 +0900
categories: [Troubleshooting, GPU]
subcategory: Troubleshooting
tags: [openstack, gpu, mig, nova, placement, troubleshooting]
---

Passthrough 계열 원인을 모두 확인했는데도 실패한다면, MIG 환경에서는 프로파일 자체가 부족한 경우를 확인해야 합니다.

## 원인

MIG(Multi-Instance GPU) 환경에서 요청한 MIG 프로파일(예: `1g.5gb`)이 GPU에 설정되어 있지 않거나, 이미 다른 인스턴스가 해당 슬라이스를 점유한 경우입니다.

```
요청: 1g.5gb MIG 슬라이스 1개
GPU 상태:
  └── 설정된 프로파일 없음 (또는 전부 사용 중)
      → No valid host found
```

## 확인 방법

```bash
# GPU에 설정된 MIG 프로파일 확인
nvidia-smi mig -lgip

# 현재 MIG 인스턴스 목록
nvidia-smi mig -lgi

# MIG 인스턴스 용량 잔여 확인
nvidia-smi
```

## 해결

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
| (1) GPU 자원 부족 | No valid host | `openstack resource provider inventory list` |
| (2) Flavor 설정 오류 | 일반 호스트에 배치 시도 | `openstack flavor show` |
| (3) PCI Alias 불일치 | 스케줄러 호스트 탐색 실패 | `nova.conf` alias 확인 |
| (4) Placement 미등록 | 후보 호스트 없음 | `openstack resource provider list` |
| (5) MIG Profile 부족 | MIG 슬라이스 없음 | `nvidia-smi mig -lgip` |

---

## 다른 원인들

- **(1)**: [GPU 자원 부족]({% post_url 2026-06-30-gpu-instance-failure-resource-shortage %})
- **(2)**: [Flavor extra_specs 설정 오류]({% post_url 2026-06-30-gpu-instance-failure-flavor-config %})
- **(3)**: [PCI Alias 불일치]({% post_url 2026-06-30-gpu-instance-failure-pci-alias-mismatch %})
- **(4)**: [Placement 리소스 클래스 미등록]({% post_url 2026-06-30-gpu-instance-failure-placement-missing %})
