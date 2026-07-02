---
title: "[Study] Kubernetes 학습 (2) — OpenStack GPU 인스턴스 위에 K8s 구성 (kubeadm + NVIDIA Device Plugin + DCGM Exporter)"
date: 2026-07-01 10:00:00 +0900
categories: [Study, Kubernetes]
subcategory: Study
tags: [kubernetes, k8s, kubeadm, openstack, gpu, nvidia, device-plugin, dcgm, prometheus, mig]
---

## 개요

minikube로 K8s 기본 개념을 익힌 뒤([Kubernetes 학습 (1)]({% post_url 2026-07-01-k8s-minikube-basics %})), GPU 워크로드까지 다뤄보기 위해 회사 OpenStack 환경의 GPU 인스턴스 위에 직접 K8s 클러스터를 구성했습니다.

```
물리 서버 (GPU 장착)
└── OpenStack (Nova + GPU Passthrough/MIG)
    └── VM (GPU 할당된 인스턴스)
        └── K8s 클러스터 (kubeadm)
            └── Pod (GPU 사용)
```

---

## 인스턴스 구성

- Control Plane용 VM 1개 (일반 인스턴스)
- Worker Node용 VM 1~2개 (GPU 인스턴스, Passthrough / MIG)
- OS: Ubuntu 22.04

---

## 1. kubeadm으로 클러스터 구성

### Control Plane

```bash
apt install kubeadm kubelet kubectl
kubeadm init
```

### Worker Node (GPU 인스턴스)

```bash
kubeadm join <control-plane-ip>:6443 --token ...
```

---

## 2. NVIDIA Device Plugin 설치

K8s가 GPU를 리소스로 인식하려면 NVIDIA Device Plugin이 필요합니다.

```bash
kubectl apply -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/main/deployments/static/nvidia-device-plugin.yml
```

### GPU 인식 확인

```bash
kubectl get nodes -o wide
kubectl describe node <gpu-node> | grep nvidia
```

---

## 3. GPU Pod 실행

Pod spec의 `resources.limits`에 `nvidia.com/gpu`를 지정하면 스케줄러가 GPU를 가진 노드에 Pod를 배치합니다.

```yaml
resources:
  limits:
    nvidia.com/gpu: 1
```

GPU가 있는 노드에만 Pod를 스케줄링하기 위해 Node Selector, Taint & Toleration을 함께 사용합니다. OpenStack에서 MIG/Passthrough로 GPU를 VM에 나눠본 경험 덕분에, K8s 레벨의 GPU 리소스 요청·스케줄링 개념은 자연스럽게 이어졌습니다.

---

## 4. DCGM Exporter + Prometheus 연동

GPU 메트릭 수집을 위해 DCGM Exporter를 DaemonSet으로 배포하고 Prometheus에 연동합니다.

- DCGM Exporter DaemonSet 배포
- Prometheus target에 DCGM Exporter 등록
- GPU 사용률 / 메모리 등 메트릭 수집 확인

---

## 정리 — 자주 나오는 질문

**Q. K8s를 사용해본 경험이 있나요?**

minikube로 기본 실습을 해봤고, 회사 OpenStack 환경에서 GPU 인스턴스 위에 kubeadm으로 K8s를 직접 구성해봤습니다. NVIDIA Device Plugin 설치 후 GPU Pod를 띄우는 것까지 확인했습니다.

**Q. GPU를 K8s에서 어떻게 할당하나요?**

NVIDIA Device Plugin을 통해 K8s가 GPU를 리소스로 인식하게 하고, Pod spec의 `resources.limits`에 `nvidia.com/gpu: 1`로 요청합니다. OpenStack에서 MIG, Passthrough 방식으로 GPU를 VM에 할당해본 경험이 있어서 GPU 가상화 개념 자체는 익숙합니다.

**Q. DaemonSet은 언제 쓰나요?**

모든 노드에 동일한 Pod를 배포해야 할 때 씁니다. 모니터링 에이전트(node_exporter, DCGM Exporter)나 로그 수집기를 각 노드에 배포할 때 적합합니다.

---

## 기술 스택

| 역할 | 기술 |
|------|------|
| 클러스터 구성 | kubeadm |
| 인프라 | OpenStack (Nova, GPU Passthrough/MIG) |
| GPU | NVIDIA A100, H100, B300 |
| GPU 연동 | NVIDIA Device Plugin, DCGM Exporter |
| OS | Ubuntu 22.04 |
| 모니터링 | Prometheus |

---

## 시리즈 구성

- **(1)**: minikube로 로컬 클러스터 구성 및 기본 오브젝트 실습
- **(2)**: OpenStack GPU 인스턴스 위에 Kubernetes 구성 (현재)
