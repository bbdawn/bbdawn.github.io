---
title: "[Project] Kubernetes 학습 (1) — minikube로 로컬 클러스터 구성 및 기본 오브젝트 실습"
date: 2026-07-01 09:00:00 +0900
categories: [Project, Kubernetes]
subcategory: Project
tags: [kubernetes, k8s, minikube, pod, deployment, service, configmap, daemonset, prometheus]
---

## 개요

기본 K8s 운영 능력을 확보하기 위해 학습을 시작했습니다. GPU 없이도 핵심 개념을 익힐 수 있도록 로컬 macOS 환경에 minikube로 단일 노드 클러스터를 구성하고, Pod부터 모니터링 연동까지 기본 오브젝트를 하나씩 실습했습니다.

```
[로컬 macOS]
  minikube (단일 노드 클러스터)
    └── Pod → Deployment → Service
                └── ConfigMap 연동
    └── DaemonSet (node_exporter)
                └── Prometheus 메트릭 수집
```

GPU 워크로드까지 다루는 실전 환경 구성은 다음 글([Kubernetes 학습 (2)]({% post_url 2026-07-01-k8s-openstack-gpu-kubeadm %}))에서 이어집니다.

---

## 핵심 개념 정리

### 핵심 개념

| 개념 | 설명 |
|------|------|
| Pod | K8s에서 가장 작은 배포 단위. 컨테이너 1개 이상을 묶은 것 |
| Node | Pod가 실행되는 서버 (Worker Node) |
| Cluster | Node들의 집합 |
| Control Plane | 클러스터 관리 (API Server, Scheduler, etcd, Controller Manager) |
| Namespace | 클러스터 내 논리적 분리 단위 |

### 워크로드

| 오브젝트 | 설명 |
|----------|------|
| Deployment | Pod 복제본 관리, 롤링 업데이트 |
| ReplicaSet | Pod 개수 유지 |
| DaemonSet | 모든 Node에 Pod 1개씩 배포 (모니터링 에이전트 배포에 사용) |
| StatefulSet | 상태 있는 애플리케이션 (DB 등) |
| Job / CronJob | 배치 작업 |

### 네트워크

| 오브젝트 | 설명 |
|----------|------|
| Service | Pod에 접근하는 방법 (ClusterIP / NodePort / LoadBalancer) |
| Ingress | 외부 HTTP 트래픽 라우팅 |
| DNS | 서비스 이름으로 내부 통신 |

### 설정 / 시크릿

| 오브젝트 | 설명 |
|----------|------|
| ConfigMap | 설정값 외부화 |
| Secret | 민감한 값 (패스워드, 토큰 등) |

### 스토리지

| 오브젝트 | 설명 |
|----------|------|
| PersistentVolume (PV) | 스토리지 리소스 |
| PersistentVolumeClaim (PVC) | Pod에서 스토리지 요청 |

---

## 환경 구성

```bash
brew install minikube
minikube start
kubectl get nodes
```

---

## 실습 내용

### 1. nginx Pod 직접 띄우기

가장 단순한 단위인 Pod를 직접 생성해 동작을 확인합니다.

```bash
kubectl run nginx --image=nginx
kubectl get pods
```

### 2. Deployment로 Pod 3개 복제

Deployment를 통해 ReplicaSet이 Pod 개수를 유지하는 것을 확인합니다.

```bash
kubectl create deployment nginx-deploy --image=nginx --replicas=3
kubectl get pods -o wide
```

### 3. Service로 외부 노출 (ClusterIP / NodePort)

```bash
kubectl expose deployment nginx-deploy --port=80 --type=ClusterIP
kubectl expose deployment nginx-deploy --port=80 --type=NodePort
minikube service nginx-deploy --url
```

### 4. ConfigMap으로 환경변수 분리

```bash
kubectl create configmap nginx-config --from-literal=ENV=dev
kubectl get configmap nginx-config -o yaml
```

### 5. Pod 장애 복구 확인

Pod를 의도적으로 종료시켜 Deployment/ReplicaSet이 자동으로 재생성하는 동작을 확인합니다.

```bash
kubectl delete pod <pod-name>
kubectl get pods -w
```

### 6. node_exporter DaemonSet 배포

모든 노드에 동일한 모니터링 에이전트를 배포하는 DaemonSet의 용도를 실습으로 확인합니다.

```bash
kubectl apply -f node-exporter-daemonset.yaml
kubectl get daemonset
```

### 7. Prometheus 연동

node_exporter가 노출하는 메트릭을 Prometheus가 수집하도록 연동하고, GPU 환경에서도 동일한 방식(DCGM Exporter)이 적용된다는 점을 확인했습니다.

---

## 기본 명령어 레퍼런스

```bash
kubectl get pods
kubectl get nodes
kubectl describe pod <name>
kubectl logs <pod>
kubectl exec -it <pod> -- bash
```

---

## 기술 스택

| 역할 | 기술 |
|------|------|
| 로컬 클러스터 | minikube |
| CLI | kubectl |
| 모니터링 | node_exporter, Prometheus |
| OS | macOS |

---

## 시리즈 구성

- **(1)**: minikube로 로컬 클러스터 구성 및 기본 오브젝트 실습 (현재)
- **(2)**: OpenStack GPU 인스턴스 위에 Kubernetes 구성 (kubeadm + NVIDIA Device Plugin + DCGM Exporter)
