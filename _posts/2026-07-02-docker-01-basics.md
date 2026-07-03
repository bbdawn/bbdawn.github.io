---
title: "[Study] Docker 학습 (1) — 기본 개념: 컨테이너, 이미지, Dockerfile"
date: 2026-07-02 09:00:00 +0900
categories: [Study, Docker]
subcategory: Study
tags: [docker, container, image, dockerfile, volume, network]
---

## 개요

OpenStack에서 VM 단위로 인프라를 다뤄왔지만, 애플리케이션 배포 단위인 컨테이너를 기본기부터 정리할 필요가 있어 Docker 학습을 시작했습니다. 이미지·컨테이너 개념부터 Dockerfile 작성, 볼륨·네트워크까지 기본 오브젝트를 하나씩 실습했습니다.

```
[로컬 환경]
  Docker Engine
    └── Image → Container 실행
                  └── Volume (데이터 영속화)
                  └── Network (컨테이너 간 통신)
```

실무에서 자주 쓰는 Compose, 멀티 스테이지 빌드, 레지스트리 운영은 다음 글([Docker 학습 (2)]({% post_url 2026-07-02-docker-02-practical %}))에서 다룹니다.

***

## 핵심 개념 정리

### 컨테이너 vs 가상머신

| 구분 | 가상머신 (VM) | 컨테이너 |
| --- | --------- | ---- |
| 격리 단위 | Hypervisor 위 Guest OS 전체 | 호스트 커널 공유, 프로세스 격리 |
| 부팅 속도 | 수십 초 \~ 수 분 | 수백 ms \~ 수 초 |
| 자원 오버헤드 | 큼 (OS 전체 포함) | 작음 (바이너리 + 라이브러리만) |
| 이식성 | 이미지 용량 큼 | 레이어 기반, 가볍고 빠르게 배포 |

OpenStack에서 다뤄온 Nova 인스턴스(VM)와 비교하면, 컨테이너는 커널을 공유하는 대신 훨씬 가볍게 애플리케이션 단위로 격리한다는 차이가 있습니다.

### 핵심 오브젝트

| 개념 | 설명 |
| --- | --- |
| Image | 컨테이너 실행에 필요한 파일 시스템 + 실행 설정을 담은 읽기 전용 템플릿 |
| Container | 이미지를 실행한 인스턴스 (프로세스 + 격리된 파일 시스템) |
| Dockerfile | 이미지를 빌드하는 방법을 정의한 텍스트 파일 |
| Layer | 이미지를 구성하는 각 명령어 단위의 읽기 전용 계층 (캐시 재사용의 단위) |
| Registry | 이미지를 저장·배포하는 저장소 (Docker Hub, 사설 레지스트리 등) |
| Volume | 컨테이너 생명주기와 분리된 영속 데이터 저장 공간 |
| Network | 컨테이너 간, 컨테이너-호스트 간 통신을 위한 가상 네트워크 |

***

## 환경 구성

```bash
docker --version
docker run hello-world
```

`hello-world` 컨테이너가 정상적으로 메시지를 출력하면 Docker Engine이 이미지를 pull → 컨테이너 생성 → 실행 → 종료까지 문제없이 수행한 것입니다.

![image.png](/assets/img/posts/1783052784348-image.png)

***

## 실습 내용

### 1\. 이미지 받고 컨테이너 실행하기

```bash
docker pull nginx
docker run -d --name web -p 8080:80 nginx
docker ps
curl http://localhost:8080
```

### 2\. Dockerfile 작성 및 빌드

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
CMD ["node", "server.js"]
```

```bash
docker build -t my-app:1.0 .
docker run -d -p 3000:3000 my-app:1.0
```

레이어 캐시를 활용하기 위해 자주 바뀌지 않는 `package.json` 설치 단계를 소스 코드 복사보다 먼저 배치하는 것이 핵심입니다.

### 3\. 볼륨으로 데이터 영속화

컨테이너를 삭제해도 데이터가 남아야 하는 경우(DB 등) 볼륨을 사용합니다.

```bash
docker volume create app-data
docker run -d -v app-data:/var/lib/data --name db postgres
docker volume inspect app-data
```

### 4\. 네트워크로 컨테이너 간 통신

```bash
docker network create app-net
docker run -d --network app-net --name db postgres
docker run -d --network app-net --name web my-app:1.0
```

같은 네트워크에 속한 컨테이너는 컨테이너 이름을 호스트명처럼 사용해 서로 통신할 수 있습니다 (`db:5432`).

### 5\. 컨테이너 로그/상태 확인

```bash
docker logs -f web
docker inspect web
docker exec -it web sh
```

***

## 기본 명령어 레퍼런스

```bash
docker ps -a
docker images
docker stop <container>
docker rm <container>
docker rmi <image>
docker system prune
```

***

## 기술 스택

| 역할 | 기술 |
| --- | --- |
| 컨테이너 런타임 | Docker Engine |
| 이미지 빌드 | Dockerfile |
| 데이터 영속화 | Docker Volume |
| 네트워킹 | Docker Network (bridge) |

***

## 시리즈 구성

* **(1)**: 기본 개념 — 컨테이너, 이미지, Dockerfile (현재)
* **(2)**: 실무 활용 — Docker Compose, 멀티 스테이지 빌드, 레지스트리
* **(3)**: 실습 프로젝트 — Docker Compose로 GPU 모니터링 스택 구성