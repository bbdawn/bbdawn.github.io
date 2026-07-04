---
title: "[Study] 네트워크 구성 — 외부/내부 네트워크부터 로드밸런서, 공유 볼륨까지"
date: 2026-07-02 09:00:00 +0900
categories: [Study, Openstack]
subcategory: Study
tags: [openstack, neutron, octavia, manila, network, loadbalancer, hands-on]
---

## 개요

콘트라베이스(OpenStack 기반 Private Cloud 관리 포탈) UI만으로 네트워크 → 인스턴스 → 유동 IP → 로드밸런서 → 공유 볼륨까지 하나의 서비스 환경을 처음부터 끝까지 구성해보는 실습입니다. Neutron, Octavia, Manila를 실제 화면에서 어떤 순서로, 어떤 개념으로 엮어 쓰는지 정리했습니다.

![contrabass network 구성도](/assets/img/posts/contrabass-network-diagram.png)
_실습에서 구성할 전체 네트워크 토폴로지_

---

## 1. 외부/내부 네트워크 생성 (Neutron)

**외부 네트워크**: `10.255.42.0/24` (이미 구성되어 있는 공용 대역)

**내부 네트워크(세그먼트) 생성**

`VPC > 세그먼트 > 세그먼트 생성`

| 항목 | 값 |
|------|-----|
| 세그먼트 유형 | 내부 세그먼트 |
| 이름 | jh.lee2-network |
| 프로젝트 | admin |
| 설정 | 외부 라우팅 ON, 포트 보안 ON |

**서브넷 생성**

`VPC > 세그먼트 > 서브넷 생성`

| 항목 | 값 |
|------|-----|
| 이름 | jh.lee2-subnet |
| CIDR | 192.168.20.0/24 |
| DHCP | ON |

> **세그먼트(네트워크)**: VM들이 붙는 가상의 L2 네트워크 — 집 안의 같은 와이파이 공간이라고 생각하면 됩니다.
> **서브넷**: 그 네트워크 안에서 쓰는 IP 대역 규칙 (예: `192.168.x.x`).

---

## 2. 인스턴스 생성 (Nova)

볼륨 스냅샷을 소스로 VM 2대를 생성합니다.

| 이름 | 소스 | 유형 | 세그먼트 |
|------|------|------|----------|
| jh.lee2-vm-1 | 볼륨 스냅샷 (jh.lee2-vm1-snapshot) | 2C2M | jh.lee2-network |
| jh.lee2-vm-2 | 볼륨 스냅샷 (jh.lee2-vm2-snapshot) | 2C2M | jh.lee2-network |

VM 접속 후 DNS 설정을 추가합니다.

```bash
ssh root@<유동IP>
# 접속 후
vi /etc/resolv.conf
# nameserver 8.8.8.8   ← 추가
```

---

## 3. 유동 IP 생성 (Neutron)

`VPC > 유동 IP > 생성`, 할당 세그먼트로 외부 네트워크(`10.255.42.0/24`)를 지정해 필요한 개수만큼 생성합니다.

> **유동 IP**: 내부 VM에 외부에서 접근할 수 있도록 매핑해주는 공인 IP.

![유동 IP 목록](/assets/img/posts/contrabass-network-floating-ip-list.png)
_생성된 유동 IP와 연동 대상(LB, VM1, VM2) 목록_

---

## 4. 라우터로 외부/내부 네트워크 연결 (Neutron)

**라우터 생성**

`VPC > 라우터 > 생성`

| 항목 | 값 |
|------|-----|
| 이름 | jh.lee2-router |
| 외부 라우팅 | ON (세그먼트: 10.255.42.0/24) |

**라우터 인터페이스 연결**

`VPC > 라우터 > 라우터 상세 > IP 대역 > IP 대역 추가`

| 항목 | 값 |
|------|-----|
| 세그먼트 | jh.lee2-network |
| SNAT 활성화 | ON |

> **라우터**: 내부 네트워크와 외부 네트워크를 연결하는 L3 장비 — 집 안 와이파이를 인터넷과 연결해주는 공유기 역할.

---

## 5. VM에 유동 IP 연동 및 테스트

`VPC > 유동 IP > 작업 > 자원 연동`으로 VM 포트에 유동 IP를 매핑하고, 외부에서 SSH 접속이 되는지 확인합니다.

---

## 6. 로드밸런서 생성 (Octavia)

`로드밸런서 생성`

| 항목 | 값 |
|------|-----|
| 이름 | jh.lee2-loadbalancer |
| 세그먼트 | jh.lee2-network |
| 풀 알고리즘 | ROUND_ROBIN |
| 풀 멤버 포트 | 80 (가중치 1) |
| 헬스 체크 | 설정 |

> **로드밸런서**: 여러 VM으로 트래픽을 분산시키는 장치 — 요청을 자동으로 나눠주는 교통 정리 담당.

## 7. 로드밸런서에 유동 IP 연동 / 8. 테스트

로드밸런서에도 유동 IP를 연동한 뒤, 해당 IP로 접속했을 때 VM 1과 VM 2로 ROUND_ROBIN 분산이 되는지 확인합니다.

---

## 9. 공유 볼륨 생성 (Manila)

`스토리지 > 공유 파일 > 공유 볼륨 생성`

| 항목 | 값 |
|------|-----|
| 이름 | poc-vol-share-01 |
| 구분 | 프로젝트 소유 |
| 프로토콜 | CEPHFS |
| 공유 타입 | default_share_type |
| 용량 | 1GB |

**메타데이터**로 마운트 옵션을 지정합니다.

| key | value |
|-----|-------|
| `__mount_options` | `fs=cephfs` |

![공유 볼륨 생성 화면](/assets/img/posts/contrabass-network-share-create.png)
_CEPHFS 프로토콜로 공유 볼륨을 생성하는 화면_

> **Manila 공유 볼륨**: 여러 VM이 동시에 접근할 수 있는 공유 파일 스토리지 — 여러 컴퓨터가 같이 쓰는 공용 드라이브(폴더).

**엑세스 규칙 설정**

| 항목 | 값 |
|------|-----|
| 엑세스 유형 | cephx |
| 엑세스 레벨 | read-write |
| 엑세스 경로 | alias |

![엑세스 규칙 추가](/assets/img/posts/contrabass-network-share-access-rule-add.png)
_엑세스 규칙 추가 다이얼로그_

![엑세스 규칙 목록](/assets/img/posts/contrabass-network-share-access-rule-list.png)
_생성된 엑세스 규칙 (엑세스 키 값은 블로그 게재를 위해 가렸습니다)_

### VM에서 mount

공유 볼륨 상세에서 접속 위치(monitor IP 목록)와 엑세스 규칙의 secret 키를 확인한 뒤, VM 콘솔에서 마운트합니다.

![공유 볼륨 상세 — 접속 위치](/assets/img/posts/contrabass-network-share-detail.png)
_마운트에 필요한 monitor IP 및 CephFS 경로 확인_

```bash
mkdir -p /share
cd /share

mount -t ceph <mon-ip-1>:6789,<mon-ip-2>:6789,<mon-ip-3>:6789:/volumes/_nogroup/<share-id>/<subvolume-id> \
  /share -o name=alias,secret='<엑세스 키 값>'
```

확인 및 해제:

```bash
mount | grep /share
umount /share
```

---

## 10. 공유 볼륨 테스트

VM 1, VM 2 양쪽에서 동일한 볼륨을 마운트한 뒤, 한쪽에서 쓴 파일이 다른 쪽에서도 즉시 보이는지 확인합니다.

```bash
cd /share
touch test.txt
ls
```

두 VM 모두 같은 `/share` 디렉터리를 통해 동일한 파일에 접근할 수 있으면 정상 동작입니다. (`rw` 권한이면 read/write, `ro`면 read만 가능)

![공유 볼륨 마운트 테스트](/assets/img/posts/contrabass-network-share-mount-test.png)
_VM1, VM2 양쪽 콘솔에서 같은 공유 파일이 동일하게 보이는 것을 확인_

![VM1, VM2 공유 스토리지 연결](/assets/img/posts/contrabass-network-vm-illustration.png)
_서로 다른 VM이지만 같은 Manila 공유 볼륨을 마운트하고 있는 상태_

---

## 정리

| 개념 | 한 줄 설명 |
|------|-----------|
| Network (세그먼트) | 집 안에 있는 같은 와이파이 공간 |
| Subnet | 그 와이파이 안에서 쓰는 IP 주소 범위 규칙 |
| Router | 와이파이를 인터넷과 연결해주는 공유기 |
| Floating IP | 내부 컴퓨터를 외부에서도 접속 가능하게 해주는 인터넷 주소 |
| LoadBalancer | 여러 서버로 요청을 자동으로 나눠주는 교통 정리 담당 |
| Manila | 여러 컴퓨터가 같이 쓰는 공용 드라이브(공유 폴더) |

---

## 기술 스택

| 역할 | 기술 |
|------|------|
| 네트워크 | Neutron (세그먼트, 서브넷, 라우터, 유동 IP) |
| 컴퓨트 | Nova |
| 로드밸런서 | Octavia |
| 공유 스토리지 | Manila (CephFS) |
| 플랫폼 | 콘트라베이스 (OpenStack 기반 Private Cloud) |
