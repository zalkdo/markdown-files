---
name: explain
description: 코드를 한국어로 설명합니다 (요약, 비유, 다이어그램, 흐름, 예제, 주의점)
context: fork
agent: Explore
allowed-tools:
  - Read
  - Grep
  - Glob
---

# /explain - 코드 설명

## 실행 방법
1. 대상 파일/함수를 Read로 읽기
2. 관련 파일을 Glob/Grep으로 탐색
3. `./format.md` 형식으로 설명 출력

## 컨텍스트 로딩
- RxJS 관련: `./examples/rxjs.md` 참조
- React Hook: `./examples/react-hook.md` 참조
- Next.js: `./examples/nextjs.md` 참조
- 프로젝트 패턴: `./patterns/codebase.md` 참조
