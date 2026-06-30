---
title: "[Feature] Octavia TERMINATED_HTTPS SSL Offloading 구현"
date: 2026-06-30 15:00:00 +0900
categories: [Project, Openstack]
subcategory: Features
tags: [openstack, octavia, loadbalancer, barbican, ssl, https, p12, java, spring]
---

## 개요

Octavia 로드밸런서의 Listener를 `TERMINATED_HTTPS` 프로토콜로 생성하면, 클라이언트와 로드밸런서 사이의 TLS를 로드밸런서에서 종료(Offloading)하고, 백엔드 서버로는 HTTP로 전달합니다.

```
[클라이언트]
    ↕ HTTPS (TLS)
[Octavia Load Balancer]  ← 인증서 적용 (SSL Offloading)
    ↕ HTTP
[백엔드 서버]
```

인증서는 OpenStack **Barbican** (Key Management Service)에 저장하고, Octavia가 이를 참조합니다.

---

## 구성 요소

| 서비스 | 역할 |
|--------|------|
| Octavia | 로드밸런서 / Listener 관리 |
| Barbican | 인증서(Secret) 저장 및 관리 |

---

## Listener 생성 흐름

```
1. Listener 프로토콜 선택 → TERMINATED_HTTPS
2. Barbican Secret 목록 조회
3. 사용할 인증서(Secret) 선택
4. Listener 생성 요청 (default_tls_container_ref 포함)
```

### Barbican Secret 목록 조회

```java
List<BarbicanSecret> secrets = barbicanClient.listSecrets().stream()
    .filter(s -> "application/octet-stream".equals(s.getContentType()))
    .collect(toList());
```

### Listener 생성 요청

```java
ListenerCreateRequest request = ListenerCreateRequest.builder()
    .loadbalancerId(loadbalancerId)
    .protocol("TERMINATED_HTTPS")
    .protocolPort(443)
    .name(listenerName)
    .defaultTlsContainerRef(selectedSecretRef)  // Barbican Secret URL
    .build();

octaviaClient.createListener(request);
```

---

## 사용자 친화적 .p12 업로드 구현

### 기존 방식의 불편함

Barbican에 인증서를 등록하려면 `.p12` 파일을 **직접 base64로 인코딩**한 뒤 API로 전송해야 합니다.

```bash
# 기존 방식 — 사용자가 직접 수행
base64 -w 0 certificate.p12 > certificate.b64
# 이 base64 문자열을 Barbican API에 직접 전송
```

실제 사용자 입장에서는 번거롭고 실수가 생기기 쉬운 과정입니다.

### 개선: 포탈에서 .p12 직접 업로드

포탈에서 `.p12` 파일을 업로드하면, **백엔드가 base64 변환 및 Barbican 등록까지 자동으로 처리**합니다.

```
[사용자]
  .p12 파일 업로드
       ↓
[백엔드]
  1. .p12 파일 수신 (MultipartFile)
  2. Base64 인코딩
  3. Barbican Secret 생성 API 호출
  4. 생성된 Secret URL 반환
       ↓
[사용자]
  Secret 목록에서 선택 → Listener 생성
```

### 구현 코드

```java
@PostMapping("/secrets/upload")
public ResponseEntity<SecretUploadResponse> uploadCertificate(
        @RequestParam("file") MultipartFile file,
        @RequestParam("name") String secretName) {

    // 1. .p12 파일을 base64로 인코딩
    byte[] fileBytes = file.getBytes();
    String base64Payload = Base64.getEncoder().encodeToString(fileBytes);

    // 2. Barbican Secret 생성
    BarbicanSecretCreateRequest request = BarbicanSecretCreateRequest.builder()
        .name(secretName)
        .secretType("opaque")
        .payloadContentType("application/octet-stream")
        .payloadContentEncoding("base64")
        .payload(base64Payload)
        .build();

    BarbicanSecret created = barbicanClient.createSecret(request);

    return ResponseEntity.ok(SecretUploadResponse.builder()
        .secretRef(created.getSecretRef())
        .name(created.getName())
        .build());
}
```

---

## 전체 UI 흐름

```
[Listener 생성 화면]
  프로토콜 선택: TERMINATED_HTTPS
       ↓
  인증서 선택 영역 활성화
  ├── 기존 Secret 선택 (Barbican 목록)
  └── 새 인증서 업로드 (.p12 파일 선택)
           ↓ 업로드 시 자동 Barbican 등록
       ↓
  Listener 생성 완료
```

---

## 처리 결과

- `TERMINATED_HTTPS` Listener가 Amphora에 인증서를 주입하여 TLS 처리
- 백엔드 서버는 HTTP로만 통신하면 되므로 인증서 관리 부담 없음
- `.p12` 직접 업로드로 base64 변환 과정을 사용자에게 노출하지 않음
