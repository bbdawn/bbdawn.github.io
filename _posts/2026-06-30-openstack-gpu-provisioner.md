---
title: "OpenStack GPU 프로비저닝 자동화 (Terraform + Ansible)"
date: 2026-06-30 05:00:00 +0900
categories: [Project, GPU]
tags: [terraform, ansible, gpu, openstack, nvidia, dcgm, iac]
---

## 개요

OpenStack 환경에서 GPU 인스턴스를 자동으로 구성하는 Infrastructure as Code 솔루션입니다.

기존에 수동으로 진행하던 nvidia-driver, dcgm-exporter, qemu-guest-agent 설치 과정을 Ansible 기반으로 자동화했으며, **멱등성(Idempotency)을 보장**하여 몇 번을 실행해도 동일한 결과를 보장합니다.

---

## 기술 구조

```
Terraform → OpenStack VM 프로비저닝
    ↓
Ansible Roles
    ├── nvidia  : NVIDIA 드라이버 설치
    ├── dcgm    : DCGM Exporter 설치
    └── qemu    : qemu-guest-agent 설치
```

- **그룹 변수**: driver 버전 및 설정값 중앙 관리
- **사이트 플레이북**: 통합 실행 스크립트

---

## 주요 특징

| 특징 | 설명 |
|------|------|
| 역할 기반 구조 분리 | 각 소프트웨어를 독립 Role로 관리 |
| 변수화된 설정 | 버전·환경을 변수로 관리 |
| Handler 자동 재시작 | 설정 변경 시 서비스 자동 재시작 |
| 멱등성 보장 | 반복 실행해도 동일 결과 |

---

## 사용 절차

```bash
# 1. Terraform으로 VM 프로비저닝
terraform init && terraform apply

# 2. Ansible로 GPU 환경 구성
ansible-playbook -i inventory/hosts.ini site.yml
```

---

## 지원 환경

- **OS**: Ubuntu 22.04
- **GPU**: NVIDIA A100, H100, B300
