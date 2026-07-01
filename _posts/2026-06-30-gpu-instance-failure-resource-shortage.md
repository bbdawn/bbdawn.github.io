---
title: "[Troubleshooting] GPU Instance 생성 실패 원인(1) : GPU 자원 부족"
date: 2026-06-30 06:00:00 +0900
categories: [Troubleshooting, GPU]
subcategory: Troubleshooting
tags: [openstack, gpu, nova, placement, troubleshooting]
---

OpenStack에서 GPU 인스턴스 생성 시 `No valid host was found` 오류가 발생했습니다. 가장 먼저 의심해야 할 원인은 GPU 자원 자체가 부족한 경우입니다.

## 원인

GPU 호스트의 물리 슬롯이 모두 사용 중이거나, 다른 인스턴스에 이미 할당된 경우입니다.

```
nova-scheduler → No valid host found
  └── 모든 Compute 노드의 GPU 리소스 할당량 초과
```

## 확인 방법

```bash
# Placement에서 GPU 리소스 잔여량 확인
openstack resource provider list
openstack resource provider inventory list <provider-uuid>

# 현재 GPU 사용 현황 (DCGM)
dcgmi group -l
nvidia-smi
```

## 해결

- 기존 인스턴스 삭제 또는 GPU 반납 후 재시도
- GPU 호스트 추가 증설 후 nova-compute 재시작

---

## 다른 원인들

- **(2)**: [Flavor extra_specs 설정 오류]({% post_url 2026-06-30-gpu-instance-failure-flavor-config %})
- **(3)**: [PCI Alias 불일치]({% post_url 2026-06-30-gpu-instance-failure-pci-alias-mismatch %})
- **(4)**: [Placement 리소스 클래스 미등록]({% post_url 2026-06-30-gpu-instance-failure-placement-missing %})
- **(5)**: [MIG Profile 부족]({% post_url 2026-06-30-gpu-instance-failure-mig-profile-shortage %})
