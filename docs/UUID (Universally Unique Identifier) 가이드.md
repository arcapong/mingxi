# UUID (Universally Unique Identifier) 가이드

## 1. 개요
UUID(Universally Unique Identifier, 범용 고유 식별자)는 128비트 길이를 가지는 식별자로, 중앙 집중형 관리 체계 없이도 전 세계적으로 고유한 ID를 생성하기 위해 설계되었습니다. IETF RFC 9562 표준을 기준으로 v1부터 v8까지 정의되어 있습니다.

---

## 2. UUID 버전별 비교

*이탤릭*으로 표시한 버전은 실무에서 거의 사용하지 않는 버전입니다.

| 버전 | 생성 원리 | 주요 특징 | 추천 사용처 |
| :--- | :--- | :--- | :--- |
| **UUIDv1** | 타임스탬프 + MAC 주소 | 타임스탬프를 포함하지만 필드 배치상(`time_low`가 앞) 문자열 정렬이 시간순과 일치하지 않음(이를 재배열해 고친 것이 v6). MAC 주소 노출로 보안 위험 존재. | 레거시 시스템 |
| *UUIDv2* | 타임스탬프 + POSIX UID/GID | DCE Security 전용. 명세가 느슨하고 타임스탬프 정밀도가 크게 희생되어 충돌 위험이 높음. | 사실상 미사용 (DCE 환경) |
| **UUIDv3** | MD5 해시 | 네임스페이스와 문자열을 MD5로 해시 처리. 동일 입력 시 동일 ID 반환. | 확정적(Deterministic) 고정 ID |
| **UUIDv4** | 순수 무작위 (Random) | 122비트를 완전 무작위 난수로 채움. 예측 불가능하고 보안성이 높으나 RDB PK로 사용 시 B-Tree 인덱스 단편화 발생. | 세션 ID, 보안 토큰, 일반 무작위 ID |
| **UUIDv5** | SHA-1 해시 | v3와 동일하며 MD5 대신 SHA-1 해시를 적용. | v3 대체용 고정 ID |
| *UUIDv6* | 타임스탬프 + MAC 주소 (재배열) | v1과 동일한 정보를 상위 비트부터 배치해 시간순 정렬이 가능하도록 개선. v1 데이터와의 호환이 필요할 때만 사용하고, 신규 시스템은 v7 권장. | v1 마이그레이션 |
| **UUIDv7** | Unix 타임스탬프 + 난수 | 상위 48비트에 Unix Epoch 시간(밀리초)을 포함. 시간순 정렬(Time-ordered)이 가능하여 DB 인덱스 성능 최적화. | **RDB Primary Key (PK)**, 로그 ID |
| *UUIDv8* | 커스텀 (자유 형식) | 버전/변형 비트를 제외한 122비트를 구현자가 자유롭게 정의하는 실험/벤더 전용 버전. 상호운용성 없음. | 벤더 특화 ID, 실험 |

---

## 3. UUIDv4 vs UUIDv7

### 3.1 비트 구조 (RFC 9562)

두 버전 모두 128비트 중 버전(4비트)과 변형(2비트)을 제외한 **122비트를 무엇으로 채우는가**가 차이의 전부입니다. v4는 122비트 전부 난수, v7은 상위 48비트를 타임스탬프로 치환한 구조입니다.

**UUIDv4 — 난수 122비트**

| 필드 | 비트 수 | 내용 |
| :--- | :--- | :--- |
| `random_a` | 48 | 난수 |
| `ver` | 4 | 버전 `0100` (=4) |
| `random_b` | 12 | 난수 |
| `var` | 2 | 변형(variant) `10` |
| `random_c` | 62 | 난수 |

**UUIDv7 — 타임스탬프 48비트 + 난수 74비트**

| 필드 | 비트 수 | 내용 |
| :--- | :--- | :--- |
| `unix_ts_ms` | 48 | Unix Epoch 밀리초 타임스탬프 (빅엔디언) |
| `ver` | 4 | 버전 `0111` (=7) |
| `rand_a` | 12 | 난수 (선택: 서브밀리초 정밀도 또는 카운터) |
| `var` | 2 | 변형(variant) `10` |
| `rand_b` | 62 | 난수 (선택: 단조 증가 카운터 포함) |

문자열 표기에서의 위치 (`r` = 난수, `t` = 타임스탬프, `V` ∈ {8, 9, a, b}):

```
UUIDv4: rrrrrrrr-rrrr-4rrr-Vrrr-rrrrrrrrrrrr
UUIDv7: tttttttt-tttt-7rrr-Vrrr-rrrrrrrrrrrr
```

3번째 그룹의 첫 글자가 버전 숫자 그대로라서, UUID 문자열만 봐도 버전을 즉시 판별할 수 있습니다.

### 3.2 RDB Primary Key 관점: B-Tree 인덱스 구조 및 작동 원리
MySQL(InnoDB)의 Primary Key는 데이터 레코드가 PK 순서대로 물리적으로 저장되는 **클러스터드 인덱스(Clustered Index)** 구조를 가집니다. B-Tree 페이지(기본 16KB)가 가득 찬 상태에서 중간 위치에 데이터가 비집고 들어가면 **페이지 분할(Page Split)**이 발생하며 디스크 I/O 폭증 및 단편화(Fragmentation)가 발생합니다.

> **참고:** PostgreSQL은 테이블이 힙(Heap)에 저장되어 클러스터드 인덱스는 아니지만, PK의 B-Tree 인덱스 자체에서 동일한 페이지 분할 문제가 발생하므로 아래 논의가 그대로 적용됩니다.

#### 3.2.1 UUIDv4 사용 시 문제점
* **Random Write:** 완전 무작위 생성으로 인해 B-Tree 중간 노드에 계속 삽입됩니다.
* **빈번한 Page Split:** 페이지 분할이 전 방위적으로 발생하여 디스크 I/O가 폭증하고 인덱스 단편화로 디스크 공간이 낭비됩니다.
* **캐시 히트율 급감:** 메모리(Buffer Pool) 외 과거 페이지를 읽어와야 하므로 데이터 증가 시 Insert 속도가 급감합니다.

#### 3.2.2 UUIDv7 사용 시 이점
* **Sequential Write (Append-Only):** 상위 48비트가 밀리초 단위 타임스탬프이므로 B-Tree 맨 오른쪽 끝에 순차 삽입됩니다.
* **Page Split 최소화:** 페이지 분할이 거의 없어 인덱스 밀도가 극대화되고 디스크 I/O가 대폭 감소합니다.
* **높은 캐시 효율:** 최신 인덱스 페이지 위주로 메모리에 유지되어 대용량 Insert에서도 안정된 성능을 보입니다.

### 3.3 항목별 비교

| 항목 | **UUIDv4** | **UUIDv7** |
| :--- | :--- | :--- |
| **난수(엔트로피)** | 122비트 | 74비트 |
| **시간순 정렬** | 불가 | 가능 (밀리초 단위) |
| **충돌 확률** | 2¹²²분의 1 수준 | 같은 밀리초 내에서만 2⁷⁴분의 1 수준 (사실상 무시 가능) |
| **정보 노출** | 없음 | **생성 시각 노출** (상위 48비트로 역산 가능) |
| **예측 가능성** | 완전 예측 불가 | 타임스탬프 부분은 예측 가능 |
| **B-Tree 삽입 패턴** | 랜덤 (페이지 분할 다수) | 순차 (Append-Only) |
| **동일 밀리초 내 순서** | 해당 없음 | 기본은 미보장 (구현체의 카운터/서브밀리초 옵션으로 보장 가능) |
| **적합 용도** | 보안 토큰, 세션 ID, 외부 노출 ID | RDB PK, 로그/이벤트 ID |

> **핵심:** 정렬이 필요하면 v7, 생성 시각까지 숨겨야 하면 v4. URL 등 외부에 노출되는 리소스 ID에 v7을 쓰면 생성 시점이 드러나므로, 민감한 경우 v4를 쓰거나 내부 PK(v7)와 공개용 ID(v4)를 분리합니다.

---

## 4. 언어 및 DB별 UUIDv7 구현 가이드

### 4.1 TypeScript
```typescript
import { v7 as uuidv7 } from 'uuid';

// UUIDv7 생성
const id: string = uuidv7();

// 타임스탬프 추출
function getTimestampFromUUIDv7(uuidStr: string): Date {
  const hexTimestamp = uuidStr.replace(/-/g, '').slice(0, 12);
  return new Date(parseInt(hexTimestamp, 16));
}
```

### 4.2 Python
```python
# Python 3.9 ~ 3.13 (uuid6 패키지 사용) / Python 3.14+ (uuid.uuid7() 내장)
from uuid6 import uuid7

my_uuid = uuid7()
timestamp_ms = my_uuid.int >> 80
```

### 4.3 Java
```java
// com.github.f4b6a3:uuid-creator 의존성 활용
import com.github.f4b6a3.uuid.UuidCreator;
import java.util.UUID;

UUID uuidV7 = UuidCreator.getTimeOrderedEpoch();
long timestampMs = uuidV7.getMostSignificantBits() >>> 16;
```

### 4.4 PostgreSQL
* **PostgreSQL 18+:** `DEFAULT uuidv7()` 내장 함수 지원.
* **PostgreSQL 17 이하:** PL/pgSQL 사용자 정의 함수 생성 후 `DEFAULT generate_uuidv7()` 적용.

### 4.5 MySQL / MariaDB
* `BINARY(16)` 타입에 `UUID_TO_BIN()` 및 커스텀 Stored Function(`uuidv7()`)을 적용하여 저장 용량 및 인덱스 크기를 최소화.
* **실무 권장:** DB 부하 분산 및 쿼리 효율을 위해 **애플리케이션(TypeScript/Java/Python 등) 레이어에서 ID를 생성하여 전송하는 패턴**을 권장.