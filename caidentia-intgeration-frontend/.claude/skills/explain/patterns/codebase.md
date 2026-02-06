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
