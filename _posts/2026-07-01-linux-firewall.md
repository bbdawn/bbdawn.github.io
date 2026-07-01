---
title: "[Feature] Linux 네트워크 접근 제어 2편 — iptables / nftables / firewalld"
date: 2026-07-01 11:30:00 +0900
categories: [Features, Linux]
subcategory: Features
tags: [linux, iptables, nftables, firewalld, firewall, netfilter, access-control, security]
---

## 개요

리눅스 방화벽은 커널의 **Netfilter** 프레임워크 위에서 동작합니다. 사용자 공간에서 Netfilter를 제어하는 도구가 `iptables`, `nftables`, `firewalld`이며, 세 도구 모두 결국 같은 커널 기능을 사용합니다.

```
사용자 공간                       커널 공간
┌─────────────┐                  ┌──────────────────┐
│  iptables   │                  │                  │
│  nftables   │ ──── 시스템콜 ──→│   Netfilter      │
│  firewalld  │                  │  (패킷 필터링)   │
└─────────────┘                  └──────────────────┘
```

---

## iptables

### 구조

iptables는 **테이블(table)** → **체인(chain)** → **규칙(rule)** 계층으로 구성됩니다.

| 테이블 | 역할 |
|--------|------|
| `filter` | 기본 테이블. 패킷 허용/차단 |
| `nat` | IP/포트 변환 (SNAT, DNAT, MASQUERADE) |
| `mangle` | 패킷 헤더 수정 |
| `raw` | 연결 추적(conntrack) 제외 처리 |

| 기본 체인 | 동작 시점 |
|-----------|-----------|
| `INPUT` | 로컬 호스트로 들어오는 패킷 |
| `OUTPUT` | 로컬 호스트에서 나가는 패킷 |
| `FORWARD` | 다른 호스트로 전달되는 패킷 (라우터 역할) |
| `PREROUTING` | 라우팅 결정 전 (nat/mangle) |
| `POSTROUTING` | 라우팅 결정 후 (nat/mangle) |

### 기본 사용법

```bash
# 규칙 목록 확인
iptables -L -n -v
iptables -L -n -v --line-numbers   # 줄 번호 포함

# 특정 테이블 확인
iptables -t nat -L -n -v
```

```bash
# INPUT 체인 기본 정책 설정
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# 특정 IP/포트 허용
iptables -A INPUT -s 192.168.1.0/24 -p tcp --dport 22 -j ACCEPT
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# 이미 연결된 세션은 허용 (상태 추적)
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# 루프백 허용
iptables -A INPUT -i lo -j ACCEPT

# 특정 IP 차단
iptables -A INPUT -s 10.0.0.5 -j DROP

# 규칙 삽입 (맨 앞에)
iptables -I INPUT 1 -s 192.168.1.100 -j ACCEPT

# 규칙 삭제
iptables -D INPUT -s 10.0.0.5 -j DROP
# 또는 줄 번호로 삭제
iptables -D INPUT 3
```

### NAT 설정

```bash
# MASQUERADE: 나가는 패킷의 소스 IP를 출구 인터페이스 IP로 변경 (SNAT 동적)
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# DNAT: 들어오는 패킷을 내부 서버로 포워드
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 192.168.1.10:80

# IP 포워딩 커널 설정 (라우터로 동작 시 필요)
echo 1 > /proc/sys/net/ipv4/ip_forward
# 영구 적용
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
```

### 규칙 저장 및 복원

```bash
# 저장
iptables-save > /etc/iptables/rules.v4

# 복원
iptables-restore < /etc/iptables/rules.v4

# 재부팅 후 자동 적용 (Debian/Ubuntu)
apt install iptables-persistent
```

---

## nftables

iptables의 후속 도구로 RHEL 8, Debian 10 이후 기본 방화벽입니다. **하나의 명령어(`nft`)** 로 모든 프로토콜(IPv4/IPv6/ARP)을 통합 관리하며, 문법이 더 일관됩니다.

### 구조

iptables와 달리 테이블/체인을 직접 생성합니다.

```bash
# 현재 ruleset 전체 확인
nft list ruleset
```

### 기본 설정 예시

```bash
# 테이블 생성
nft add table inet filter

# 체인 생성 (기본 정책 DROP)
nft add chain inet filter input  '{ type filter hook input priority 0 ; policy drop ; }'
nft add chain inet filter output '{ type filter hook output priority 0 ; policy accept ; }'
nft add chain inet filter forward '{ type filter hook forward priority 0 ; policy drop ; }'

# 룰 추가
nft add rule inet filter input iif lo accept
nft add rule inet filter input ct state established,related accept
nft add rule inet filter input ip saddr 192.168.1.0/24 tcp dport 22 accept
nft add rule inet filter input tcp dport { 80, 443 } accept

# 특정 IP 차단
nft add rule inet filter input ip saddr 10.0.0.5 drop
```

### 설정 파일로 관리

```bash
# /etc/nftables.conf 예시
#!/usr/sbin/nft -f

flush ruleset

table inet filter {
    chain input {
        type filter hook input priority 0 ; policy drop ;

        iif lo accept
        ct state established,related accept
        ip saddr 192.168.1.0/24 tcp dport 22 accept
        tcp dport { 80, 443 } accept
    }
    chain forward {
        type filter hook forward priority 0 ; policy drop ;
    }
    chain output {
        type filter hook output priority 0 ; policy accept ;
    }
}
```

```bash
# 설정 적용
nft -f /etc/nftables.conf

# systemd 서비스로 자동 적용
systemctl enable --now nftables
```

---

## firewalld

**zone 기반의 동적 방화벽** 관리 도구입니다. 백엔드로 nftables(또는 iptables)를 사용하며, 서비스 재시작 없이 규칙을 실시간으로 변경할 수 있습니다.

### zone 개념

네트워크 인터페이스나 IP 대역을 "신뢰 수준" 별로 zone에 배정합니다.

| zone | 기본 정책 |
|------|-----------|
| `drop` | 들어오는 모든 패킷 차단, 응답 없음 |
| `block` | 들어오는 패킷 거부 (ICMP 거부 메시지 반환) |
| `public` | 공개 네트워크, 선택된 서비스만 허용 |
| `external` | 외부 네트워크, MASQUERADE 포함 |
| `dmz` | DMZ 구간, 제한된 내부 접근 |
| `work` | 업무 네트워크, 더 넓은 신뢰 |
| `home` | 홈 네트워크 |
| `internal` | 내부 네트워크 |
| `trusted` | 모든 연결 허용 |

### 기본 사용법

```bash
# 상태 확인
firewall-cmd --state
firewall-cmd --list-all
firewall-cmd --list-all-zones

# 현재 기본 zone 확인
firewall-cmd --get-default-zone

# 서비스 허용 (--permanent 없으면 재시작 시 초기화)
firewall-cmd --permanent --add-service=ssh
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https

# 포트 직접 허용
firewall-cmd --permanent --add-port=8080/tcp
firewall-cmd --permanent --add-port=5000-5100/tcp

# 설정 적용 (reload)
firewall-cmd --reload
```

### zone에 인터페이스/소스 배정

```bash
# 인터페이스를 특정 zone에 배정
firewall-cmd --permanent --zone=internal --add-interface=eth1

# IP 대역을 특정 zone에 배정
firewall-cmd --permanent --zone=trusted --add-source=192.168.1.0/24

# zone에 서비스 허용
firewall-cmd --permanent --zone=internal --add-service=mysql
firewall-cmd --reload
```

### Rich Rule (세밀한 제어)

```bash
# 특정 IP에서 SSH만 허용
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.1.100" service name="ssh" accept'

# 특정 IP 차단
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.0.0.5" drop'

# 접근 시 로그 기록
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.0.0.0/8" service name="ssh" log prefix="SSH-ATTEMPT" level="info" accept'

firewall-cmd --reload
```

### MASQUERADE (NAT)

```bash
# external zone에서 MASQUERADE 활성화 (기본 활성화되어 있음)
firewall-cmd --permanent --zone=external --add-masquerade

# 포트 포워딩
firewall-cmd --permanent --zone=external \
  --add-forward-port=port=8080:proto=tcp:toport=80:toaddr=192.168.1.10

firewall-cmd --reload
```

---

## 세 도구 비교

| 항목 | iptables | nftables | firewalld |
|------|----------|----------|-----------|
| 등장 시기 | 1998 | 2014 | 2011 |
| 문법 | 명령어 옵션 방식 | 선언적 DSL | 서비스/zone 추상화 |
| IPv4/IPv6 통합 | 별도 (`ip6tables`) | 단일 명령어 | 단일 명령어 |
| 동적 변경 | 재적용 필요 | 재적용 필요 | 실시간 반영 |
| 백엔드 | Netfilter 직접 | Netfilter 직접 | nftables (또는 iptables) |
| 적합한 환경 | 레거시, 정밀 제어 | 신규 서버, 복잡한 규칙 | RHEL/CentOS, 운영 자동화 |
| 주요 배포판 기본 | Ubuntu < 20.04 | Debian 10+, Ubuntu 20.04+ | RHEL 7+, CentOS 7+ |

### 현재 백엔드 확인

```bash
# firewalld가 어떤 백엔드를 쓰는지
firewall-cmd --info-zone=public 2>/dev/null | head -5
cat /etc/firewalld/firewalld.conf | grep FirewallBackend
```

---

## 계층적 접근 제어 구성 예시

```
외부 트래픽
    │
    ▼
[firewalld / iptables]  ← 포트/IP 수준 필터링
    │
    ▼
[TCP Wrappers]          ← 서비스별 호스트 기반 필터링 (libwrap 지원 서비스)
    │
    ▼
[sshd / vsftpd 등]      ← 서비스 자체 인증 및 접근 제어
```

방화벽에서 1차로 포트를 차단하고, TCP Wrappers에서 2차로 호스트를 필터링하면 더 안전한 접근 제어가 가능합니다.
