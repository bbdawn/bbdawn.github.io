---
title: "[Feature] OpenStack GPU 인스턴스 생성 및 모니터링 (4) — GPU 인스턴스 검증 도구 설치: nvidia-driver, DCGM Exporter, gpu-burn"
date: 2026-07-02 11:00:00 +0900
categories: [Project, GPU]
subcategory: Feature
tags: [openstack, gpu, nvidia-driver, dcgm-exporter, gpu-burn, ansible, golden-image]
---

이전 글([3편]({% post_url 2026-07-02-gpu-instance-host-precheck %}))에서 호스트 사전 검증을 통과한 뒤 인스턴스를 생성했다면, 이번엔 그 인스턴스가 실제로 GPU를 정상적으로 사용할 수 있는지 검증할 차례입니다.

## 개발 배경

호스트 검증을 통과하고 인스턴스가 정상적으로 생성되었다고 해서, 인스턴스 내부에서 GPU가 실제로 정상 동작한다는 보장은 없습니다. 드라이버 설치 문제, MIG/Passthrough 전달 과정의 설정 누락 등으로 인스턴스 내부에서 GPU 인식이 안 되거나 성능이 기대에 못 미치는 경우가 있어, 생성 직후 표준화된 방식으로 동작을 검증하는 작업이 필요했습니다.

---

## 설치·검증 대상

| 도구 | 목적 |
|------|------|
| nvidia-driver | 인스턴스에서 GPU를 인식하기 위한 드라이버 |
| DCGM Exporter | GPU 메트릭 수집 (모니터링 연동, 5편에서 사용) |
| gpu-burn | GPU에 부하를 걸어 정상 동작 여부를 검증하는 스트레스 테스트 도구 |

---

## 검증 절차

```
인스턴스 생성 완료
        ↓
nvidia-driver 설치
        ↓
nvidia-smi 로 GPU 인식 확인
        ↓
dcgm-exporter 설치 및 실행
        ↓
gpu-burn 실행 (부하 테스트)
        ↓
정상 동작 확인 → 검증 완료
```

### 드라이버 설치 및 확인

```bash
# nvidia-driver 설치 후
nvidia-smi
```

GPU가 인스턴스에 정상적으로 전달됐다면 모델명과 메모리 크기가 출력됩니다. Passthrough/MIG 모드에 따라 보이는 디바이스 개수와 형태가 다르므로, (1)~(3)편에서 설정한 모드와 일치하는지 함께 확인합니다.

### DCGM Exporter 설치

GPU 메트릭을 외부에서 수집할 수 있도록 인스턴스에 DCGM Exporter를 설치하고 실행합니다. 이 데이터는 (5)편의 모니터링 API에서 사용됩니다.

### gpu-burn으로 부하 테스트

```bash
./gpu_burn 60   # 60초간 GPU에 부하를 걸어 연산 정상 여부 확인
```

단순히 드라이버가 인식하는 것과, 실제로 연산이 정상적으로 도는 것은 다른 문제이기 때문에 gpu-burn을 통해 실제 연산 부하를 걸어 이상 유무(에러, 온도, 처리량)를 확인합니다.

---

## 향후 계획: Golden Image / Ansible로 확장

지금은 인스턴스 생성 후 수동으로 위 절차를 수행하지만, 검증이 끝난 이 설치 순서는 그대로 두 가지 방향으로 자동화될 예정입니다.

- **Golden Image**: nvidia-driver, DCGM Exporter를 미리 설치한 이미지를 만들어 생성 시점부터 바로 사용 가능하게 함
- **Ansible**: 이미지에 포함하기 어려운 설정(버전 조합, 환경별 값)은 Ansible playbook으로 인스턴스 생성 직후 자동 적용

이번에 정리한 설치·검증 절차가 이 playbook/이미지 빌드 스크립트의 기준(base)이 됩니다.

---

## 기술 스택

| 역할 | 기술 |
|------|------|
| GPU 드라이버 | NVIDIA Driver |
| 모니터링 에이전트 | DCGM Exporter |
| 부하 테스트 | gpu-burn |
| 향후 자동화 | Ansible, Golden Image |

---

## 시리즈 구성

- **(1)**: [프로젝트 개요]({% post_url 2026-06-30-gpu-instance %})
- **(2)**: [OpenStack Version × Host OS 조합별 nova.conf / Flavor Metadata 자동 생성]({% post_url 2026-07-02-gpu-flavor-metadata-automation %})
- **(3)**: [인스턴스 생성 전 호스트 GPU 상태·설정 검증]({% post_url 2026-07-02-gpu-instance-host-precheck %})
- **(4)**: GPU 인스턴스 검증 도구 설치 — nvidia-driver, DCGM Exporter, gpu-burn (현재)
- **(5)**: [GPU 모니터링 API 개발 — Prometheus 연동]({% post_url 2026-07-02-gpu-monitoring-api %})
