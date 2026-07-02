---
title: "[Study] Docker 학습 (2) — 실무 활용: Docker Compose, 멀티 스테이지 빌드, 레지스트리"
date: 2026-07-02 10:00:00 +0900
categories: [Study, Docker]
subcategory: Study
tags: [docker, docker-compose, multi-stage-build, registry, security, optimization]
---

## 개요

기본 개념([Docker 학습 (1)]({% post_url 2026-07-02-docker-01-basics %}))을 익힌 뒤, 실무에서 실제로 자주 마주치는 여러 컨테이너 조합 실행(Compose), 이미지 경량화(멀티 스테이지 빌드), 사설 레지스트리 운영, 운영 시 고려사항을 정리했습니다.

```
docker-compose.yml
  ├── app (멀티 스테이지 빌드로 경량화된 이미지)
  ├── db
  └── cache
        └── 사설 레지스트리에 push → 배포 환경에서 pull
```

---

## Docker Compose로 여러 컨테이너 관리

컨테이너를 하나씩 `docker run`으로 띄우면 네트워크·볼륨·환경변수를 매번 수동으로 맞춰야 합니다. Compose는 이 구성을 YAML 하나로 선언적으로 관리합니다.

```yaml
services:
  web:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DB_HOST=db
    depends_on:
      - db
  db:
    image: postgres:16
    volumes:
      - db-data:/var/lib/postgresql/data
    environment:
      - POSTGRES_PASSWORD=example

volumes:
  db-data:
```

```bash
docker compose up -d
docker compose ps
docker compose logs -f web
docker compose down
```

`depends_on`은 시작 순서만 보장하고 애플리케이션이 실제로 준비됐는지는 보장하지 않기 때문에, 실무에서는 `healthcheck`를 함께 정의해 의존 서비스가 준비된 뒤 시작하도록 합니다.

```yaml
db:
  image: postgres:16
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U postgres"]
    interval: 5s
    retries: 5
```

---

## 멀티 스테이지 빌드로 이미지 경량화

빌드 도구(컴파일러, devDependencies 등)가 최종 이미지에 그대로 남으면 이미지 용량이 커지고 공격 표면도 늘어납니다. 멀티 스테이지 빌드로 빌드 단계와 실행 단계를 분리합니다.

```dockerfile
# 1단계: 빌드
FROM node:20 AS build
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# 2단계: 실행 (빌드 산출물만 복사)
FROM node:20-alpine
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules ./node_modules
CMD ["node", "dist/server.js"]
```

```bash
docker build -t my-app:2.0 .
docker images  # 단일 스테이지 대비 용량 비교
```

---

## 사설 레지스트리 push / pull

사내 배포 환경에서는 Docker Hub 대신 사설 레지스트리를 씁니다.

```bash
docker tag my-app:2.0 registry.internal:5000/my-app:2.0
docker push registry.internal:5000/my-app:2.0

# 배포 서버에서
docker pull registry.internal:5000/my-app:2.0
docker run -d registry.internal:5000/my-app:2.0
```

CI 파이프라인에서는 커밋 해시나 태그를 이미지 태그로 사용해 어떤 소스로 빌드된 이미지인지 추적할 수 있게 합니다.

```bash
docker build -t registry.internal:5000/my-app:$(git rev-parse --short HEAD) .
```

---

## 운영 시 고려사항

| 항목 | 내용 |
|------|------|
| 비root 유저 실행 | Dockerfile에 `USER` 지정, 컨테이너 탈출 시 피해 범위 축소 |
| 리소스 제한 | `--memory`, `--cpus`로 노이지 네이버 문제 방지 |
| 로그 관리 | 로그 드라이버 설정(`json-file` max-size) 없으면 디스크 고갈 위험 |
| 시크릿 관리 | 환경변수 대신 Docker secret / Vault 등으로 민감정보 분리 |
| 이미지 스캔 | `docker scout` 등으로 베이스 이미지 취약점 점검 |

```dockerfile
FROM node:20-alpine
RUN addgroup -S app && adduser -S app -G app
USER app
```

```bash
docker run -d --memory=512m --cpus=1 my-app:2.0
```

---

## 정리 — 자주 나오는 질문

**Q. Docker Compose와 Kubernetes는 어떻게 다른가요?**

Compose는 단일 호스트에서 여러 컨테이너를 선언적으로 관리하는 도구이고, K8s는 여러 노드에 걸친 스케줄링·자동 복구·오토스케일링까지 다루는 오케스트레이션 플랫폼입니다. 로컬 개발이나 단일 서버 배포에는 Compose로 충분하지만, 다중 노드 운영이 필요해지면 K8s로 넘어가게 됩니다.

**Q. 이미지 용량을 줄이는 방법은?**

멀티 스테이지 빌드로 빌드 전용 도구를 최종 이미지에서 제외하고, alpine 같은 경량 베이스 이미지를 사용하며, `.dockerignore`로 불필요한 파일이 빌드 컨텍스트에 포함되지 않게 합니다.

**Q. 컨테이너 보안에서 가장 먼저 신경써야 할 것은?**

root가 아닌 유저로 실행하는 것입니다. 컨테이너가 탈취당하더라도 root 권한이 없으면 호스트에 미치는 영향을 크게 줄일 수 있습니다.

---

## 기술 스택

| 역할 | 기술 |
|------|------|
| 다중 컨테이너 관리 | Docker Compose |
| 이미지 빌드 | 멀티 스테이지 Dockerfile |
| 이미지 저장소 | 사설 Docker Registry |
| 리소스 제어 | Docker 리소스 제한 옵션 |

---

## 시리즈 구성

- **(1)**: 기본 개념 — 컨테이너, 이미지, Dockerfile
- **(2)**: 실무 활용 — Docker Compose, 멀티 스테이지 빌드, 레지스트리 (현재)
- **(3)**: 실습 프로젝트 — Docker Compose로 GPU 모니터링 스택 구성
