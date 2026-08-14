# CLAUDE.md

## 프로젝트 개요

개발하면서 공부한 내용을 정리하는 **문서 저장소**입니다. 코드는 없고 `docs/` 아래 한국어 마크다운 문서가 전부입니다.

- 문서 사이트: https://arcapong.github.io/mingxi/ (MkDocs Material + GitHub Pages)
- `main`에 push하면 [.github/workflows/deploy-docs.yml](.github/workflows/deploy-docs.yml)이 자동 빌드/배포합니다.

## 문서 추가 절차

1. `docs/`에 마크다운 파일 작성 (파일명은 한국어 제목 그대로, 공백 포함 가능)
2. [mkdocs.yml](mkdocs.yml)의 `nav`에 항목 추가 (API/보안 등 주제별 섹션)
3. [docs/index.md](docs/index.md)와 [README.md](README.md)의 문서 목록에 한 줄 요약과 함께 링크 추가
4. 커밋 & push → 자동 배포

로컬 미리보기: `pip install mkdocs-material` 후 `mkdocs serve`. push 전 `mkdocs build --strict`로 검증할 것.

## 문서 스타일

- 표 중심의 간결한 정리, 핵심은 `> **핵심:** ...` 인용구로 강조
- 섹션 구분은 `---` 수평선 사용
- **하위 리스트는 반드시 4칸 들여쓰기** — MkDocs의 Python-Markdown은 2~3칸을 중첩으로 인식하지 못하고 형제 항목으로 풀어버림

## 사이트 커스터마이징

- 색상은 [docs/stylesheets/extra.css](docs/stylesheets/extra.css)의 `--md-*` 변수로 관리 (브랜드 퍼플 `#5b3fd4`)
- **주의:** `mkdocs.yml` palette의 `primary: custom` / `accent: custom`을 지우면 안 됨 — 테마가 `<body>`에 색 변수를 정의해서 `extra.css`의 `:root` 오버라이드가 무시됨
- 파비콘/로고: `docs/assets/favicon.svg`(퍼플), `docs/assets/logo.svg`(헤더용 흰색) — 같은 문서 아이콘의 색상 변형

## 커밋 규칙

- 커밋 메시지는 한국어로, **Co-Authored-By 트레일러 넣지 않기**
- 이 저장소는 개인 계정 `arcapong <arcapong@gmail.com>`으로 커밋합니다. 새 머신에서는 repo-local 설정 필요:
  - `git config user.name arcapong && git config user.email arcapong@gmail.com`
  - push 자격증명도 개인 계정으로 (개인 SSH 키를 `git config core.sshCommand`로 지정하거나 개인 토큰 사용)
