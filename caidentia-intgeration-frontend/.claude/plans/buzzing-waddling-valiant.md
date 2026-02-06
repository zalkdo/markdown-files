# Skill 파일 생성 계획 (토큰 효율성 최적화)

## 목표
토큰 효율성을 최적화한 `/explain` 코드 설명 skill 생성

## 최적화 전략

| 전략 | 효과 |
|------|------|
| 파일 분리 | 필요한 파일만 로드 (200→55~110 tokens) |
| `context: fork` | 서브에이전트에서 격리 실행 |
| `allowed-tools` | Read, Grep, Glob만 허용 (읽기 전용) |
| 조건부 로딩 | 코드 유형에 따라 관련 예제만 로드 |

## 디렉토리 구조

```
.claude/skills/explain/
├── SKILL.md           # 최소 진입점 (~15줄)
├── format.md          # 출력 형식 템플릿 (~40줄)
├── examples/
│   ├── rxjs.md        # RxJS 패턴 가이드
│   ├── react-hook.md  # React Hook 가이드
│   └── nextjs.md      # Next.js 패턴 가이드
└── patterns/
    └── codebase.md    # 프로젝트 특화 패턴
```

## 생성할 파일

### 1. SKILL.md (최소 진입점)

```yaml
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
```

### 2. format.md (출력 형식)

```markdown
# 코드 설명 출력 형식

## 필수 섹션

### 1. 한 줄 요약
> [코드가 무엇을 하는지 한 문장으로]

### 2. 비유
> 이 코드는 [일상적 비유]와 같습니다. [비유 설명]

### 3. ASCII 다이어그램
```
[구조/흐름을 ASCII로 시각화]
┌─────────┐     ┌─────────┐
│ Input   │────▶│ Process │────▶ Output
└─────────┘     └─────────┘
```

### 4. 핵심 흐름
1. **[단계1]**: 설명
2. **[단계2]**: 설명

### 5. 예제 코드
```typescript
// 사용 예시
```

### 6. 주의할 점
- **[주의1]**: 설명

## 언어 규칙
- 모든 설명은 한국어, 코드/기술 용어는 영어 유지
```

### 3. examples/rxjs.md

```markdown
# RxJS 패턴 설명 가이드

## ReactiveStreamsProcessor 다이어그램
```
┌──────────────────────────────────────────────────┐
│              ReactiveStreamsProcessor             │
│  ┌─────────┐    ┌─────────┐    ┌──────────────┐ │
│  │ publish │───▶│ Subject │───▶│ subscribers  │ │
│  └─────────┘    └─────────┘    └──────────────┘ │
└──────────────────────────────────────────────────┘
```

## 비유
- Subject → "방송국" (여러 청취자에게 동시 전송)
- publish → "라디오 송출"
- subscribe → "라디오 채널 맞추기"

## 핵심 개념
- Message = {group, topic, payload}
- group: 수신자 그룹 필터링
- topic: 이벤트 종류 구분
```

### 4. examples/react-hook.md

```markdown
# React Hook 설명 가이드

## Custom Hook 다이어그램
```
┌─────────────────────────────────┐
│           useXxx()              │
│  ┌─────────┐   ┌─────────────┐  │
│  │ Context │──▶│ useCallback │  │
│  │   or    │   │ useMemo     │  │
│  │ State   │   │ useEffect   │  │
│  └─────────┘   └─────────────┘  │
│         ↓                       │
│  return { state, actions }      │
└─────────────────────────────────┘
```

## 비유
- useContext → "공용 게시판에서 정보 가져오기"
- useCallback → "전화번호 저장 (매번 새로 외우지 않음)"
- useEffect → "자동 알림 설정"

## 핵심 포인트
- 의존성 배열 [] 의미 설명 필수
- cleanup 함수 언급
```

### 5. examples/nextjs.md

```markdown
# Next.js 15 App Router 설명 가이드

## 레이아웃 구조
```
app/
├── layout.tsx      ─── 전체 앱 감싸기
├── page.tsx        ─── / 경로
└── [route]/
    └── page.tsx    ─── /[route] 경로
```

## 비유
- layout.tsx → "건물의 기본 구조"
- page.tsx → "각 방의 가구 배치"
- 'use client' → "클라이언트 전용 기능 필요"

## 주의점
- Server vs Client Component 구분
- 'use client' 지시어 의미
```

### 6. patterns/codebase.md

```markdown
# Caidentia 프로젝트 패턴

## 아키텍처
```
GlobalContext (Providers)
    ├── ThemeProvider
    ├── MantineProvider
    ├── QueryClientContext
    └── ReactiveContext ←── RxJS 이벤트 버스
```

## 컴포넌트 통신
```
useAlert/useConfirm ──publish──▶ ReactiveStreamsProcessor
                                        ↓
AppAlertModal/AppConfirmModal ◀──subscribe──
```

## 참조 파일
- 전역 상태: `@/contexts/GlobalContext.tsx`
- RxJS 코어: `@/lib/reactive/reactivestreams.ts`
- Hook 패턴: `@/hooks/useReactive.ts`
```

## 토큰 효율성 비교

| 시나리오 | 단일 파일 | 분리 구조 | 절감률 |
|---------|----------|----------|-------|
| 간단한 유틸 함수 | 200 tokens | 55 tokens | 72% |
| RxJS 코드 설명 | 200 tokens | 80 tokens | 60% |
| 프로젝트 패턴 설명 | 200 tokens | 110 tokens | 45% |

## 검증 방법

1. `/explain` 명령어로 skill 호출
2. 다양한 코드 유형으로 테스트:
   - `hooks/useReactive.ts` (RxJS + Hook)
   - `components/AppLayout.tsx` (Next.js)
   - `contexts/GlobalContext.tsx` (프로젝트 패턴)
3. 출력이 format.md 형식을 따르는지 확인
