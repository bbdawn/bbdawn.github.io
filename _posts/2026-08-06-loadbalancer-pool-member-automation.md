---
title: "[Project] Terraform + Ansible로 LoadBalancer Pool Member 자동 구축하기"
date: 2026-08-06 10:00:00 +0900
categories: [Project, Openstack]
subcategory: Project
tags: [openstack, terraform, ansible, loadbalancer, pool-member, nginx, iac, automation]
---

## 개요

OpenStack Octavia 로드밸런서의 Pool Member 역할을 할 VM 2대를 Terraform으로 생성하고, VM이 뜨자마자 Ansible로 nginx를 설치 + 서버별로 다른 페이지를 배포하는 과정을 자동화했습니다.

`terraform apply` 한 번만 실행하면 아래 흐름이 순서대로 이어집니다.

```
terraform apply
   │
   ▼
OpenStack에 pool-member-1, pool-member-2 VM 생성
   │
   ▼
VM의 IP를 읽어 ansible/inventory.ini 자동 생성
   │
   ▼
ansible-playbook install-nginx.yml 자동 실행
   │
   ▼
nginx 설치 + pool member별 index.html 배포
```

VM마다 다른 색상의 페이지를 붙여서, 로드밸런서가 두 pool member에 트래픽을 어떻게 분배하는지 브라우저에서 바로 눈으로 확인할 수 있게 만드는 것이 목적입니다.

***

## 디렉토리 구조

```
terraform-ansible-auto/
├── terraform/                  # OpenStack VM 프로비저닝 + Ansible 자동 실행
│   ├── main.tf                 # pool-member-1/2 생성 + inventory.ini 생성 + ansible-playbook 실행
│   ├── variables.tf
│   ├── outputs.tf
│   ├── deploy.sh                # init/fmt/validate/plan/apply 자동 실행 스크립트
│   └── terraform.tfvars        # 실제 값 (gitignore)
└── ansible/                    # DNS 설정 + nginx 설치 + pool member별 페이지 배포
    ├── ansible.cfg
    ├── inventory.ini            # terraform이 자동 생성 (수동 편집 불필요)
    ├── install-nginx.yml
    └── files/
        ├── vm1.html              # pool-member-1 전용 (Blue Server)
        └── vm2.html              # pool-member-2 전용 (Red Server)
```

***

## Terraform — VM 생성 + Ansible 자동 실행

### main.tf

`pool-member-1`, `pool-member-2` 두 인스턴스를 생성하고, 두 VM의 IP가 확정되는 시점에 `local_file`로 Ansible inventory를 생성한 뒤 `null_resource`로 Ansible playbook까지 이어서 실행합니다.

```hcl
resource "openstack_compute_instance_v2" "pool-member-1" {
  name      = "pool-member-1"
  flavor_id = var.flavor_id
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

# pool-member-2도 동일한 구조로 생성
```

VM 생성 후 IP를 읽어 Ansible inventory를 자동으로 만듭니다.

```hcl
resource "local_file" "ansible_inventory" {
  filename = "../ansible/inventory.ini"

  content = <<EOF
[pool_members]
pool-member-1 ansible_host=${openstack_compute_instance_v2.pool-member-1.access_ip_v4}
pool-member-2 ansible_host=${openstack_compute_instance_v2.pool-member-2.access_ip_v4}

[pool_members:vars]
ansible_user=ubuntu
ansible_connection=ssh
ansible_become=true
EOF
}
```

inventory 파일이 만들어지면 바로 이어서 `ansible-playbook`을 실행합니다.

```hcl
resource "null_resource" "run_ansible" {
  depends_on = [local_file.ansible_inventory]

  provisioner "local-exec" {
    working_dir = "../ansible"
    command     = "ansible-playbook install-nginx.yml"
  }
}
```

### variables.tf / terraform.tfvars

```hcl
variable "flavor_id"   { description = "인스턴스 플레이버 ID" }
variable "key_pair"    { description = "OpenStack keypair name" }
variable "image_id"    { description = "Ubuntu 22.04 이미지 ID" }
variable "network_id"  { description = "인스턴스를 연결할 네트워크 ID" }
variable "volume_size" { description = "부팅 볼륨 크기(GB)"; type = number }
```

```hcl
# terraform.tfvars (실제 UUID는 환경마다 다르므로 예시로 대체)
flavor_id   = "<flavor-uuid>"
key_pair    = "keypair"
image_id    = "<ubuntu-22.04-image-uuid>"
network_id  = "<network-uuid>"
volume_size = 20
```

### outputs.tf

```hcl
output "pool_member_ips" {
  value = {
    "pool-member-1" = openstack_compute_instance_v2.pool-member-1.access_ip_v4
    "pool-member-2" = openstack_compute_instance_v2.pool-member-2.access_ip_v4
  }
  description = "IP list for ansible inventory"
}
```

### 실행

```bash
cd terraform
terraform init
terraform apply
```

`terraform apply`가 끝나면 OpenStack 대시보드에서 두 VM이 함께 올라온 것을 확인할 수 있습니다.

![pool-member-1, pool-member-2 인스턴스 생성 결과](/assets/img/posts/pool-member-openstack-instances.png)

***

## Ansible — nginx 설치 + pool member별 페이지 배포

### install-nginx.yml

`vars.poolmember_pages`에서 인벤토리 호스트명과 배포할 파일을 매핑하고, `copy` 태스크가 `inventory_hostname` 기준으로 알맞은 파일을 골라 배포합니다.

```yaml
---
- name: Configure DNS and install nginx
  hosts: pool_members
  become: yes

  vars:
    poolmember_pages:
      pool-member-1: vm1.html
      pool-member-2: vm2.html

  pre_tasks:
    - name: Wait for SSH connection
      wait_for_connection:
        timeout: 300
        sleep: 5

  tasks:
    - name: Configure DNS
      copy:
        dest: /etc/resolv.conf
        content: |
          nameserver 8.8.8.8

    - name: Stop unattended-upgrades service
      systemd:
        name: unattended-upgrades
        state: stopped
      ignore_errors: yes

    - name: Disable unattended-upgrades service
      systemd:
        name: unattended-upgrades
        enabled: no
      ignore_errors: yes

    - name: Install nginx
      apt:
        name: nginx
        state: present
        update_cache: yes
        lock_timeout: 300

    - name: Start nginx service
      service:
        name: nginx
        state: started
        enabled: yes

    - name: Deploy pool member index page
      copy:
        src: "files/{{ poolmember_pages[inventory_hostname] }}"
        dest: /var/www/html/index.html
        owner: www-data
        group: www-data
        mode: '0644'
      when: inventory_hostname in poolmember_pages
```

### files/ — pool member별 페이지

| 파일 | 대상 호스트 | 내용 |
|------|------------|------|
| `vm1.html` | pool-member-1 | 🚀 VM1 Blue Server |
| `vm2.html` | pool-member-2 | 🔥 VM2 Red Server |

두 파일은 배경색(파란색/빨간색)과 타이틀만 다르고 구조는 동일합니다.

```html
<!-- vm1.html -->
<div class="title">🚀 VM1</div>
<div class="subtitle">BLUE SERVER</div>
```

```html
<!-- vm2.html -->
<div class="title">🔥 VM2</div>
<div class="subtitle">RED SERVER</div>
```

### ansible.cfg

```ini
[defaults]
inventory = inventory.ini
host_key_checking = False
private_key_file = ~/.ssh/keypair.pem
```

***

## 실행 결과

`terraform apply`로 실행된 `ansible-playbook install-nginx.yml` 결과입니다. 두 pool member 모두 정상적으로 nginx 설치와 페이지 배포까지 끝난 것을 확인할 수 있습니다.

![ansible-playbook install-nginx.yml 실행 결과](/assets/img/posts/pool-member-ansible-playbook-run.png)

브라우저로 각 VM에 접속하면 pool-member-1은 파란색, pool-member-2는 빨간색 화면으로 구분됩니다.

```bash
terraform output pool_member_ips

curl http://<pool-member-1_IP>/   # 🚀 VM1 Blue Server
curl http://<pool-member-2_IP>/   # 🔥 VM2 Red Server
```

![pool-member-1(Blue), pool-member-2(Red) 페이지](/assets/img/posts/pool-member-nginx-pages.png)

***

## 트러블슈팅

- **SSH 인증 실패 (`Permission denied`)**: Ubuntu 22.04 이미지는 `root` 직접 SSH 로그인이 막혀 있음.
  → `inventory.ini`의 `ansible_user`를 `root` → `ubuntu`로 변경하고, sudo 권한 상승을 위해 `ansible_become=true` 추가.
- **`private_key_file`이 적용 안 되던 문제**: `ansible.cfg`에서 `private_key_file` 옵션을 `[ssh_connection]` 섹션에 뒀더니 무시됨.
  → `[defaults]` 섹션으로 이동해야 정상 적용됨. 또한 `%(HOME)s` 보간이 동작하지 않아 `~/.ssh/<keypair>.pem` 형태로 직접 경로를 지정해야 함.

***

## 정리

| 항목 | 이전 (수동) | 이후 (Terraform + Ansible) |
|------|------------|----------------------------|
| VM 생성 | 대시보드에서 VM 2대 직접 생성 | `terraform apply` 한 번 |
| Ansible inventory | IP 확인 후 수동 작성 | `local_file` 리소스로 자동 생성 |
| nginx 설치 + 페이지 배포 | VM마다 SSH 접속 후 수동 설정 | `null_resource`로 apply 시 자동 실행 |
| pool member 구분 | 접속해서 직접 확인 | VM별 페이지 색상으로 즉시 구분 |

`terraform apply` 한 번으로 VM 생성부터 로드밸런서 pool member 동작 확인까지 이어지는 구조를 만들어, 반복적인 테스트 환경 구축 시간을 크게 줄였습니다.

***

## 관련 글

- [Octavia API로 삭제 안 되는 LB, 운영 자동화하기](/posts/loadbalancer-automation/)
