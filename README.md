# Noumenon

파이널 판타지 14 정보 검색 서비스.

> 설계 원칙과 문서는 [`docs/`](./docs)를 참조하세요.

## 구성

- `apps/front/` — Frontend (Vite + React 19, TypeScript).
- `packages/gleaner/` — 데이터 추출 (Rust). raw EXD/EXH 파일로부터 서비스용 JSON과 `.d.ts`를 생성.
- `docs/` — Phase A~D 설계 문서.

## 기술 스택

- **런타임/매니저**: Bun 1.3.4
- **모노레포 도구**: Turborepo 2.x
- **포맷팅/린팅**: Biome 1.x
- **타입체커**: TypeScript 5.x
- **Git hooks**: Lefthook

## 로컬 개발

```bash
# 의존성 설치
bun install

# Git hooks 활성화 (최초 1회)
bunx lefthook install

# 타입 검사
bun run check-types

# 포맷 + 린트
bun run format-and-lint

# 전체 빌드
bun run build

# 개발 서버 (frontend)
bun run dev

# 테스트
bun run test
```

## 라이선스

[MIT](./LICENSE)
