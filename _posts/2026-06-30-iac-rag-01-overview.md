---
title: "[Project] GPU IaC 자동화 + RAG 파이프라인 (1) — 프로젝트 개요"
date: 2026-06-30 11:00:00 +0900
categories: [Project, GPU]
subcategory: Project
tags: [openstack, gpu, terraform, ansible, iac, automation, nvidia, rag, ollama, qdrant, llm]
---

## 개요

OpenStack 환경에서 GPU 인스턴스를 자동으로 구성하고, 그 위에 RAG(Retrieval-Augmented Generation) 파이프라인까지 구축하는 프로젝트입니다.

```
[IaC 자동화]
  Terraform → GPU / Qdrant 인스턴스 프로비저닝
  Ansible   → NVIDIA 드라이버 / Ollama / Qdrant 환경 구성
      ↓
[RAG 파이프라인]
  Ollama (GPU 인스턴스) → 임베딩 생성 + LLM 답변
  Qdrant (일반 인스턴스) → 벡터 저장 + 유사도 검색
```

---

## 개발 배경

### 반복되는 GPU 환경 구성

새 GPU 인스턴스를 생성할 때마다 아래 과정을 수동으로 반복했습니다.

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

GPU 모델, OS 버전, 드라이버 버전에 따라 명령어가 달라져 실수가 발생하기 쉬웠고, 여러 인스턴스를 동시에 구성할 때 일관성을 보장하기 어려웠습니다.

### RAG 파이프라인의 필요성

OpenStack, GPU 운영 문서가 여러 곳에 흩어져 있고, 특정 오류 메시지나 설정값을 검색하는 데 시간이 걸렸습니다. 이 문서들을 로컬 LLM 기반으로 질의응답할 수 있도록 RAG 파이프라인을 구성했습니다.

---

## 전체 아키텍처

```
Terraform (인프라 프로비저닝)
  → GPU 인스턴스 생성 (OpenStack)
  → Qdrant 인스턴스 생성
  ↓
Ansible (환경 구성)
  → GPU 인스턴스: NVIDIA 드라이버 + DCGM + qemu-guest-agent + Ollama
  → 일반 인스턴스: Docker + Qdrant
  ↓
Python (RAG 파이프라인)
  → 문서 임베딩 및 Qdrant 저장
  → 질의응답 (로컬 PC에서 실행)
  ↓
모니터링
  → DCGM → Prometheus → Grafana
```

### 인스턴스 구성

```
[GPU 인스턴스]          [일반 인스턴스]
  Ollama                  Qdrant
  - 임베딩 생성            - 벡터 저장
  - 답변 생성              - 유사도 검색
```

### Python 실행 위치

별도 VM 없이 **로컬 PC에서 직접** Ollama·Qdrant API를 호출합니다.

```
[내 PC]
  Python 스크립트
  ├── → HTTP → GPU 인스턴스 (Ollama, port 11434)
  └── → HTTP → Qdrant 인스턴스 (port 6333)
```

OpenStack Security Group에서 아래 포트를 열어야 합니다.

| 인스턴스 | 포트 | 용도 |
|----------|------|------|
| GPU 인스턴스 | 11434 | Ollama API |
| Qdrant 인스턴스 | 6333 | Qdrant API |

---

## repo 구조

```
iac-rag-pipeline/
├── terraform/
│   ├── main.tf          # GPU/Qdrant 인스턴스 생성
│   ├── variables.tf
│   └── outputs.tf       # IP 출력 → Ansible inventory로
├── ansible/
│   ├── inventory/
│   │   └── hosts.ini
│   ├── roles/
│   │   ├── nvidia/      # NVIDIA 드라이버 설치
│   │   ├── dcgm/        # DCGM Exporter 설치
│   │   ├── qemu/        # qemu-guest-agent 설치
│   │   ├── ollama/      # Ollama + 모델 설치
│   │   └── qdrant/      # Docker + Qdrant 설치
│   ├── group_vars/
│   │   └── all.yml      # 버전 및 설정값 중앙 관리
│   └── site.yml
├── docs/                # 임베딩할 문서
├── embed.py             # 문서 → 벡터 저장
├── rag.py               # 질의응답
├── requirements.txt
└── README.md
```

---

## 기술 스택

| 역할 | 기술 |
|------|------|
| IaC | Terraform, Ansible |
| 인프라 | OpenStack (Nova, Neutron) |
| GPU | NVIDIA A100, H100, B300 |
| OS | Ubuntu 22.04 |
| LLM | Ollama (nomic-embed-text, phi3) |
| 벡터 DB | Qdrant (Docker) |
| 언어 | Python |
| 모니터링 | DCGM → Prometheus → Grafana |

---

## 시리즈 구성

- **(1)**: 프로젝트 개요 (현재)
- **(2)**: Terraform으로 인스턴스 프로비저닝
- **(3)**: Ansible로 GPU 환경 구성
- **(4)**: Ollama 설치 및 임베딩
- **(5)**: Qdrant + RAG 파이프라인 구성
