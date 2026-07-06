---
title: "[Project] Octavia API로 삭제 안 되는 LB, 운영 자동화하기"
date: 2026-06-30 10:00:00 +0900
categories: [Project, Openstack]
subcategory: Project
tags: [openstack, octavia, loadbalancer, automation, amphora]
---

## 문제 상황

OpenStack Octavia 운영 중 API로 삭제되지 않는 비정상 로드밸런서가 반복적으로 발생했습니다.

```
DELETE /v2/lbaas/loadbalancers/{lb_id}

→ 409 Conflict
→ "LB is in ERROR state and cannot be deleted"

또는 아무 응답 없이 PENDING_DELETE 상태에서 멈춤
```

원인은 Amphora 인스턴스 장애, 네트워크 이슈 등 다양했으며, 상태가 꼬이면 API로는 해결이 불가능했습니다.

---

## 기존 수동 처리 방식

비정상 LB가 발생할 때마다 아래 과정을 직접 수행했습니다.

```bash
# 1. LB 상태 확인
openstack loadbalancer show <lb_id>

# 2. Amphora 상태 확인
openstack loadbalancer amphora list --loadbalancer <lb_id>

# 3. Octavia DB에서 직접 상태 조회
mysql -u octavia -p octavia
SELECT id, name, provisioning_status, operating_status FROM load_balancer WHERE id='<lb_id>';

# 4. 하위 리소스까지 직접 삭제 SQL 작성
DELETE FROM member WHERE pool_id IN (...);
DELETE FROM pool WHERE load_balancer_id='<lb_id>';
DELETE FROM listener WHERE load_balancer_id='<lb_id>';
DELETE FROM load_balancer WHERE id='<lb_id>';
```

LB → Listener → Pool → Member 계층 구조를 파악하고 순서에 맞게 SQL을 작성해야 해서 시간이 오래 걸리고 실수 위험이 있었습니다.

---

## 자동화 방향

반복되는 작업을 줄이기 위해 아래 기능을 하나의 도구로 정리했습니다.

1. **LB 계층 구조 시각화**
2. **비정상 LB 삭제 SQL 자동 생성**
3. **Amphora 상태 조회 및 Failover 명령 제공**
4. **Octavia 서비스 상태 확인 및 재시작**

---

## Octavia Manager

이 네 가지 기능을 인터랙티브 웹 페이지로 정리했습니다. LB ID를 입력하면 브라우저에서 바로 삭제 SQL을 단계별로 생성해주고, 자주 쓰는 명령어는 클릭 한 번으로 복사할 수 있습니다.

**[Octavia Manager 바로가기 →](/octavia-manager/)**

| 구성 | 내용 |
|------|------|
| 구조 | LoadBalancer → Listener → Pool → Member 계층을 시각적으로 정리 |
| CLI 명령어 / 로그 | 리소스별 조회 명령어, 상태값 사전, 로그 확인 위치 |
| 안 지워지는 LB 삭제 | PENDING_* → ERROR 전환, cascade 삭제, ID 입력 시 단계별 삭제 SQL 자동 생성 |
| 서비스 상태/재시작 | Octavia systemd 서비스 관리 |
| 트러블슈팅 | 진단 흐름 및 자주 발생하는 장애 케이스 |

### 삭제 SQL 자동 생성

기존에는 LB 계층 구조를 파악해서 순서에 맞게 SQL을 손으로 작성해야 했는데, 이제는 LB ID 하나만 입력하면 참조 무결성을 지키는 순서대로 단계별 SQL이 만들어집니다. 각 단계는 개별적으로 복사할 수 있어서, 한 번에 실행하지 않고 단계마다 결과를 확인하며 진행할 수 있습니다.

---

## 효과

| 항목 | 이전 | 이후 |
|------|------|------|
| 계층 구조 파악 | 여러 API 호출 수동 조합 | 시각화된 구조로 즉시 확인 |
| 삭제 SQL 작성 | 수동 작성 (실수 위험) | ID 입력만으로 자동 생성 |
| 명령어 조회 | 문서/메모 검색 | 카테고리별 정리, 클릭 복사 |
| 장애 대응 시간 | 30분 이상 | 5분 이내 |

---

## 기술 스택

| 역할 | 기술 |
|------|------|
| 프론트엔드 | HTML, CSS, Vanilla JavaScript (정적, 백엔드 없음) |
| 대상 인프라 | OpenStack Octavia, MySQL |
