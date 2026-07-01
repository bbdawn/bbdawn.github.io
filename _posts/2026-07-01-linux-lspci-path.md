---
title: "[Study] lspci 경로 인식 문제와 심볼릭 링크 임시 해결"
date: 2026-07-01 12:00:00 +0900
categories: [Study, Linux]
subcategory: Study
tags: [linux, lspci, pci, path, symlink, symbolic-link, gpu, troubleshooting]
---

## 문제 상황

GPU 호스트 관리 도구(mole)에서 호스트 목록의 GPU 인식 여부 항목이 `No`로 표시되는 현상이 발생했다. 확인해보니 도구 내부에서 `lspci`를 실행할 때 명령어를 찾지 못하는 것이 원인이었다.

```
lspci: command not found
```

---

## 원인: lspci가 PATH에 없는 경우

`lspci`는 PCI 장치 목록을 출력하는 명령어로, 패키지 이름은 `pciutils`다.

설치되어 있더라도 실행 파일 경로가 현재 사용자의 `PATH`에 포함되지 않아 인식이 안 되는 경우가 있다.

```bash
# lspci가 어디 있는지 찾기
which lspci
# 결과 없으면:

find / -name lspci 2>/dev/null
# 예시 결과:
# /usr/sbin/lspci
# /usr/bin/lspci  ← 없을 수도 있음
```

일반 사용자 세션이나 특정 실행 환경(systemd 서비스, SSH non-login 쉘 등)에서는 `/usr/sbin`이 PATH에서 빠지는 경우가 있다.

```bash
echo $PATH
# /usr/local/bin:/usr/bin:/bin  ← /usr/sbin 없음
```

---

## 임시 해결: 심볼릭 링크 설정

`/usr/bin`처럼 PATH에 포함된 경로에 심볼릭 링크를 걸어 어느 환경에서든 `lspci`를 찾을 수 있게 한다.

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

---

## 다른 해결 방법들

### PATH에 /usr/sbin 추가 (사용자 환경)

```bash
# ~/.bashrc 또는 ~/.profile에 추가
export PATH="$PATH:/usr/sbin"
source ~/.bashrc
```

### lspci 패키지 재설치

```bash
# Debian/Ubuntu
sudo apt install pciutils

# RHEL/CentOS
sudo dnf install pciutils
```

### 절대 경로로 직접 호출

도구 코드에서 `lspci` 대신 `/usr/sbin/lspci`처럼 절대 경로를 사용하면 PATH 의존성 없이 실행할 수 있다.

```python
import subprocess
result = subprocess.run(["/usr/sbin/lspci"], capture_output=True, text=True)
```

---

## 근본 해결: mole 추가 개발 필요

현재 심볼릭 링크는 임시 조치다. mole에서 lspci를 실행할 때 PATH에 의존하지 않고 실행 파일을 동적으로 탐색하도록 수정이 필요하다.

```
개선 방향:
- which / shutil.which() 로 lspci 경로 동적 탐색
- 탐색 실패 시 /usr/sbin/lspci, /usr/bin/lspci 순으로 fallback
- 경로를 찾지 못할 경우 호스트 목록에 'No (lspci not found)' 등 명확한 메시지 표시
```

| 상태 | 항목 |
|------|------|
| 완료 | 심볼릭 링크로 임시 해결 |
| 예정 | mole에서 lspci 경로 동적 탐색 로직 추가 |
| 예정 | 탐색 실패 시 사용자에게 명확한 오류 메시지 표시 |
