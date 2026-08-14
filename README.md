# mingxi

개발하면서 공부한 내용을 정리하는 문서 저장소입니다.

📖 **문서 사이트: https://arcapong.github.io/mingxi/**

## 문서 목록

### API

- [RESTful API 설계 및 API 자격증명(Credentials)](<docs/RESTful API 설계 및 API 자격증명(Credentials).md>) — API 자격증명 전달 방식, 서비스 공용/유저별 토큰 정책, RESTful URI 설계 규칙, Server-to-Server 연동 시 토큰 관리 로직

### 보안

- [랜덤 시크릿(토큰) 및 인코딩](<docs/랜덤 시크릿(토큰) 및 인코딩.md>) — 용도별(JWT 서명, AES 암호화, Bearer 토큰) 시크릿 키 생성 `openssl` 명령어와 인코딩 형식 제약
- [인증(Authentication), 인가(Authorization), OAuth 2.0, OIDC](<docs/인증(Authentication), 인가(Authorization), OAuth 2.0, OIDC.md>) — 인증 vs 인가(401/403), OAuth 2.0 역할·Grant·토큰, OIDC ID Token, 구글·카카오·네이버 간편로그인이 쓰는 스택 매핑

### 데이터베이스

- [UUID (Universally Unique Identifier) 가이드](<docs/UUID (Universally Unique Identifier) 가이드.md>) — 버전별(v1~v8) 비교, RDB PK 관점의 UUIDv4 vs v7 (B-Tree 페이지 분할), 언어/DB별 UUIDv7 구현
