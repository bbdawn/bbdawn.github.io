---
title: "[Study] Kubernetes 자주 쓰는 명령어 정리"
date: 2026-07-02 12:00:00 +0900
categories: [Study, "Docker & Kubernetes"]
subcategory: Study
tags: [kubernetes, kubectl, cheatsheet, cli]
---

## 개요

minikube 실습([Kubernetes 학습 (1)]({% post_url 2026-07-01-k8s-minikube-basics %}))과 OpenStack GPU 인스턴스 위 클러스터 구성([Kubernetes 학습 (2)]({% post_url 2026-07-01-k8s-openstack-gpu-kubeadm %}))을 진행하면서 반복해서 찾아보게 되는 `kubectl` 명령어를 용도별로 정리했습니다.

---

## 조회 (get / describe)

```bash
kubectl get pods
kubectl get pods -o wide
kubectl get pods -A                      # 모든 네임스페이스
kubectl get pods -l app=web              # 라벨 셀렉터로 필터
kubectl get all -n <namespace>

kubectl describe pod <pod-name>
kubectl describe node <node-name>
kubectl get events --sort-by=.metadata.creationTimestamp
```

`describe`는 리소스 상태 변화(Events)까지 함께 보여주기 때문에, Pod가 `Pending`/`CrashLoopBackOff`일 때 원인을 파악하는 첫 단계로 가장 먼저 사용합니다.

---

## 로그 / 접속

```bash
kubectl logs <pod-name>
kubectl logs -f <pod-name>               # 실시간 tail
kubectl logs <pod-name> -c <container>   # 컨테이너 여러 개인 Pod
kubectl logs --previous <pod-name>       # 재시작 전 로그 (CrashLoopBackOff 원인 파악)

kubectl exec -it <pod-name> -- bash
kubectl exec -it <pod-name> -- sh        # alpine 등 bash 없는 이미지

kubectl port-forward pod/<pod-name> 8080:80
kubectl port-forward svc/<service-name> 8080:80
```

---

## 리소스 생성 / 수정 / 삭제

```bash
kubectl apply -f deployment.yaml
kubectl apply -f ./manifests/             # 디렉터리 전체 적용

kubectl delete -f deployment.yaml
kubectl delete pod <pod-name>
kubectl delete pod <pod-name> --grace-period=0 --force   # 강제 삭제

kubectl edit deployment <name>            # 즉시 편집 (에디터 오픈)
kubectl patch deployment <name> -p '{"spec":{"replicas":3}}'
```

---

## 스케일링 / 롤아웃

```bash
kubectl scale deployment <name> --replicas=3

kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name>
kubectl rollout undo deployment/<name>              # 직전 버전으로 롤백
kubectl rollout undo deployment/<name> --to-revision=2
kubectl rollout restart deployment/<name>           # 이미지 태그 안 바뀐 변경 재배포
```

---

## 네임스페이스 / 컨텍스트

```bash
kubectl config get-contexts
kubectl config current-context
kubectl config use-context <context-name>

kubectl config set-context --current --namespace=<namespace>
kubectl get ns
```

여러 클러스터(minikube, 사내 OpenStack GPU 클러스터 등)를 오가며 작업할 때는 `use-context`로 전환하지 않으면 엉뚱한 클러스터에 명령이 나가는 사고가 나기 쉬워, 작업 전 `current-context` 확인이 습관이 되어야 합니다.

---

## 디버깅에 자주 쓰는 조합

```bash
# Pod가 Pending일 때 스케줄링 실패 원인 확인
kubectl describe pod <pod-name> | grep -A5 Events

# 특정 노드에 배치된 Pod만 조회
kubectl get pods -A -o wide --field-selector spec.nodeName=<node-name>

# 리소스 사용량 확인 (metrics-server 필요)
kubectl top nodes
kubectl top pods

# 임시 디버깅용 Pod 띄우기
kubectl run tmp-shell --rm -it --image=busybox -- sh

# YAML로 현재 정의 뽑아보기 (수정 없이 확인만)
kubectl get deployment <name> -o yaml
```

GPU 노드에서는 `kubectl describe node <gpu-node> | grep nvidia`로 Device Plugin이 리소스를 정상적으로 광고하고 있는지 확인하는 것이 GPU Pod가 `Pending`에 머물 때 가장 먼저 확인하는 지점입니다.

---

## alias

매번 `kubectl`을 타이핑하는 대신 셸 alias를 등록해두면 반복 작업이 훨씬 빨라집니다.

```bash
alias k=kubectl
alias kgp='kubectl get pods'
alias kgpa='kubectl get pods -A'
alias kl='kubectl logs -f'
alias kdp='kubectl describe pod'

# kubectl 자동완성 (bash)
source <(kubectl completion bash)
complete -o default -F __start_kubectl k
```

---

## 기술 스택

| 역할 | 기술 |
|------|------|
| CLI | kubectl |
| 클러스터 | minikube, kubeadm (OpenStack GPU 인스턴스) |
| 메트릭 조회 | metrics-server (`kubectl top`) |
