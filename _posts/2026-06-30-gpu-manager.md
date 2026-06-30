---
title: "OpenStack GPU 호스트 관리 TUI 도구 개발"
date: 2026-06-30 08:00:00 +0900
categories: [Project, OpenStack]
tags: [openstack, gpu, mig, go, tui, nvidia, libvirt]
---

## 개요

OpenStack 환경에서 GPU 호스트를 관리하는 TUI(Terminal UI) 도구입니다.

---

## 개발 배경

MIG 방식에서는 VM이 삭제된 후에도 mdev 장치가 sysfs에 남는 **orphan(ghost) 문제**가 발생합니다. 이를 효율적으로 탐지하고 정리하기 위해 개발했습니다.

---

## 주요 기능

| 기능 | 설명 |
|------|------|
| GPU 할당 방식 안내 | Passthrough/MIG 설정 가이드 |
| mdev orphan 탐지 및 정리 | `mdevctl stop` 자동 실행 |
| VM-GPU 매핑 | OpenStack VM과 libvirt VM 연결 확인 |
| GPU 사용 현황 | VM별 GPU 할당 상태 파악 |
| MIG 프로파일 관리 | GI/CI 조회 및 관리 |
| nvidia-smi 조회 | GPU 상태 실시간 확인 |

---

## orphan mdev 처리 흐름

```
VM 삭제 감지
    ↓
sysfs에서 잔존 mdev 장치 탐지
    ↓
mdevctl stop으로 정리
```

---

## 기술 스택

| 항목 | 내용 |
|------|------|
| 언어 | Go (단일 바이너리) |
| OS | Ubuntu 22.04 |
| 인프라 | OpenStack (Nova/libvirt) |
| GPU | NVIDIA A100, H100, B300 |

---

## 사용 방법

```bash
./gpu-manager
```

OpenStack 연동 시 openrc 경로를 환경변수로 지정하여 실행합니다.
