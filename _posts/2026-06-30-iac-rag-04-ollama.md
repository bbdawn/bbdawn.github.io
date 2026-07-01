---
title: "[Project] GPU IaC 자동화 + RAG 파이프라인 (4) — Ollama 설치 및 임베딩"
date: 2026-06-30 11:30:00 +0900
categories: [Project, GPU]
subcategory: Project
tags: [gpu, ollama, llm, rag, nvidia, ansible, embedding]
---

## 개요

GPU 인스턴스에 Ollama를 설치하고 임베딩 모델과 LLM 모델을 준비합니다. Ansible Role로 자동화하고, 설치 후 GPU가 실제로 사용되는지 검증합니다.

---

## Ansible Role: ollama

```yaml
# roles/ollama/tasks/main.yml

- name: Ollama 설치 스크립트 실행
  shell: curl -fsSL https://ollama.com/install.sh | sh
  args:
    creates: /usr/local/bin/ollama  # 이미 설치돼 있으면 건너뜀

- name: Ollama 서비스 시작 및 활성화
  systemd:
    name: ollama
    state: started
    enabled: yes

- name: 임베딩 모델 다운로드 (nomic-embed-text)
  shell: ollama pull nomic-embed-text
  environment:
    OLLAMA_HOST: "0.0.0.0:11434"  # 외부 접근 허용

- name: LLM 모델 다운로드 (phi3)
  shell: ollama pull phi3
```

---

## GPU 사용 확인

Ollama 설치 후 GPU가 인식되는지 확인합니다.

```bash
# Ollama가 GPU를 사용하는지 확인
ollama run phi3 "안녕" &
nvidia-smi  # GPU 메모리 사용량 증가 확인
```

```
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 535.xx      Driver Version: 535.xx      CUDA Version: 12.x      |
|-------------------------------+----------------------+----------------------|
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
|   0  NVIDIA A100...       Off | 00000000:xx:00.0 Off |                    0 |
+-------------------------------+----------------------+----------------------|
| Processes:                                                                  |
|    PID   Type   Process name             GPU Memory Usage                   |
| ollama    C     /usr/local/bin/ollama    ~4000MiB                           |
+-----------------------------------------------------------------------------+
```

---

## 모델 준비 확인

```bash
# 설치된 모델 목록
ollama list

# 출력 예시:
# NAME                    ID              SIZE    MODIFIED
# phi3:latest             ...             2.2 GB  ...
# nomic-embed-text:latest ...             274 MB  ...
```

---

## API 직접 테스트

로컬 PC에서 Ollama API가 정상 동작하는지 확인합니다.

```bash
# 임베딩 생성 테스트
curl -s http://<GPU_INSTANCE_IP>:11434/api/embeddings \
  -d '{"model": "nomic-embed-text", "prompt": "OpenStack Neutron이란?"}' \
  | python3 -c "import sys,json; e=json.load(sys.stdin)['embedding']; print(f'벡터 차원: {len(e)}, 첫 3개 값: {e[:3]}')"

# 출력 예시:
# 벡터 차원: 768, 첫 3개 값: [0.023, -0.011, 0.045]
```

```bash
# LLM 응답 시간 측정
curl -w "\n응답시간: %{time_total}s\n" \
  http://<GPU_INSTANCE_IP>:11434/api/chat \
  -d '{"model": "phi3", "messages": [{"role": "user", "content": "안녕"}], "stream": false}'
```

---

## GPU 모니터링

DCGM으로 Ollama 실행 중 GPU 사용률을 실시간으로 확인합니다.

```bash
# GPU 사용률 실시간 모니터링
dcgmi dmon -s u

# 출력 예시:
# Entity  GUTIL  MCUTIL
# GPU 0    87%     62%
```

---

## 사용 모델 정리

| 모델 | 용도 | 크기 |
|------|------|------|
| nomic-embed-text | 문서 → 벡터 임베딩 | 274 MB |
| phi3 | 질문 답변 생성 | 2.2 GB |
