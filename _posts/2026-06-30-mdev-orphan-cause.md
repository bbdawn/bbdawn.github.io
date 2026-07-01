---
title: "[Troubleshooting] MIG 환경에서 mdev orphan이 반복되는 이유"
date: 2026-06-30 07:00:00 +0900
categories: [Troubleshooting, GPU]
subcategory: Troubleshooting
tags: [openstack, gpu, mig, mdev, orphan, nova, libvirt, troubleshooting]
---

## 개요

OpenStack MIG 환경에서 VM을 삭제해도 mdev 장치가 sysfs에 그대로 남는 현상이 반복됐습니다. GPU 슬라이스가 실제로는 비어 있지만 점유된 것으로 인식되어 새 VM에 할당할 수 없는 상태가 됩니다.

---

## mdev 장치의 정상 생명주기

OpenStack이 MIG 기반 GPU VM을 생성하고 삭제하는 정상 흐름은 아래와 같습니다.

```
[VM 생성]
nova-compute
  → mdevctl start -u <uuid> -t <mig-profile>   # mdev 장치 생성
  → libvirt attach-device                        # VM에 mdev 연결
  → VM 실행

[VM 삭제]
nova-compute
  → libvirt detach-device                        # VM에서 mdev 분리
  → mdevctl stop -u <uuid>                       # mdev 장치 제거
  → sysfs에서 장치 사라짐
```

orphan은 마지막 단계—`mdevctl stop`—가 실행되지 못할 때 발생합니다.

---

## orphan이 생기는 케이스

### 케이스 1. nova-compute 비정상 종료

VM 삭제 요청을 처리하는 도중 nova-compute가 crash되면, mdev 정리 단계에 진입하지 못합니다.

```
nova-compute 처리 흐름:
  1. libvirt detach-device ✓
  2. mdevctl stop          ← 여기서 nova-compute crash → 실행 안 됨
```

재시작 후 nova-compute는 이미 Nova DB에서 삭제된 VM을 재처리하지 않기 때문에 mdev는 정리되지 않고 남습니다.

### 케이스 2. libvirt detach 실패

libvirt가 VM에서 mdev 장치를 분리하는 데 실패하면, nova-compute는 이후 `mdevctl stop`을 호출하지 않고 흐름을 중단합니다.

```
libvirt detach-device → 오류 반환
nova-compute → 예외 처리 후 종료
mdevctl stop → 호출되지 않음
```

libvirt detach 실패는 VM이 이미 내부적으로 응답 불가 상태이거나, 도메인 XML과 실제 하드웨어 상태가 불일치할 때 발생합니다.

### 케이스 3. 강제 VM 삭제 (force-delete)

`nova force-delete` 또는 Nova API를 통한 강제 삭제는 정상 cleanup 흐름을 건너뜁니다.

```
nova force-delete
  → Nova DB에서 VM 즉시 제거
  → nova-compute의 teardown 로직 skip
  → mdev 장치 잔존
```

긴급 상황에서 force-delete를 사용할 경우 반드시 수동으로 mdev 정리가 필요합니다.

### 케이스 4. 정리 작업 타임아웃

mdevctl stop 또는 libvirt detach 작업이 nova-compute의 타임아웃 내에 완료되지 않으면 정리가 중단됩니다. GPU 부하가 높은 상황이나 sysfs I/O 지연 시 발생할 수 있습니다.

---

## orphan 확인 방법

```bash
# 1. sysfs에 남아있는 mdev 장치 전체 목록
ls /sys/bus/mdev/devices/

# 2. libvirt가 현재 인식하는 VM 목록
virsh list --all

# 3. 각 mdev UUID가 libvirt 도메인 XML에 참조되는지 확인
virsh dumpxml <domain-name> | grep -i mdev

# 4. Nova에서 인스턴스 목록 (OpenStack)
openstack server list --all-projects --host <compute-node>
```

sysfs에는 존재하지만 libvirt와 Nova 어디에도 연결되지 않은 mdev UUID가 orphan입니다.

---

## 정리 방법

```bash
# orphan mdev 정리
mdevctl stop -u <orphan-uuid>

# 정리 후 확인
ls /sys/bus/mdev/devices/
```

---

## 정리

mdev orphan의 근본 원인은 **Nova의 VM 삭제 흐름과 실제 GPU 장치 정리 사이의 간극**입니다. Nova는 DB 상태 기준으로 VM을 삭제하지만, GPU 하드웨어 레벨의 mdev 장치 정리는 nova-compute가 직접 수행하기 때문에 중간 단계에서 실패가 발생하면 정리가 누락됩니다.

이 문제가 반복되는 환경에서는 주기적으로 orphan 여부를 점검하고 정리하는 별도 운영 도구가 필요합니다.
