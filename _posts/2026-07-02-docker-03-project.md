---
title: "[Study] Docker 학습 (3) — 실습 프로젝트: Docker Compose로 컨테이너 모니터링 스택 구성"
date: 2026-07-02 11:00:00 +0900
categories: [Study, Docker]
subcategory: Study
tags: [docker, docker-compose, cadvisor, node-exporter, prometheus, grafana, monitoring]
---

## 개요

기본 개념([Docker 학습 (1)]({% post_url 2026-07-02-docker-01-basics %}))과 실무 활용([Docker 학습 (2)]({% post_url 2026-07-02-docker-02-practical %}))을 익힌 뒤, 배운 내용을 직접 써보기 위해 **컨테이너/호스트 모니터링 스택**(cAdvisor + node_exporter + Prometheus + Grafana)을 Docker Compose 하나로 구성하는 실습을 진행했습니다.

```
Docker Host
  └── docker-compose up
        ├── cadvisor       → 컨테이너별 CPU/메모리/네트워크 메트릭 노출 (포트 8080)
        ├── node-exporter  → 호스트 레벨 메트릭 노출 (포트 9100)
        ├── prometheus     → cadvisor·node-exporter 스크레이핑
        └── grafana        → Prometheus 데이터소스로 대시보드 시각화
```

특정 하드웨어(GPU 등) 없이 어떤 환경에서든 재현할 수 있는 조합으로, "메트릭 노출 → Prometheus 수집 → Grafana 시각화"라는 모니터링 파이프라인의 기본 골격을 익히는 데 목적을 두었습니다.

---

## docker-compose.yml 구성

```yaml
services:
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.49.1
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro

  node-exporter:
    image: prom/node-exporter:latest
    ports:
      - "9100:9100"

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
    depends_on:
      - cadvisor
      - node-exporter

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-data:/var/lib/grafana
    depends_on:
      - prometheus

volumes:
  grafana-data:
```

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
```

Compose 내부에서는 컨테이너 이름(`cadvisor`, `node-exporter`)이 곧 호스트명이 되므로, IP 대신 서비스 이름으로 스크레이핑 대상을 지정할 수 있습니다 — 1편에서 다룬 Docker 네트워크의 컨테이너 간 통신 방식이 그대로 적용됩니다.

---

## 실행 및 확인

```bash
docker compose up -d
docker compose ps

# cAdvisor 메트릭 직접 확인
curl http://localhost:8080/metrics | grep container_cpu_usage_seconds_total

# node_exporter 메트릭 직접 확인
curl http://localhost:9100/metrics | grep node_memory_MemAvailable_bytes

# Prometheus에서 타겟 상태 확인
open http://localhost:9090/targets
```

Grafana에 접속(`http://localhost:3000`)해 Prometheus를 데이터소스로 등록하고, 커뮤니티 대시보드를 import하면 바로 시각화됩니다.

```bash
open http://localhost:3000
# Configuration → Data Sources → Prometheus 추가 (URL: http://prometheus:9090)
# Dashboards → Import → 1860 입력 (Node Exporter Full)
```

`Node Exporter Full`(dashboard ID `1860`)은 가장 널리 쓰이는 호스트 모니터링 대시보드로, CPU/메모리/디스크/네트워크를 즉시 시각화해줍니다. cAdvisor 메트릭은 컨테이너별 리소스 사용량 패널을 직접 구성하며 PromQL을 익히는 용도로 다뤘습니다.

---

## 트러블슈팅

**macOS에서 cAdvisor 메트릭이 부정확하거나 일부만 나오는 경우**

cAdvisor는 공식적으로 Linux를 대상으로 하며, `/sys`, `/var/lib/docker` 같은 호스트 경로를 직접 마운트해 메트릭을 수집합니다. macOS의 Docker Desktop은 실제로는 경량 VM 위에서 컨테이너가 돌기 때문에, 호스트 경로를 그대로 마운트해도 cAdvisor가 완전한 메트릭을 얻지 못하는 경우가 있습니다. 컨테이너 단위 CPU/메모리 상대 비교 정도는 문제없지만, 정확한 수치가 필요하다면 Linux 환경(서버, VM)에서 확인하는 것이 안전합니다.

**Prometheus target이 DOWN 상태인 경우**

Compose 네트워크 안에서 서비스명으로 통신하는지 확인합니다. `prometheus.yml`에 컨테이너 IP를 직접 넣으면 컨테이너 재시작 시 IP가 바뀌어 target을 잃어버리므로, 반드시 서비스명(`cadvisor:8080`, `node-exporter:9100`)을 사용해야 합니다.

---

## 정리 — 자주 나오는 질문

**Q. GPU 모니터링 대신 컨테이너/호스트 모니터링으로 구성한 이유는?**

회사에서 다루는 DCGM Exporter + Prometheus 조합은 실제 GPU 하드웨어가 있어야만 재현할 수 있어 학습 실습으로는 재현성이 떨어집니다. cAdvisor + node_exporter는 특별한 하드웨어 없이 어디서나 실행할 수 있으면서도 "메트릭 노출 → Prometheus 수집 → Grafana 시각화"라는 동일한 파이프라인 구조를 그대로 연습할 수 있어 이 조합으로 바꿨습니다.

**Q. Kubernetes에서 다룬 node_exporter와 이번 실습의 차이는?**

[Kubernetes 학습 (1)]({% post_url 2026-07-01-k8s-minikube-basics %})에서는 node_exporter를 DaemonSet으로 모든 노드에 배포했는데, 이번에는 단일 호스트에서 Docker Compose로 같은 컴포넌트를 실행했습니다. 오케스트레이션 레이어만 다를 뿐 "메트릭 노출 → Prometheus 수집"이라는 핵심 흐름은 동일합니다.

**Q. 컨테이너 모니터링에 cAdvisor를 쓰는 이유는?**

Docker 자체에도 `docker stats` 명령어가 있지만 히스토리 조회나 알림 설정이 안 됩니다. cAdvisor는 컨테이너별 메트릭을 Prometheus 포맷으로 노출해주기 때문에, Prometheus·Grafana와 조합하면 컨테이너 리소스 사용량을 시계열로 추적하고 임계치 알림까지 구성할 수 있습니다.

---

## 기술 스택

| 역할 | 기술 |
|------|------|
| 컨테이너 오케스트레이션 | Docker Compose |
| 컨테이너 메트릭 수집 | cAdvisor |
| 호스트 메트릭 수집 | node_exporter |
| 메트릭 저장 | Prometheus |
| 시각화 | Grafana |

---

## 시리즈 구성

- **(1)**: 기본 개념 — 컨테이너, 이미지, Dockerfile
- **(2)**: 실무 활용 — Docker Compose, 멀티 스테이지 빌드, 레지스트리
- **(3)**: 실습 프로젝트 — Docker Compose로 컨테이너 모니터링 스택 구성 (현재)
