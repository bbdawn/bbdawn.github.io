---
title: "[Troubleshooting] Loadbalancer: o-hm0 인터페이스 없어서 생성 실패"
date: 2026-07-01 10:00:00 +0900
categories: [Troubleshooting, Openstack]
subcategory: Troubleshooting
tags: [openstack, octavia, loadbalancer, o-hm0, health-manager, amphora, troubleshooting]
---

## 오류 메시지

```
Loadbalancer provisioning_status: ERROR
```

Octavia Loadbalancer를 생성했을 때 Amphora 인스턴스는 뜨는데 `provisioning_status`가 `ERROR`로 빠지는 경우, 또는 Health Manager 로그에 아래와 같은 메시지가 반복되는 경우.

```
ERROR octavia.amphorae.drivers.haproxy.rest_api_driver [...] Failed to get status from amphora
```

---

## o-hm0이란

`o-hm0`은 **Octavia Health Manager**가 Amphora 인스턴스와 통신하기 위해 사용하는 네트워크 인터페이스입니다.

```
[Controller / Network Node]
      o-hm0  ─────────────────────────────────────────┐
       │                                               │
       │  lb-mgmt-net (관리 전용 네트워크)             │
       │                                               │
  [Amphora-Active]                            [Amphora-Standby]
   haproxy 실행                                haproxy 실행
```

### 역할 요약

| 역할 | 설명 |
|------|------|
| Health Check | Amphora에 주기적으로 heartbeat를 보내 생존 여부 확인 |
| 설정 전달 | Listener, Pool, Member 설정을 REST API로 Amphora에 push |
| 상태 수신 | Amphora가 보내는 heartbeat UDP 패킷 수신 (기본 포트 5555) |

`o-hm0`이 없으면 Health Manager가 Amphora에 도달할 수 없어 **설정 전달 자체가 불가능**해집니다. Amphora 인스턴스가 정상적으로 생성되더라도 Loadbalancer는 `ERROR` 상태가 됩니다.

---

## 확인 방법

```bash
ip link show o-hm0
```

정상이라면 다음과 같이 출력됩니다.

```
5: o-hm0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether fa:16:3e:xx:xx:xx brd ff:ff:ff:ff:ff:ff
```

인터페이스가 없으면 다음과 같이 출력됩니다.

```
Device "o-hm0" does not exist.
```

### Health Manager 로그 추가 확인

```bash
journalctl -u octavia-health-manager -n 100 --no-pager
# 또는
tail -f /var/log/octavia/health-manager.log
```

---

## 원인

`o-hm0`은 **Octavia 설치 시 한 번만 생성**하도록 설계된 인터페이스입니다. 다음 상황에서 사라질 수 있습니다.

| 원인 | 상황 |
|------|------|
| 노드 재부팅 | 재부팅 후 자동 복구 설정이 없으면 사라짐 |
| `ip link delete` 실수 | 운영 중 수동 삭제 |
| 초기 설치 누락 | Octavia 설치 시 `create_mgmt_network` 단계 스킵 |
| Neutron port 삭제 | lb-mgmt-net의 Health Manager port가 삭제됨 |

---

## 해결 방법

### 1. Health Manager port 및 인터페이스 재생성

Octavia Health Manager에 할당된 Neutron port를 확인하고, 그 MAC 주소를 이용해 `o-hm0`을 다시 생성합니다.

#### 1단계: Health Manager용 Neutron port 확인

```bash
# lb-mgmt-net에서 Health Manager에 할당된 port 확인
openstack port list --network lb-mgmt-net

# 또는 octavia.conf에서 직접 확인
grep -i "bind_ip\|health_manager" /etc/octavia/octavia.conf
```

#### 2단계: port가 없으면 새로 생성

```bash
# lb-mgmt-net ID 확인
MGMT_NET_ID=$(openstack network show lb-mgmt-net -f value -c id)

# Health Manager용 port 생성
openstack port create \
  --network $MGMT_NET_ID \
  --device-owner Octavia:health-mgr \
  --security-group lb-health-mgr-sec-grp \
  octavia-health-manager-port
```

#### 3단계: port의 MAC 주소로 o-hm0 생성

```bash
# port의 MAC 주소 확인
HM_PORT_MAC=$(openstack port show octavia-health-manager-port -f value -c mac_address)
HM_PORT_ID=$(openstack port show octavia-health-manager-port -f value -c id)

# o-hm0 인터페이스 생성
sudo ip link add o-hm0 type veth peer name o-hm0@if

# 또는 OVS/linuxbridge 환경에 따라 tap 방식으로 생성
sudo ip link add o-hm0 type dummy
sudo ip link set o-hm0 address $HM_PORT_MAC
sudo ip link set o-hm0 up
```

#### 4단계: IP 주소 부여

```bash
# Health Manager bind_ip 확인
HM_IP=$(grep bind_ip /etc/octavia/octavia.conf | awk -F= '{print $2}' | tr -d ' ')

sudo ip addr add ${HM_IP}/24 dev o-hm0
sudo ip link set o-hm0 up
```

#### 5단계: Octavia 재시작

```bash
sudo systemctl restart octavia-health-manager octavia-worker
```

---

### 2. 재부팅 후에도 유지되도록 systemd 설정 (재발 방지)

`/etc/systemd/network/` 또는 `networkd-dispatcher`를 이용해 o-hm0을 부팅 시 자동 복구하도록 설정합니다.

```bash
# /etc/octavia/post-up.d/o-hm0-up.sh 같은 스크립트를 작성해두면 편리합니다
cat << 'EOF' > /usr/local/bin/octavia-hm-interface.sh
#!/bin/bash
HM_MAC="fa:16:3e:xx:xx:xx"   # 실제 MAC으로 교체
HM_IP="192.168.200.1"        # 실제 bind_ip로 교체

if ! ip link show o-hm0 > /dev/null 2>&1; then
    ip link add o-hm0 type dummy
    ip link set o-hm0 address $HM_MAC
    ip addr add ${HM_IP}/24 dev o-hm0
    ip link set o-hm0 up
fi
EOF
chmod +x /usr/local/bin/octavia-hm-interface.sh
```

---

## 케이스 요약

| 증상 | 원인 | 해결 |
|------|------|------|
| Loadbalancer ERROR, Amphora는 정상 | o-hm0 없어서 Health Manager → Amphora 통신 불가 | o-hm0 재생성 + octavia 재시작 |
| 재부팅 후 반복 발생 | o-hm0 지속성 설정 없음 | 부팅 스크립트 등록 |
| Amphora heartbeat timeout | o-hm0 IP 잘못 설정 | octavia.conf의 bind_ip와 일치하는지 확인 |
