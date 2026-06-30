---
title: "[Project] Octavia 로드밸런서 TUI 관리 도구 개발"
date: 2026-06-30 07:00:00 +0900
categories: [Project, Openstack]
subcategory: Projects
tags: [openstack, octavia, loadbalancer, go, tui, amphora]
---

## 개요

OpenStack Octavia 기반 로드밸런서 운영 및 장애 처리를 위한 TUI(Terminal UI) 관리 도구입니다.

---

## 개발 배경

Octavia 운영 중 API로 삭제되지 않는 비정상 LB가 발생할 경우, 수동으로 상태를 파악하고 DB에서 직접 처리해야 하는 어려움이 있었습니다.

현황 파악, 장애 처리 명령 생성, 로그 조회를 하나의 도구로 통합했습니다.

---

## 주요 기능

### 계층 구조 트리 조회

```
Load Balancer
└── Listener
    └── Pool
        └── Member
```

### 장애 처리

- Amphora 상태 조회 및 Failover 명령
- **비정상 LB 삭제 SQL 자동 생성**
- Octavia 서비스 상태 확인 및 재시작 명령

---

## 기술 스택

| 항목 | 내용 |
|------|------|
| 언어 | Go (단일 바이너리, 외부 의존성 없음) |
| 인프라 | OpenStack Octavia |

---

## 사용 방법

```bash
./loadbalancer-manager
```

OpenStack 연동을 위해 openrc 파일 경로를 설정합니다.

```bash
# 환경변수로 지정
export LB_MANAGER_OPENSTACK_OPENRC=~/openrc

# 또는 기본 경로 사용
~/contrabass-openrc
```
