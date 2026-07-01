---
title: "[Troubleshooting] GPU Instance 생성 실패 원인(4) : Placement 리소스 클래스 미등록"
date: 2026-06-30 06:30:00 +0900
categories: [Troubleshooting, GPU]
subcategory: Troubleshooting
tags: [openstack, gpu, nova, placement, troubleshooting]
---

Flavor와 nova.conf의 alias가 모두 일치하는데도 실패한다면, Placement 서비스에 GPU 리소스 클래스가 등록되어 있는지 확인해야 합니다.

## 원인

Compute 노드가 GPU를 보유하고 있어도, Placement 서비스에 해당 리소스 클래스가 등록되지 않은 경우입니다. nova-scheduler는 Placement에서 할당 가능한 리소스를 조회하기 때문에, 등록이 누락되면 후보 호스트 목록에서 제외됩니다.

```
nova-scheduler
  → Placement API 조회
    → GPU 리소스 클래스 없음
      → 해당 Compute 노드 후보 제외
```

## 확인 방법

```bash
# Resource Provider 목록 확인
openstack resource provider list

# 특정 노드의 인벤토리 확인
openstack resource provider inventory list <provider-uuid>
```

GPU가 있는 노드에 `CUSTOM_GPU_*` 또는 `PGPU` 클래스가 없으면 미등록 상태입니다.

## 해결

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

## 다른 원인들

- **(1)**: [GPU 자원 부족]({% post_url 2026-06-30-gpu-instance-failure-resource-shortage %})
- **(2)**: [Flavor extra_specs 설정 오류]({% post_url 2026-06-30-gpu-instance-failure-flavor-config %})
- **(3)**: [PCI Alias 불일치]({% post_url 2026-06-30-gpu-instance-failure-pci-alias-mismatch %})
- **(5)**: [MIG Profile 부족]({% post_url 2026-06-30-gpu-instance-failure-mig-profile-shortage %})
