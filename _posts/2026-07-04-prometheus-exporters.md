---
title: "[Study] Prometheus Exporter 정리 — node_exporter, libvirt_exporter, openstack_exporter, dcgm_exporter"
date: 2026-07-04 09:00:00 +0900
categories: [Study, Openstack]
subcategory: Study
tags: [openstack, prometheus, monitoring, node-exporter, libvirt-exporter, openstack-exporter, dcgm-exporter]
---

콘트라베이스 운영/모니터링 과정에서 실제로 다뤄본 Prometheus Exporter 4가지를 정리했습니다. 대상 계층이 서로 달라서, 하나의 Compute 호스트를 완전히 들여다보려면 이 중 여러 개를 함께 써야 하는 경우가 많습니다.

## node_exporter

- **대상**: 호스트 자체 (OS 레벨)
- **주요 메트릭**: CPU, 메모리, 디스크, 네트워크 사용량
- **특징**: 별도 종속성 없이 바이너리/컨테이너 하나만 실행하면 되는 가장 기본적인 exporter. Prometheus 생태계 진입점으로 가장 많이 쓰입니다.

## libvirt_exporter

- **대상**: libvirt/KVM으로 관리되는 VM (Nova Compute 호스트)
- **주요 메트릭**: VM(도메인)별 CPU 사용률, 메모리, 디스크·네트워크 I/O
- **연동 방식**: libvirtd 소켓에 접속해 도메인 목록과 상태를 조회
- **실무 연계**: [GPU mdev orphan 자동 탐지 TUI 도구]({% post_url 2026-06-30-gpu-host-manager-tui %})에서 다룬 sysfs ↔ libvirt ↔ Nova 관계와 같은 선상에서, VM 자체의 리소스 사용량을 모니터링하는 데 씁니다. Compute 호스트 하나에 VM이 여러 대 떠 있을 때, 호스트 전체 지표(node_exporter)만으로는 어떤 VM이 자원을 많이 쓰는지 구분할 수 없는데 libvirt_exporter가 이 간극을 메웁니다.

## openstack_exporter

- **대상**: OpenStack API 자체 (Nova, Cinder, Neutron 등)
- **주요 메트릭**: 서비스별 인스턴스/볼륨 개수, 상태별 집계, 쿼터 사용량, API 응답 시간
- **연동 방식**: OpenStack REST API를 주기적으로 호출해 집계한 뒤 Prometheus 포맷으로 노출
- **특징**: 개별 호스트가 아니라 "클러스터 전체 관점"의 지표를 제공합니다. 예를 들어 "지금 전체 리전에 ACTIVE 상태인 인스턴스가 몇 개인지"는 node_exporter/libvirt_exporter로는 알 수 없고 openstack_exporter가 담당하는 영역입니다.

## dcgm_exporter

- **대상**: NVIDIA GPU
- **주요 메트릭**: GPU 사용률, 메모리, 온도, ECC 에러 등
- **실무 연계**: [GPU 모니터링 API 개발 — Prometheus 연동]({% post_url 2026-07-01-gpu-monitoring-api %})에서 다룬 파이프라인의 핵심 컴포넌트입니다.

---

## 계층 구조로 보기

```
[Compute Host]
  ├── node_exporter     → 호스트 자체(OS) 메트릭
  ├── libvirt_exporter  → 그 호스트에서 도는 VM별 메트릭
  └── dcgm_exporter     → GPU 인스턴스라면 GPU 메트릭까지

[별도 위치 / API 게이트웨이]
  └── openstack_exporter → 개별 호스트가 아닌, 클러스터 전체의 서비스 단위 지표

              모두 → Prometheus → Grafana / 자체 개발 모니터링 API
```

## 비교

| Exporter | 대상 | 계층 | 실행 조건 |
|----------|------|------|-----------|
| node_exporter | 호스트 자체 | OS / 하드웨어 | 어디서든 실행 가능, 의존성 없음 |
| libvirt_exporter | KVM/QEMU VM | 하이퍼바이저 | libvirtd 접근 필요 |
| openstack_exporter | OpenStack API | 서비스 / 클러스터 | OpenStack API 엔드포인트·자격 증명 필요 |
| dcgm_exporter | NVIDIA GPU | 하드웨어 | NVIDIA GPU + 드라이버 필요 |

---

## 기술 스택

| 역할 | 기술 |
|------|------|
| 메트릭 수집 | node_exporter, libvirt_exporter, openstack_exporter, dcgm_exporter |
| 저장 | Prometheus |
| 시각화 | Grafana / 자체 개발 모니터링 API |
| 인프라 | OpenStack (Nova, libvirt/KVM) |
