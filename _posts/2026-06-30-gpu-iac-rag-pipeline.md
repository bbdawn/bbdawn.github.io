---
title: "[Project] GPU 인스턴스 프로비저닝 & RAG 파이프라인 구성 (OpenStack + Terraform + Ansible + Ollama + Qdrant)"
date: 2026-06-30 11:00:00 +0900
categories: [Project, GPU]
subcategory: Projects
tags: [openstack, gpu, terraform, ansible, iac, automation, nvidia, dcgm, rag, ollama, qdrant, python, llm]
---

## 개요

OpenStack 환경에서 GPU 인스턴스를 자동으로 구성하고, 그 위에 RAG(Retrieval-Augmented Generation) 파이프라인까지 구축하는 전체 과정을 정리합니다.

```
[IaC 자동화]
  Terraform → GPU/Qdrant 인스턴스 프로비저닝
  Ansible   → NVIDIA 드라이버 / Ollama / Qdrant 환경 구성
      ↓
[RAG 파이프라인]
  Ollama (GPU 인스턴스) → 임베딩 생성 + LLM 답변
  Qdrant (일반 인스턴스) → 벡터 저장 + 유사도 검색
```

---

## 1부: GPU 환경 자동화

### 문제 상황

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

GPU 모델, OS 버전, 드라이버 버전에 따라 명령어가 달라져 실수가 생기기 쉬웠고, 여러 인스턴스를 동시에 구성할 때 일관성을 보장하기 어려웠습니다.

### 기존 수동 처리 방식

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

### 자동화 방향

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

### 구현

#### Terraform — 인스턴스 생성

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

#### Ansible — 역할 기반 구조

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

#### 멱등성 보장

몇 번을 실행해도 동일한 결과를 보장합니다. 이미 설치된 경우 건너뜁니다.

```yaml
- name: NVIDIA 드라이버 설치
  apt:
    name: "nvidia-driver-{{ nvidia_driver_version }}"
    state: present   # 이미 설치돼 있으면 건너뜀
```

### 주요 특징

| 특징 | 설명 |
|------|------|
| 역할 기반 구조 분리 | 각 소프트웨어를 독립 Role로 관리 |
| 변수화된 설정 | 버전·환경을 변수로 관리 |
| Handler 자동 재시작 | 설정 변경 시 서비스 자동 재시작 |
| 멱등성 보장 | 반복 실행해도 동일 결과 |

### 효과

| 항목 | 이전 | 이후 |
|------|------|------|
| 인스턴스당 구성 시간 | 20~30분 (수동) | 5분 이내 (자동) |
| 버전 관리 | 명령어에 직접 입력 | 변수 파일 한 곳에서 관리 |
| 일관성 | 실수 가능 | 멱등성 보장 |
| 다중 인스턴스 | 대수만큼 반복 | 병렬 실행 가능 |

---

## 2부: RAG 파이프라인 구성

### 시나리오: OpenStack 문서 기반 질의응답 시스템

```
[문서 준비] → [임베딩 생성] → [벡터 DB 저장]
                                      ↓
[사용자 질문] → [질문 임베딩] → [유사 문서 검색] → [LLM 답변]
```

### 환경 구성

OpenStack 인스턴스 2개가 필요합니다.

```
[GPU 인스턴스]          [일반 인스턴스]
  Ollama                  Qdrant
  - 임베딩 생성            - 벡터 저장
  - 답변 생성              - 유사도 검색
```

### Step 1. GPU 인스턴스 생성

기존 `gpu-instance` 기능으로 MIG 인스턴스를 생성합니다.

- MIG 슬라이스 1개 할당
- RAM 16GB 이상

### Step 2. Ollama 설치 (GPU 인스턴스)

```bash
# Ollama 설치
curl -fsSL https://ollama.com/install.sh | sh

# GPU 사용 확인
ollama run llama3 "안녕"

# 임베딩 모델 다운로드
ollama pull nomic-embed-text

# LLM 모델 다운로드
ollama pull phi3
```

### Step 3. Qdrant 설치 (일반 인스턴스)

```bash
docker run -d \
  --name qdrant \
  -p 6333:6333 \
  -v qdrant_storage:/qdrant/storage \
  qdrant/qdrant

# 실행 확인
curl http://localhost:6333
```

### Step 4. 문서 준비

OpenStack 공식 문서나 사내 운영 문서를 준비합니다.

```
docs/
├── neutron.txt     # Neutron 개념 정리
├── nova.txt        # Nova 개념 정리
├── cinder.txt      # Cinder 개념 정리
└── gpu-setup.txt   # GPU 설정 가이드
```

### Step 5. 문서 임베딩 및 저장

```python
# embed.py
from ollama import Client
from qdrant_client import QdrantClient
from qdrant_client.models import VectorParams, Distance
import os

ollama = Client(host="http://<GPU_INSTANCE_IP>:11434")
qdrant = QdrantClient(host="<QDRANT_INSTANCE_IP>", port=6333)

# 컬렉션 생성
qdrant.create_collection(
    collection_name="openstack-docs",
    vectors_config=VectorParams(size=768, distance=Distance.COSINE)
)

def split_text(text, chunk_size=200):
    sentences = text.split(".")
    chunks = []
    current = ""
    for s in sentences:
        current += s + "."
        if len(current) >= chunk_size:
            chunks.append(current.strip())
            current = ""
    if current:
        chunks.append(current.strip())
    return chunks

doc_id = 0
for filename in os.listdir("docs/"):
    with open(f"docs/{filename}", "r") as f:
        text = f.read()

    chunks = split_text(text)

    for chunk in chunks:
        response = ollama.embeddings(model="nomic-embed-text", prompt=chunk)
        embedding = response["embedding"]

        qdrant.upsert(
            collection_name="openstack-docs",
            points=[{
                "id": doc_id,
                "vector": embedding,
                "payload": {"text": chunk, "source": filename}
            }]
        )
        doc_id += 1
        print(f"저장 완료: {filename} - {chunk[:50]}...")

print("모든 문서 임베딩 완료")
```

### Step 6. 질의응답 (RAG)

```python
# rag.py
from ollama import Client
from qdrant_client import QdrantClient

ollama = Client(host="http://<GPU_INSTANCE_IP>:11434")
qdrant = QdrantClient(host="<QDRANT_INSTANCE_IP>", port=6333)

def rag(query):
    # 1. 질문을 벡터로 변환
    response = ollama.embeddings(model="nomic-embed-text", prompt=query)
    query_vector = response["embedding"]

    # 2. Qdrant에서 유사한 문서 검색
    results = qdrant.search(
        collection_name="openstack-docs",
        query_vector=query_vector,
        limit=3
    )

    # 3. 컨텍스트 구성
    context = "\n".join([r.payload["text"] for r in results])
    print(f"\n[검색된 문서]\n{context}\n")

    # 4. LLM으로 답변 생성
    response = ollama.chat(
        model="phi3",
        messages=[
            {"role": "system", "content": "주어진 문서를 참고해서 질문에 답변하세요."},
            {"role": "user", "content": f"문서:\n{context}\n\n질문: {query}"}
        ]
    )

    return response["message"]["content"]

while True:
    query = input("\n질문: ")
    if query == "exit":
        break
    print(f"\n답변: {rag(query)}")
```

### Step 7. 모니터링

```bash
# GPU 사용률 확인 (DCGM)
dcgmi dmon -s u

# Ollama 응답 시간 확인
curl -w "\n응답시간: %{time_total}s\n" \
  http://<GPU_INSTANCE_IP>:11434/api/embeddings \
  -d '{"model": "nomic-embed-text", "prompt": "test"}'
```

### 실행 결과 예시

```
질문: OpenStack에서 네트워크 담당 서비스가 뭐야?

[검색된 문서]
Neutron은 OpenStack의 네트워크 서비스입니다.
Network, Subnet, Router, Port를 관리합니다...

답변: OpenStack에서 네트워크를 담당하는 서비스는
Neutron입니다. Neutron은 Network, Subnet, Router,
Port, Floating IP, Security Group을 관리합니다.
```

---

## 전체 흐름

```
Terraform (인프라 프로비저닝)
  → GPU 인스턴스 생성 (OpenStack)
  → Qdrant 인스턴스 생성
  ↓
Ansible (환경 구성)
  → GPU 인스턴스: NVIDIA 드라이버 + DCGM + Ollama 설치
  → 일반 인스턴스: Docker + Qdrant 설치
  ↓
Python (RAG 파이프라인)
  → 문서 임베딩 및 저장
  → 질의응답
  ↓
모니터링
  → DCGM → Prometheus → Grafana
```

### Terraform — 인스턴스 프로비저닝

```hcl
# main.tf

resource "openstack_compute_instance_v2" "gpu" {
  name      = "rag-gpu"
  flavor_id = var.gpu_flavor_id
  image_id  = var.image_id
  key_pair  = var.key_pair

  network {
    name = var.network_name
  }
}

resource "openstack_compute_instance_v2" "qdrant" {
  name      = "rag-qdrant"
  flavor_id = var.flavor_id
  image_id  = var.image_id
  key_pair  = var.key_pair

  network {
    name = var.network_name
  }
}

output "gpu_ip" {
  value = openstack_compute_instance_v2.gpu.access_ip_v4
}

output "qdrant_ip" {
  value = openstack_compute_instance_v2.qdrant.access_ip_v4
}
```

### Ansible — 환경 구성

```yaml
# roles/ollama/tasks/main.yml
- name: Ollama 설치
  shell: curl -fsSL https://ollama.com/install.sh | sh

- name: Ollama 서비스 시작
  systemd:
    name: ollama
    state: started
    enabled: yes

- name: 임베딩 모델 다운로드
  shell: ollama pull nomic-embed-text

- name: LLM 모델 다운로드
  shell: ollama pull phi3
```

```yaml
# roles/qdrant/tasks/main.yml
- name: Docker 설치
  apt:
    name: docker.io
    state: present
    update_cache: yes

- name: Qdrant 컨테이너 실행
  docker_container:
    name: qdrant
    image: qdrant/qdrant
    state: started
    restart_policy: always
    ports:
      - "6333:6333"
    volumes:
      - qdrant_storage:/qdrant/storage
```

```yaml
# site.yml
- hosts: gpu
  roles:
    - nvidia
    - dcgm
    - qemu
    - ollama

- hosts: qdrant
  roles:
    - qdrant
```

### 전체 실행 순서

```bash
# 1. Terraform으로 인스턴스 생성
cd terraform && terraform init && terraform apply

# 2. Ansible로 환경 구성
cd ../ansible && ansible-playbook -i inventory/hosts.ini site.yml

# 3. RAG 파이프라인 실행
pip install ollama qdrant-client
python embed.py   # 문서 임베딩
python rag.py     # 질의응답
```

---

## Python 실행 위치

별도 VM 없이 **로컬 PC에서 실행**하면 됩니다.

```
[내 PC]
  Python 스크립트
  ├── → HTTP → GPU 인스턴스 (Ollama, port 11434)
  └── → HTTP → Qdrant 인스턴스 (port 6333)
```

OpenStack Security Group에서 포트를 열어야 합니다.

| 인스턴스 | 포트 | 용도 |
|----------|------|------|
| GPU 인스턴스 | 11434 | Ollama API |
| Qdrant 인스턴스 | 6333 | Qdrant API |

---

![RAG 파이프라인 전체 구성](/assets/img/posts/rag-pipeline-overview.png)

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
