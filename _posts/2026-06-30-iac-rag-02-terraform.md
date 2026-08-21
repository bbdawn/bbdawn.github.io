---
title: "[Project] GPU IaC 자동화 + RAG 파이프라인 (2) — Terraform으로 인스턴스 프로비저닝"
date: 2026-06-30 11:10:00 +0900
categories: [Project, "GPU"]
subcategory: Project
tags: [openstack, gpu, terraform, iac, automation]
---

## 개요

Terraform으로 OpenStack에 GPU 인스턴스(`gpu-vm`)와 Qdrant 인스턴스(`qdrant-vm`), 총 2대를 프로비저닝합니다. 두 인스턴스가 서로 다른 flavor를 쓸 수 있도록 변수를 분리했고, 생성된 IP는 output으로 뽑아 Ansible inventory 파일에 자동으로 기록되도록 구성했습니다.

***

## 디렉토리 구조

```
03-terraform-ansible-gpu/
├── ansible/
│   └── inventory.ini     # terraform이 apply 시 자동 생성 (git 추적 제외)
└── terraform/
    ├── main.tf            # gpu-vm, qdrant-vm 리소스 정의
    ├── variables.tf       # 변수 선언
    ├── outputs.tf         # IP 출력
    ├── terraform.tfvars   # 실제 값 입력 (gitignore 처리)
    └── .gitignore
```

***

## main.tf

```hcl
terraform {
  required_providers {
    openstack = {
      source = "terraform-provider-openstack/openstack"
    }
    local = {
      source = "hashicorp/local"
    }
    null = {
      source = "hashicorp/null"
    }
    time = {
      source = "hashicorp/time"
    }
  }
}

# ~/.config/openstack/clouds.yaml 참조
provider "openstack" {
  cloud = "openstack"
}

# gpu-vm으로 사용할 인스턴스
resource "openstack_compute_instance_v2" "gpu-vm" {
  name      = "gpu-vm"
  flavor_id = var.gpu_flavor_id
  key_pair  = var.key_pair

  user_data = <<EOF
#cloud-config
runcmd:
  - sed -i 's/^PubkeyAuthentication no/PubkeyAuthentication yes/' /etc/ssh/sshd_config
  - systemctl restart ssh
EOF

  block_device {
    uuid                  = var.image_id
    source_type           = "image"
    destination_type      = "volume"
    volume_size           = var.volume_size
    boot_index            = 0
    delete_on_termination = true
  }

  network {
    uuid = var.network_id
  }

  security_groups = ["default"]
}

# qdrant-vm으로 사용할 인스턴스
resource "openstack_compute_instance_v2" "qdrant-vm" {
  name      = "qdrant-vm"
  flavor_id = var.qdrant_flavor_id
  key_pair  = var.key_pair

  user_data = <<EOF
#cloud-config
runcmd:
  - sed -i 's/^PubkeyAuthentication no/PubkeyAuthentication yes/' /etc/ssh/sshd_config
  - systemctl restart ssh
EOF

  block_device {
    uuid                  = var.image_id
    source_type           = "image"
    destination_type      = "volume"
    volume_size           = var.volume_size
    boot_index            = 0
    delete_on_termination = true
  }

  network {
    uuid = var.network_id
  }

  security_groups = ["default"]
}

# 생성된 IP를 Ansible inventory 파일로 그대로 흘려보냄
resource "local_file" "ansible_inventory" {
  filename = "../ansible/inventory.ini"

  content = <<EOF
[instance]
gpu-vm ansible_host=${openstack_compute_instance_v2.gpu-vm.access_ip_v4}
qdrant-vm ansible_host=${openstack_compute_instance_v2.qdrant-vm.access_ip_v4}

[instance:vars]
ansible_user=ubuntu
ansible_connection=ssh
ansible_become=true
EOF
}

# NVIDIA driver / Qdrant 설치 플레이북이 아직 없어서 당분간 비활성화
# resource "null_resource" "run_ansible" {
#   depends_on = [local_file.ansible_inventory]
#
#   provisioner "local-exec" {
#     working_dir = "../ansible"
#     command     = "ansible-playbook install-xxx.yml"
#   }
# }
```

두 인스턴스 모두 볼륨 부팅(`block_device`) 방식으로 만들고, `user_data`로 cloud-init 단계에서 pubkey 인증을 강제로 켜줍니다. `local_file` 리소스는 실제로 뭘 실행하는 게 아니라 인벤토리 텍스트 파일 하나만 쓰는 거라, apply 순서와 무관하게 안전하게 둘 수 있습니다.

***

## variables.tf

```hcl
variable "gpu_flavor_id" {
  description = "gpu instance flavor id"
}

variable "qdrant_flavor_id" {
  description = "qdrant instance flavor id"
}

variable "key_pair" {
  description = "OpenStack keypair name"
  type        = string
}

variable "image_id" {
  description = "Ubuntu 22.04 image id"
}

variable "network_id" {
  description = "Openstack network id"
}

variable "volume_size" {
  description = "booting volume size(GB)"
  type        = number
}
```

처음엔 `flavor_id` 변수 하나를 두 인스턴스가 같이 썼는데, GPU와 Qdrant는 필요한 스펙이 다르기 때문에 `gpu_flavor_id` / `qdrant_flavor_id`로 분리했습니다.

***

## terraform.tfvars

```hcl
gpu_flavor_id    = "<GPU 플레이버 ID>"
qdrant_flavor_id = "<Qdrant용 일반 플레이버 ID>"
key_pair         = "<keypair 이름>"
image_id         = "<Ubuntu 22.04 이미지 ID>"
network_id       = "<네트워크 ID>"
volume_size      = 20
```

> 이 파일은 실제 인프라 값(플레이버/이미지/네트워크 ID)이 들어가기 때문에 `.gitignore`로 커밋 대상에서 뺐습니다. GPU 플레이버 ID는 Horizon 콘솔의 **관리자 > 컴퓨트 > 플레이버**에서 확인합니다.
>
> ⚠️ 아직 정리 중인 부분: `gpu_flavor_id`와 `qdrant_flavor_id`를 변수로는 분리해놨지만, 실제 값을 아직 다르게 채워 넣지 못해서 지금은 두 인스턴스가 같은 플레이버로 뜨고 있습니다. Qdrant용 플레이버를 확정하는 대로 값만 바꿔서 재적용할 예정입니다.

***

## outputs.tf

```hcl
output "instance" {
  value = {
    "gpu-vm"    = openstack_compute_instance_v2.gpu-vm.access_ip_v4
    "qdrant-vm" = openstack_compute_instance_v2.qdrant-vm.access_ip_v4
  }

  description = "IP list for ansible inventory"
}
```

처음엔 `gpu_ip`, `qdrant_ip`를 따로 뽑을 계획이었는데, 결과적으로는 맵 하나(`instance`)로 합쳐서 출력하도록 정리했습니다. `terraform output instance`로 전체를 보거나, `terraform output -json instance`로 스크립트에서 파싱해 쓸 수 있습니다.

***

## .gitignore

```
# IDE
.idea/
.vscode/

# Terraform
.terraform/
.terraform.lock.hcl
*.tfstate
*.tfstate.*
*.tfvars
!terraform.tfvars.example
crash.log
crash.*.log
override.tf
override.tf.json
*_override.tf
*_override.tf.json

# OS
.DS_Store
```

`terraform.tfstate`엔 생성된 인스턴스의 실제 IP·리소스 ID가 그대로 저장되고, `.terraform/`엔 다운로드한 provider 바이너리가 들어가기 때문에 둘 다 git에 올릴 이유가 없습니다. `terraform.tfvars`도 실제 인프라 값이라 같이 제외했습니다.

***

## 실행

```bash
cd terraform

# 최초 1회 (provider 설치) — provider나 backend를 바꾼 게 아니면 매번 안 해도 됨
terraform init

# 코드 포맷 정리 (선택)
terraform fmt

# 문법 검사, 실제 리소스 존재 여부는 검사 안 함 (선택)
terraform validate

# 뭐가 바뀌는지 미리 확인
terraform plan

# 실제로 생성
terraform apply
```

***

## 트러블슈팅: `Invalid key_name provided`

첫 apply 때 두 인스턴스 모두 아래 에러로 실패했습니다.

```
Error: Error creating OpenStack server: ... got 400 instead:
{"badRequest": {"code": 400, "message": "Invalid key_name provided."}}
```

**원인**: `terraform.tfvars`에 넣은 `key_pair` 이름 자체는 맞았지만, OpenStack의 Nova keypair는 **사용자(user) 단위로 소유**되는 리소스입니다. `clouds.yaml`에 설정된 인증 계정이 그 keypair를 만든 계정과 다르면 "그런 키페어 없다"는 400을 받게 됩니다.

**해결**: `~/.config/openstack/clouds.yaml`의 `username`을 keypair를 실제로 갖고 있는 계정으로 바꿔주니 바로 해결됐습니다.

```yaml
clouds:
  openstack:
    auth:
      auth_url: "https://<controller-ip>:15000/v3"
      username: "<keypair를 소유한 계정>"
      password: "..."
      project_name: "admin"
      user_domain_name: "Default"
      project_domain_name: "Default"
```

***

## 결과

두 인스턴스 모두 정상적으로 생성되고 IP가 할당됐습니다. `local_file.ansible_inventory`도 같이 생성되면서 `../ansible/inventory.ini`에 두 VM의 IP가 자동으로 기록됩니다.

![terraform apply 결과](/assets/img/posts/terraform-apply-gpu-qdrant-vm.png)

***

## 남은 작업

- `gpu_flavor_id` / `qdrant_flavor_id`에 실제로 다른 값 채워 넣기 (현재는 동일한 플레이버로 임시 적용된 상태)
- GPU 드라이버 / Qdrant 설치용 Ansible 플레이북 작성 후 `null_resource.run_ansible` 다시 활성화
- 다음 편에서는 이 인벤토리를 받아서 Ansible로 실제 소프트웨어를 설치하는 과정을 다룹니다.
