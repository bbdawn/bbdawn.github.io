---
title: "[Feature] OpenStack GPU 인스턴스 생성 및 모니터링"
date: 2026-06-30 01:00:00 +0900
categories: [Project, GPU]
subcategory: Feature
tags: [openstack, gpu, mig, dcgm, prometheus, monitoring]
---

## 개요

OpenStack 환경에서 GPU 인스턴스를 일관되게 생성하고, 호스트/인스턴스 단위의 GPU 모니터링을 구현한 프로젝트입니다.

## 개발 배경

GPU 할당 방식, OS 버전, OpenStack 버전에 따라 Flavor metadata 구성이 달라져 수동 설정 시 실수가 빈번했습니다. 이를 자동화하여 일관성을 확보하기 위해 개발했습니다.

---

## 주요 기능

### 1. GPU 인스턴스 생성

세 가지 호스트 모드를 지원합니다.

| 모드 | 설명 |
|------|------|
| Passthrough (MIG 미활성화) | GPU 전체 할당 |
| Passthrough (MIG 활성화) | GPU 전체 할당 + MIG 구성 |
| MIG | GPU 슬라이스 단위 할당 |

OS/OpenStack 버전별로 적합한 metadata 구성 방식을 자동으로 선택합니다.

### 2. 사전 검증

- `nova.conf` 설정 검증
- Placement API를 통한 자원 가용성 검증

### 3. GPU 모니터링

DCGM Exporter를 활용해 호스트 및 인스턴스 단위 GPU 메트릭을 수집합니다.

```
DCGM Exporter → Prometheus → Grafana
```

Prometheus HTTP API를 통해 host/instance 단위로 데이터를 가공·시각화합니다.

---

## 지원 환경

| 항목 | 지원 버전 |
|------|-----------|
| OS | Ubuntu 22.04, SUSE 9, Ubuntu 24.04 |
| OpenStack | Yoga, Caracal, Epoxy |
| GPU | NVIDIA A100, H100, B300 |
| 모니터링 | DCGM Exporter, Prometheus |
