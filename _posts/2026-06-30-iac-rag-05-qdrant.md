---
title: "[Project] GPU IaC 자동화 + RAG 파이프라인 (5) — Qdrant + RAG 파이프라인 구성"
date: 2026-06-30 11:40:00 +0900
categories: [Project, GPU]
subcategory: Project
tags: [qdrant, rag, ollama, llm, python, vector-db, embedding]
---

## 개요

Qdrant를 일반 인스턴스에 Docker로 실행하고, Python으로 문서를 임베딩해 저장한 뒤, RAG 방식으로 질의응답하는 파이프라인 전체를 구성합니다.

---

## 벡터 데이터베이스란?

벡터 DB는 데이터를 **숫자 배열(벡터)** 형태로 저장하고, 벡터 간의 **유사도**를 기반으로 검색하는 데이터베이스입니다. 텍스트, 이미지, 오디오 같은 비정형 데이터를 AI 모델이 **임베딩(embedding)** 으로 변환한 뒤 저장합니다.

```
"고양이" → [0.12, 0.87, 0.34, ...]  # 수백~수천 차원의 벡터
"강아지" → [0.11, 0.85, 0.31, ...]  # 유사한 의미 → 유사한 벡터
"자동차" → [0.91, 0.02, 0.77, ...]  # 다른 의미 → 먼 벡터
```

### 핵심 원리

- **유사도 검색 (ANN, Approximate Nearest Neighbor)**: "이것과 가장 비슷한 것을 찾아줘"
- 거리 측정 방식: 코사인 유사도, 유클리드 거리, 내적(dot product) 등

### 일반 DB와의 차이

| | 관계형 DB | 벡터 DB |
|--|-----------|---------|
| 검색 방식 | 정확한 값 일치 | 유사도 기반 |
| 질의 예시 | `WHERE name = '고양이'` | "고양이와 비슷한 것" |
| 데이터 형태 | 테이블, 행/열 | 고차원 벡터 |

### 주요 활용 사례

| 활용 | 설명 |
|------|------|
| RAG (Retrieval-Augmented Generation) | LLM에 외부 지식을 연결할 때 |
| 시맨틱 검색 | 키워드가 아닌 의미 기반 검색 |
| 추천 시스템 | 유사 상품/콘텐츠 추천 |
| 중복 감지 | 유사 문서/이미지 탐지 |
| 얼굴 인식 | 얼굴 임베딩 비교 |

### 주요 벡터 DB 종류

| 이름 | 특징 |
|------|------|
| **Pinecone** | 완전 관리형, SaaS |
| **Weaviate** | 오픈소스, 멀티모달 지원 |
| **Chroma** | 로컬 개발에 간편, Python 친화적 |
| **Qdrant** | Rust 기반, 고성능 |
| **pgvector** | PostgreSQL 확장, 기존 DB에 추가 |
| **Milvus** | 대규모 엔터프라이즈용 |

이 프로젝트에서는 Rust 기반의 고성능 오픈소스인 **Qdrant**를 벡터 DB로 선택했습니다.

---

## Ansible Role: qdrant

```yaml
# roles/qdrant/tasks/main.yml

- name: Docker 설치
  apt:
    name: docker.io
    state: present
    update_cache: yes

- name: Docker 서비스 시작
  systemd:
    name: docker
    state: started
    enabled: yes

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

```bash
# 실행 확인
curl http://<QDRANT_IP>:6333
# {"title":"qdrant - vector search engine","version":"..."}
```

---

## RAG 파이프라인 전체 흐름

```
[문서 준비] → [임베딩 생성] → [Qdrant 저장]
                                    ↓
[사용자 질문] → [질문 임베딩] → [유사 문서 검색] → [LLM 답변]
```

---

## 문서 준비

OpenStack 운영 문서나 사내 가이드를 텍스트로 준비합니다.

```
docs/
├── neutron.txt     # Neutron 개념 정리
├── nova.txt        # Nova 개념 정리
├── cinder.txt      # Cinder 개념 정리
└── gpu-setup.txt   # GPU 설정 가이드
```

---

## embed.py — 문서 임베딩 및 저장

```python
from ollama import Client
from qdrant_client import QdrantClient
from qdrant_client.models import VectorParams, Distance
import os

ollama = Client(host="http://<GPU_INSTANCE_IP>:11434")
qdrant = QdrantClient(host="<QDRANT_IP>", port=6333)

# 컬렉션 생성 (nomic-embed-text 벡터 차원: 768)
qdrant.create_collection(
    collection_name="openstack-docs",
    vectors_config=VectorParams(size=768, distance=Distance.COSINE)
)

def split_text(text, chunk_size=200):
    sentences = text.split(".")
    chunks, current = [], ""
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
    for chunk in split_text(text):
        response = ollama.embeddings(model="nomic-embed-text", prompt=chunk)
        qdrant.upsert(
            collection_name="openstack-docs",
            points=[{
                "id": doc_id,
                "vector": response["embedding"],
                "payload": {"text": chunk, "source": filename}
            }]
        )
        doc_id += 1
        print(f"저장: {filename} — {chunk[:50]}...")

print(f"완료: {doc_id}개 청크 저장")
```

---

## rag.py — 질의응답

```python
from ollama import Client
from qdrant_client import QdrantClient

ollama = Client(host="http://<GPU_INSTANCE_IP>:11434")
qdrant = QdrantClient(host="<QDRANT_IP>", port=6333)

def rag(query):
    # 1. 질문 임베딩
    response = ollama.embeddings(model="nomic-embed-text", prompt=query)

    # 2. 유사 문서 검색
    results = qdrant.search(
        collection_name="openstack-docs",
        query_vector=response["embedding"],
        limit=3
    )
    context = "\n".join([r.payload["text"] for r in results])
    print(f"\n[검색된 문서]\n{context}\n")

    # 3. LLM 답변 생성
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

## 실행

```bash
pip install ollama qdrant-client

# 문서 임베딩
python embed.py

# 질의응답
python rag.py
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

## 전체 실행 순서

```bash
# 1. Terraform으로 인스턴스 생성 (2편)
cd terraform && terraform init && terraform apply

# 2. Ansible로 환경 구성 (3, 4, 5편)
cd ../ansible && ansible-playbook -i inventory/hosts.ini site.yml

# 3. RAG 파이프라인 실행 (로컬 PC)
pip install ollama qdrant-client
python embed.py   # 문서 임베딩
python rag.py     # 질의응답
```
