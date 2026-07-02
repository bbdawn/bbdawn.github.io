---
title: "[Feature] Placement 리소스 프로바이더 연동 — GPU 인벤토리 관리"
date: 2026-07-02 11:00:00 +0900
categories: [Feature, Openstack]
subcategory: Feature
tags: [openstack, placement, nova, gpu, resource-provider, java, spring]
---

이 기능은 [OpenStack GPU 인스턴스 생성 및 모니터링 (1) — 프로젝트 개요]({% post_url 2026-06-30-gpu-instance %}) 시리즈의 기반이 되는 부분입니다. GPU 인스턴스가 스케줄링되려면 그 전에 Placement에 GPU 리소스가 정확히 등록되어 있어야 합니다.

## 개요

Placement는 OpenStack에서 "어떤 호스트가 어떤 자원을 얼마나 갖고 있는지"를 관리하는 서비스입니다. Nova 스케줄러는 인스턴스를 배치할 때 Placement에 먼저 질의해서 후보 호스트를 좁힙니다. GPU처럼 표준 자원(vCPU, RAM, Disk)이 아닌 커스텀 자원은 Placement에 **리소스 프로바이더(Resource Provider)** 와 **인벤토리(Inventory)** 로 별도 등록해야 스케줄링 대상이 됩니다.

```
[GPU Compute 호스트]
      ↓ (GPU 리소스 등록 필요)
[Placement: Resource Provider + Inventory]
      ↓
[Nova Scheduler]
      ↓ 조회
후보 호스트 목록에 포함 여부 결정
```

## 개발 범위

콘트라베이스가 Placement API를 통해 다룬 CRUD는 다음과 같습니다.

| 기능 | 설명 |
|------|------|
| Resource Provider 조회 | 호스트별로 등록된 리소스 프로바이더 목록/상세 조회 |
| Inventory 등록·수정 | GPU 리소스 클래스(`PGPU`, `CUSTOM_GPU_*`)의 total/reserved/allocation_ratio 등록 |
| Usage 조회 | 리소스 프로바이더별 현재 사용량 조회 (몇 개 할당 중인지) |
| Aggregate 연동 | 리소스 프로바이더를 Aggregate에 묶어 특정 GPU 모델별로 스케줄링 후보를 분리 |

```java
// GPU Resource Provider의 인벤토리 조회
ResourceProviderInventory inventory = placementClient
    .getInventory(resourceProviderUuid, "PGPU");

// GPU 사용량 조회 (스케줄링 가능 여부 판단에 사용)
ResourceProviderUsage usage = placementClient
    .getUsage(resourceProviderUuid);

boolean hasAvailableGpu = inventory.getTotal() - usage.getUsed("PGPU") > 0;
```

## 인스턴스 생성 화면에서의 활용

호스트 목록에 GPU 잔여 수량을 표시할 때, Placement의 인벤토리·사용량 조회 결과를 그대로 활용합니다. GPU가 물리적으로 있어도 Placement에 등록이 안 되어 있으면 스케줄링 후보에서 제외되는데, 이 케이스는 [GPU Instance 생성 실패 원인(4) : Placement 리소스 클래스 미등록]({% post_url 2026-06-30-gpu-instance-failure-placement-missing %})에서 다룬 문제와 직접 연결됩니다.

## 기술 스택

| 역할 | 기술 |
|------|------|
| 백엔드 | Java, Spring Boot |
| 인프라 연동 | OpenStack Placement API, Nova |
