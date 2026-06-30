---
title: "OpenStack 주요 서비스 정리"
date: 2026-06-30 02:00:00 +0900
categories: [Study, Openstack]
subcategory: Features
tags: [openstack, neutron, octavia, nova, barbican, keystone]
---

## Neutron (네트워크)

기본 구성 요소: Network, Subnet, Router, Port, Floating IP, Security Groups

주요 구현:
- Floating IP 연동 가능 여부 표출
- Port별 Security Group 목록 API 개발

---

## Octavia (로드 밸런서)

구성 요소: Load Balancer, Listener, Pool, Pool Member, Health Monitor, L7 Policy

주요 구현:
- SSL Offloading 기능 (Barbican 연동)
- `.p12` 파일 직접 업로드 지원
- base64 자동 변환 처리

문제 해결 경험:
- Amphora 인스턴스 active-standby 설정 이슈
- API로 삭제되지 않는 LB 처리 방법

---

## 기타 서비스 요약

| 서비스 | 역할 |
|--------|------|
| **Nova** | 컴퓨팅 리소스 관리 |
| **Glance** | 이미지 관리 |
| **Cinder** | 블록 스토리지 |
| **Manila** | 공유 파일시스템 |
| **Placement** | 리소스 제공자 관리 |
| **Barbican** | 키 관리 |
| **Keystone** | 인증 및 권한 관리 |

---

## 학습 범위

각 서비스별로 아래 항목을 다루었습니다.

- 개념 및 아키텍처
- 동작 방식
- 데이터베이스 테이블 구조
- 주요 로그
- 문제 해결 사례
- 직접 실습
