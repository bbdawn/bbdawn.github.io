---
title: "OpenStack GPU 인스턴스 기반 RAG 파이프라인 구성"
date: 2026-06-30 11:00:00 +0900
categories: [Study, GPU]
subcategory: Hands-on
tags: [openstack, gpu, rag, ollama, qdrant, terraform, ansible, python, llm]
---

## 시나리오: OpenStack 문서 기반 질의응답 시스템

### 전체 그림

```
[문서 준비] → [임베딩 생성] → [벡터 DB 저장]
                                      ↓
[사용자 질문] → [질문 임베딩] → [유사 문서 검색] → [LLM 답변]
```

---

## 환경 구성

OpenStack 인스턴스 2개가 필요합니다.

```
[GPU 인스턴스]          [일반 인스턴스]
  Ollama                  Qdrant
  - 임베딩 생성            - 벡터 저장
  - 답변 생성              - 유사도 검색
```

---

## Step 1. GPU 인스턴스 생성

기존 `gpu-instance` 기능으로 MIG 인스턴스를 생성합니다.

- MIG 슬라이스 1개 할당
- RAM 16GB 이상

---

## Step 2. Ollama 설치 (GPU 인스턴스)

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

---

## Step 3. Qdrant 설치 (일반 인스턴스)

```bash
docker run -d \
  --name qdrant \
  -p 6333:6333 \
  -v qdrant_storage:/qdrant/storage \
  qdrant/qdrant

# 실행 확인
curl http://localhost:6333
```

---

## Step 4. 문서 준비

OpenStack 공식 문서나 사내 운영 문서를 준비합니다.

```
docs/
├── neutron.txt     # Neutron 개념 정리
├── nova.txt        # Nova 개념 정리
├── cinder.txt      # Cinder 개념 정리
└── gpu-setup.txt   # GPU 설정 가이드
```

---

## Step 5. 문서 임베딩 및 저장

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

---

## Step 6. 질의응답 (RAG)

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

---

## Step 7. 모니터링

```bash
# GPU 사용률 확인 (DCGM)
dcgmi dmon -s u

# Ollama 응답 시간 확인
curl -w "\n응답시간: %{time_total}s\n" \
  http://<GPU_INSTANCE_IP>:11434/api/embeddings \
  -d '{"model": "nomic-embed-text", "prompt": "test"}'
```

---

## 실행 결과 예시

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

## Terraform + Ansible 연동

### 전체 흐름

```
Terraform (인프라 프로비저닝)
  → GPU 인스턴스 생성 (OpenStack)
  → Qdrant 인스턴스 생성
  ↓
Ansible (환경 구성)
  → GPU 인스턴스: Ollama + 모델 설치
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

## 기술 스택

| 역할 | 기술 |
|------|------|
| IaC | Terraform, Ansible |
| 인프라 | OpenStack (Nova, Neutron) |
| LLM | Ollama (nomic-embed-text, phi3) |
| 벡터 DB | Qdrant (Docker) |
| 언어 | Python |
| 모니터링 | DCGM → Prometheus → Grafana |

---

## repo 구조

```
rag-pipeline/
├── terraform/
│   ├── main.tf          # GPU/Qdrant 인스턴스 생성
│   ├── variables.tf
│   └── outputs.tf       # IP 출력 → Ansible inventory로
├── ansible/
│   ├── inventory/
│   │   └── hosts.ini
│   ├── roles/
│   │   ├── ollama/      # Ollama + 모델 설치
│   │   └── qdrant/      # Docker + Qdrant 설치
│   └── site.yml
├── docs/                # 임베딩할 문서
├── embed.py             # 문서 → 벡터 저장
├── rag.py               # 질의응답
├── requirements.txt
└── README.md
```
