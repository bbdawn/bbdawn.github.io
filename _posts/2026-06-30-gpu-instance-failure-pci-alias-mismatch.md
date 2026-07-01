---
title: "[Troubleshooting] GPU Instance 생성 실패 원인(3) : PCI Alias 불일치 (Flavor ↔ nova.conf)"
date: 2026-06-30 06:20:00 +0900
categories: [Troubleshooting, GPU]
subcategory: Troubleshooting
tags: [openstack, gpu, nova, pci, flavor, troubleshooting]
---

Flavor에 `pci_passthrough:alias`가 설정되어 있어도 실패한다면, 그 값이 Compute 노드의 실제 alias와 일치하는지 확인해야 합니다.

## 원인

Flavor의 `pci_passthrough:alias` 값과 Compute 노드 `nova.conf`의 alias 이름이 다른 경우입니다.

```
Flavor extra_specs:  pci_passthrough:alias = "gpu-a100:1"
nova.conf alias:     name = "gpu_a100"          ← 이름 불일치
```

nova-scheduler는 alias 이름으로 호스트를 탐색하기 때문에, 불일치 시 유효한 호스트를 찾지 못합니다.

## 확인 방법

```bash
# Compute 노드에서 확인
grep -A5 "\[pci\]" /etc/nova/nova.conf
```

```ini
[pci]
alias = {"name": "gpu-a100", "product_id": "20b0", "vendor_id": "10de", "device_type": "type-PCI"}
```

## 해결

Flavor의 alias 값과 `nova.conf`의 `name` 필드를 동일하게 맞춥니다.

```bash
# nova.conf 수정 후 nova-compute 재시작
sudo systemctl restart nova-compute
```

---

## 다른 원인들

- **(1)**: [GPU 자원 부족]({% post_url 2026-06-30-gpu-instance-failure-resource-shortage %})
- **(2)**: [Flavor extra_specs 설정 오류]({% post_url 2026-06-30-gpu-instance-failure-flavor-config %})
- **(4)**: [Placement 리소스 클래스 미등록]({% post_url 2026-06-30-gpu-instance-failure-placement-missing %})
- **(5)**: [MIG Profile 부족]({% post_url 2026-06-30-gpu-instance-failure-mig-profile-shortage %})
