---
title: "[Study] lspci 경로 인식 문제와 심볼릭 링크 임시 해결"
date: 2026-07-01 12:00:00 +0900
categories: [Linux]
subcategory: Study
tags: [linux, lspci, pci, path, symlink, symbolic-link, gpu, troubleshooting]
---

## 문제 상황

콘트라베이스 제품에서 물리 호스트에 설치하는 에이전트인 **mole**은 물리 호스트의 하드웨어 정보를 수집하고, `qemu-guest-agent`가 설치된 VM에도 접근해 정보를 가져오는 역할을 한다.

mole이 GPU 인식 여부를 확인하기 위해 내부적으로 `lspci`를 실행하는데, 호스트 목록에서 GPU 항목이 `No`로 표시되는 문제가 발생했다. 확인해보니 mole이 실행되는 환경에서 `lspci`를 찾지 못하는 것이 원인이었다.

```
lspci: command not found
```

***

## 원인: lspci가 PATH에 없는 경우

`lspci`는 PCI 장치 목록을 출력하는 명령어로, 패키지 이름은 `pciutils`다.

설치되어 있더라도 실행 파일 경로가 실행 환경의 `PATH`에 포함되지 않아 인식이 안 되는 경우가 있다.

```bash
# lspci가 어디 있는지 찾기
which lspci
# 결과 없으면:

find /usr -name lspci 2>/dev/null
# 예시 결과:
# /usr/sbin/lspci
```

에이전트처럼 systemd 서비스나 SSH non-login 쉘 형태로 실행되는 환경에서는 `/usr/sbin`이 PATH에서 빠지는 경우가 많다.

```bash
echo $PATH
# /usr/local/bin:/usr/bin:/bin  ← /usr/sbin 없음
```

***

## 임시 해결: 심볼릭 링크 설정

`/usr/bin`처럼 PATH에 포함된 경로에 심볼릭 링크를 걸어 어느 실행 환경에서든 `lspci`를 찾을 수 있게 한다.

```bash
# lspci 실제 경로 확인
find /usr -name lspci 2>/dev/null
# /usr/sbin/lspci

# /usr/bin에 심볼릭 링크 생성
sudo ln -s /usr/sbin/lspci /usr/bin/lspci

# 확인
which lspci
# /usr/bin/lspci

lspci | grep -i nvidia
# 정상 출력되면 해결
```

***

## 근본 해결: mole 추가 개발 필요

현재 심볼릭 링크는 물리 호스트마다 수동으로 설정해야 하는 임시 조치다. mole 자체에서 lspci 경로를 동적으로 탐색하도록 수정이 필요하다.

```
개선 방향:
- shutil.which('lspci') 등으로 경로 동적 탐색
- 탐색 실패 시 /usr/sbin/lspci, /sbin/lspci 순으로 fallback 시도
- 경로를 찾지 못할 경우 호스트 목록에 명확한 상태 메시지 표시
```

| 상태 | 항목 |
| --- | --- |
| 완료 | 물리 호스트에 심볼릭 링크 설정으로 임시 해결 |
| 예정 | mole에서 lspci 경로 동적 탐색 로직 추가 |
| 예정 | 탐색 실패 시 호스트 목록에 명확한 오류 상태 표시 |
