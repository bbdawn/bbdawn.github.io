---
title: "[Feature] Floating IP 연동 가능 여부 표출"
date: 2026-06-30 14:00:00 +0900
categories: [Features, Openstack]
subcategory: Feature
tags: [openstack, neutron, floating-ip, network, router, java, spring]
---

## 문제 상황

OpenStack Neutron에서 Floating IP를 포트에 연동할 때, **라우터가 연결되지 않은 네트워크의 포트**와 연동을 시도하면 실패합니다.

```
FloatingIP 연동 실패
  └── ExternalGatewayForFloatingIPNotFound
       "포트가 속한 네트워크가 외부 네트워크에 연결된 라우터를 통해 접근 가능하지 않음"
```

기존 UI에서는 포트 목록을 그대로 보여줬기 때문에, 사용자가 연동 불가능한 포트를 선택해 오류를 만나는 상황이 반복됐습니다.

---

## 연동 가능 조건 분석

Floating IP 연동이 성공하려면 아래 경로가 모두 연결되어 있어야 합니다.

```
[External Network]
       ↕  (External Gateway)
   [Router]
       ↕  (Router Interface)
   [Network]
       ↕
    [Port]  ← FloatingIP 연동 대상
```

### 조건 체크리스트

| 조건 | 확인 방법 |
|------|----------|
| 포트가 속한 네트워크 확인 | `port.network_id` |
| 해당 네트워크에 라우터 인터페이스 존재 여부 | `router interface` 조회 |
| 라우터에 External Gateway 설정 여부 | `router.external_gateway_info` |

세 조건을 모두 만족하는 포트만 Floating IP 연동이 가능합니다.

---

## 구현 방법

### 1. 라우팅 가능한 네트워크 목록 추출

Neutron API를 통해 라우터와 인터페이스 정보를 수집합니다.

```java
// 1. External Gateway가 설정된 라우터 목록 조회
List<Router> routers = neutronClient.getRouters().stream()
    .filter(r -> r.getExternalGatewayInfo() != null)
    .collect(toList());

// 2. 해당 라우터들의 인터페이스(서브넷) 조회
Set<String> routableNetworkIds = new HashSet<>();
for (Router router : routers) {
    List<Port> routerPorts = neutronClient.getPorts().stream()
        .filter(p -> p.getDeviceId().equals(router.getId()))
        .filter(p -> p.getDeviceOwner().equals("network:router_interface"))
        .collect(toList());

    routerPorts.stream()
        .map(Port::getNetworkId)
        .forEach(routableNetworkIds::add);
}
```

### 2. 포트 목록 조회 시 연동 가능 여부 표시

포트 목록을 반환할 때 `routableNetworkIds` 기준으로 연동 가능 여부를 함께 반환합니다.

```java
List<PortResponse> portList = ports.stream()
    .map(port -> PortResponse.builder()
        .id(port.getId())
        .name(port.getName())
        .networkId(port.getNetworkId())
        .fixedIps(port.getFixedIps())
        .floatingIpAssociatable(routableNetworkIds.contains(port.getNetworkId()))
        .build())
    .collect(toList());
```

### 3. 응답 구조

```json
{
  "ports": [
    {
      "id": "port-uuid-1",
      "name": "vm-port-1",
      "networkId": "net-uuid-a",
      "fixedIps": [{ "ipAddress": "10.0.1.5", "subnetId": "subnet-uuid" }],
      "floatingIpAssociatable": true
    },
    {
      "id": "port-uuid-2",
      "name": "vm-port-2",
      "networkId": "net-uuid-b",
      "fixedIps": [{ "ipAddress": "192.168.10.3", "subnetId": "subnet-uuid-2" }],
      "floatingIpAssociatable": false
    }
  ]
}
```

---

## 연동 불가 케이스 정리

| 케이스 | 원인 |
|--------|------|
| 네트워크에 라우터 인터페이스 없음 | 라우터와 연결된 서브넷이 없어 외부 통신 불가 |
| 라우터에 External Gateway 미설정 | 라우터가 외부 네트워크로 나가는 경로 없음 |
| 포트가 External Network에 직접 연결됨 | Floating IP는 Internal Network 포트에만 연동 가능 |

---


## 효과

- 연동 불가 포트를 선택했을 때 발생하던 오류 사전 차단
- 사용자가 UI에서 연동 가능한 포트만 선택할 수 있도록 안내
- Neutron API 호출 실패로 인한 불필요한 에러 로그 감소


