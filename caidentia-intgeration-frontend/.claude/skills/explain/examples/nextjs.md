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
