---
title: "[Project] Docker 학습 (3) — 실습 프로젝트: Docker Compose로 GPU 모니터링 스택 구성"
date: 2026-07-02 11:00:00 +0900
categories: [Project, Docker]
subcategory: Project
tags: [docker, docker-compose, gpu, dcgm, prometheus, grafana, monitoring, nvidia]
---

## 개요

기본 개념([Docker 학습 (1)]({% post_url 2026-07-02-docker-01-basics %}))과 실무 활용([Docker 학습 (2)]({% post_url 2026-07-02-docker-02-practical %}))을 익힌 뒤, 배운 내용을 실제 업무와 맞닿은 형태로 적용해보기 위해 GPU 인스턴스 모니터링 스택(DCGM Exporter + Prometheus + Grafana)을 Docker Compose 하나로 구성하는 실습을 진행했습니다.

```
GPU 인스턴스 (NVIDIA GPU 장착)
  └── docker-compose up
        ├── dcgm-exporter  → GPU 메트릭 노출 (포트 9400)
        ├── prometheus     → dcgm-exporter 스크레이핑
        └── grafana        → Prometheus 데이터소스로 대시보드 시각화
```

회사에서는 GPU 모니터링을 별도 API·대시보드로 직접 개발해 운영하고 있는데, Docker만으로 동일한 파이프라인의 기본 골격을 얼마나 빠르게 구성할 수 있는지 확인하는 데 목적을 두었습니다.

---

## 사전 준비: nvidia-container-toolkit

일반 컨테이너는 호스트의 GPU에 접근할 수 없습니다. GPU 인스턴스에 `nvidia-container-toolkit`을 설치해 Docker가 GPU 리소스를 컨테이너에 노출하도록 설정합니다.

```bash
distribution=$(. /etc/os-release; echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | \
  sudo tee /etc/apt/sources.list.d/nvidia-docker.list

sudo apt-get update && sudo apt-get install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

```bash
docker run --rm --gpus all nvidia/cuda:12.4.0-base-ubuntu22.04 nvidia-smi
```

`nvidia-smi` 출력이 정상적으로 나오면 컨테이너가 호스트 GPU를 인식한 것입니다.

---

## docker-compose.yml 구성

```yaml
services:
  dcgm-exporter:
    image: nvcr.io/nvidia/k8s/dcgm-exporter:3.3.5-3.4.1-ubuntu22.04
    runtime: nvidia
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    ports:
      - "9400:9400"

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
    depends_on:
      - dcgm-exporter

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
  - job_name: 'dcgm-exporter'
    static_configs:
      - targets: ['dcgm-exporter:9400']
```

Compose 내부에서는 컨테이너 이름(`dcgm-exporter`)이 곧 호스트명이 되므로, IP 대신 서비스 이름으로 스크레이핑 대상을 지정할 수 있습니다 — 1편에서 다룬 Docker 네트워크의 컨테이너 간 통신 방식이 그대로 적용됩니다.

---

## 실행 및 확인

```bash
docker compose up -d
docker compose ps

# DCGM Exporter 메트릭 직접 확인
curl http://localhost:9400/metrics | grep DCGM_FI_DEV_GPU_UTIL

# Prometheus에서 타겟 상태 확인
open http://localhost:9090/targets
```

Grafana에 접속(`http://localhost:3000`)해 Prometheus를 데이터소스로 등록하고, NVIDIA 공식 DCGM Exporter 대시보드(dashboard ID `12239`)를 import하면 GPU 사용률·메모리·온도 등을 즉시 시각화할 수 있습니다.

```bash
open http://localhost:3000
# Configuration → Data Sources → Prometheus 추가 (URL: http://prometheus:9090)
# Dashboards → Import → 12239 입력
```

---

## 트러블슈팅

**컨테이너가 GPU를 인식하지 못하는 경우**

```bash
docker info | grep -i runtime
```

출력에 `nvidia`가 없다면 `/etc/docker/daemon.json`에 nvidia 런타임이 기본으로 등록되지 않은 것입니다. `nvidia-ctk runtime configure` 실행 후 Docker 데몬을 재시작하면 해결됩니다.

**Prometheus target이 DOWN 상태인 경우**

Compose 네트워크 안에서 서비스명으로 통신하는지 확인합니다. `prometheus.yml`에 컨테이너 IP를 직접 넣으면 컨테이너 재시작 시 IP가 바뀌어 target을 잃어버리므로, 반드시 서비스명(`dcgm-exporter:9400`)을 사용해야 합니다.

---

## 정리 — 자주 나오는 질문

**Q. 이 실습이 실무 GPU 모니터링과 어떻게 연결되나요?**

회사에서는 Prometheus로 수집한 DCGM 메트릭을 자체 개발한 API가 다시 가공해 GPU Host/Instance 모니터링 화면에 노출하는 구조로 운영하고 있습니다. 이번 실습에서는 그 앞단인 "GPU 메트릭 수집 → Prometheus 저장 → 시각화"까지의 기본 파이프라인을 Docker Compose로 재현해봤습니다.

**Q. 컨테이너에서 GPU를 쓰려면 무엇이 필요한가요?**

호스트에 NVIDIA 드라이버가 설치되어 있어야 하고, Docker가 GPU 디바이스를 컨테이너에 전달할 수 있도록 `nvidia-container-toolkit`을 설치해 nvidia 런타임을 등록해야 합니다.

**Q. Kubernetes에서 다룬 DCGM Exporter와 이번 실습의 차이는?**

[Kubernetes 학습 (2)]({% post_url 2026-07-01-k8s-openstack-gpu-kubeadm %})에서는 DCGM Exporter를 DaemonSet으로 모든 노드에 배포했는데, 이번에는 단일 호스트에서 Docker Compose로 같은 컴포넌트를 실행했습니다. 오케스트레이션 레이어만 다를 뿐 "GPU 메트릭 노출 → Prometheus 수집"이라는 핵심 흐름은 동일합니다.

---

## 기술 스택

| 역할 | 기술 |
|------|------|
| 컨테이너 오케스트레이션 | Docker Compose |
| GPU 컨테이너 런타임 | nvidia-container-toolkit |
| GPU 메트릭 수집 | DCGM Exporter |
| 메트릭 저장 | Prometheus |
| 시각화 | Grafana |

---

## 시리즈 구성

- **(1)**: 기본 개념 — 컨테이너, 이미지, Dockerfile
- **(2)**: 실무 활용 — Docker Compose, 멀티 스테이지 빌드, 레지스트리
- **(3)**: 실습 프로젝트 — Docker Compose로 GPU 모니터링 스택 구성 (현재)
