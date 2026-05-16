# OAuth2 Authorization Server với FIDO2/WebAuthn

---

## Metadata

| Trường | Giá trị |
|---|---|
| **Spec ID** | AUTH-FIDO2-001 |
| **Version** | 1.0.0 |
| **Trạng thái** | Draft — Chờ Security Lead approve |
| **Tác giả** | _(điền tên BA/Tech Lead)_ |
| **Reviewer** | Security Lead · Backend Lead · QA Lead · PM |
| **Ngày tạo** | 2026-05-16 |
| **Ngày cập nhật** | 2026-05-16 |
| **Target release** | Sprint 8 — Q3 2026 |
| **Jira Epic** | AUTH-120 |
| **Confluence** | /spaces/ENG/auth/oauth2-fido2 |

### Change log

| Ver | Ngày | Tác giả | Thay đổi |
|---|---|---|---|
| 1.0.0 | 2026-05-16 | — | Hoàn thiện: Thêm ADR, Account Recovery, Traceability Matrix, UI Behavior, i18n |
| 0.3.0 | 2026-05-16 | — | Bổ sung User Stories, NFR, OpenAPI, Migration plan, Glossary |
| 0.2.0 | 2026-05-10 | — | Thêm Registration flow, DB schema, Security constraints |
| 0.1.0 | 2026-05-05 | — | Draft ban đầu: kiến trúc tổng thể, Auth flow |

---

## Mục lục

1. [Business Context](#1-business-context)
2. [Scope](#2-scope)
3. [Dependencies](#3-dependencies)
4. [Glossary](#4-glossary)
5. [Architecture](#5-architecture)
6. [User Stories & Acceptance Criteria](#6-user-stories--acceptance-criteria)
7. [Auth Flows](#7-auth-flows)
8. [API Contract (OpenAPI)](#8-api-contract-openapi)
9. [Database Schema](#9-database-schema)
10. [Permission & Role Model](#10-permission--role-model)
11. [Security Requirements](#11-security-requirements)
12. [Non-Functional Requirements](#12-non-functional-requirements)
13. [UI/UX Behavior Spec](#13-uiux-behavior-spec)
14. [Account Recovery Flow](#14-account-recovery-flow)
15. [Migration Plan](#15-migration-plan)
16. [Architecture Decision Records (ADR)](#16-architecture-decision-records-adr)
17. [Traceability Matrix](#17-traceability-matrix)
18. [Edge Cases & Escalation](#18-edge-cases--escalation)

---

## 1. Business Context

Hệ thống xác thực hiện tại dùng username/password. Theo báo cáo Q1 2026:

- **34%** ticket support liên quan đến quên mật khẩu
- **2/3** security incident xuất phát từ phishing tấn công credential
- Chi phí reset password ước tính **$12/lần** (nhân lực support)

Spec này định nghĩa việc tích hợp **FIDO2/WebAuthn (passkey)** vào Authorization Server, loại bỏ hoàn toàn password trong luồng xác thực chính:

- Private key không bao giờ rời thiết bị → phishing về lý thuyết là không thể
- Không có password database → không thể bị credential stuffing
- UX nhanh hơn: xác thực bằng sinh trắc học thay vì gõ password

**Tiêu chuẩn áp dụng:**
- OAuth2: [RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749) (2012-10) + [RFC 7636 PKCE](https://datatracker.ietf.org/doc/html/rfc7636) (2015-09)
- WebAuthn: [W3C WebAuthn Level 2](https://www.w3.org/TR/webauthn-2/) (2021-04-08)
- CTAP2: [FIDO Alliance CTAP 2.1](https://fidoalliance.org/specs/fido-v2.1-ps-20210615/fido-client-to-authenticator-protocol-v2.1-ps-20210615.html) (2021-06)
- OpenID Connect: [OIDC Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)

---

## 2. Scope

### Trong scope (v1)

| # | Feature |
|---|---|
| S-01 | Đăng ký passkey lần đầu (Registration ceremony) |
| S-02 | Xác thực bằng passkey (Authentication ceremony) |
| S-03 | OAuth2 Authorization Code + PKCE flow |
| S-04 | Quản lý nhiều passkey trên một tài khoản |
| S-05 | Revoke passkey từ phía user |
| S-06 | Account recovery khi mất tất cả passkey |
| S-07 | Refresh token rotation |
| S-08 | Token introspection cho Resource Server |
| S-09 | Admin revoke credential |
| S-10 | Audit log cho mọi auth event |
| S-11 | WebAuthn attestation conveyance `none` cho registration v1; không validate FIDO Metadata Service (MDS3) |

### Ngoài scope (v1)

| # | Feature | Lý do |
|---|---|---|
| O-01 | Enterprise SSO / SAML federation | Scope riêng — AUTH-200 |
| O-02 | Passkey cross-device sync (iCloud/Google) | Phụ thuộc OS, không kiểm soát được |
| O-03 | Admin portal UI quản lý credential | UI sprint riêng |
| O-04 | Step-up auth cho high-risk action | Phase 2 |
| O-05 | Client Credentials grant | Không có use case hiện tại |
| O-06 | FIDO2 Attestation verification với FIDO MDS3, trusted roots và provenance phần cứng | Phase 2; v1 chọn `attestation: "none"` để giảm operational complexity và không dùng attestation statement làm bằng chứng provenance |
| O-07 | Biometrics enrollment UI | Phụ thuộc OS/device |

---

## 3. Dependencies

| ID | Service | Loại | Mô tả |
|---|---|---|---|
| DEP-01 | AUTH-089 — Email verification | Hard | User phải verify email trước khi đăng ký passkey |
| DEP-02 | INFRA-044 — Redis cluster | Hard | Lưu challenge, session state; TTL management |
| DEP-03 | SEC-012 — Key management (HSM) | Hard | Lưu JWT signing key |
| DEP-04 | FE-201 — WebAuthn browser SDK | Hard | Client-side WebAuthn API wrapper |
| DEP-05 | MOB-033 — iOS/Android passkey API | Hard | Mobile native passkey support |
| DEP-06 | OPS-017 — Alerting pipeline | Soft | Alert khi sign_count anomaly, 5xx spike |
| DEP-07 | DATA-055 — Audit log pipeline | Soft | Splunk ingestion, 12-tháng retention |

---

## 4. Glossary

| Thuật ngữ | Định nghĩa |
|---|---|
| **Passkey** | Phương thức xác thực thay password. Private key lưu trong thiết bị (không xuất được), server chỉ lưu public key. Đăng nhập bằng sinh trắc học/PIN. |
| **Authenticator** | Thiết bị/phần mềm lưu private key. Hai loại: Platform (Touch ID, Face ID, Windows Hello) và Roaming (YubiKey, USB key). |
| **Relying Party (RP)** | Trang web/ứng dụng muốn xác thực user — trong spec này là Authorization Server. rpId = domain name. |
| **Challenge** | 32 bytes ngẫu nhiên server tạo, gửi cho authenticator để ký. Unique mỗi lần, TTL 5 phút, chống replay attack. |
| **Assertion** | Kết quả authenticator trả về khi login: signature trên challenge + authenticatorData. Server dùng public key để verify. |
| **Attestation** | Quá trình xác minh danh tính authenticator khi đăng ký passkey lần đầu. |
| **Relying Party ID (rpId)** | Domain name của RP, dùng để bind passkey. Passkey của "yourdomain.com" không dùng được trên domain khác. |
| **Sign count** | Bộ đếm tăng mỗi lần authenticator ký. Server so sánh để phát hiện clone attack. |
| **Discoverable credential** | Passkey lưu user handle, cho phép login không cần nhập username. Spec này yêu cầu `residentKey=required`. |
| **AAGUID** | Authenticator Attestation GUID — định danh model authenticator (không phải thiết bị cụ thể). |
| **COSE** | CBOR Object Signing and Encryption — định dạng encode public key trong FIDO2. |
| **ES256** | ECDSA với P-256 curve, algorithm ID = -7. Ưu tiên dùng vì key nhỏ, nhanh. |
| **RS256** | RSA-PKCS1v15 với SHA-256, algorithm ID = -257. Dùng khi device không hỗ trợ EC. |
| **clientDataJSON** | JSON authenticator tạo: type, challenge, origin. Server verify để chống phishing. |
| **authenticatorData** | Binary blob: rpIdHash, flags (UP, UV), sign count, credential data. |
| **Authorization Code** | Mã tạm thời (10 phút, one-time) server cấp sau FIDO2 auth, client dùng đổi lấy token. |
| **Access token** | JWT ngắn hạn (15 phút) để gọi Resource Server API. |
| **Refresh token** | Opaque token dài hạn (7 ngày) để lấy access token mới, không cần re-auth. |
| **PKCE** | Proof Key for Code Exchange — cơ chế chống đánh cắp authorization code cho public client. |
| **AMR** | Authentication Methods References — claim JWT cho biết phương thức auth đã dùng. Spec này: `amr=["fido2"]`. |
| **Ceremony** | Thuật ngữ FIDO2 cho một quy trình hoàn chỉnh: Registration ceremony hoặc Authentication ceremony. |
| **UP flag** | User Presence — bit trong authenticatorData, = 1 khi user đã interact với authenticator. |
| **UV flag** | User Verification — bit trong authenticatorData, = 1 khi user đã verify PIN/biometric. Spec yêu cầu UV=1. |

---

## 5. Architecture

### Các thành phần chính

```
┌─────────────────────────────────────────────────────────────┐
│                    Authorization Server                      │
│                                                             │
│  ┌─────────────────┐      ┌──────────────────────┐         │
│  │  OAuth2 Engine  │◄────►│   FIDO2 Engine        │         │
│  │  /authorize     │      │  /register/begin|finish│        │
│  │  /token         │      │  /auth/begin|finish    │        │
│  │  /introspect    │      │  /credentials          │        │
│  └────────┬────────┘      └──────────┬─────────────┘        │
│           │                          │                       │
│  ┌────────▼────────┐      ┌──────────▼─────────────┐        │
│  │ Session Manager │      │    Token Factory         │        │
│  │ Challenge store │      │  JWT/opaque token gen    │        │
│  └────────┬────────┘      └──────────┬─────────────┘        │
│           │                          │                       │
│  ┌────────▼────────┐      ┌──────────▼─────────────┐        │
│  │   User Store    │      │    Token Store (Redis)   │        │
│  │   (PostgreSQL)  │      │  auth_code, refresh_token│        │
│  └─────────────────┘      └─────────────────────────┘        │
└─────────────────────────────────────────────────────────────┘
        ▲                                   ▲
        │ Auth Code + PKCE                  │ JWT validate
   ┌────┴──────┐                    ┌───────┴───────┐
   │  Client   │                    │ Resource Server│
   │ SPA/Mobile│                    │  /api/*        │
   └───────────┘                    └───────────────┘
```

### Tech stack

| Layer | Technology |
|---|---|
| Runtime | Node.js 20 LTS / Go 1.22 |
| Framework | Fastify 4.x hoặc Gin |
| FIDO2 library | `@simplewebauthn/server` (Node) hoặc `go-webauthn/webauthn` |
| Database | PostgreSQL 16 |
| Cache / Session | Redis 7.2 (Sentinel mode) |
| Key management | AWS KMS / HashiCorp Vault |
| Observability | OpenTelemetry → Datadog |
| Container | Docker + Kubernetes (HPA enabled) |

---

## 6. User Stories & Acceptance Criteria

> **Format:** `TC-{US_ID}-{số}` = Test Case ID dùng để link Jira test management

---

### US-AUTH-001 — Đăng ký passkey

**Story:** Là người dùng đã xác minh email, tôi muốn đăng ký passkey trên thiết bị của mình, để đăng nhập mà không cần nhớ mật khẩu.

**Priority:** High | **Jira:** AUTH-121

#### TC-001-01 — Đăng ký thành công

```
Given: User đã xác minh email, chưa có passkey, browser/device hỗ trợ WebAuthn
When:  POST /fido2/register/begin với user_id hợp lệ
       → User hoàn thành sinh trắc học trên thiết bị
       → POST /fido2/register/finish với WebAuthn registration response hợp lệ (`attestation=none`)
Then:  HTTP 200
       Body: { "credential_id": "...", "message": "Passkey đã được thêm thành công" }
       DB: fido2_credentials có bản ghi mới với sign_count=0
       Email xác nhận được gửi đến user
```

**Sample payload /register/finish:**
```json
{
  "id": "mfaFAs7FKdPlBDqo...",
  "rawId": "mfaFAs7FKdPlBDqo...",
  "type": "public-key",
  "response": {
    "attestationObject": "o2NmbXRkbm9uZWd...",
    "clientDataJSON": "eyJ0eXBlIjoid2Vi...",
    "transports": ["internal"]
  }
}
```

#### TC-001-02 — Đăng ký thiết bị đã tồn tại

```
Given: User đã có passkey trên thiết bị A (credential_id = X đã lưu trong DB)
When:  User thử register lại cùng thiết bị A
Then:  HTTP 400
       Body: { "error": "credential_already_registered",
               "message": "Thiết bị này đã được đăng ký" }
       DB: Không tạo bản ghi mới
```

#### TC-001-03 — Challenge hết hạn (5 phút)

```
Given: User đã gọi /register/begin thành công (challenge được lưu Redis TTL 5m)
When:  User gọi /register/finish sau 5 phút 1 giây
Then:  HTTP 400
       Body: { "error": "challenge_expired",
               "message": "Phiên đăng ký đã hết hạn, vui lòng thử lại" }
       DB: Không có bản ghi mới
       Redis: Challenge đã bị xóa tự động
```

#### TC-001-04 — Email chưa xác minh

```
Given: User tồn tại nhưng email chưa verified
When:  POST /fido2/register/begin
Then:  HTTP 403
       Body: { "error": "email_not_verified",
               "message": "Vui lòng xác minh email trước khi đăng ký passkey" }
```

#### TC-001-05 — Rate limit đăng ký

```
Given: Cùng IP gọi /register/begin 10 lần trong 60 giây
When:  Request thứ 11 trong cùng 60 giây
Then:  HTTP 429
       Header: Retry-After: 60
       Body: { "error": "rate_limit_exceeded" }
```

---

### US-AUTH-002 — Đăng nhập bằng passkey

**Story:** Là người dùng đã có passkey, tôi muốn đăng nhập bằng vân tay hoặc Face ID, để truy cập ứng dụng nhanh và an toàn.

**Priority:** High | **Jira:** AUTH-122

#### TC-002-01 — Đăng nhập thành công

```
Given: User có passkey hợp lệ
       GET /oauth2/authorize với PKCE đúng chuẩn (S256)
When:  POST /fido2/auth/finish với assertion hợp lệ
Then:  HTTP 302 redirect → redirect_uri?code={auth_code}&state={state}
       DB: sign_count tăng thêm 1, last_used_at = now()
       Redis: Transaction/challenge bị đánh dấu used và xóa ngay sau khi issue code
       Redis: auth_code:{code} chứa các OAuth field đã bind từ transaction
```

**Sample payload /auth/finish:**

> Khuyến nghị: FIDO2 UI gửi request trong cùng browser session đã nhận cookie `__Host-auth_tx` (HttpOnly, Secure, SameSite=Lax/Strict) từ `GET /oauth2/authorize`. Nếu UI/native client cần reference rõ ràng, truyền thêm `auth_transaction_id` do server cấp; server vẫn phải đối chiếu với server-side session trước khi chấp nhận.

```json
{
  "auth_transaction_id": "atx_01HZY...",
  "id": "mfaFAs7FKdPlBDqo...",
  "rawId": "mfaFAs7FKdPlBDqo...",
  "type": "public-key",
  "response": {
    "authenticatorData": "SZYN5YgOjGh0NBc...",
    "clientDataJSON": "eyJ0eXBlIjoid2Vi...",
    "signature": "MEUCIQDSVgRFBbzK...",
    "userHandle": "dXNlci0xMjM0NTY..."
  }
}
```

#### TC-002-02 — Signature sai (tampered)

```
Given: User gọi /auth/begin thành công
When:  User gửi /auth/finish với signature bị thay đổi 1 byte
Then:  HTTP 401
       Body: { "error": "invalid_signature" }
       Log: WARN | { user_id, ip, user_agent, timestamp, credential_id, auth_transaction_id }
       Redis: Transaction/challenge bị đánh dấu used/failed và xóa (không thể retry cùng challenge)
```

#### TC-002-03 — Sign count giảm (clone attack)

```
Given: DB lưu sign_count = 5 cho credential X
When:  Authenticator trả về sign_count = 3 trong assertion
Then:  HTTP 401
       Body: { "error": "sign_count_anomaly" }
       DB: credential status = "suspended"
       Alert: #security-alerts nhận thông báo ngay lập tức
       Log: ERROR | { credential_id, expected_min: 6, received: 3 }
```

#### TC-002-04 — Origin mismatch (phishing)

```
Given: rpId = "yourdomain.com"
When:  clientDataJSON.origin = "https://evil-phishing.com"
Then:  HTTP 400
       Body: { "error": "origin_mismatch" }
       Log: ERROR | { received_origin, expected_origin, ip }
```

#### TC-002-05 — Rate limit authentication

```
Given: Cùng IP gọi /auth/begin 5 lần trong 60 giây
When:  Request thứ 6 trong cùng 60 giây
Then:  HTTP 429
       Header: Retry-After: 60
       Body: { "error": "rate_limit_exceeded" }
```

#### TC-002-06 — UV flag = 0 (user không verify PIN/biometric)

```
Given: Server policy userVerification = "required"
When:  authenticatorData.flags.UV = 0
Then:  HTTP 400
       Body: { "error": "user_verification_required",
               "message": "Thiết bị yêu cầu xác minh sinh trắc học hoặc PIN" }
```

#### TC-002-07 — Synced passkey (backup_state = true) cho high-assurance scope

```
Given: User request scope = "admin:write"
       Credential được dùng có backup_state = true (synced passkey)
When:  /auth/finish thành công về mặt signature
Then:  HTTP 403
       Body: { "error": "insufficient_assurance",
               "message": "Scope này yêu cầu hardware-bound passkey" }
```

---

### US-AUTH-003 — Quản lý danh sách passkey

**Story:** Là người dùng đã đăng nhập, tôi muốn xem và xóa các passkey đã đăng ký, để kiểm soát thiết bị nào có thể truy cập tài khoản.

**Priority:** Medium | **Jira:** AUTH-123

#### TC-003-01 — Xem danh sách passkey

```
Given: User đã đăng nhập (valid access_token), có 2 passkey
When:  GET /fido2/credentials
Then:  HTTP 200
       Body: array gồm các object:
       {
         "credential_id": "mfaFAs...(masked, last 8 chars)",
         "aaguid": "uuid",
         "transports": ["internal"],
         "backup_eligible": false,
         "backup_state": false,
         "created_at": "2026-05-01T08:00:00Z",
         "last_used_at": "2026-05-16T10:30:00Z"
       }
       KHÔNG trả về: public_key_cose, sign_count
```

#### TC-003-02 — Xóa passkey (còn ≥ 2)

```
Given: User có 2 passkey A và B
When:  DELETE /fido2/credentials/{credential_id_A}
Then:  HTTP 200
       Body: { "message": "Passkey đã được xóa" }
       DB: credential A bị xóa, chỉ còn B
       Email: Thông báo "Một passkey đã bị xóa khỏi tài khoản của bạn"
```

#### TC-003-03 — Không cho xóa passkey cuối

```
Given: User chỉ còn 1 passkey
When:  DELETE /fido2/credentials/{last_credential_id}
Then:  HTTP 409
       Body: { "error": "last_credential",
               "message": "Không thể xóa passkey cuối. Thêm passkey mới trước." }
```

---

### US-AUTH-004 — Admin revoke credential

**Story:** Là Admin/Support T2, tôi muốn vô hiệu hóa passkey của một tài khoản, để xử lý tài khoản bị xâm phạm hoặc user mất thiết bị.

**Priority:** High | **Jira:** AUTH-124

#### TC-004-01 — Admin revoke tất cả credential

```
Given: Caller role = ADMIN, scope = admin:write
       user_id tồn tại, có 3 credentials
When:  DELETE /admin/users/{user_id}/credentials
Then:  HTTP 200
       DB: 3 credentials bị xóa
       Redis: Tất cả active session của user bị invalidate
       Audit log: { admin_id, user_id, action:"revoke_all", timestamp, ip }
       Email user: "Tất cả passkey đã bị revoke bởi quản trị viên"
```

#### TC-004-02 — Non-admin bị từ chối

```
Given: Caller role = USER
When:  DELETE /admin/users/{user_id}/credentials
Then:  HTTP 403
       Body: { "error": "insufficient_scope" }
```

---

### US-AUTH-005 — Account Recovery

**Story:** Là người dùng đã mất tất cả thiết bị có passkey, tôi muốn lấy lại quyền truy cập tài khoản, để không bị mất tài khoản vĩnh viễn.

**Priority:** Critical | **Jira:** AUTH-125

#### TC-005-01 — Recovery thành công qua email OTP

```
Given: User không có passkey nào còn hoạt động
When:  POST /auth/recovery/begin với { "email": "user@example.com" }
       → User nhận OTP 6 số qua email (TTL 10 phút)
       → POST /auth/recovery/verify với { "email", "otp": "123456" }
       → User đăng ký passkey mới qua /fido2/register flow
Then:  HTTP 200 tại mỗi bước
       DB: Passkey mới được tạo
       Audit log: { action: "account_recovery", user_id, timestamp, ip }
       Email: "Tài khoản của bạn đã được khôi phục"
```

#### TC-005-02 — OTP sai 3 lần liên tiếp

```
Given: User đã request recovery OTP
When:  User nhập OTP sai 3 lần liên tiếp
Then:  HTTP 429 ở lần thứ 3
       Body: { "error": "max_otp_attempts",
               "message": "Quá nhiều lần thử. Vui lòng yêu cầu OTP mới." }
       Redis: OTP hiện tại bị xóa (phải request OTP mới)
```

#### TC-005-03 — OTP hết hạn

```
Given: User đã request recovery OTP
When:  User nhập OTP sau 10 phút 1 giây
Then:  HTTP 400
       Body: { "error": "otp_expired" }
```

---

## 7. Auth Flows

### 7.1 Authorization Code Flow + FIDO2

`GET /oauth2/authorize` là điểm tạo **server-side authentication transaction**. Sau khi validate `client_id`, `redirect_uri`, `scope`, `state`, PKCE và `nonce` (nếu dùng OIDC), Authorization Server tạo `auth_transaction_id` (ví dụ `atx_...`, entropy tối thiểu 128-bit), bind nó với `session_id` hiện tại và lưu toàn bộ OAuth context trong Redis. FIDO2 UI không được tự tái tạo OAuth field từ query string; mọi bước `/fido2/auth/begin` và `/fido2/auth/finish` phải tham chiếu lại transaction đã bind.

```
Client                  Auth Server              FIDO2 Engine         Redis                 Authenticator
  │                          │                        │                  │                         │
  │── GET /oauth2/authorize ─►│                        │                  │                         │
  │   (client_id, scope,      │                        │                  │                         │
  │    redirect_uri, PKCE,    │                        │                  │                         │
  │    state, nonce?)         │                        │                  │                         │
  │                           │── Validate OAuth req ─►│                  │                         │
  │                           │── Create auth_transaction_id + session_id │                         │
  │                           │── Store oauth ctx + transaction ─────────►│ TTL 5m                  │
  │◄── 302 FIDO2 UI + Set-Cookie __Host-auth_tx ───────│                  │                         │
  │    (optional tx_ref)      │                        │                  │                         │
  │                           │                        │                  │                         │
  │── POST /fido2/auth/begin ─►│                        │                  │                         │
  │   (same cookie, optional  │── Validate transaction/session binding ──►│                         │
  │    auth_transaction_id)   │── Create FIDO2 challenge ────────────────►│ same auth_tx, TTL 5m    │
  │◄── PublicKeyCredentialReq + auth_transaction_id ───│                  │                         │
  │                           │                        │                  │                         │
  │── navigator.credentials.get() ───────────────────────────────────────────────────────────────►│
  │                           │                        │                  │          Sign(challenge, │
  │                           │                        │                  │          private_key)    │
  │◄──────────────────────────────────────────────────────────────────────── AssertionResponse ───│
  │                           │                        │                  │                         │
  │── POST /fido2/auth/finish ─►│                        │                  │                         │
  │   (assertion + same       │── Load bound transaction/challenge ──────►│                         │
  │    transaction ref)       │── Reject if missing/expired/mismatch/used │                         │
  │                           │── Verify assertion ───►│                  │                         │
  │                           │   1. clientDataJSON type, challenge, origin                         │
  │                           │   2. authenticatorData rpIdHash, UV flag                            │
  │                           │   3. sign_count > stored_count                                       │
  │                           │   4. verify(pubkey, sig, authData||clientHash)                       │
  │                           │◄── VERIFIED ───────────│                  │                         │
  │                           │── Update sign_count ──►│(DB)              │                         │
  │                           │── Issue auth_code only for bound tx ─────►│ auth_code copies bound OAuth ctx
  │                           │── Mark tx used/delete challenge ────────►│ anti-replay             │
  │◄── HTTP 302 + auth_code ──│                        │                  │                         │
  │                           │                        │                  │                         │
  │── POST /oauth2/token ─────►│                        │                  │                         │
  │   (code, code_verifier)   │── Verify code_verifier against copied PKCE                         │
  │                           │   SHA256(verifier) == code_challenge                                 │
  │◄── access_token (15m) ────│                        │                  │                         │
  │    refresh_token (7d)     │                        │                  │                         │
  │    id_token               │                        │                  │                         │
```

**Transaction binding rules:**

- `auth_transaction_id` được tạo duy nhất tại `GET /oauth2/authorize`; không chấp nhận ID do client tự sinh.
- Redis transaction phải lưu: `client_id`, `redirect_uri` đã validate/canonicalized, `scope`, `state`, `code_challenge`, `code_challenge_method`, `nonce` (nếu OIDC), `session_id`, `created_at`, `expires_at`, `status=pending`, và FIDO2 challenge metadata (`challenge_b64`, `rp_id`, `origin`, `allow_credentials`, `user_verification`, `challenge_created_at`).
- FIDO2 UI ưu tiên dùng cookie server-side `__Host-auth_tx`/`session_id` với `HttpOnly`, `Secure`, `SameSite=Lax` hoặc `Strict`. `auth_transaction_id` trong body/query chỉ là transaction reference để debug/mobile-deep-link và phải khớp với cookie/session đã bind.
- `/fido2/auth/begin` chỉ tạo FIDO2 auth challenge cho transaction `pending` còn hạn và cùng `session_id`; nếu transaction đã có challenge chưa hết hạn thì rotate challenge cũ và đánh dấu cũ là unusable.
- `/fido2/auth/finish` chỉ được issue authorization code khi assertion hợp lệ **và** transaction vẫn `pending`, cùng `session_id`, cùng `auth_transaction_id`, challenge chưa hết hạn và chưa dùng. Khi tạo `auth_code:{code}`, server copy OAuth field đã bind từ transaction (`client_id`, `redirect_uri`, `scope`, `state`, `code_challenge`, `code_challenge_method`, `nonce`) thay vì lấy từ request `/finish`.
- Sau khi issue code hoặc khi verification thất bại không thể retry an toàn, transaction/challenge phải được đánh dấu `used`/`failed` bằng thao tác atomic (ví dụ Redis Lua/SET NX state transition) rồi xóa hoặc giữ tombstone TTL ngắn để chặn replay.

**Reject cases bắt buộc:**

| Trường hợp | HTTP | Error | Ghi chú |
|---|---:|---|---|
| Thiếu cookie/session hoặc thiếu transaction reference khi endpoint yêu cầu | 400 | `missing_auth_transaction` | Không bắt đầu WebAuthn nếu không có transaction OAuth đã bind |
| `auth_transaction_id` không tồn tại hoặc TTL hết hạn | 400 | `auth_transaction_expired` | User phải restart từ `/oauth2/authorize` |
| Transaction không khớp `session_id`, cookie, `client_id` context hoặc challenge trong assertion | 400 | `auth_transaction_mismatch` | Log WARN, không issue code |
| Transaction/challenge đã `used`, `failed` hoặc có replay cùng assertion | 400 | `auth_transaction_replayed` | Không cho retry cùng challenge |
| `/finish` hợp lệ về FIDO2 nhưng transaction chưa được bind từ `/authorize` | 400 | `unbound_auth_transaction` | Không tạo `auth_code` |

### 7.2 FIDO2 Verification Logic (RFC W3C §7.2)

```
Input: assertionResponse = { id, rawId, response: { authenticatorData, clientDataJSON, signature, userHandle } }

Step 1: Parse clientDataJSON
  C = JSON.parse(base64url_decode(clientDataJSON))
  Assert C.type === "webauthn.get"
  Assert bytes_equal(base64url_decode(C.challenge), stored_challenge)  // byte compare, NOT string
  Assert C.origin === "https://yourdomain.com"

Step 2: Compute hash
  clientDataHash = SHA-256(clientDataJSON_bytes)

Step 3: Parse authenticatorData
  rpIdHash = authenticatorData[0:32]
  flags = authenticatorData[32]
  signCount = authenticatorData[33:37] (big-endian uint32)

Step 4: Verify authenticatorData
  Assert rpIdHash === SHA-256("yourdomain.com")
  Assert flags.UP === 1       // User Presence required
  Assert flags.UV === 1       // User Verification required (policy: required)
  
Step 5: Verify sign count
  If signCount > 0 AND signCount <= stored_sign_count:
    → REJECT: sign_count_anomaly
    → Suspend credential, alert security
  Else:
    → stored_sign_count = signCount  // update DB

Step 6: Verify signature
  verificationData = authenticatorData || clientDataHash
  publicKey = fetch_public_key(credential_id)  // from DB, COSE format
  Assert verify_signature(publicKey, signature, verificationData) === true

Step 7: On success
  → Update sign_count in DB
  → Atomically mark auth_tx as used and delete/expire FIDO2 challenge metadata
  → Issue authorization_code only from the bound auth_tx OAuth context
  → Persist auth_code:{code} with copied client_id, redirect_uri, scope, state, PKCE, nonce
```

### 7.3 Registration Ceremony

```
Step 1: POST /fido2/register/begin
  Server tạo challenge = random_bytes(32), lưu Redis key="reg:{session_id}", TTL=5m
  Trả về PublicKeyCredentialCreationOptions:
    - excludeCredentials: [tất cả credential_id của user]  // tránh đăng ký trùng
    - authenticatorSelection.residentKey = "required"       // discoverable credential
    - authenticatorSelection.userVerification = "required"
    - attestation = "none"                         // v1 không yêu cầu attestation statement

Step 2: Client gọi navigator.credentials.create(options)
  Authenticator tạo key pair, lưu private key trong authenticator
  Trả về attestationObject + clientDataJSON; với policy `none`, browser không cung cấp
  attestation statement/cert chain nào có thể dùng để chứng minh provenance phần cứng.

Step 3: POST /fido2/register/finish
  Server verify registration theo WebAuthn: challenge, origin, rpIdHash, type, UV flag,
  credential_id uniqueness, public_key_cose và thuật toán được phép.
  Server KHÔNG verify attestation statement, KHÔNG gọi FIDO MDS3, KHÔNG tin AAGUID/cert
  như bằng chứng provenance phần cứng trong v1.
  Server lưu: credential_id, public_key_cose, aaguid (diagnostic only), transports, backup_eligible, backup_state
  Server xóa challenge khỏi Redis
  Server gửi email xác nhận
```

---

## 8. API Contract (OpenAPI)

```yaml
openapi: 3.1.0
info:
  title: OAuth2 + FIDO2 Authorization Server
  version: 1.0.0
  description: |
    Authorization Server hỗ trợ OAuth2 Authorization Code + PKCE (RFC 6749, RFC 7636)
    và FIDO2/WebAuthn passkey authentication (W3C WebAuthn Level 2).
    
    Tất cả endpoint yêu cầu TLS 1.3.
    Base URL production: https://auth.yourcompany.com
    Base URL staging: https://auth-staging.yourcompany.com

servers:
  - url: https://auth.yourcompany.com
  - url: https://auth-staging.yourcompany.com

tags:
  - name: oauth2
  - name: fido2-registration
  - name: fido2-authentication
  - name: credentials
  - name: recovery
  - name: admin

components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
    IntrospectionAuth:
      type: oauth2
      description: |
        Confidential/service client authentication for Resource Servers.
        Use a client credential or service account access token carrying scope `tokens:introspect`;
        do not use an end-user access token to authenticate this endpoint.
      flows:
        clientCredentials:
          tokenUrl: /oauth2/token
          scopes:
            tokens:introspect: Validate and introspect access tokens for Resource Servers.
    AdminAuth:
      type: oauth2
      description: Admin bearer token; accepted on introspection only as an explicit admin override.
      flows:
        clientCredentials:
          tokenUrl: /oauth2/token
          scopes:
            admin:write: Administrative write access.

  schemas:
    Error:
      type: object
      required: [error]
      properties:
        error: { type: string }
        message: { type: string }
        request_id: { type: string, format: uuid }

    # --- Registration ---
    RegisterBeginRequest:
      type: object
      required: [user_id]
      properties:
        user_id: { type: string, format: uuid }
        username: { type: string, example: user@company.com }
        display_name: { type: string, example: Nguyễn Văn A }

    RegisterBeginResponse:
      type: object
      properties:
        challenge: { type: string, format: base64url, description: 32-byte random, TTL 5m }
        rp:
          type: object
          properties:
            id: { type: string, example: yourcompany.com }
            name: { type: string, example: Your Company }
        user:
          type: object
          properties:
            id: { type: string, format: base64url }
            name: { type: string }
            displayName: { type: string }
        pubKeyCredParams:
          type: array
          items:
            type: object
            properties:
              type: { type: string, enum: [public-key] }
              alg: { type: integer, description: "ES256=-7, RS256=-257" }
        timeout: { type: integer, example: 300000, description: 5 phút }
        excludeCredentials:
          type: array
          items:
            type: object
            properties:
              id: { type: string, format: base64url }
              type: { type: string, enum: [public-key] }
              transports:
                type: array
                items: { type: string, enum: [internal, hybrid, usb, nfc, ble] }
        authenticatorSelection:
          type: object
          properties:
            userVerification: { type: string, enum: [required] }
            residentKey: { type: string, enum: [required] }
        attestation: { type: string, enum: [none], default: none, description: "v1 dùng attestation conveyance none; server không validate FIDO MDS3 hoặc dùng attestation statement làm bằng chứng provenance phần cứng" }

    RegisterFinishRequest:
      type: object
      required: [id, rawId, type, response]
      properties:
        id: { type: string, format: base64url }
        rawId: { type: string, format: base64url }
        type: { type: string, enum: [public-key] }
        response:
          type: object
          required: [attestationObject, clientDataJSON]
          properties:
            attestationObject: { type: string, format: base64url }
            clientDataJSON: { type: string, format: base64url }
            transports:
              type: array
              items: { type: string, enum: [internal, hybrid, usb, nfc, ble] }

    RegisterFinishResponse:
      type: object
      properties:
        credential_id: { type: string, format: base64url }
        message: { type: string, example: Passkey đã được thêm thành công }

    # --- Authentication ---
    AuthBeginRequest:
      type: object
      description: |
        FIDO2 UI phải gọi endpoint này trong cùng server-side browser session đã được tạo bởi
        GET /oauth2/authorize. Cookie HttpOnly/SameSite là nguồn bind chính; auth_transaction_id
        chỉ là transaction reference rõ ràng cho UI/native client và phải khớp với session.
      properties:
        auth_transaction_id:
          type: string
          pattern: '^atx_[A-Za-z0-9_-]{22,}$'
          description: Optional khi cookie __Host-auth_tx hiện diện; bắt buộc cho native/deep-link flow.
        user_id: { type: string, format: uuid, description: Để trống nếu dùng discoverable credential }

    AuthBeginResponse:
      type: object
      properties:
        auth_transaction_id:
          type: string
          description: Server-issued transaction reference đã bind với session; echo lại để UI gửi /finish khi cần.
        challenge: { type: string, format: base64url }
        timeout: { type: integer, example: 300000 }
        rpId: { type: string, example: yourcompany.com }
        allowCredentials:
          type: array
          items:
            type: object
            properties:
              id: { type: string, format: base64url }
              type: { type: string, enum: [public-key] }
              transports: { type: array, items: { type: string } }
        userVerification: { type: string, enum: [required] }

    AuthFinishRequest:
      type: object
      required: [id, rawId, type, response]
      description: |
        Hoàn tất assertion trong transaction đã bind. Server chỉ issue authorization code nếu
        auth_transaction_id/cookie/session khớp với transaction pending và challenge chưa dùng.
      properties:
        auth_transaction_id:
          type: string
          pattern: '^atx_[A-Za-z0-9_-]{22,}$'
          description: Optional khi cookie __Host-auth_tx đủ để lookup; nếu truyền thì phải khớp tuyệt đối.
        id: { type: string, format: base64url }
        rawId: { type: string, format: base64url }
        type: { type: string, enum: [public-key] }
        response:
          type: object
          required: [authenticatorData, clientDataJSON, signature]
          properties:
            authenticatorData: { type: string, format: base64url }
            clientDataJSON: { type: string, format: base64url }
            signature: { type: string, format: base64url }
            userHandle: { type: string, format: base64url }

    # --- Credential ---
    CredentialInfo:
      type: object
      properties:
        credential_id: { type: string, description: Masked - last 8 chars only }
        aaguid: { type: string, format: uuid }
        transports: { type: array, items: { type: string } }
        backup_eligible: { type: boolean }
        backup_state: { type: boolean }
        created_at: { type: string, format: date-time }
        last_used_at: { type: string, format: date-time }

    # --- Token ---
    TokenRequest:
      type: object
      required: [grant_type]
      properties:
        grant_type: { type: string, enum: [authorization_code, refresh_token] }
        code: { type: string }
        redirect_uri: { type: string, format: uri }
        code_verifier: { type: string, minLength: 43, maxLength: 128 }
        client_id: { type: string }
        refresh_token: { type: string }

    TokenResponse:
      type: object
      properties:
        access_token: { type: string }
        token_type: { type: string, enum: [Bearer] }
        expires_in: { type: integer, example: 900, description: 15 phút }
        refresh_token: { type: string }
        scope: { type: string }
        id_token: { type: string }

    # --- Recovery ---
    RecoveryBeginRequest:
      type: object
      required: [email]
      properties:
        email: { type: string, format: email }

    RecoveryVerifyRequest:
      type: object
      required: [email, otp]
      properties:
        email: { type: string, format: email }
        otp: { type: string, pattern: '^\d{6}$' }

paths:
  /fido2/register/begin:
    post:
      summary: Bắt đầu đăng ký passkey
      operationId: fido2RegisterBegin
      tags: [fido2-registration]
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/RegisterBeginRequest' }
      responses:
        '200':
          description: PublicKeyCredentialCreationOptions
          content:
            application/json:
              schema: { $ref: '#/components/schemas/RegisterBeginResponse' }
        '400': { description: Request không hợp lệ }
        '403':
          description: Email chưa xác minh
          content:
            application/json:
              example: { error: email_not_verified, message: Vui lòng xác minh email trước }
        '429':
          description: Rate limit
          headers:
            Retry-After: { schema: { type: integer } }

  /fido2/register/finish:
    post:
      summary: Hoàn tất đăng ký passkey
      operationId: fido2RegisterFinish
      tags: [fido2-registration]
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/RegisterFinishRequest' }
      responses:
        '200':
          description: Passkey đã được lưu
          content:
            application/json:
              schema: { $ref: '#/components/schemas/RegisterFinishResponse' }
        '400':
          description: Challenge hết hạn hoặc WebAuthn registration response không hợp lệ
          content:
            application/json:
              examples:
                expired: { value: { error: challenge_expired } }
                duplicate: { value: { error: credential_already_registered } }

  /fido2/auth/begin:
    post:
      summary: Bắt đầu xác thực bằng passkey
      operationId: fido2AuthBegin
      tags: [fido2-authentication]
      requestBody:
        content:
          application/json:
            schema: { $ref: '#/components/schemas/AuthBeginRequest' }
      responses:
        '200':
          description: PublicKeyCredentialRequestOptions kèm transaction reference đã bind
          content:
            application/json:
              schema: { $ref: '#/components/schemas/AuthBeginResponse' }
        '400':
          description: Transaction thiếu, hết hạn, không khớp hoặc replay
          content:
            application/json:
              examples:
                missing_tx: { value: { error: missing_auth_transaction } }
                expired_tx: { value: { error: auth_transaction_expired } }
                mismatch_tx: { value: { error: auth_transaction_mismatch } }
                replayed_tx: { value: { error: auth_transaction_replayed } }
        '429':
          description: Rate limit — 5 lần/phút/IP
          headers:
            Retry-After: { schema: { type: integer } }

  /fido2/auth/finish:
    post:
      summary: Hoàn tất xác thực, nhận authorization_code
      operationId: fido2AuthFinish
      tags: [fido2-authentication]
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/AuthFinishRequest' }
      responses:
        '302':
          description: |
            Redirect về client kèm authorization_code. Code chỉ được issue cho transaction đã bind;
            auth_code:{code} phải copy client_id, redirect_uri, scope, state, PKCE và nonce từ Redis transaction.
          headers:
            Location: { schema: { type: string, format: uri } }
        '400':
          description: Origin mismatch, challenge expired, UV=0, hoặc auth transaction không hợp lệ
          content:
            application/json:
              examples:
                origin: { value: { error: origin_mismatch } }
                expired: { value: { error: challenge_expired } }
                uv: { value: { error: user_verification_required } }
                missing_tx: { value: { error: missing_auth_transaction } }
                expired_tx: { value: { error: auth_transaction_expired } }
                mismatch_tx: { value: { error: auth_transaction_mismatch } }
                replayed_tx: { value: { error: auth_transaction_replayed } }
                unbound_tx: { value: { error: unbound_auth_transaction } }
        '401':
          description: Signature không hợp lệ hoặc sign_count anomaly
          content:
            application/json:
              examples:
                sig: { value: { error: invalid_signature } }
                counter: { value: { error: sign_count_anomaly } }
        '403':
          description: Insufficient assurance (synced passkey cho high-assurance scope)
          content:
            application/json:
              example: { error: insufficient_assurance }

  /fido2/credentials:
    get:
      summary: Lấy danh sách passkey của user
      operationId: listCredentials
      tags: [credentials]
      security: [{ BearerAuth: [] }]
      responses:
        '200':
          description: Danh sách credential
          content:
            application/json:
              schema:
                type: array
                items: { $ref: '#/components/schemas/CredentialInfo' }
        '401': { description: Token không hợp lệ }

  /fido2/credentials/{credential_id}:
    delete:
      summary: Xóa một passkey
      operationId: deleteCredential
      tags: [credentials]
      security: [{ BearerAuth: [] }]
      parameters:
        - name: credential_id
          in: path
          required: true
          schema: { type: string, format: base64url }
      responses:
        '200': { description: Đã xóa thành công }
        '404': { description: Credential không tồn tại hoặc không thuộc về user }
        '409':
          description: Passkey cuối cùng, không thể xóa
          content:
            application/json:
              example: { error: last_credential }

  /oauth2/authorize:
    get:
      summary: Bắt đầu Authorization Code flow
      operationId: authorize
      tags: [oauth2]
      parameters:
        - { name: response_type, in: query, required: true, schema: { type: string, enum: [code] } }
        - { name: client_id, in: query, required: true, schema: { type: string } }
        - { name: redirect_uri, in: query, required: true, schema: { type: string, format: uri } }
        - { name: scope, in: query, required: true, schema: { type: string, example: "openid profile email" } }
        - { name: state, in: query, required: true, schema: { type: string, minLength: 16 } }
        - { name: code_challenge, in: query, required: true, schema: { type: string, format: base64url } }
        - { name: code_challenge_method, in: query, required: true, schema: { type: string, enum: [S256] } }
        - { name: nonce, in: query, required: false, schema: { type: string, minLength: 16 }, description: Bắt buộc khi scope chứa openid nếu policy OIDC yêu cầu nonce }
      responses:
        '302':
          description: |
            Redirect đến FIDO2 authentication UI sau khi tạo auth_transaction_id server-side,
            bind với session_id và lưu OAuth context đã validate trong Redis.
          headers:
            Location:
              schema: { type: string, format: uri }
              description: FIDO2 UI URL; có thể chứa tx_ref ngắn, không chứa OAuth secret/context.
            Set-Cookie:
              schema: { type: string }
              description: __Host-auth_tx=<opaque>; HttpOnly; Secure; SameSite=Lax hoặc Strict; TTL 5m.
        '400': { description: Request không hợp lệ (missing param, invalid redirect_uri) }

  /oauth2/token:
    post:
      summary: Đổi authorization_code lấy token
      operationId: token
      tags: [oauth2]
      requestBody:
        required: true
        content:
          application/x-www-form-urlencoded:
            schema: { $ref: '#/components/schemas/TokenRequest' }
      responses:
        '200':
          description: Token response
          content:
            application/json:
              schema: { $ref: '#/components/schemas/TokenResponse' }
        '400':
          description: Grant không hợp lệ
          content:
            application/json:
              examples:
                invalid_code: { value: { error: invalid_grant } }
                invalid_verifier: { value: { error: invalid_client } }

  /oauth2/introspect:
    post:
      summary: Validate token (dùng cho Resource Server)
      description: |
        Resource Server MUST authenticate as a confidential/service client.
        Required scope is `tokens:introspect`; `admin:write` is accepted only for Admin clients that need
        to call this endpoint. End-user access tokens MUST NOT be used as the caller credential.
      operationId: introspect
      tags: [oauth2]
      security:
        - IntrospectionAuth: [tokens:introspect]
        - AdminAuth: [admin:write]
      requestBody:
        required: true
        content:
          application/x-www-form-urlencoded:
            schema:
              type: object
              properties:
                token: { type: string }
      responses:
        '200':
          description: Token info
          content:
            application/json:
              schema:
                type: object
                properties:
                  active: { type: boolean }
                  sub: { type: string, format: uuid }
                  scope: { type: string }
                  exp: { type: integer }
                  amr: { type: array, items: { type: string } }
        '401':
          description: Missing or invalid Resource Server credential
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Error' }
              examples:
                missing_credential:
                  value:
                    error: invalid_client
                    message: Missing bearer token or client authentication for token introspection.
                    request_id: 018f9d0a-91b4-7cc6-a5d2-4b0ed2b7159d
        '403':
          description: Caller is authenticated but lacks the required introspection scope
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Error' }
              examples:
                missing_scope:
                  value:
                    error: insufficient_scope
                    message: Required scope `tokens:introspect` or admin override scope `admin:write`.
                    request_id: 018f9d0b-2254-73a5-9c15-3bb7dcda6e62

  /oauth2/revoke:
    post:
      summary: Revoke token
      operationId: revoke
      tags: [oauth2]
      security: [{ BearerAuth: [] }]
      responses:
        '200': { description: Token đã bị revoke }

  /.well-known/oauth-authorization-server:
    get:
      summary: OAuth2 Server Metadata (RFC 8414)
      tags: [oauth2]
      responses:
        '200': { description: Server metadata }

  /.well-known/jwks.json:
    get:
      summary: JSON Web Key Set — public keys để verify JWT
      tags: [oauth2]
      responses:
        '200': { description: JWKS }

  /auth/recovery/begin:
    post:
      summary: Bắt đầu account recovery
      operationId: recoveryBegin
      tags: [recovery]
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/RecoveryBeginRequest' }
      responses:
        '200': { description: OTP đã được gửi (luôn trả 200 dù email không tồn tại — tránh enumeration) }
        '429': { description: Rate limit — 3 lần/giờ/email }

  /auth/recovery/verify:
    post:
      summary: Verify OTP, nhận recovery session token
      operationId: recoveryVerify
      tags: [recovery]
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/RecoveryVerifyRequest' }
      responses:
        '200':
          description: Recovery token để tiếp tục đăng ký passkey mới
          content:
            application/json:
              schema:
                type: object
                properties:
                  recovery_token: { type: string, description: Short-lived token, TTL 15m }
        '400': { description: OTP sai hoặc hết hạn }
        '429': { description: Max OTP attempts — 3 lần }

  /admin/users/{user_id}/credentials:
    delete:
      summary: Admin revoke tất cả credential của user
      operationId: adminRevokeAllCredentials
      tags: [admin]
      security: [{ AdminAuth: [admin:write] }]
      parameters:
        - name: user_id
          in: path
          required: true
          schema: { type: string, format: uuid }
      responses:
        '200': { description: Tất cả credential đã bị revoke }
        '403': { description: Không đủ quyền }
        '404': { description: User không tồn tại }
```

---

## 9. Database Schema

```sql
-- =====================================================
-- Users
-- =====================================================
CREATE TABLE users (
  user_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email            TEXT UNIQUE NOT NULL,
  display_name     TEXT,
  email_verified   BOOLEAN DEFAULT false,
  created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  mfa_required     BOOLEAN DEFAULT true,
  account_status   TEXT NOT NULL DEFAULT 'active'
                   CHECK (account_status IN ('active', 'suspended', 'recovering'))
);

CREATE INDEX idx_users_email ON users(email);

-- =====================================================
-- FIDO2 Credentials (Passkeys)
-- =====================================================
CREATE TABLE fido2_credentials (
  credential_id        BYTEA PRIMARY KEY,
  user_id              UUID NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
  public_key_cose      BYTEA NOT NULL,
  sign_count           BIGINT NOT NULL DEFAULT 0,
  aaguid               UUID,
  transports           TEXT[],          -- ['internal','hybrid','usb','nfc','ble']
  backup_eligible      BOOLEAN NOT NULL DEFAULT false,
  backup_state         BOOLEAN NOT NULL DEFAULT false,
  attestation_format   TEXT,            -- v1 expected 'none'; diagnostic only, không dùng làm provenance
  status               TEXT NOT NULL DEFAULT 'active'
                       CHECK (status IN ('active', 'suspended', 'revoked')),
  created_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
  last_used_at         TIMESTAMPTZ,
  suspended_at         TIMESTAMPTZ,
  suspend_reason       TEXT
);

CREATE INDEX idx_fido2_user ON fido2_credentials(user_id);
CREATE INDEX idx_fido2_status ON fido2_credentials(user_id, status);

-- Business rule cho backup_eligible / backup_state:
-- backup_eligible = true: Authenticator hỗ trợ sync (ví dụ: iCloud Keychain)
-- backup_state = true: Passkey hiện đang được sync lên cloud
-- Khi backup_state = true VÀ scope yêu cầu high-assurance (admin:write):
--   → Từ chối auth, yêu cầu hardware-bound passkey

-- =====================================================
-- OAuth2 Clients
-- =====================================================
CREATE TABLE oauth2_clients (
  client_id                    TEXT PRIMARY KEY,
  client_secret_hash           TEXT,    -- NULL nếu public client
  redirect_uris                TEXT[] NOT NULL,
  scopes                       TEXT[] NOT NULL,
  grant_types                  TEXT[] NOT NULL DEFAULT ARRAY['authorization_code'],
  token_endpoint_auth_method   TEXT NOT NULL DEFAULT 'none',
                               -- 'none' = public client, 'client_secret_post' = confidential
  require_pkce                 BOOLEAN NOT NULL DEFAULT true,
  access_token_ttl_seconds     INTEGER NOT NULL DEFAULT 900,    -- 15 phút
  refresh_token_ttl_seconds    INTEGER NOT NULL DEFAULT 604800, -- 7 ngày
  created_at                   TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- =====================================================
-- Refresh Tokens
-- =====================================================
CREATE TABLE refresh_tokens (
  token_hash       TEXT PRIMARY KEY,  -- SHA-256 của token gốc
  user_id          UUID NOT NULL REFERENCES users(user_id),
  client_id        TEXT NOT NULL REFERENCES oauth2_clients(client_id),
  scope            TEXT NOT NULL,
  family_id        UUID NOT NULL DEFAULT gen_random_uuid(),
                   -- Tất cả token từ cùng một auth session dùng chung family_id
                   -- Nếu token đã revoked được dùng lại → revoke toàn bộ family
  issued_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  expires_at       TIMESTAMPTZ NOT NULL,
  revoked          BOOLEAN NOT NULL DEFAULT false,
  revoked_at       TIMESTAMPTZ,
  replaced_by      TEXT   -- token_hash của token thay thế (rotation)
);

CREATE INDEX idx_rt_user ON refresh_tokens(user_id);
CREATE INDEX idx_rt_family ON refresh_tokens(family_id);
CREATE INDEX idx_rt_expires ON refresh_tokens(expires_at) WHERE NOT revoked;

-- =====================================================
-- Audit Log
-- =====================================================
CREATE TABLE auth_audit_log (
  log_id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  event_type       TEXT NOT NULL,
                   -- 'fido2.register.begin' | 'fido2.register.finish'
                   -- 'fido2.auth.begin' | 'fido2.auth.finish'
                   -- 'fido2.auth.failed' | 'token.issued' | 'token.revoked'
                   -- 'credential.deleted' | 'credential.suspended'
                   -- 'account.recovery.begin' | 'account.recovery.complete'
                   -- 'admin.credential.revoke'
  user_id          UUID REFERENCES users(user_id),
  client_id        TEXT,
  credential_id    TEXT,  -- Masked: last 8 chars only
  ip_address       INET NOT NULL,
  user_agent       TEXT,
  result           TEXT NOT NULL CHECK (result IN ('success', 'failure')),
  error_code       TEXT,
  metadata         JSONB,  -- extra context, không chứa PII
  created_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_user ON auth_audit_log(user_id, created_at DESC);
CREATE INDEX idx_audit_event ON auth_audit_log(event_type, created_at DESC);
-- Retention policy: partition by month, drop partitions > 12 tháng

-- =====================================================
-- Redis Keys (không phải DB table — documentation only)
-- =====================================================
-- Key: "challenge:reg:{session_id}"
--   Value: { challenge_b64, user_id, created_at, rp_id, origin }
--   TTL: 300s
--
-- Key: "auth_tx:{auth_transaction_id}"
--   Value: {
--     session_id, status: "pending" | "used" | "failed", created_at, expires_at,
--     client_id, redirect_uri_validated, scope, state,
--     code_challenge, code_challenge_method, nonce,
--     fido2: {
--       challenge_b64, challenge_created_at, rp_id, origin,
--       allow_credentials, user_verification, timeout_ms
--     }
--   }
--   TTL: 300s while pending; tombstone TTL 60s after used/failed to reject replay
--
-- Key: "challenge:auth:{session_id}"
--   Deprecated: chỉ dùng migration. Auth challenge mới phải nằm trong auth_tx:{auth_transaction_id}
--   để bind OAuth context + FIDO2 challenge atomically. TTL: 300s
--
-- Key: "auth_code:{code}"
--   Value: {
--     user_id, client_id, redirect_uri, scope, state,
--     code_challenge, code_challenge_method, nonce,
--     auth_transaction_id, session_id, amr: ["fido2"], issued_at
--   }
--   TTL: 600s; tất cả OAuth field phải copy từ auth_tx đã bind, không lấy từ /fido2/auth/finish
--
-- Key: "recovery_otp:{email_hash}"     Value: { otp_hash, attempts }      TTL: 600s
-- Key: "recovery_token:{token_hash}"   Value: { user_id }                 TTL: 900s
-- Key: "session:{session_id}"          Value: { user_id, created_at, auth_transaction_id? } TTL: 3600s
```

---

## 10. Permission & Role Model

### Roles

| Role | Scope | Mô tả |
|---|---|---|
| End user | `openid profile email fido2:register fido2:auth credentials:read credentials:delete offline_access` | Người dùng cuối |
| Support T2 | `support:read credentials:read audit:read:own` | Nhân viên hỗ trợ tier 2 |
| Admin | `admin:read admin:write credentials:revoke tokens:revoke oauth2:clients:manage audit:read` + optional `admin:write` override for `/oauth2/introspect` | Quản trị hệ thống |
| Service account | `tokens:introspect` | Machine-to-machine Resource Server client |

**Introspection rule:** `/oauth2/introspect` requires caller scope `tokens:introspect`. Resource Servers MUST call it with a confidential/service client credential, not an end-user access token. `admin:write` is also accepted only as an Admin override when operators need to introspect tokens.

### Ma trận quyền

| Endpoint | User | Support | Admin | Service |
|---|:---:|:---:|:---:|:---:|
| POST /fido2/register/begin | ✓ | ✗ | ✗ | ✗ |
| POST /fido2/register/finish | ✓ | ✗ | ✗ | ✗ |
| POST /fido2/auth/begin | ✓ | ✗ | ✗ | ✗ |
| POST /fido2/auth/finish | ✓ | ✗ | ✗ | ✗ |
| GET /fido2/credentials | ✓ own | ✓ read-only | ✓ any | ✗ |
| DELETE /fido2/credentials/:id | ✓ own, ≥2 left | ✗ | ✓ any | ✗ |
| DELETE /admin/users/:id/credentials | ✗ | ✗ | ✓ | ✗ |
| GET /oauth2/authorize | ✓ | ✗ | ✗ | ✗ |
| POST /oauth2/token | ✓ | ✗ | ✗ | ✗ |
| POST /oauth2/introspect | ✗ | ✗ | ✓ (`admin:write` override) | ✓ (`tokens:introspect`) |
| POST /oauth2/revoke | ✓ own | ✗ | ✓ any | ✗ |
| GET /.well-known/jwks.json | ✓ | ✓ | ✓ | ✓ |
| GET /admin/users/:id | ✗ | ✓ | ✓ | ✗ |
| POST /admin/oauth2/clients | ✗ | ✗ | ✓ | ✗ |
| GET /admin/audit-logs | ✗ | ✓ own-user | ✓ | ✗ |
| POST /auth/recovery/* | ✓ | ✗ | ✗ | ✗ |

---

## 11. Security Requirements

### Transport

- TLS 1.3 bắt buộc cho mọi endpoint; TLS 1.2 chỉ chấp nhận với cipher suite ECDHE+AES-GCM
- HSTS: `max-age=31536000; includeSubDomains; preload`
- Certificate pinning cho mobile client (SHA-256 pin của leaf cert)

### FIDO2 Policy

| Setting | Giá trị | Lý do |
|---|---|---|
| `userVerification` | `required` | Bắt buộc PIN/biometric, không chỉ user presence |
| `rpId` | `yourcompany.com` | Không wildcard subdomain |
| `timeout` | `300000` ms | 5 phút — đủ cho user thao tác |
| `attestation` | `none` | v1 không thu thập/verify attestation statement, không gọi FIDO MDS3, và không dùng AAGUID/cert chain làm bằng chứng provenance phần cứng |
| `residentKey` | `required` | Discoverable credential — login không cần nhập username |
| Hardware provenance | Không được chứng minh bằng attestation trong v1 | Scope high-assurance như `admin:write` không được grant chỉ dựa trên attestation statement; cần policy bổ sung hoặc Phase 2 MDS3 |

### Rate Limiting

| Endpoint | Limit | Window | Key |
|---|---|---|---|
| POST /fido2/auth/begin | 5 req | 60s | IP |
| POST /fido2/auth/finish | 10 req | 60s | user_id |
| POST /fido2/register/begin | 10 req | 60s | IP |
| POST /oauth2/token | 20 req | 60s | client_id |
| POST /auth/recovery/begin | 3 req | 3600s | email_hash |
| POST /auth/recovery/verify | 3 req | 600s | email_hash |

### PKCE

- Chỉ chấp nhận `S256` — từ chối `plain`
- `code_verifier`: 43–128 ký tự, alphabet `[A-Z a-z 0-9 - . _ ~]`
- Challenge lưu Redis TTL 10 phút, xóa ngay sau khi dùng

### Session & Challenge

- Challenge xóa ngay sau verify (dù pass hay fail)
- Challenge bind với `session_id` của browser — không accept challenge từ session khác
- Không lưu challenge in-memory — phải dùng Redis để support rolling deploy

### Token

```
Access token:
  Format: JWT (ES256 preferred, RS256 fallback)
  TTL: 900 giây (15 phút)
  Claims bắt buộc: sub, iss, aud, exp, iat, jti, scope, amr=["fido2"]
  Signing key: P-256 hoặc RSA-2048, rotate mỗi 90 ngày
  Key ID (kid) trong header để Resource Server biết dùng key nào từ JWKS

Refresh token:
  Format: 32 bytes random, lưu dưới dạng SHA-256 hash trong DB
  TTL: 7 ngày sliding, absolute max 30 ngày
  Rotation: mỗi lần dùng → tạo token mới (cùng family_id), revoke token cũ
  Reuse detection: token đã revoked được dùng lại → revoke toàn bộ family_id

ID token (OpenID Connect):
  Claims bổ sung: name, email, email_verified, amr
```

### Audit & Monitoring

- Mọi auth event phải có log với: `event_type, user_id (nếu có), ip, user_agent, result, error_code, timestamp (UTC)`
- Không log: `clientDataJSON` raw, `public_key_cose`, token value
- Credential ID trong log: masked (last 8 chars only)
- Retention: 12 tháng, không thể xóa từ app layer
- Alert ngay lập tức khi: `sign_count_anomaly`, `5xx rate > 0.1%/5min`, `recovery attempts spike`

### SAST / DAST

- Semgrep (SAST) chạy mỗi PR — block merge nếu có Critical
- OWASP ZAP (DAST) chạy mỗi release — block nếu có High unfixed
- GitGuardian scan mọi commit — không có secret trong source code

---

## 12. Non-Functional Requirements

### Performance

| ID | Yêu cầu | Metric | Mức | Đo bằng |
|---|---|---|---|---|
| NFR-P-01 | Latency FIDO2 auth | p50 < 80ms, p99 < 300ms | MUST | Datadog APM |
| NFR-P-02 | Latency /oauth2/token | p50 < 50ms, p99 < 150ms | MUST | Datadog APM |
| NFR-P-03 | Throughput | 500 auth/s peak, 100 reg/s peak | MUST | k6 load test |
| NFR-P-04 | Redis challenge lookup | p99 < 5ms | SHOULD | Redis MONITOR |

### Availability & Reliability

| ID | Yêu cầu | Metric | Mức | Đo bằng |
|---|---|---|---|---|
| NFR-A-01 | Uptime SLA | 99.9%/tháng (≤43 phút downtime) | MUST | Uptime Robot |
| NFR-A-02 | Error rate | 5xx < 0.1% trong 5 phút sliding window | MUST | Datadog monitor |
| NFR-A-03 | Graceful degradation | Redis down → 503 + Retry-After trong < 2s | MUST | Chaos test |
| NFR-A-04 | Zero-downtime deploy | Rolling deploy, không drop request | MUST | Deploy pipeline |

### Security NFR

| ID | Yêu cầu | Metric | Mức | Đo bằng |
|---|---|---|---|---|
| NFR-S-01 | Key rotation | JWT signing key rotate mỗi 90 ngày | MUST | HSM audit log |
| NFR-S-02 | Pentest | 1 lần/năm, 0 Critical/High unfixed khi go-live | MUST | Pentest report |
| NFR-S-03 | Audit retention | 12 tháng, không thể xóa từ app | MUST | Splunk policy |
| NFR-S-04 | SAST/DAST | 0 Critical, < 3 High merged vào main | MUST | CI gate |
| NFR-S-05 | Secret scanning | 0 secret trong code hoặc log | MUST | GitGuardian |

### Compatibility & Scalability

| ID | Yêu cầu | Metric | Mức | Đo bằng |
|---|---|---|---|---|
| NFR-C-01 | Browser | Chrome 108+, Safari 16+, Firefox 119+, Edge 108+ | MUST | BrowserStack |
| NFR-C-02 | Mobile | iOS 16+, Android 9+ | MUST | Device lab |
| NFR-C-03 | Horizontal scaling | 2 → 10 instance trong < 3 phút khi CPU > 70% | MUST | k8s HPA |
| NFR-C-04 | DB connection | Max 20 conn/instance, PgBouncer, timeout 30s | SHOULD | PgBouncer stats |
| NFR-C-05 | Observability | 100% request có trace_id (OpenTelemetry) | SHOULD | Datadog |

---

## 13. UI/UX Behavior Spec

> Phần này dành cho Frontend team và QA. Backend không implement UI nhưng cần biết các state để trả đúng error code.

### 13.1 Registration Flow — UI States

| State | Trigger | UI behavior |
|---|---|---|
| `idle` | Trang settings load | Hiển thị nút "Thêm passkey" nếu device hỗ trợ WebAuthn; hiển thị danh sách passkey hiện có |
| `checking_support` | User click "Thêm passkey" | Gọi `PublicKeyCredential.isUserVerifyingPlatformAuthenticatorAvailable()` |
| `not_supported` | Browser/device không support | Hiển thị message: "Thiết bị này không hỗ trợ passkey. Vui lòng dùng thiết bị khác." Không show nút thêm. |
| `loading` | Đang gọi /register/begin | Spinner trên nút, disabled nút để tránh double-click |
| `authenticating` | navigator.credentials.create() đang chạy | Native browser/OS dialog hiển thị. FE không can thiệp. |
| `user_cancelled` | User dismiss native dialog | Hiển thị toast: "Đăng ký đã bị hủy" — không error |
| `verifying` | Đang gọi /register/finish | Spinner với text "Đang xác minh..." |
| `success` | HTTP 200 từ /register/finish | Toast success, danh sách passkey refresh, email confirmation note |
| `error_duplicate` | `credential_already_registered` | "Thiết bị này đã được đăng ký. Vui lòng dùng thiết bị khác." |
| `error_expired` | `challenge_expired` | "Phiên đăng ký hết hạn. Vui lòng thử lại." + tự động restart flow |
| `error_generic` | Lỗi khác | "Có lỗi xảy ra. Mã lỗi: {request_id}" + link "Liên hệ hỗ trợ" |

### 13.2 Authentication Flow — UI States

| State | Trigger | UI behavior |
|---|---|---|
| `idle` | Trang login load | Hiển thị nút "Đăng nhập bằng passkey" nếu có passkey cho domain này |
| `auto_fill_available` | Browser detect discoverable credential | Browser hiển thị auto-fill suggestion tự động (không cần user action) |
| `loading` | User click nút | Spinner, gọi /auth/begin |
| `authenticating` | navigator.credentials.get() chạy | Native dialog. FE không can thiệp. |
| `user_cancelled` | User dismiss | Toast: "Đăng nhập đã bị hủy" |
| `verifying` | Gọi /auth/finish | Spinner "Đang xác thực..." |
| `success` | HTTP 302 | Redirect theo Location header |
| `error_invalid` | `invalid_signature` | "Xác thực không thành công. Vui lòng thử lại." (không nói lý do cụ thể — tránh lộ info) |
| `error_anomaly` | `sign_count_anomaly` | "Phát hiện hoạt động bất thường. Tài khoản của bạn đã bị tạm khóa bảo mật. Vui lòng liên hệ hỗ trợ." |
| `error_rate_limit` | HTTP 429 | "Quá nhiều lần thử. Vui lòng chờ {Retry-After} giây." + countdown timer |

### 13.3 Fallback khi WebAuthn không hỗ trợ

```
if (!window.PublicKeyCredential) {
  // Browser hoàn toàn không hỗ trợ WebAuthn
  → Hiển thị: "Trình duyệt này không hỗ trợ passkey.
               Vui lòng dùng Chrome 108+, Safari 16+, hoặc Firefox 119+."
  → Không show passkey option
}

if (!await PublicKeyCredential.isUserVerifyingPlatformAuthenticatorAvailable()) {
  // Device không có platform authenticator (Touch ID, Face ID, Windows Hello)
  → Vẫn show option đăng ký (user có thể dùng USB security key)
  → Hint text: "Bạn có thể dùng USB security key nếu thiết bị không có sinh trắc học"
}
```

### 13.4 Localization — Error Messages

| Error code | Tiếng Việt | English |
|---|---|---|
| `email_not_verified` | Vui lòng xác minh email trước khi đăng ký passkey | Please verify your email before registering a passkey |
| `credential_already_registered` | Thiết bị này đã được đăng ký | This device is already registered |
| `challenge_expired` | Phiên đăng ký hết hạn. Vui lòng thử lại | Session expired. Please try again |
| `invalid_signature` | Xác thực không thành công. Vui lòng thử lại | Authentication failed. Please try again |
| `sign_count_anomaly` | Phát hiện hoạt động bất thường. Tài khoản bị tạm khóa bảo mật | Suspicious activity detected. Account temporarily locked |
| `origin_mismatch` | Yêu cầu từ nguồn không hợp lệ | Request from invalid origin |
| `user_verification_required` | Thiết bị yêu cầu xác minh sinh trắc học hoặc PIN | Device requires biometric or PIN verification |
| `last_credential` | Không thể xóa passkey cuối. Thêm passkey mới trước | Cannot delete last passkey. Add a new one first |
| `rate_limit_exceeded` | Quá nhiều lần thử. Vui lòng chờ {seconds} giây | Too many attempts. Please wait {seconds} seconds |
| `insufficient_assurance` | Tính năng này yêu cầu passkey được lưu trên thiết bị vật lý | This feature requires a device-bound passkey |
| `otp_expired` | Mã xác minh đã hết hạn. Vui lòng yêu cầu mã mới | Verification code expired. Please request a new one |
| `max_otp_attempts` | Quá nhiều lần nhập sai. Vui lòng yêu cầu mã mới | Too many incorrect attempts. Please request a new code |

---

## 14. Account Recovery Flow

> **Critical path** — user bị mất tất cả passkey. Phải hoạt động nhưng không được là con đường bypass bảo mật.

### Điều kiện

- User có tài khoản với email verified
- Tất cả fido2_credentials bị mất/xóa/suspended
- User còn nhớ email đăng ký

### Flow chi tiết

```
Step 1: User truy cập /login → click "Không thể đăng nhập?"
Step 2: POST /auth/recovery/begin { email: "user@example.com" }
  Server LUÔN trả 200 (dù email tồn tại hay không — tránh email enumeration)
  Nếu email tồn tại: tạo OTP 6 số, TTL 10 phút, max 3 attempts
  Nếu email không tồn tại: không làm gì (silent)
  Rate limit: 3 request/giờ/email_hash

Step 3: User nhận email với OTP
  Subject: "Khôi phục tài khoản [Your Company]"
  Body: OTP 6 số + cảnh báo "Không chia sẻ mã này với ai"
  TTL: 10 phút

Step 4: POST /auth/recovery/verify { email, otp: "123456" }
  Nếu sai: attempts++; nếu attempts >= 3 → xóa OTP, yêu cầu request lại
  Nếu đúng: tạo recovery_token (32 bytes random, TTL 15 phút), return
  Audit log: { event: "account.recovery.otp.verified", user_id, ip }

Step 5: Dùng recovery_token để gọi /fido2/register/begin
  Header: Authorization: Bearer {recovery_token}
  Server: verify recovery_token, tiến hành registration ceremony bình thường

Step 6: Sau khi đăng ký passkey mới thành công:
  Xóa recovery_token
  Audit log: { event: "account.recovery.complete", user_id, ip }
  Email: "Tài khoản của bạn đã được khôi phục. Passkey mới đã được thêm."
```

### Security constraints của Recovery Flow

- OTP không được gửi qua SMS (dễ bị SIM swap) — chỉ email
- Recovery_token chỉ dùng để đăng ký passkey mới, không thể dùng để gọi API khác
- Recovery_token chỉ dùng được 1 lần
- Nếu email không nhận được: user phải liên hệ Support T2 với identity verification ngoài hệ thống
- Admin không được bypass recovery flow (phải qua OTP) — tránh insider threat
- Mọi bước recovery đều được ghi audit log với ip, user_agent

### Acceptance Criteria — xem US-AUTH-005

---

## 15. Migration Plan

### Strategy: Coexist → Nudge → Deprecate → Sunset

#### Phase 1 — Coexist (Sprint 8–9, 4 tuần)

**Mục tiêu:** Deploy FIDO2 mà không ảnh hưởng user hiện tại

- Deploy FIDO2 engine với feature flag `fido2_enabled=false`
- DB migration: thêm bảng `fido2_credentials`, `auth_audit_log` — không drop gì
- Bật cho internal users (5%) để smoke test
- Password login vẫn là primary — không thay đổi UX
- Thêm banner opt-in "Thử đăng nhập bằng passkey" trên trang settings

**Go/No-go:** 0 critical bug, p99 < 300ms, FIDO2 adoption > 80% internal users

#### Phase 2 — Nudge (Sprint 10–11, 4 tuần)

**Mục tiêu:** Tăng adoption tự nguyện

- Bật FIDO2 cho 100% users (`fido2_registration_open=true`)
- Password login vẫn hoạt động song song
- Prompt "Muốn kích hoạt passkey?" sau mỗi lần login bằng password
- Email campaign: hướng dẫn đăng ký passkey
- Indicator "Tài khoản bảo vệ bằng passkey ✓" trên dashboard
- Training Support T2 về recovery flow

**Go/No-go:** ≥ 40% user active đã có ít nhất 1 passkey

#### Phase 3 — Deprecate (Sprint 12–13, 4 tuần)

**Mục tiêu:** Làm rõ password sắp bị loại bỏ

- Users có passkey: ẩn password login option
- Users chưa có passkey: warning banner đỏ trên mọi trang
- Email thông báo: "Mật khẩu sẽ bị vô hiệu hóa vào [ngày sunset]"
- Mở recovery flow (email OTP) để users không bị lock out
- Freeze: không cho phép đổi/reset password (`password_login_disabled=true` cho users có passkey)

**Go/No-go:** ≥ 85% user active có passkey; support ticket < 50/tuần

#### Phase 4 — Sunset (Sprint 14+)

**Mục tiêu:** Loại bỏ hoàn toàn password

- `password_sunset_global=true`
- Users chưa có passkey sau ngày X: account suspended, gửi recovery email
- Drop cột `password_hash` khỏi DB (sau khi backup + audit)
- Xóa tất cả password-related endpoints, middleware, dependency bcrypt
- Update security policy: "No password authentication"

**Go/No-go:** 100% auth qua FIDO2; 0 password-related security ticket trong 2 tuần

### Feature Flags

| Flag | Default | Bật tại | Mô tả |
|---|---|---|---|
| `fido2_enabled` | false | Phase 1 | Kill switch toàn bộ FIDO2 |
| `fido2_registration_open` | false | Phase 1 (internal) → Phase 2 (all) | Cho phép đăng ký passkey |
| `password_login_disabled` | false | Phase 3 | Tắt password với users có passkey |
| `password_sunset_global` | false | Phase 4 | Tắt password toàn bộ |

### Rollback Plan

**Rollback nhanh (< 15 phút, reversible):**
1. Toggle `fido2_enabled=false` trên LaunchDarkly — không cần deploy
2. Hệ thống tự fallback về password login
3. Notify #on-call và #security-alerts
4. Giữ nguyên data credentials trong DB
5. Post-mortem trong 48h trước khi retry

**Không rollback được (irreversible):**
- Sau khi drop cột `password_hash` (Phase 4) → chỉ restore từ backup
- Sau khi xóa bcrypt dependency → cần thêm lại và redeploy

---

## 16. Architecture Decision Records (ADR)

> ADR ghi lại lý do đằng sau các quyết định kỹ thuật quan trọng. Giúp team mới hiểu tại sao, không chỉ là cái gì.

---

### ADR-001: Chọn ES256 thay vì RS256 làm thuật toán signing mặc định

**Status:** Accepted  
**Date:** 2026-05-10  
**Deciders:** Backend Lead, Security Lead

**Context:**  
FIDO2 và JWT đều hỗ trợ nhiều thuật toán ký. Cần chọn mặc định cho signing key của JWT access token và ưu tiên trong `pubKeyCredParams`.

**Quyết định:** Dùng ES256 (ECDSA P-256) làm mặc định, RS256 (RSA-2048) làm fallback.

**Lý do:**
- Key size nhỏ hơn nhiều: EC P-256 key = 256 bits vs RSA = 2048 bits → JWT nhỏ hơn
- Nhanh hơn RS256 khoảng 3–5x trên hardware hiện đại
- Tương đương security level: EC-256 ≈ RSA-3072 về mặt bảo mật
- Support rộng: Chrome 108+, Safari 16+, Firefox 119+ đều support
- AWS KMS và HashiCorp Vault đều có native support EC P-256

**Trade-offs:**
- Một số legacy device (Android < 9, iOS < 16) không support EC → dùng RS256 fallback
- `pubKeyCredParams` phải list cả hai: `[{alg: -7}, {alg: -257}]`

---

### ADR-002: Access token TTL = 15 phút, không phải 30 hay 60 phút

**Status:** Accepted  
**Date:** 2026-05-10  
**Deciders:** Security Lead, Backend Lead

**Context:**  
Access token là stateless JWT, một khi issued không thể revoke trước khi hết hạn (trừ khi có blacklist). Cần cân bằng giữa UX (refresh ít) và security (minimize window nếu token bị stolen).

**Quyết định:** TTL = 900 giây (15 phút).

**Lý do:**
- 15 phút là sweet spot phổ biến trong industry (Google, GitHub, Auth0 default)
- Nếu token bị stolen, attacker có tối đa 15 phút trước khi token hết hạn
- Với refresh token rotation, UX không bị ảnh hưởng — client auto-refresh
- 30 phút: window quá lớn nếu token bị intercept
- 5 phút: quá nhiều refresh request, tăng tải server

**Trade-offs:**
- Mobile app phải implement token refresh logic
- Đường truyền chậm (3G) có thể gây friction nếu nhiều refresh

---

### ADR-003: Lưu refresh token dưới dạng hash, không plain text

**Status:** Accepted  
**Date:** 2026-05-10  
**Deciders:** Security Lead

**Context:**  
Refresh token cần được lưu trong DB để có thể revoke. Câu hỏi: lưu plain text hay hash?

**Quyết định:** Lưu SHA-256 hash của refresh token, không lưu plain text.

**Lý do:**
- Nếu DB bị dump, attacker không có refresh token để dùng
- SHA-256 đủ nhanh cho lookup mà không cần bcrypt (refresh token đã là random bytes, không cần key stretching)
- Pattern tương tự GitHub Personal Access Token, Stripe API key

**Implementation:** `token_hash = hex(SHA-256(token_bytes))`, lookup bằng `WHERE token_hash = ?`

---

### ADR-004: Challenge lưu Redis, không in-memory

**Status:** Accepted  
**Date:** 2026-05-08  
**Deciders:** Backend Lead, Infra Lead

**Context:**  
FIDO2 challenge cần được lưu tạm (TTL 5 phút) để verify khi user submit response. Có thể lưu in-memory hoặc Redis.

**Quyết định:** Lưu trong Redis với TTL automatic.

**Lý do:**
- Rolling deploy: nếu lưu in-memory, request /begin và /finish có thể hit 2 instance khác nhau → challenge not found
- Auto-expiry: Redis TTL tự xử lý cleanup, không cần cron job
- Horizontal scaling: nhiều instance share cùng Redis cluster → consistent state
- Redis Sentinel đảm bảo HA

**Trade-offs:**
- Thêm dependency vào Redis — nếu Redis down, FIDO2 auth không hoạt động
- Mitigation: circuit breaker với timeout 1s, graceful 503 response

---

### ADR-005: Không hỗ trợ PKCE `plain` method

**Status:** Accepted  
**Date:** 2026-05-05  
**Deciders:** Security Lead

**Context:**  
RFC 7636 định nghĩa hai method: `plain` (code_verifier = code_challenge) và `S256` (code_challenge = BASE64URL(SHA256(verifier))).

**Quyết định:** Chỉ chấp nhận `S256`, từ chối `plain` với HTTP 400.

**Lý do:**
- `plain` method không có security benefit — attacker intercept challenge_code là có verifier
- `S256` là standard mà mọi modern OAuth2 client support
- RFC 9700 (2024) khuyến nghị không dùng `plain`

---

### ADR-006: Chọn `attestation: "none"` cho v1 và defer FIDO MDS3

**Status:** Accepted\
**Date:** 2026-05-16\
**Deciders:** Security Lead, Backend Lead, Infra Lead

**Context:**
Registration có thể yêu cầu attestation để xác minh model authenticator và provenance phần cứng qua FIDO Metadata Service v3 (MDS3). Lựa chọn này tăng assurance, nhưng kéo theo vận hành trusted roots, refresh metadata định kỳ, cache, xử lý outage, revocation/status report và latency khi registration. v1 cần passkey rollout ổn định, latency thấp và ít dependency vận hành.

**Quyết định:** v1 dùng WebAuthn `attestation: "none"`. Server không validate FIDO MDS3, không xử lý attestation certificate chain/trusted roots, và không dùng attestation statement/AAGUID/cert chain làm bằng chứng provenance phần cứng.

**Hệ quả bảo mật:**
- Registration vẫn verify các yêu cầu WebAuthn core: challenge, origin, rpIdHash, type, UV flag, credential uniqueness, public key và thuật toán được phép.
- AAGUID và transports chỉ dùng cho diagnostic/risk analytics, không dùng để grant assurance level.
- Không thể chứng minh bằng cryptographic metadata rằng credential đến từ hardware authenticator cụ thể trong v1.

**Tác động đến high-assurance scope (`admin:write`):**
- `admin:write` không được cấp chỉ vì registration response có attestation-like data; v1 coi dữ liệu đó là không đáng tin cho provenance.
- Nếu business bắt buộc hardware provenance cho `admin:write`, endpoint phải trả `insufficient_assurance` cho đến khi có policy ngoài luồng được Security phê duyệt hoặc Phase 2 MDS3 được triển khai.
- Tín hiệu như `backup_state=false` có thể dùng để giảm rủi ro synced passkey, nhưng không thay thế MDS3 attestation verification.

**Trade-offs:**
- Ưu điểm: giảm dependency vận hành, tránh failure mode khi MDS/cache/trusted roots lỗi, registration nhanh và dễ rollout hơn.
- Nhược điểm: không có provenance phần cứng trong v1; một số use case high-assurance phải bị hạn chế hoặc cần exception được audit.

**Follow-up Phase 2 (ngoài scope v1):**
Khi đưa MDS3 vào scope, cần thiết kế metadata refresh schedule, signed TOC verification, cache TTL/stale behavior, failure handling, trusted roots management, status report handling và monitoring.

---

## 17. Traceability Matrix

> Bảng này link từng User Story → Acceptance Criteria → Endpoint → DB Table → NFR để đảm bảo không có gì bị bỏ sót khi requirement thay đổi.

| Story ID | AC / TC | Endpoint | DB Table | NFR | Jira |
|---|---|---|---|---|---|
| US-AUTH-001 | TC-001-01 | POST /fido2/register/begin, POST /fido2/register/finish | fido2_credentials, auth_audit_log | NFR-P-01, NFR-S-03 | AUTH-121 |
| US-AUTH-001 | TC-001-02 | POST /fido2/register/finish | fido2_credentials | — | AUTH-121 |
| US-AUTH-001 | TC-001-03 | POST /fido2/register/finish | — (Redis) | — | AUTH-121 |
| US-AUTH-001 | TC-001-04 | POST /fido2/register/begin | users | — | AUTH-121 |
| US-AUTH-001 | TC-001-05 | POST /fido2/register/begin | — (Rate limiter) | NFR-S-04 | AUTH-121 |
| US-AUTH-002 | TC-002-01 | POST /fido2/auth/begin, POST /fido2/auth/finish, GET /oauth2/authorize | fido2_credentials, auth_audit_log | NFR-P-01, NFR-P-03 | AUTH-122 |
| US-AUTH-002 | TC-002-02 | POST /fido2/auth/finish | auth_audit_log | NFR-S-03 | AUTH-122 |
| US-AUTH-002 | TC-002-03 | POST /fido2/auth/finish | fido2_credentials, auth_audit_log | NFR-S-03 | AUTH-122 |
| US-AUTH-002 | TC-002-04 | POST /fido2/auth/finish | auth_audit_log | NFR-S-03 | AUTH-122 |
| US-AUTH-002 | TC-002-05 | POST /fido2/auth/begin | — (Rate limiter) | — | AUTH-122 |
| US-AUTH-002 | TC-002-06 | POST /fido2/auth/finish | — | — | AUTH-122 |
| US-AUTH-002 | TC-002-07 | POST /fido2/auth/finish | fido2_credentials | — | AUTH-122 |
| US-AUTH-003 | TC-003-01 | GET /fido2/credentials | fido2_credentials | — | AUTH-123 |
| US-AUTH-003 | TC-003-02 | DELETE /fido2/credentials/:id | fido2_credentials | — | AUTH-123 |
| US-AUTH-003 | TC-003-03 | DELETE /fido2/credentials/:id | fido2_credentials | — | AUTH-123 |
| US-AUTH-004 | TC-004-01 | DELETE /admin/users/:id/credentials | fido2_credentials, auth_audit_log | NFR-S-03 | AUTH-124 |
| US-AUTH-004 | TC-004-02 | DELETE /admin/users/:id/credentials | — | — | AUTH-124 |
| US-AUTH-005 | TC-005-01 | POST /auth/recovery/begin, POST /auth/recovery/verify, POST /fido2/register/finish | users, fido2_credentials, auth_audit_log | NFR-S-03 | AUTH-125 |
| US-AUTH-005 | TC-005-02 | POST /auth/recovery/verify | — (Redis) | — | AUTH-125 |
| US-AUTH-005 | TC-005-03 | POST /auth/recovery/verify | — (Redis) | — | AUTH-125 |

### NFR Traceability

| NFR ID | Liên quan đến Story | Được test bởi | Đo bằng |
|---|---|---|---|
| NFR-P-01 | US-AUTH-002 | TC-002-01 (performance) | Datadog APM p99 |
| NFR-P-03 | US-AUTH-001, US-AUTH-002 | k6 load test | k6 results |
| NFR-A-01 | Tất cả | Uptime monitoring | Uptime Robot |
| NFR-S-03 | US-AUTH-002, US-AUTH-004, US-AUTH-005 | Audit log check | Splunk query |
| NFR-C-01 | US-AUTH-001, US-AUTH-002 | BrowserStack | Test report |

---

## 18. Edge Cases & Escalation

| Tình huống | Phát hiện | Xử lý tự động | Escalation |
|---|---|---|---|
| Sign count giảm (potential clone) | /fido2/auth/finish | HTTP 401, suspend credential | Alert #security-alerts ngay lập tức |
| Origin mismatch (phishing attempt) | /fido2/auth/finish | HTTP 400 | Log ERROR, monitor spike |
| Challenge timeout | /fido2/auth/finish | HTTP 400 `challenge_expired` | Không, user restart flow |
| PR > 500 dòng thay đổi | — | Không có | NEEDS_HUMAN review |
| Attestation statement/provenance phần cứng xuất hiện hoặc thiếu | /fido2/register/finish | Với policy `attestation=none`, chỉ verify WebAuthn core fields; không dùng statement/AAGUID/cert chain để chứng minh hardware provenance | Log INFO nếu client gửi dữ liệu ngoài kỳ vọng |
| Request `admin:write` nhưng credential không có provenance được policy chấp nhận | /fido2/auth/finish hoặc /oauth2/authorize | HTTP 403 `insufficient_assurance`; v1 không coi attestation statement là bằng chứng, Phase 2 MDS3 mới có thể nâng assurance | Security review nếu business cần allowlist tạm |
| Redis down | Mọi FIDO2 endpoint | HTTP 503 + Retry-After, circuit breaker | PagerDuty alert |
| DB connection exhausted | Mọi endpoint | HTTP 503, queue request tối đa 30s | PagerDuty alert |
| Synced passkey cho high-assurance | /fido2/auth/finish | HTTP 403 `insufficient_assurance` | Không, user dùng hardware key |
| Recovery OTP leak (user report) | Manual | Admin suspend account | Security team investigate |
| JWKS rotation missed | — | Monitoring alert | On-call engineer rotate key |
| Refresh token reuse detected | POST /oauth2/token | Revoke toàn bộ family_id | Log WARN, optional user notify |

---

*Spec ID: AUTH-FIDO2-001 · Version 1.0.0 · 2026-05-16*  
*Trạng thái: Draft — Chờ Security Lead và Backend Lead approve*
