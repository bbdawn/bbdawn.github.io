---
title: "[Feature] OpenStack GPU 인스턴스 생성 및 모니터링 (5) — GPU 모니터링 API 개발: Prometheus 연동"
date: 2026-07-01 12:00:00 +0900
categories: [Project, GPU]
subcategory: Feature
tags: [openstack, gpu, dcgm-exporter, prometheus, monitoring, api]
---

![Prometheus에서 DCGM_FI_DEV_SM_CLOCK 메트릭을 조회한 결과 - job 라벨로 baremetal_dcgm_exporter와 dcgm_exporter가 구분됨](/assets/img/posts/gpu-monitoring-api-prometheus-query.png)
_Prometheus에 적재된 DCGM 메트릭. job 라벨로 GPU Host(baremetal_dcgm_exporter)와 GPU VM(dcgm_exporter) 수집 경로가 구분된다._

이전 글([4편]({% post_url 2026-07-01-gpu-instance-verification-tools %}))에서 인스턴스에 DCGM Exporter를 설치했다면, 이번 글은 그 메트릭을 실제로 조회 가능한 API로 만드는 과정입니다.

## 개발 배경

GPU 모니터링은 **GPU Host Monitoring**과 **GPU VM Monitoring** 두 갈래로 구성됩니다. 메트릭을 노출하는 지점(DCGM Exporter)은 같지만, Host는 Prometheus가 직접 접근할 수 있는 반면 VM은 네트워크상 직접 접근이 어려워 수집 경로가 다릅니다.

- **GPU Host Monitoring**: 호스트에 DCGM Exporter를 설치해 GPU 메트릭을 노출합니다.
- **GPU VM Monitoring**: VM에 DCGM Exporter와 qemu-guest-agent를 설치합니다. 모든 물리 호스트에 설치되는 Contrabass agent **mole**이 qemu-guest-agent를 통해 VM 내부에 접근해 DCGM Exporter 메트릭을 수집합니다.

```
[GPU Host]                       [GPU 인스턴스(VM)]
  DCGM Exporter                    DCGM Exporter
       │                           qemu-guest-agent
       │                                  │
       │                                  │ mole이 qemu-guest-agent로 접근해
       │                                  │ DCGM Exporter 메트릭 수집
       │                                  ▼
       │                     mole (Contrabass Agent, 전 호스트 설치)
       │                                  │
       └────────────────┬─────────────────┘
                         ▼
               Prometheus (호스트)   ← 수집·적재: 데이터엔지니어 담당
                         ▼
               Prometheus HTTP API
                         ▼
     GPU 모니터링 API (단건 조회 / 시계열 조회)   ← 본인 개발
                         ▼
              UI (GPU Host / GPU VM)
```

DCGM Exporter 메트릭을 mole로 수집해 Prometheus에 적재하는 부분은 데이터엔지니어가 구성했고, 저는 Prometheus에 쌓인 데이터를 **Prometheus HTTP API로 조회해서 UI에서 쓸 수 있는 형태로 가공하는 API**를 Host/VM 각각에 대해 개발했습니다.

---

## API 구성

### 1. 단건 조회 (현재 상태)

Prometheus의 instant query API(`/api/v1/query`)를 사용해 특정 시점의 GPU 사용률, 메모리, 온도 등 현재 상태값을 조회합니다.

```
GET /api/v1/query?query=DCGM_FI_DEV_GPU_UTIL{instance="<gpu-host>"}
```

호스트 단위, 인스턴스 단위로 각각 필터링할 수 있도록 쿼리를 구성했습니다.

### 2. 시계열 조회 (그래프)

Prometheus의 range query API(`/api/v1/query_range`)를 사용해 일정 기간 동안의 메트릭 추이를 조회합니다.

```
GET /api/v1/query_range?query=...&start=<ts>&end=<ts>&step=<interval>
```

UI에서 시간 범위(최근 1시간/6시간/1일 등)를 선택하면 해당 range로 조회해 그래프 데이터를 반환합니다.

### 호스트 / 인스턴스 단위 필터링

동일한 GPU 메트릭이라도 "호스트 전체 관점"과 "특정 인스턴스 관점"에서 다르게 조회되어야 하므로, label(host, instance UUID 등) 기준으로 쿼리를 분리해 구성했습니다.

---

## 데이터 검증

API가 반환하는 값이 실제 GPU 상태와 일치하는지 직접 확인하는 작업을 진행했습니다.

```bash
# GPU 호스트에서
nvidia-smi

# GPU 인스턴스 내부에서
nvidia-smi
```

위 명령어로 직접 확인한 값과, 개발한 API가 Prometheus를 통해 반환하는 값을 비교해서 오차나 지연(scrape interval에 따른 시차) 여부를 확인했습니다. 이 과정에서 label 매핑이 잘못되어 다른 호스트의 값이 섞이는 문제, scrape 주기로 인한 순간적인 값 지연 등을 발견하고 API 쿼리 조건을 조정했습니다.

---

## 효과

| 항목 | 이전 | 이후 |
|------|------|------|
| GPU 상태 확인 | 호스트/인스턴스 접속 후 명령어 실행 | UI에서 즉시 조회 |
| 추이 확인 | 불가능 (순간값만 확인 가능) | 시계열 그래프로 확인 |
| 신뢰도 | 접속 가능한 사람만 확인 가능 | API로 통합 노출, 직접 값 검증 완료 |

---

## 기술 스택

| 역할 | 기술 |
|------|------|
| 메트릭 노출 | DCGM Exporter |
| VM 메트릭 접근 | mole (Contrabass Agent) + qemu-guest-agent |
| 수집·적재 | Prometheus (데이터엔지니어 구성) |
| 조회 API | Prometheus HTTP API (query / query_range) |
| 검증 | nvidia-smi, dcgmi |

---

## 시리즈 구성

- **(1)**: [프로젝트 개요]({% post_url 2026-06-30-gpu-instance %})
- **(2)**: [OpenStack Version × Host OS 조합별 nova.conf / Flavor Metadata 자동 생성]({% post_url 2026-07-01-gpu-flavor-metadata-automation %})
- **(3)**: [인스턴스 생성 전 호스트 GPU 상태·설정 검증]({% post_url 2026-07-01-gpu-instance-host-precheck %})
- **(4)**: [GPU 인스턴스 검증 도구 설치 — nvidia-driver, DCGM Exporter, gpu-burn]({% post_url 2026-07-01-gpu-instance-verification-tools %})
- **(5)**: GPU 모니터링 API 개발 — Prometheus 연동 (현재)
