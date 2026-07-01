---
title: "[Project] OpenStack GPU 호스트 관리 TUI 도구 개발"
date: 2026-06-30 08:00:00 +0900
categories: [Project, GPU]
subcategory: Project
tags: [openstack, gpu, mig, mdev, orphan, go, tui, nvidia, libvirt]
---

## 개발 배경

MIG 환경에서 mdev orphan 문제가 반복적으로 발생했습니다. 장애 발생 시마다 아래 과정을 Compute 호스트마다 직접 수행해야 했습니다.

```bash
virsh list --all                         # libvirt VM 목록
ls /sys/bus/mdev/devices/               # mdev 장치 목록
# → UUID 수작업 비교 후 orphan 판별
mdevctl stop -u <orphan-uuid>           # 정리

nvidia-smi mig -lgip                    # MIG 프로파일 확인
nvidia-smi mig -lgi                     # GI 목록 확인
nvidia-smi mig -cgi <profile-id> -C    # GI/CI 생성
```

GPU 호스트가 여러 대일 때 각 호스트에 SSH 접속해서 반복해야 했고, mdev UUID와 VM을 매핑하는 과정에서 실수가 발생하기 쉬웠습니다. sysfs ↔ libvirt ↔ Nova를 자동으로 비교하는 도구가 필요하다는 결론으로 TUI 도구 개발을 시작했습니다.

---

## 기능 구성

```
GPU Manager (TUI)
├── mdev Orphan 탐지 및 정리
│   └── sysfs ↔ libvirt ↔ Nova 자동 비교 후 orphan 식별
├── MIG GI / CI 관리
│   └── 생성·삭제 명령 즉시 조회 및 실행
└── GPU 명령어 레퍼런스
    └── 자주 사용하는 nvidia-smi / DCGM / libvirt 명령 모음
```

---

## 화면 1. mdev Orphan 탐지

sysfs에 남아있는 mdev 장치를 Nova/libvirt와 비교하여 orphan 여부를 자동으로 판별합니다.

```
┌─ mdev Orphan Detection ─────────────────────────────────────────┐
│                                                                   │
│  mdev UUID                               VM              Status  │
│  ──────────────────────────────────────  ──────────────  ──────  │
│  550e8400-e29b-41d4-a716-446655440000    (없음)          ORPHAN  │
│  6ba7b810-9dad-11d1-80b4-00c04fd430c8   instance-0001   ACTIVE  │
│  7f3bc120-1a2b-43c4-d5e6-778899001122   instance-0002   ACTIVE  │
│                                                                   │
│  ──────────────────────────────────────────────────────────────  │
│  [정리 명령]                                                       │
│  mdevctl stop -u 550e8400-e29b-41d4-a716-446655440000            │
└───────────────────────────────────────────────────────────────────┘
```

### orphan 탐지 로직

```
sysfs mdev 목록 수집
      ↓
각 mdev UUID → libvirt 도메인 XML 참조 여부 확인
      ↓                              ↓
  참조 있음 → ACTIVE          참조 없음 → Nova 확인
                                     ↓
                         Nova에도 없음 → ORPHAN 판정
                                     ↓
                         mdevctl stop 명령 자동 생성
```

---

## 화면 2. MIG GI / CI 관리

현재 GPU에 설정된 GI/CI 목록을 보여주고, 생성·삭제에 필요한 명령어를 즉시 확인할 수 있습니다.

```
┌─ MIG Instance Management ──────────────────────────────────────┐
│                                                                  │
│  GPU 0: NVIDIA A100 80GB                                        │
│                                                                  │
│  ┌─ GI (GPU Instance) ─────────┐  ┌─ CI (Compute Instance) ──┐ │
│  │  ID   Profile     Status    │  │  ID  GI ID  Profile       │ │
│  │   1   1g.10gb     Used      │  │   0    1    1c.1g.10gb    │ │
│  │   2   2g.20gb     Used      │  │   0    2    1c.2g.20gb    │ │
│  │   3   (empty)     Free      │  └──────────────────────────┘ │
│  └─────────────────────────────┘                                │
│                                                                  │
│  ── 명령어 레퍼런스 ──────────────────────────────────────────   │
│  GI 생성   nvidia-smi mig -cgi <profile-id> -C                 │
│  GI 삭제   nvidia-smi mig -dgi -gi <gi-id>                     │
│  CI 생성   nvidia-smi mig -cci -gi <gi-id>                     │
│  CI 삭제   nvidia-smi mig -dci -ci <ci-id> -gi <gi-id>         │
│  전체 삭제  nvidia-smi mig -dgi (모든 GI/CI 제거)               │
└──────────────────────────────────────────────────────────────────┘
```

주요 MIG 프로파일 ID (A100 기준):

| Profile ID | 프로파일 | 슬라이스 |
|------------|---------|--------|
| 0 | 7g.80gb | 전체 |
| 5 | 4g.40gb | 1/2 |
| 14 | 2g.20gb | 1/4 |
| 19 | 1g.10gb | 1/8 |

---

## 화면 3. GPU 명령어 레퍼런스

자주 쓰는 GPU 관련 명령어를 한 화면에서 조회하고 복사할 수 있습니다.

```
┌─ GPU Commands ───────────────────────────────────────────────────┐
│                                                                    │
│  [상태 확인]                                                       │
│  nvidia-smi                       GPU 전체 현황 (온도/메모리/사용률) │
│  nvidia-smi -L                    GPU 목록                         │
│  nvidia-smi -q -d MEMORY          메모리 상세                      │
│  nvidia-smi mig -lgip             MIG 프로파일 목록                │
│  nvidia-smi mig -lgi              GI 목록                          │
│  nvidia-smi mig -lci              CI 목록                          │
│                                                                    │
│  [DCGM]                                                            │
│  dcgmi dmon -s u                  GPU 사용률 실시간 모니터링        │
│  dcgmi diag -r 1                  기본 진단                        │
│  dcgmi discovery -l               DCGM이 인식한 GPU 목록           │
│                                                                    │
│  [MIG 활성화]                                                      │
│  nvidia-smi -i 0 -mig 1           GPU 0 MIG 모드 활성화            │
│  nvidia-smi mig -cgi 19,19 -C     1g.10gb 슬라이스 2개 생성        │
│                                                                    │
│  [libvirt / mdev]                                                  │
│  virsh list --all                 VM 목록 (running + shut off)     │
│  virsh dumpxml <domain>           VM XML (mdev UUID 확인)          │
│  ls /sys/bus/mdev/devices/        sysfs mdev 장치 목록             │
│  mdevctl list                     mdev 장치 상태                   │
│  mdevctl stop -u <uuid>           mdev 장치 정리                   │
└────────────────────────────────────────────────────────────────────┘
```

---

## 구현

Go 언어로 단일 바이너리 TUI를 개발했습니다. 별도 설치 없이 바이너리 하나만 복사하면 실행됩니다.

```bash
./gpu-manager
```

OpenStack 연동이 필요한 기능(Nova VM 목록 비교)은 openrc를 환경변수로 지정합니다.

```bash
source openrc && ./gpu-manager
```

---

## 효과

| 항목 | 이전 | 이후 |
|------|------|------|
| orphan 탐지 | SSH 접속 후 수동 비교 (10~20분) | 자동 탐지 (즉시) |
| 정리 명령 | UUID 직접 입력 | 자동 생성 후 확인 |
| MIG 명령 조회 | 문서 검색 | TUI에서 즉시 확인 |
| 실수 가능성 | UUID 오타, 잘못된 mdev 정리 위험 | 자동 매핑으로 제거 |
| 배포 | - | 단일 바이너리, 설치 불필요 |

---

## 기술 스택

- **언어**: Go (단일 바이너리)
- **인프라**: OpenStack Nova, libvirt, sysfs
- **GPU**: NVIDIA A100, H100, B300 (MIG)
- **OS**: Ubuntu 22.04
