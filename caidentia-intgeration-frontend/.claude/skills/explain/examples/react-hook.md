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
