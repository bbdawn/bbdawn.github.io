---
title: "[Troubleshooting] GPU Instance 생성 실패 원인(2) : Flavor extra_specs 설정 오류"
date: 2026-06-30 06:10:00 +0900
categories: [Troubleshooting, GPU]
subcategory: Troubleshooting
tags: [openstack, gpu, nova, flavor, pci, troubleshooting]
---

GPU 자원은 충분한데도 `No valid host was found`가 발생한다면, Flavor 설정 자체를 의심해야 합니다.

## 원인

Flavor의 `extra_specs`에 `pci_passthrough:alias` 항목이 없거나 잘못된 값으로 설정된 경우입니다. nova-scheduler가 GPU를 요구하는 Flavor임을 인식하지 못해 일반 호스트에 배치를 시도합니다.

```
nova-scheduler → GPU 없는 호스트에 배치 시도 → 실패
```

## 확인 방법

```bash
openstack flavor show <flavor-id> --format json | grep extra_specs
```

올바른 extra_specs 예시:

```
pci_passthrough:alias = gpu-a100:1
```

## 해결

```bash
openstack flavor set <flavor-id> \
  --property pci_passthrough:alias="gpu-a100:1"
```

alias 이름은 `nova.conf`의 `[pci] alias` 설정과 정확히 일치해야 합니다. 일치 여부는 다음 글([3편]({% post_url 2026-06-30-gpu-instance-failure-pci-alias-mismatch %}))에서 다룹니다.

---

## 다른 원인들

- **(1)**: [GPU 자원 부족]({% post_url 2026-06-30-gpu-instance-failure-resource-shortage %})
- **(3)**: [PCI Alias 불일치]({% post_url 2026-06-30-gpu-instance-failure-pci-alias-mismatch %})
- **(4)**: [Placement 리소스 클래스 미등록]({% post_url 2026-06-30-gpu-instance-failure-placement-missing %})
- **(5)**: [MIG Profile 부족]({% post_url 2026-06-30-gpu-instance-failure-mig-profile-shortage %})
