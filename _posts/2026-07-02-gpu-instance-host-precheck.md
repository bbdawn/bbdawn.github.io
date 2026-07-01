---
title: "[Feature] OpenStack GPU 인스턴스 생성 및 모니터링 (3) — 인스턴스 생성 전 호스트 GPU 상태·설정 검증"
date: 2026-07-02 10:00:00 +0900
categories: [Project, GPU]
subcategory: Feature
tags: [openstack, gpu, mig, passthrough, nova, precheck, validation]
---

이전 글([2편]({% post_url 2026-07-02-gpu-flavor-metadata-automation %}))에서 OS/OpenStack Version 조합별로 nova.conf와 Flavor metadata를 자동 생성하는 기능을 다뤘습니다. 이번 글은 실제로 그 설정이 **호스트에 정상 반영되어 있는지**를 인스턴스 생성 전에 확인하는 기능입니다.

## 개발 배경

(2)편에서 만든 자동 생성 기능으로 설정을 내려보내도, 다음과 같은 이유로 실제 호스트 상태가 기대와 다를 수 있습니다.

- 호스트에 GPU 카드가 물리적으로 꽂혀있지 않은 경우
- GPU는 있지만 BIOS/드라이버 설정이 안 맞아 MIG/Passthrough 모드가 기대와 다르게 인식되는 경우
- nova.conf가 자동 생성 이후 재적용(서비스 재시작 등)되지 않아 실제 설정과 어긋난 경우

이 상태를 모르고 인스턴스 생성을 시도하면, 스케줄링 단계에서 실패하거나 최악의 경우 인스턴스는 생성되지만 GPU를 사용할 수 없는 상태로 떠버립니다. 그래서 인스턴스 생성 화면에서 호스트 목록을 보여줄 때, 각 호스트의 GPU 상태를 미리 검증해서 노출하는 기능을 개발했습니다.

---

## 검증 항목

인스턴스 생성 화면에서 호스트 목록을 조회할 때 호스트별로 아래 3가지를 확인합니다.

| 항목 | 확인 내용 |
|------|-----------|
| GPU 카드 유무 | 호스트에 실제 GPU 디바이스가 존재하는지 |
| 동작 모드 | MIG인지 Passthrough인지, 몇 개의 슬라이스/디바이스가 있는지 |
| nova.conf 정합성 | (2)편에서 생성된 기대 설정값과 호스트의 실제 설정값이 일치하는지 |

---

## 동작 흐름

```
[UI] 인스턴스 생성 화면 → 호스트 목록 조회
        ↓
[Backend] 호스트별 GPU 디바이스 조회
        ↓
   GPU 카드 있음?  ── No  → "GPU 없음" 표시, 선택 비활성화
        │ Yes
        ↓
   모드 판별 (MIG / Passthrough)
        ↓
   nova.conf 기대값 vs 실제값 비교
        │
   ┌────┴────┐
  일치        불일치
   │           │
"생성 가능"   "설정 불일치" 경고 + 상세 사유 표시
```

### GPU 카드/모드 조회

호스트의 GPU 디바이스 정보(PCI 디바이스, MIG 인스턴스 목록)를 조회해 카드 유무와 현재 모드를 판별합니다.

### nova.conf 정합성 체크

(2)편에서 해당 호스트의 OS/OpenStack 버전 조합에 대해 기대되는 nova.conf 값을 알고 있으므로, 호스트에서 실제 적용된 값을 가져와 비교합니다. 값이 다르면 "설정은 생성됐지만 미적용" 같은 상태를 구체적으로 안내합니다.

---

## UI 표시

호스트 목록에서 상태별로 구분되어 표시됩니다.

```
호스트명          GPU        모드           상태
──────────────    ───────    ───────────    ─────────────
compute-gpu-01    A100 ×4    MIG            생성 가능
compute-gpu-02    A100 ×4    Passthrough    생성 가능
compute-gpu-03    H100 ×2    MIG            설정 불일치 ⚠
compute-gpu-04    -          -              GPU 없음
```

---

## 효과

| 항목 | 이전 | 이후 |
|------|------|------|
| 문제 발견 시점 | 인스턴스 생성 실패 후 | 호스트 선택 단계에서 사전 확인 |
| 원인 파악 | SSH 접속 후 수동 확인 | UI에서 바로 확인 |
| 잘못된 호스트 선택 | 가능 (생성 시도 후 실패) | 선택 단계에서 차단 |

---

## 기술 스택

| 역할 | 기술 |
|------|------|
| 백엔드 | OpenStack Nova, Placement API |
| GPU 조회 | PCI 디바이스 조회, nvidia-smi / MIG 정보 |
| 검증 대상 | nova.conf |

---

## 시리즈 구성

- **(1)**: [프로젝트 개요]({% post_url 2026-06-30-gpu-instance %})
- **(2)**: [OpenStack Version × Host OS 조합별 nova.conf / Flavor Metadata 자동 생성]({% post_url 2026-07-02-gpu-flavor-metadata-automation %})
- **(3)**: 인스턴스 생성 전 호스트 GPU 상태·설정 검증 (현재)
- **(4)**: [GPU 인스턴스 검증 도구 설치 — nvidia-driver, DCGM Exporter, gpu-burn]({% post_url 2026-07-02-gpu-instance-verification-tools %})
- **(5)**: [GPU 모니터링 API 개발 — Prometheus 연동]({% post_url 2026-07-02-gpu-monitoring-api %})
