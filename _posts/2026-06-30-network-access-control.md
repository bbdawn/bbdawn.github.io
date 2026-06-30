---
title: "Network Access Control"
date: 2026-06-30 04:01:00 +0900
categories: [Study, Network]
tags: [network, iptables, nftables, firewalld, security-group, tcp-wrappers, linux]
---

## TCP Wrappers

```bash
# 허용
/etc/hosts.allow

# 차단
/etc/hosts.deny
```

---

## Firewall

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
