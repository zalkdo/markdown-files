# Token 최적화 보고서

**작성일**: 2026-02-07
**프로젝트**: MyApp (Spring Boot Hexagonal Architecture)

---

## 📊 현재 Token 사용 현황

### Context 사이즈 분석 (from /context 출력)

| 카테고리 | Token 사용량 | 비율 |
|---------|-------------|------|
| System prompt | 3k | 1.5% |
| System tools | 16.5k | 8.3% |
| MCP tools | 7.2k | 3.6% |
| Custom agents | 1.2k | 0.6% |
| Memory files | 18.9k | 9.5% |
| Skills | 1.5k | 0.8% |
| Messages | 45.6k | 22.8% |
| **Free space** | 73k | 36.5% |
| Autocompact buffer | 33k | 16.5% |

**총 사용**: 102k / 200k tokens (51%)

---

## 🎯 최적화 기회

### 1. Memory Files 최적화 (18.9k tokens → 목표: 12k tokens)

#### 현재 상태
- `C:\Users\jh952\.claude\CLAUDE.md`: 6.6k tokens (글로벌 OMC 설정)
- `CLAUDE.md`: 5.6k tokens (프로젝트 가이드 - 구버전)
- `.claude\CLAUDE.md`: 6.6k tokens (중복)

#### 문제점
1. **중복**: 프로젝트 루트의 `CLAUDE.md`가 업데이트되기 전 버전
2. **불필요한 로딩**: 3개 파일이 모두 로드됨 (총 18.9k)
3. **내용 중복**: 일반 가이드가 중복 포함

#### 최적화 방안

**A. 파일 구조 정리**
```
C:\Users\jh952\.claude\CLAUDE.md  (6.6k) ← 글로벌 OMC 설정 유지
F:\workplace\claude\CLAUDE.md     (8.5k) ← 프로젝트별 내용 (업데이트됨)
F:\workplace\claude\.claude\CLAUDE.md ← 삭제 또는 심볼릭 링크
```

**B. 프로젝트 CLAUDE.md 압축 (8.5k → 6k tokens)**

압축 대상:
- 아키텍처 다이어그램 (ASCII art) 제거 → 간단한 텍스트로
- 코드 예시 압축 (주석 제거, 핵심만)
- 중복 설명 제거 (예: 명명 규칙 표 → 간단한 목록)
- Anti-patterns 섹션 압축

**절약 예상**: 18.9k → 12.6k (6.3k tokens 절약, 33% 감소)

---

### 2. Application DTO OpenAPI 어노테이션 제거 (아키텍처 개선 + Token 절약)

#### 현재 문제
```java
// application/dto/UserResponse.java
@Schema(description = "사용자 응답")  // ← Application 레이어가 OpenAPI에 의존
public record UserResponse(
    @Schema(description = "사용자 ID", example = "user-123")
    String id,

    @Schema(description = "이메일", example = "user@example.com")
    String email,

    @Schema(description = "이름", example = "홍길동")
    String name
) {}
```

**문제점**:
1. Application 레이어가 OpenAPI 라이브러리에 의존 (아키텍처 위반)
2. 3개 파일 × 평균 15줄 = 45줄 불필요한 어노테이션
3. 파일 크기 약 40% 증가

#### 해결 방안

**옵션 A: HTTP Response DTO 별도 생성 (권장)**

```java
// application/dto/UserResponse.java (어노테이션 제거)
public record UserResponse(
    String id,
    String email,
    String name
) {
    public static UserResponse from(User user) { ... }
}

// adapter/inbound/rest/UserHttpResponse.java (신규 생성)
@Schema(description = "사용자 응답")
public record UserHttpResponse(
    @Schema(description = "사용자 ID", example = "user-123")
    String id,

    @Schema(description = "이메일", example = "user@example.com")
    String email,

    @Schema(description = "이름", example = "홍길동")
    String name
) {
    public static UserHttpResponse from(UserResponse response) {
        return new UserHttpResponse(
            response.id(),
            response.email(),
            response.name()
        );
    }
}
```

**Controller 변경**:
```java
@GetMapping("/{id}")
public UserHttpResponse findUser(@PathVariable String id) {
    UserResponse response = findUserUseCase.execute(id);
    return UserHttpResponse.from(response);  // 변환 추가
}
```

**장점**:
- ✅ 아키텍처 원칙 준수 (Application 레이어 독립성)
- ✅ Application DTO 파일 크기 40% 감소
- ✅ 명확한 레이어 분리

**단점**:
- ⚠️ DTO 클래스 증가 (3개 → 6개)
- ⚠️ 변환 로직 추가 (Controller에서 한 줄)

**옵션 B: 현행 유지 + 문서화**

Application DTO에 `@Schema`를 허용하되, CLAUDE.md에 명시적으로 문서화:
- "Application Response DTO는 Controller에서 직접 반환되므로 예외적으로 OpenAPI 어노테이션 허용"

**Token 절약 예상**:
- 옵션 A: Application DTO 3개 × 15줄 = 45줄 제거 → 약 500 tokens 절약
- 옵션 B: 0 tokens (변화 없음)

**권장**: 옵션 A (아키텍처 개선 우선)

---

### 3. Skills 파일 최적화 (1.5k tokens → 목표: 1k tokens)

#### 현재 상태
5개 스킬 파일, 각 2-3.5KB

#### 최적화 방안
- 코드 예시 축약 (주석 제거)
- 중복 설명 제거
- Markdown 테이블 압축

**절약 예상**: 1.5k → 1k (500 tokens 절약, 33% 감소)

---

### 4. Custom Agents 로딩 최적화 (1.2k tokens - 현재 양호)

OMC 플러그인의 모든 에이전트(33개)가 로드되지만 1.2k tokens만 사용 중.
→ 추가 최적화 불필요

---

### 5. MCP Tools 최적화 (7.2k tokens - 현재 양호)

41개 MCP tool이 로드되지만 7.2k tokens만 사용 중.
→ 추가 최적화 불필요

---

## 📈 최적화 요약

| 항목 | 현재 | 최적화 후 | 절약 |
|------|------|----------|------|
| Memory files | 18.9k | 12.6k | **-6.3k** (33%) |
| Application DTOs | 포함 | 어노테이션 제거 | **-0.5k** (압축) |
| Skills | 1.5k | 1k | **-0.5k** (33%) |
| **총계** | **102k** | **~95k** | **-7k** (7% 감소) |

**최종 사용률**: 95k / 200k (47.5%) ← 51%에서 감소

---

## ✅ 즉시 실행 가능한 액션

### 우선순위 1: 중복 파일 제거
```bash
# .claude/CLAUDE.md 삭제 (프로젝트 루트 CLAUDE.md와 중복)
rm F:\workplace\claude\.claude\CLAUDE.md
```
**절약**: ~6k tokens

### 우선순위 2: CLAUDE.md 압축
- ASCII art 다이어그램 제거 → 텍스트 설명
- 코드 예시 주석 제거
- 중복 섹션 통합

**절약**: ~2k tokens

### 우선순위 3: Application DTO OpenAPI 어노테이션 제거
- HTTP Response DTO를 Adapter 레이어에 별도 생성
- Controller에서 변환 로직 추가
- Application DTO에서 `@Schema` 제거

**절약**: ~0.5k tokens
**부가 효과**: 아키텍처 원칙 준수

### 우선순위 4: Skills 파일 압축
- 코드 예시 축약
- 중복 설명 제거

**절약**: ~0.5k tokens

---

## 🎯 장기 최적화 전략

### 1. 프로젝트별 컨텍스트 로딩
현재 모든 메모리 파일이 자동 로드됨. 프로젝트별로 필요한 파일만 로드하도록 설정:

```yaml
# .claude/config.yml
context:
  files:
    - CLAUDE.md  # 프로젝트 가이드만
    - AGENTS.md  # 필요시에만
```

### 2. 동적 Skills 로딩
모든 스킬을 항상 로드하지 않고, 트리거 패턴 매칭 시에만 로드

### 3. Lazy Agent Loading
33개 에이전트 정의를 필요할 때만 로드

---

## 📝 실행 계획

1. **즉시 실행** (5분):
   - `.claude/CLAUDE.md` 삭제
   - 절약: 6k tokens

2. **단기 실행** (1시간):
   - CLAUDE.md 압축 (다이어그램, 예시 축약)
   - 절약: 2k tokens

3. **중기 실행** (3시간):
   - Application DTO 리팩토링 (HTTP Response DTO 분리)
   - Skills 파일 압축
   - 절약: 1k tokens

**총 절약**: 9k tokens (현재의 9%)

---

## 🔍 모니터링

최적화 후 `/context` 명령으로 확인:
- Memory files: 18.9k → 12k 목표
- 전체 사용률: 51% → 48% 이하 목표

---

**결론**: 즉시 실행 가능한 중복 제거만으로도 6k tokens (6%) 절약 가능. 아키텍처 개선(Application DTO 리팩토링)을 포함하면 총 9k tokens 절약 달성 가능.
