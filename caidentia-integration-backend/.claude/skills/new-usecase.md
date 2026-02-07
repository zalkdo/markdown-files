---
name: new-usecase
description: Add a new Use Case to an existing Bounded Context
triggers:
  - "new use case"
  - "add use case"
  - "create use case"
---

# New Use Case 추가

기존 Bounded Context에 새로운 Use Case를 추가합니다.

## 작업 순서

1. **Use Case 정보 확인**
   - Bounded Context 이름
   - 동사 (Register, Create, Update, Delete, Find, Cancel 등)
   - 명사 (User, Order, Payment 등)
   - Use Case 이름: `{Verb}{Noun}UseCase`

2. **Application DTO 생성 (필요시)**
   - Request DTO: `{Verb}{Noun}Request.java` (record)
   - Response DTO는 기존 `{Noun}Response.java` 재사용 또는 새로 생성

3. **Use Case 클래스 생성**
   ```java
   @Service
   public class {Verb}{Noun}UseCase {
       private final {Noun}Repository repository;
       // 다른 Port 주입

       @Transactional // 또는 @Transactional(readOnly=true)
       public {Noun}Response execute({Verb}{Noun}Request request) {
           // 1. DTO → Domain
           // 2. Domain 로직 실행
           // 3. Repository 저장/조회
           // 4. Domain → Response DTO
       }
   }
   ```

4. **HTTP DTO 생성 (새 엔드포인트 필요시)**
   - HTTP Request: `{Verb}{Noun}HttpRequest.java`
   - Jakarta Validation 어노테이션 추가
   - OpenAPI @Schema 어노테이션 추가

5. **Controller 메서드 추가**
   - 기존 Controller에 새 메서드 추가
   - HTTP 메서드, 경로, 상태 코드 지정
   - @Operation, @ApiResponses 문서화

6. **테스트 생성**
   - Use Case 단위 테스트
   - Mockito로 Repository 모킹
   - 성공 케이스 + 실패 케이스

## 패턴별 Use Case

### 조회 (Find)
- `@Transactional(readOnly = true)`
- Repository에서 조회 → Response DTO 변환

### 생성 (Create/Register)
- `@Transactional`
- Domain 팩토리 메서드 호출 → Repository 저장

### 수정 (Update)
- `@Transactional`
- Repository 조회 → Domain 메서드 호출 → Repository 저장

### 삭제 (Delete/Cancel)
- `@Transactional`
- Repository 조회 → 상태 변경 또는 삭제 → Repository 저장

생성 후 테스트를 실행하여 정상 동작을 확인하세요.
