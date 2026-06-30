---
title: "네트워크 접근 제어 (Network Access Control) 정리"
date: 2026-06-30 04:00:00 +0900
categories: [Study, Network]
tags: [network, iptables, nftables, firewalld, security-group, tcp, linux]
---

## 네트워크 기초

### 프로토콜

- IP / 라우팅
- TCP / UDP
- TCP 3-way handshake

### 주소 체계

- 서브넷 마스크 / CIDR 계산
- NAT 동작 방식

### 네트워크 디버깅 도구

| 명령어 | 용도 |
|--------|------|
| `curl` | HTTP 레벨 디버깅 |
| `tcpdump` | 패킷 캡처 |
| `ss` | 소켓/포트 확인 |
| `nslookup` / `dig` | DNS 조회 |
| `netstat` | 연결 상태 확인 |

---

## Network Access Control

### TCP Wrappers

```bash
# 허용
/etc/hosts.allow

# 차단
/etc/hosts.deny
```

### Firewall

| 도구 | 설명 |
|------|------|
| `iptables` | 전통적인 리눅스 방화벽 |
| `nftables` | iptables 후속, 더 유연한 문법 |
| `firewalld` | zone 기반 동적 방화벽 관리 |

---

## 클라우드 네트워크 접근 제어

### OpenStack

- **Security Group**: 인스턴스 단위 인바운드/아웃바운드 트래픽 제어

### AWS

| 도구 | 적용 범위 |
|------|-----------|
| Network ACL | 서브넷 레벨 |
| Security Group | 인스턴스 레벨 |
