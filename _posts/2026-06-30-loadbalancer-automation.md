---
title: "[Project] Octavia API로 삭제 안 되는 LB, TUI로 운영 자동화하기"
date: 2026-06-30 10:00:00 +0900
categories: [Project, Openstack]
subcategory: Project
tags: [openstack, octavia, loadbalancer, automation, go, tui, amphora]
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

반복되는 작업을 줄이기 위해 아래 기능을 하나의 도구로 통합했습니다.

1. **LB 계층 구조 트리 조회**
2. **비정상 LB 삭제 SQL 자동 생성**
3. **Amphora 상태 조회 및 Failover 명령 제공**
4. **Octavia 서비스 상태 확인 및 재시작**

---

## 구현: LoadBalancer Manager (Go TUI)

```
LoadBalancer Manager
├── LB 목록 및 상태 조회
│   └── Load Balancer → Listener → Pool → Member 트리
├── 비정상 LB 처리
│   ├── 삭제 SQL 자동 생성
│   └── Amphora Failover 명령
└── 서비스 관리
    └── Octavia 서비스 상태 확인 / 재시작
```

### 삭제 SQL 자동 생성 예시

LB ID 입력 시 계층 구조를 분석해 순서에 맞는 SQL을 자동으로 생성합니다.

```sql
-- LoadBalancer Manager 자동 생성 SQL
DELETE FROM health_monitor WHERE pool_id IN ('pool-uuid-1');
DELETE FROM member WHERE pool_id IN ('pool-uuid-1');
DELETE FROM pool WHERE load_balancer_id = 'lb-uuid';
DELETE FROM l7policy WHERE listener_id IN ('listener-uuid-1');
DELETE FROM listener WHERE load_balancer_id = 'lb-uuid';
DELETE FROM load_balancer WHERE id = 'lb-uuid';
```

---

## 효과

| 항목 | 이전 | 이후 |
|------|------|------|
| 계층 구조 파악 | 여러 API 호출 수동 조합 | 트리뷰로 즉시 확인 |
| 삭제 SQL 작성 | 수동 작성 (실수 위험) | 자동 생성 |
| Amphora 처리 | 명령어 직접 조합 | TUI에서 선택 |
| 장애 대응 시간 | 30분 이상 | 5분 이내 |

---

## 기술 스택

- **언어**: Go (단일 바이너리, 외부 의존성 없음)
- **인프라**: OpenStack Octavia, MySQL
