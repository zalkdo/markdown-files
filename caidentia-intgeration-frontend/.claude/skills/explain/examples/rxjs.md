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
