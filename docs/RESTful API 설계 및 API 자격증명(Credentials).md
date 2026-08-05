# RESTful API 설계 및 API 자격증명(Credentials)

---

## 1. API 자격증명 전달 방식 (2가지)

| 구분 | **매 요청 ID/Secret 전송** | **Access Token 발급 후 사용 (표준)** |
| :--- | :--- | :--- |
| **특징** | 매번 Master Key(Client Secret)를 전송 | 최초 1회 인증 후, 수명이 짧은 **Access Token** 사용 |
| **장점** | 단순함, 서버 상태 관리 불필요 | **보안성 우수**, 피해 범위 최소화, Scope(권한) 제어 가능 |
| **단점** | Secret 노출 위험 매우 높음 | 토큰 저장 및 만료/재발급(Refresh) 로직 구현 필요 |
| **실무** | 지양하는 추세 (폐쇄망/구형 API 일부) | **현대 Web API의 절대적 표준** |

---

## 2. 자격증명의 단위: 서비스 공용 vs 유저별

> **핵심:** Access Token을 사용하는 것은 **기술적 메커니즘(How)**이고, 이를 서비스 전체가 공유할지 유저별로 나눌지는 **비즈니스 정책(Who)**의 문제입니다.

- **서비스 공용 토큰 (`Client Credentials`):**
  - 서비스(백엔드) 대표 토큰 **1개만 발급받아 Redis/메모리에 캐싱** 후 모든 유저 요청에서 공유.
  - *예: OpenAI, 결제 PG, 날씨 API*
- **유저별 토큰 (`Authorization Code`):**
  - 유저 각각이 직접 동의 및 인증을 거쳐 **유저 N명만큼 토큰을 발급받아 DB에 매핑 저장**.
  - *예: Google Drive, Notion, GitHub 연동*

---

## 3. 유저별 자격증명과 RESTful API 설계

유저별 자격증명을 사용하더라도 **인증(Who)과 리소스(What)를 엄격히 분리**하면 완벽한 RESTful 규격을 유지할 수 있습니다.

### RESTful 설계 3대 규칙
1. **자격증명은 HTTP Header로:** Token/Key는 URL이 아닌 `Authorization: Bearer <Token>` 헤더에 담습니다.
2. **식별자와 권한의 분리:** URL은 오직 명사형 자원(Resource)만 가리키고, 접근 권한 검증은 백엔드가 헤더의 토큰을 복호화하여 수행합니다.
3. **실무형 URI 패턴 (`GET /orders`):**
   - `/users/123/orders` (학술적/엄격한 REST): 식별자가 명확하지만 프론트엔드의 ID 사전 조회 오버헤드 존재.
   - `/me/orders` (상대적 표현): 직관적이나 URI 식별성/캐싱 측면에서 REST 원칙과 다소 긴장 관계.
   - **`GET /orders` (실무 표준 🏆):** REST의 자원 명사 규칙을 지키면서, **"현재 인증된 유저(In-context user)의 주문 목록"**을 가져오는 가장 실용적이고 범용적인 패턴.

---

## 4. Server-to-Server 오픈 API 연동 시 필수 백엔드 로직

Access Token 기반 외부 API를 Server-to-Server로 가져다 쓸 때는 다음 3가지 구현이 필수적입니다.

1. **Token Caching:** 매 요청마다 발급받지 않고 **Redis나 In-Memory에 저장**하여 재사용 (Rate Limit 및 오버헤드 방지).
2. **Proactive Refresh (사전 갱신):** 토큰 만료 시간 3~5분 전(Buffer)을 만료 시점으로 간주하고, API 호출 직전에 미리 재발급받는 안정적인 로직 구축.
3. **저장 구조의 결정:** 제공자(Provider)의 정책이 **서비스 공용이면 Redis 1개**, **유저별 위임이면 DB 유저 테이블에 저장**하도록 백엔드 구조 설계.
