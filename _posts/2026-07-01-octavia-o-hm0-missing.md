---
title: "[Troubleshooting] Loadbalancer 생성 실패 원인(2) : o-hm0 인터페이스 없어서 생성 실패"
date: 2026-07-01 10:00:00 +0900
categories: [Troubleshooting, Openstack]
subcategory: Troubleshooting
tags: [openstack, octavia, loadbalancer, o-hm0, health-manager, amphora, ovs, troubleshooting]
---

## 오류 상황

노드 재부팅 등의 작업 이후 Health Manager와 Amphora 간 통신이 끊기면서, Loadbalancer가 정상적으로 생성되지 않는 경우가 있습니다. 이때 컨트롤 노드에서 `o-hm0` 인터페이스를 조회하면 다음과 같이 나옵니다.

```bash
$ ip link show o-hm0
Device "o-hm0" does not exist.
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

| 역할 | 설명 |
|------|------|
| Health Check | Amphora에 주기적으로 heartbeat를 보내 생존 여부 확인 |
| 설정 전달 | Listener, Pool, Member 설정을 REST API로 Amphora에 push |
| 상태 수신 | Amphora가 보내는 heartbeat UDP 패킷 수신 |

`o-hm0`이 없으면 Health Manager가 Amphora에 도달할 수 없어 설정 전달 자체가 불가능해지고, Amphora 인스턴스가 정상적으로 생성되더라도 Loadbalancer는 `ERROR` 상태가 됩니다.

---

## 해결 방법

### 1단계: dhclient로 복구 시도

가장 먼저 시도해볼 수 있는 방법은 `dhclient`로 인터페이스를 되살리는 것입니다.

```bash
sudo dhclient -v o-hm0
```

### 2단계: OVS 상태 확인

1단계로 복구되지 않으면 OVS 브릿지 상태를 확인합니다. `br-int`에 `o-hm0` 포트 정의는 남아있지만 실제 device는 없는 상태(`no device`)인 경우가 대부분입니다.

```bash
ovs-vsctl show
```

### 3단계: Health Manager port 정보 조회

Neutron에 등록된 Octavia Health Manager용 포트 목록을 조회해, 각 컨트롤 노드에 매핑된 MAC 주소와 port id(iface-id)를 확인합니다.

```bash
openstack port list | grep octavia-he
```

```
| 0b982ee1-fba1-4ac8-a91e-621998da418c | octavia-health-manager-listen-port-con3 | fa:16:3e:0e:65:34 | ip_address='172.16.0.4', subnet_id='6dffc781-...' | ACTIVE |
| 619bf3e5-fdc9-42f9-9884-b38f2e09e9eb | octavia-health-manager-listen-port-con1 | fa:16:3e:06:55:23 | ip_address='172.16.0.2', subnet_id='6dffc781-...' | ACTIVE |
| 65d8971c-3ada-4e9f-a992-516153766830 | octavia-health-manager-listen-port-con2 | fa:16:3e:ac:ff:51 | ip_address='172.16.0.3', subnet_id='6dffc781-...' | ACTIVE |
```

이 중 **인터페이스가 사라진 물리 서버(컨트롤 노드)에 해당하는 port**를 찾아 MAC 주소와 port id를 확인합니다.

### 4단계: o-hm0 인터페이스 재생성

확인한 MAC 주소(`attached-mac`)와 port id(`iface-id`)를 이용해 `br-int`에 `o-hm0`을 다시 붙여줍니다.

```bash
# 기존 포트 정의 제거
ovs-vsctl del-port br-int o-hm0

# 해당 노드의 MAC / port id로 재생성
ovs-vsctl add-port br-int o-hm0 \
  -- set Interface o-hm0 type=internal \
  -- set Interface o-hm0 external-ids:iface-status=active \
  -- set Interface o-hm0 external-ids:attached-mac=fa:16:3e:3d:e7:80 \
  -- set Interface o-hm0 external-ids:iface-id=36ffa58c-37d3-4580-82f3-afa379439c49

# MTU 설정 (lb-mgmt-net 기준 1450)
ovs-vsctl set Interface o-hm0 mtu_request=1450

# 인터페이스 MAC 주소 적용
sudo ip link set dev o-hm0 address fa:16:3e:3d:e7:80

# DHCP로 IP 재할당
sudo dhclient -v o-hm0
```

> 노드가 여러 대라면 각 컨트롤 노드마다 자신의 `octavia-health-manager-listen-port-conN`에 해당하는 MAC / port id를 사용해야 합니다.

### 5단계: 재발 방지용 스크립트

부팅 시마다 반복된다면, 아래와 같이 하나의 스크립트로 묶어 `octavia-interface.service`로 등록해두면 편리합니다.

```bash
#!/bin/bash

ovs-vsctl del-port br-int o-hm0

ovs-vsctl add-port br-int o-hm0 \
  -- set Interface o-hm0 type=internal \
  -- set Interface o-hm0 external-ids:iface-status=active \
  -- set Interface o-hm0 external-ids:attached-mac=fa:16:3e:d6:d0:1d \
  -- set Interface o-hm0 external-ids:iface-id=84eeeb25-342b-4d4a-ba60-bbee2e60d870

sudo ip link set dev o-hm0 address fa:16:3e:d6:d0:1d

ovs-vsctl set Interface o-hm0 mtu_request=1450

sudo dhclient -v o-hm0

systemctl restart octavia-interface.service
```

---

## 케이스 요약

| 증상 | 원인 | 해결 |
|------|------|------|
| `ip link show o-hm0` → Device does not exist | 노드 재부팅 등으로 o-hm0 인터페이스 소실 | `dhclient -v o-hm0`로 우선 복구 시도 |
| dhclient로도 복구 안 됨 | `br-int`에 포트 정의만 남고 실제 device가 없음 (`ovs-vsctl show`로 확인) | `ovs-vsctl del-port` 후 해당 노드의 MAC/iface-id로 `add-port` 재생성 |
| Loadbalancer ERROR, Amphora는 정상 | o-hm0 없어서 Health Manager ↔ Amphora 통신 불가 | 위 재생성 절차 진행 후 정상 확인 |
| 재부팅 후 반복 발생 | o-hm0 복구 로직이 시스템에 등록되어 있지 않음 | 재생성 절차를 스크립트로 묶어 `octavia-interface.service`로 등록 |
