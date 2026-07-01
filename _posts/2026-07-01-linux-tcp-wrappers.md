---
title: "[Feature] Linux 네트워크 접근 제어 1편 — TCP Wrappers"
date: 2026-07-01 11:00:00 +0900
categories: [Features, Linux]
subcategory: Feature
tags: [linux, tcp-wrappers, hosts.allow, hosts.deny, libwrap, access-control, security]
---

## TCP Wrappers란

TCP Wrappers는 `/etc/hosts.allow`와 `/etc/hosts.deny` 두 파일로 서비스 접근을 제어하는 호스트 기반 접근 제어(host-based access control) 메커니즘입니다.

커널이 아닌 **라이브러리(libwrap)** 수준에서 동작하며, `libwrap`을 링크한 데몬이 클라이언트 연결 요청을 받을 때 두 파일을 참조해 허용 여부를 결정합니다.

```
Client → sshd(libwrap 포함) → hosts.allow / hosts.deny 순서로 검사 → 허용 or 거부
```

---

## 동작 원리

### 검사 순서

1. `/etc/hosts.allow` 를 위에서 아래로 읽어 **첫 번째로 매칭되는 규칙을 적용**
2. `hosts.allow`에 일치하는 규칙이 없으면 `/etc/hosts.deny` 검사
3. 두 파일 어디에도 일치하지 않으면 **기본 허용(allow)**

```
hosts.allow 매칭 → 즉시 허용 (hosts.deny 검사 안 함)
hosts.allow 미매칭 → hosts.deny 검사
  └─ hosts.deny 매칭 → 즉시 거부
  └─ hosts.deny 미매칭 → 허용 (기본값)
```

### 서비스가 TCP Wrappers를 지원하는지 확인

```bash
# libwrap 링크 여부 확인
ldd $(which sshd) | grep libwrap
# libwrap.so.0 => /lib/x86_64-linux-gnu/libwrap.so.0 이 나오면 지원

# 또는
strings $(which sshd) | grep hosts
```

---

## 설정 문법

```
데몬_목록 : 클라이언트_목록 [: 옵션]
```

| 항목 | 설명 |
|------|------|
| 데몬_목록 | 서비스 이름 (예: `sshd`, `vsftpd`, `ALL`) |
| 클라이언트_목록 | IP, 대역, 호스트명, 패턴 |
| 옵션 | `spawn`, `twist`, `ALLOW`, `DENY` 등 (선택) |

### 클라이언트 표현 방법

| 표현 | 의미 |
|------|------|
| `192.168.1.1` | 특정 IP |
| `192.168.1.` | 192.168.1.0/24 대역 (끝에 `.` 포함) |
| `192.168.1.0/255.255.255.0` | CIDR 대역 (서브넷 마스크 방식) |
| `ALL` | 모든 클라이언트 |
| `LOCAL` | 같은 도메인의 호스트 |
| `KNOWN` | 역방향 DNS 조회 성공한 호스트 |
| `UNKNOWN` | 역방향 DNS 조회 실패한 호스트 |
| `PARANOID` | 정방향/역방향 DNS 불일치 호스트 |

---

## 설정 예시

### /etc/hosts.allow

```
# 특정 IP만 SSH 허용
sshd : 192.168.1.100

# 내부 대역 전체 허용
sshd : 192.168.1.

# 여러 대역 허용 (콤마로 구분)
sshd : 192.168.1. 10.0.0.

# 모든 서비스를 로컬 대역에 허용
ALL : 127.0.0.1 192.168.0.
```

### /etc/hosts.deny

```
# 허용 목록에 없는 SSH는 전부 차단 (화이트리스트 방식)
sshd : ALL

# 모든 서비스 기본 차단 (hosts.allow로만 선택 허용)
ALL : ALL
```

### 접속 차단 시 로그 남기기 (spawn 옵션)

```
sshd : 10.0.0.5 : spawn /bin/echo "$(date) DENIED %h" >> /var/log/hosts.deny.log
```

`%h`는 클라이언트 호스트명, `%a`는 클라이언트 IP로 치환됩니다.

---

## 주요 확장 변수

| 변수 | 의미 |
|------|------|
| `%a` | 클라이언트 IP |
| `%h` | 클라이언트 호스트명 (없으면 IP) |
| `%d` | 데몬 이름 |
| `%u` | 클라이언트 username (identd 응답) |
| `%s` | 서버 이름 (`%d@%H` 형식) |

---

## 규칙 테스트

```bash
# 특정 클라이언트가 특정 서비스에 접근 가능한지 확인
tcpdmatch sshd 192.168.1.100

# 출력 예시 (허용)
# client:   address  192.168.1.100
# server:   process  sshd
# matched:  /etc/hosts.allow line 1
# access:   granted

# 출력 예시 (거부)
# matched:  /etc/hosts.deny line 1
# access:   denied
```

---

## TCP Wrappers의 한계

| 항목 | 설명 |
|------|------|
| 적용 범위 | `libwrap`을 직접 사용하는 서비스에만 적용 |
| 포트 기반 제어 불가 | IP/호스트명 기준만 가능, 포트 필터링은 iptables 필요 |
| 현대 배포판 지원 축소 | Ubuntu 18.04 이후 많은 서비스가 libwrap 제거 추세 |
| systemd 환경 | `xinetd` 없이 직접 실행되는 데몬은 별도 설정 필요 |

현대 리눅스 환경에서는 TCP Wrappers 단독보다는 방화벽(iptables/nftables/firewalld)과 함께 계층적으로 사용하거나, 방화벽으로 대체하는 경우가 많습니다.
