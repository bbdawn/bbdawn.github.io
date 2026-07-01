---
title: "[Project] GPU IaC 자동화 + RAG 파이프라인 (3) — Ansible로 GPU 환경 구성"
date: 2026-06-30 11:20:00 +0900
categories: [Project, GPU]
subcategory: Project
tags: [openstack, gpu, ansible, iac, automation, nvidia, dcgm]
---

## 개요

Terraform으로 생성한 인스턴스에 Ansible로 NVIDIA 드라이버, DCGM, qemu-guest-agent를 설치합니다. 역할(Role) 기반으로 구조를 분리해 버전 관리와 재사용이 쉽도록 했습니다.

---

## 디렉토리 구조

```
ansible/
├── inventory/
│   └── hosts.ini          # Terraform output으로 채운 IP
├── roles/
│   ├── nvidia/            # NVIDIA 드라이버 설치
│   ├── dcgm/              # DCGM Exporter 설치
│   ├── qemu/              # qemu-guest-agent 설치
│   ├── ollama/            # Ollama + 모델 설치 (4편 참조)
│   └── qdrant/            # Docker + Qdrant 설치 (5편 참조)
├── group_vars/
│   └── all.yml            # 버전 및 설정값 중앙 관리
└── site.yml
```

---

## group_vars/all.yml

버전 관리는 이 파일 하나에서만 합니다.

```yaml
nvidia_driver_version: "535"
dcgm_version: "3.3.0"
cuda_keyring_deb: "https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.1-1_all.deb"
```

---

## roles/nvidia — NVIDIA 드라이버 설치

```yaml
# roles/nvidia/tasks/main.yml
- name: NVIDIA 드라이버 설치
  apt:
    name: "nvidia-driver-{{ nvidia_driver_version }}"
    state: present
    update_cache: yes
  notify: Reboot

- name: nvidia-smi 동작 확인
  command: nvidia-smi
  register: smi_result
  changed_when: false

- name: GPU 상태 출력
  debug:
    msg: "{{ smi_result.stdout_lines[0] }}"
```

```yaml
# roles/nvidia/handlers/main.yml
- name: Reboot
  reboot:
    reboot_timeout: 300
```

---

## roles/dcgm — DCGM Exporter 설치

```yaml
# roles/dcgm/tasks/main.yml
- name: CUDA keyring 다운로드
  get_url:
    url: "{{ cuda_keyring_deb }}"
    dest: /tmp/cuda-keyring.deb

- name: CUDA keyring 설치
  apt:
    deb: /tmp/cuda-keyring.deb

- name: DCGM 설치
  apt:
    name: datacenter-gpu-manager
    state: present
    update_cache: yes

- name: DCGM 서비스 시작 및 활성화
  systemd:
    name: nvidia-dcgm
    state: started
    enabled: yes

- name: DCGM Exporter 시작
  systemd:
    name: dcgm-exporter
    state: started
    enabled: yes
```

---

## roles/qemu — qemu-guest-agent 설치

```yaml
# roles/qemu/tasks/main.yml
- name: qemu-guest-agent 설치
  apt:
    name: qemu-guest-agent
    state: present
    update_cache: yes

- name: qemu-guest-agent 시작 및 활성화
  systemd:
    name: qemu-guest-agent
    state: started
    enabled: yes
```

---

## site.yml

```yaml
- hosts: gpu
  become: yes
  roles:
    - nvidia
    - dcgm
    - qemu
    - ollama   # 4편

- hosts: qdrant
  become: yes
  roles:
    - qdrant   # 5편
```

---

## 멱등성 보장

`state: present`를 사용하면 이미 설치된 경우 건너뜁니다. 몇 번을 실행해도 동일한 결과를 보장합니다.

```bash
# 안전하게 재실행 가능
ansible-playbook -i inventory/hosts.ini site.yml
```

---

## 실행 결과

| 항목 | 이전 (수동) | 이후 (Ansible) |
|------|------------|----------------|
| 인스턴스당 구성 시간 | 20~30분 | 5분 이내 |
| 버전 관리 | 명령어에 직접 입력 | `all.yml` 한 곳에서 관리 |
| 일관성 | 실수 가능 | 멱등성 보장 |
| 다중 인스턴스 | 대수만큼 반복 | 병렬 실행 |
