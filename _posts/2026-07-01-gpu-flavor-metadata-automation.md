---
title: "[Feature] OpenStack GPU 인스턴스 생성 및 모니터링 (2) — OpenStack Version × Host OS 조합별 nova.conf / Flavor Metadata 자동 생성"
date: 2026-07-01 09:00:00 +0900
categories: [Project, GPU]
subcategory: Feature
tags: [openstack, gpu, mig, passthrough, nova, flavor, metadata]
---

시리즈 첫 글은 [OpenStack GPU 인스턴스 생성 및 모니터링 (1) — 프로젝트 개요]({% post_url 2026-06-30-gpu-instance %})에서 확인할 수 있습니다.

## 개발 배경

GPU 인스턴스를 생성하려면 두 곳의 설정이 정확히 맞아야 합니다.

- **nova.conf** — Compute 호스트에서 GPU를 PCI Passthrough 또는 MIG 슬라이스로 인식/할당하기 위한 설정
- **Flavor metadata** — 어떤 GPU 리소스를 얼마나 요청할지 정의하는 extra_specs

문제는 이 두 설정의 정확한 값과 문법이 **OpenStack 버전(Yoga / Caracal / Epoxy)** 과 **Host OS(Ubuntu 22.04 / 24.04, SUSE 9)** 조합에 따라 달라진다는 점입니다. 같은 "MIG 모드"라도 버전에 따라 nova.conf의 키 이름이 바뀌거나, PCI 디바이스 지정 방식이 달라졌습니다. 운영자가 이 조합을 매번 문서를 찾아가며 수작업으로 입력하다 보니 오타나 버전 불일치로 인한 실패가 반복됐고, 이를 자동 생성하는 백엔드를 개발했습니다.

---

## 지원 대상 조합

| 항목 | 지원 값 |
|------|---------|
| Host OS | Ubuntu 22.04, Ubuntu 24.04, SUSE 9 |
| OpenStack | Yoga, Caracal, Epoxy |
| GPU 모드 | Passthrough (MIG 미활성화), Passthrough (MIG 활성화), MIG |

3가지 GPU 모드 × 3개 OS × 3개 OpenStack 버전 조합을 모두 수작업으로 관리하면 실수가 나올 수밖에 없는 규모입니다.

---

## 동작 흐름

```
[UI] 호스트 선택 + GPU 모드 선택 (Passthrough / MIG)
        ↓
[Backend] Host OS + OpenStack Version 조회
        ↓
조합 매트릭스 조회 (OS × Version × Mode → 설정 템플릿)
        ↓
┌─────────────────────────┬─────────────────────────┐
│   nova.conf 설정값 생성   │  Flavor extra_specs 생성  │
└─────────────────────────┴─────────────────────────┘
        ↓
호스트에 적용 + Flavor 등록
```

### 조합 매트릭스

OS × OpenStack Version × GPU 모드를 키로 하는 설정 템플릿을 관리하고, 요청이 들어오면 해당 조합의 템플릿을 조회해 실제 값으로 치환합니다.

```
(Ubuntu 22.04, Yoga, MIG)      → template_a
(Ubuntu 22.04, Caracal, MIG)   → template_b
(Ubuntu 24.04, Epoxy, MIG)     → template_c
(SUSE 9, Caracal, Passthrough) → template_d
...
```

### nova.conf 생성 항목

GPU 모드에 따라 아래와 같은 항목이 자동으로 채워집니다.

- Passthrough: PCI 디바이스 화이트리스트/스펙 설정
- MIG: mdev 타입 활성화 설정, MIG 프로파일 관련 설정

### Flavor metadata 생성 항목

- GPU 리소스 요청 extra_specs (모드별로 키/값 형식이 다름)
- PCI alias 또는 리소스 클래스 매핑

---

## 효과

| 항목 | 이전 | 이후 |
|------|------|------|
| 설정 방식 | 버전별 문서 참고 후 수작업 입력 | 조합 선택만으로 자동 생성 |
| 실수 가능성 | 버전 간 문법 차이로 오타/누락 발생 | 템플릿 기반으로 제거 |
| 신규 버전 대응 | 담당자 개별 학습 필요 | 매트릭스에 조합 추가만 하면 됨 |
| 소요 시간 | 조합당 10분 이상 | 즉시 |

---

## 기술 스택

| 역할 | 기술 |
|------|------|
| 백엔드 | OpenStack Nova API, Placement API |
| 대상 | nova.conf, Flavor extra_specs |
| 인프라 | OpenStack (Yoga, Caracal, Epoxy) |

---

## 시리즈 구성

- **(1)**: [프로젝트 개요]({% post_url 2026-06-30-gpu-instance %})
- **(2)**: OpenStack Version × Host OS 조합별 nova.conf / Flavor Metadata 자동 생성 (현재)
- **(3)**: [인스턴스 생성 전 호스트 GPU 상태·설정 검증]({% post_url 2026-07-01-gpu-instance-host-precheck %})
- **(4)**: [GPU 인스턴스 검증 도구 설치 — nvidia-driver, DCGM Exporter, gpu-burn]({% post_url 2026-07-01-gpu-instance-verification-tools %})
- **(5)**: [GPU 모니터링 API 개발 — Prometheus 연동]({% post_url 2026-07-01-gpu-monitoring-api %})
