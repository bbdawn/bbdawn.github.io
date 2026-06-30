---
title: "MIG 환경에서 반복되는 mdev orphan 문제, TUI로 자동화하기"
date: 2026-06-30 09:00:00 +0900
categories: [Project, GPU]
tags: [openstack, gpu, mig, mdev, orphan, automation, go, tui, libvirt]
---

## 문제 상황

OpenStack MIG 환경에서 VM을 삭제해도 mdev 장치가 sysfs에 그대로 남는 문제가 반복적으로 발생했습니다.

```
VM 삭제
  ↓
Nova에서는 삭제됨
  ↓
sysfs에 mdev 장치 잔존 (orphan)
  ↓
해당 GPU 슬라이스 재할당 불가
```

GPU 슬라이스가 실제로는 비어있지만 점유된 것으로 인식되어, 새로운 VM에 할당할 수 없는 상태가 됩니다.

---

## 기존 수동 처리 방식

장애 발생 시마다 아래 과정을 직접 수행해야 했습니다.

```bash
# 1. libvirt에서 현재 VM 목록 확인
virsh list --all

# 2. sysfs에서 mdev 장치 목록 확인
ls /sys/bus/mdev/devices/

# 3. Nova VM과 mdev UUID 매핑 수작업으로 비교
# → 어떤 mdev가 orphan인지 수동으로 판단

# 4. orphan mdev 정리
mdevctl stop -u <mdev-uuid>
```

문제는 GPU 호스트가 여러 대일 때 각 호스트에 SSH 접속해서 반복해야 했고, mdev UUID와 VM을 매핑하는 과정에서 실수가 발생하기 쉬웠습니다.

---

## 자동화 방향

반복되는 수작업을 줄이기 위해 아래 기능을 하나의 도구로 통합했습니다.

1. **OpenStack VM 목록 + libvirt VM 목록 자동 비교**
2. **sysfs mdev 장치와 VM 매핑 자동 분석**
3. **orphan mdev 자동 탐지 및 정리 명령 생성**

---

## 구현: GPU Manager (Go TUI)

Go 언어로 단일 바이너리 TUI 도구를 개발했습니다.

```
GPU Manager
├── VM-GPU 매핑 조회
│   └── OpenStack VM ↔ libvirt VM ↔ mdev UUID 연결
├── orphan mdev 탐지
│   └── Nova에 없는 mdev 자동 식별
├── 정리 명령 생성
│   └── mdevctl stop -u <uuid> 자동 생성
└── MIG 프로파일 조회
    └── GI / CI 현황 확인
```

### orphan 탐지 로직

```
sysfs mdev 목록
      ↓
각 mdev UUID → libvirt domain XML에서 참조 여부 확인
      ↓
libvirt에도 없는 mdev → OpenStack Nova에서도 확인
      ↓
Nova에도 없으면 → orphan으로 판정
      ↓
mdevctl stop 명령 자동 생성
```

---

## 효과

| 항목 | 이전 | 이후 |
|------|------|------|
| orphan 탐지 | 수동 비교 (10~20분) | 자동 탐지 (즉시) |
| 정리 명령 | 직접 작성 | 자동 생성 |
| 실수 가능성 | UUID 오타, 잘못된 mdev 정리 위험 | 자동 매핑으로 제거 |
| 배포 방식 | - | 단일 바이너리, 설치 불필요 |

---

## 기술 스택

- **언어**: Go (단일 바이너리)
- **인프라**: OpenStack Nova, libvirt, sysfs
- **GPU**: NVIDIA A100, H100, B300 (MIG)
- **OS**: Ubuntu 22.04
