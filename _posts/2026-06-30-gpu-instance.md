---
title: "[Feature] OpenStack GPU 인스턴스 생성 및 모니터링 (1) — 프로젝트 개요"
date: 2026-06-30 01:00:00 +0900
categories: [Project, GPU]
subcategory: Feature
tags: [openstack, gpu, mig, passthrough, nova, flavor, dcgm, prometheus, monitoring]
---

## 개요

OpenStack 환경에서 GPU 인스턴스를 일관되게 생성하고, 호스트/인스턴스 단위의 GPU 모니터링을 구현한 프로젝트입니다. Flavor/nova.conf 자동 생성부터 생성 전 호스트 검증, 생성 후 동작 검증, 모니터링 API까지 GPU 인스턴스의 전 과정을 다룹니다.

## 개발 배경

GPU 할당 방식(Passthrough / MIG), OS 버전, OpenStack 버전에 따라 `nova.conf` 설정과 Flavor metadata 구성이 달라져 수동 설정 시 실수가 빈번했습니다. 설정이 틀리면 인스턴스 생성 자체가 실패하거나, 생성은 되어도 GPU를 인식하지 못하는 문제가 있어 생성 전후 검증까지 포함한 전체 흐름을 자동화했습니다.

---

## 전체 흐름

```
[Flavor / nova.conf 자동 생성] ── (2)
        ↓
[호스트 GPU 상태·설정 검증] ── (3)
        ↓
[GPU 인스턴스 생성]
        ↓
[검증 도구 설치: nvidia-driver / dcgm-exporter / gpu-burn] ── (4)
        ↓
[GPU 모니터링 API: 현재 상태 / 시계열] ── (5)
```

---

## 지원 환경

| 항목 | 지원 버전 |
|------|-----------|
| OS | Ubuntu 22.04, SUSE 9, Ubuntu 24.04 |
| OpenStack | Yoga, Caracal, Epoxy |
| GPU | NVIDIA A100, H100, B300 |
| 모니터링 | DCGM Exporter, Prometheus |

---

## 시리즈 구성

- **(1)**: 프로젝트 개요 (현재)
- **(2)**: OpenStack Version × Host OS 조합별 nova.conf / Flavor Metadata 자동 생성
- **(3)**: 인스턴스 생성 전 호스트 GPU 상태·설정 검증
- **(4)**: GPU 인스턴스 검증 도구 설치 — nvidia-driver, DCGM Exporter, gpu-burn
- **(5)**: GPU 모니터링 API 개발 — Prometheus 연동
