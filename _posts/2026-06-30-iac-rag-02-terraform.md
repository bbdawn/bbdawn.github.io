---
title: "[Project] GPU IaC 자동화 + RAG 파이프라인 (2) — Terraform으로 인스턴스 프로비저닝"
date: 2026-06-30 11:10:00 +0900
categories: [Project, "GPU"]
subcategory: Project
tags: [openstack, gpu, terraform, iac, automation]
---

## 개요

Terraform으로 OpenStack에 GPU 인스턴스와 Qdrant 인스턴스를 프로비저닝합니다. 출력값(output)으로 IP를 뽑아 Ansible inventory에 바로 연결할 수 있게 구성했습니다.

***

<br>
Terraform 설치 
`brew install hashicorp/tap/terraform`
![image.png](/assets/img/posts/1783497292070-image.png)

<br>
<br>
<br>
<br>
<br>
## 디렉토리 구조

```
terraform/
├── main.tf        # 인스턴스 리소스 정의
├── variables.tf   # 변수 선언
├── outputs.tf     # IP 출력
└── terraform.tfvars  # 실제 값 (gitignore 권장)
```

***

## main.tf

```hcl
terraform {
  required_providers {
    openstack = {
      source  = "terraform-provider-openstack/openstack"
      version = "~> 1.53"
    }
  }
}

provider "openstack" {
  cloud = "openstack"  # ~/.config/openstack/clouds.yaml 참조
}

# GPU 인스턴스 (Ollama 실행)
resource "openstack_compute_instance_v2" "gpu" {
  name      = "rag-gpu"
  flavor_id = var.gpu_flavor_id
  image_id  = var.image_id
  key_pair  = var.key_pair

  network {
    name = var.network_name
  }

  security_groups = ["default", "ollama-access"]
}

# 일반 인스턴스 (Qdrant 실행)
resource "openstack_compute_instance_v2" "qdrant" {
  name      = "rag-qdrant"
  flavor_id = var.flavor_id
  image_id  = var.image_id
  key_pair  = var.key_pair

  network {
    name = var.network_name
  }

  security_groups = ["default", "qdrant-access"]
}
```

***

## variables.tf

```hcl
variable "gpu_flavor_id" {
  description = "MIG 슬라이스가 포함된 GPU 플레이버 ID"
}

variable "flavor_id" {
  description = "일반 인스턴스 플레이버 ID"
}

variable "image_id" {
  description = "Ubuntu 22.04 이미지 ID"
}

variable "key_pair" {
  description = "SSH 키페어 이름"
}

variable "network_name" {
  description = "인스턴스를 연결할 네트워크 이름"
}
```

***

## outputs.tf

```hcl
output "gpu_ip" {
  value       = openstack_compute_instance_v2.gpu.access_ip_v4
  description = "GPU 인스턴스 IP (Ansible inventory에 사용)"
}

output "qdrant_ip" {
  value       = openstack_compute_instance_v2.qdrant.access_ip_v4
  description = "Qdrant 인스턴스 IP"
}
```

***

## 실행

```bash
cd terraform

# 초기화
terraform init

# 변경 사항 미리 확인
terraform plan

# 인스턴스 생성
terraform apply

# IP 확인
terraform output gpu_ip
terraform output qdrant_ip
```

***

## Ansible inventory 연결

Terraform output을 Ansible `hosts.ini`에 바로 넣어 연결합니다.

```bash
# output 값을 hosts.ini에 기록
GPU_IP=$(terraform output -raw gpu_ip)
QDRANT_IP=$(terraform output -raw qdrant_ip)

cat > ../ansible/inventory/hosts.ini <<EOF
[gpu]
${GPU_IP} ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/id_rsa

[qdrant]
${QDRANT_IP} ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/id_rsa
EOF
```

***

## Security Group 참고

Terraform으로 Security Group을 생성하거나 OpenStack 대시보드에서 수동으로 추가합니다.

| Security Group | 인바운드 포트 | 대상 |
| -------------- | ------- | --- |
| ollama-access | 11434/tcp | 로컬 PC IP |
| qdrant-access | 6333/tcp | 로컬 PC IP |
