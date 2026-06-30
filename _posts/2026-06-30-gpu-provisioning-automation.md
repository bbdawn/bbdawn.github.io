---
title: "GPU 인스턴스 환경 구성, 매번 수동 설치에서 IaC로 자동화하기"
date: 2026-06-30 11:00:00 +0900
categories: [Project, GPU]
subcategory: Hands-on
tags: [openstack, gpu, terraform, ansible, iac, automation, nvidia, dcgm]
---

## 문제 상황

새로운 GPU 인스턴스를 생성할 때마다 아래 작업을 수동으로 반복해야 했습니다.

```
GPU 인스턴스 생성 (OpenStack)
  ↓
SSH 접속
  ↓
NVIDIA 드라이버 설치
  ↓
DCGM Exporter 설치 및 설정
  ↓
qemu-guest-agent 설치
  ↓
각 서비스 시작 및 활성화 확인
```

문제는 단순 반복 작업임에도 GPU 모델, OS 버전, 드라이버 버전에 따라 명령어가 달라져 실수가 생기기 쉬웠고, 여러 인스턴스를 동시에 구성할 때 일관성을 보장하기 어려웠습니다.

---

## 기존 수동 처리 방식

```bash
# 매번 직접 SSH 접속 후 순서대로 실행
ssh ubuntu@<gpu-instance-ip>

# NVIDIA 드라이버 설치
sudo apt-get install -y nvidia-driver-535

# DCGM 설치
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt-get install -y datacenter-gpu-manager

# qemu-guest-agent 설치
sudo apt-get install -y qemu-guest-agent
sudo systemctl enable --now qemu-guest-agent
```

드라이버 버전이 바뀌거나 인스턴스가 여러 대면 처음부터 다시 반복해야 했습니다.

---

## 자동화 방향

**Terraform + Ansible** 조합으로 인스턴스 생성부터 환경 구성까지 전 과정을 자동화했습니다.

```
Terraform
  → OpenStack GPU 인스턴스 프로비저닝
      ↓
Ansible
  ├── nvidia role : NVIDIA 드라이버 설치
  ├── dcgm role   : DCGM Exporter 설치 및 설정
  └── qemu role   : qemu-guest-agent 설치
```

---

## 구현

### Terraform — 인스턴스 생성

```hcl
resource "openstack_compute_instance_v2" "gpu" {
  name      = "gpu-instance"
  flavor_id = var.gpu_flavor_id
  image_id  = var.image_id
  key_pair  = var.key_pair

  network {
    name = var.network_name
  }
}
```

### Ansible — 역할 기반 구조

```
ansible/
├── roles/
│   ├── nvidia/   # 드라이버 버전 변수화
│   ├── dcgm/     # exporter 설정 변수화
│   └── qemu/     # guest-agent 설치
├── group_vars/
│   └── all.yml   # 버전 및 설정값 중앙 관리
└── site.yml
```

버전 관리는 `group_vars/all.yml` 한 곳에서만 수정하면 됩니다.

```yaml
# group_vars/all.yml
nvidia_driver_version: "535"
dcgm_version: "3.3.0"
```

### 멱등성 보장

몇 번을 실행해도 동일한 결과를 보장합니다. 이미 설치된 경우 건너뜁니다.

```yaml
- name: NVIDIA 드라이버 설치
  apt:
    name: "nvidia-driver-{{ nvidia_driver_version }}"
    state: present   # 이미 설치돼 있으면 건너뜀
```

---

## 전체 실행 순서

```bash
# 1. 인스턴스 생성
terraform apply

# 2. 환경 구성 (전체 자동화)
ansible-playbook -i inventory/hosts.ini site.yml
```

---

## 효과

| 항목 | 이전 | 이후 |
|------|------|------|
| 인스턴스당 구성 시간 | 20~30분 (수동) | 5분 이내 (자동) |
| 버전 관리 | 명령어에 직접 입력 | 변수 파일 한 곳에서 관리 |
| 일관성 | 실수 가능 | 멱등성 보장 |
| 다중 인스턴스 | 대수만큼 반복 | 병렬 실행 가능 |

---

## 기술 스택

- **IaC**: Terraform, Ansible
- **인프라**: OpenStack (Nova)
- **GPU**: NVIDIA A100, H100, B300
- **OS**: Ubuntu 22.04
