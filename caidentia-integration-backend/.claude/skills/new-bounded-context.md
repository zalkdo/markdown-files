---
name: new-bounded-context
description: Create a new Bounded Context with complete Hexagonal Architecture structure
triggers:
  - "new bounded context"
  - "create bounded context"
  - "add bounded context"
---

# New Bounded Context 생성

사용자가 새로운 Bounded Context 생성을 요청했습니다.

## 작업 순서

1. **Context 이름 확인**
   - Bounded Context 이름을 단수형 명사로 확인 (예: product, payment, notification)
   - PascalCase로 변환 (예: Product, Payment, Notification)

2. **패키지 구조 생성**
   ```
   src/main/java/com/example/myapp/{context}/
   ├── domain/
   │   ├── model/
   │   ├── event/
   │   ├── exception/
   │   └── repository/
   ├── application/
   │   ├── usecase/
   │   ├── dto/
   │   └── port/out/
   └── adapter/
       ├── inbound/rest/
       └── outbound/persistence/
   ```

3. **Domain Layer 생성**
   - ID Value Object: `{Context}Id.java` (record, UUID 기반)
   - Aggregate Root: `{Context}.java` (private 생성자, create(), reconstruct())
   - Status Enum (필요시): `{Context}Status.java`
   - Domain Exception: `{Context}NotFoundException.java`
   - Repository Interface: `{Context}Repository.java`

4. **Flyway Migration 생성**
   - 파일명: `V{next_number}__create_{context}_table.sql`
   - 테이블명: `{context}s` (복수형)
   - 기본 컬럼: id (VARCHAR PRIMARY KEY), status (VARCHAR), created_at, updated_at

5. **Persistence Layer 생성**
   - JPA Entity: `{Context}Entity.java` (@Entity, @Table)
   - JPA Repository: `{Context}JpaRepository.java` (extends JpaRepository)
   - Mapper: `{Context}PersistenceMapper.java` (static toEntity/toDomain)
   - Repository Impl: `{Context}RepositoryImpl.java` (@Repository, implements)

6. **Application Layer 생성**
   - Request DTO: `Create{Context}Request.java` (record)
   - Response DTO: `{Context}Response.java` (record, static from())
   - Use Case: `Create{Context}UseCase.java` (@Service, @Transactional)
   - Use Case: `Find{Context}UseCase.java` (@Service, @Transactional(readOnly=true))

7. **Adapter Inbound 생성**
   - HTTP Request: `Create{Context}HttpRequest.java` (record, @Schema, Jakarta validation)
   - Controller: `{Context}Controller.java` (@RestController, @Tag, OpenAPI annotations)

8. **테스트 생성**
   - Domain Test: `{Context}Test.java` (JUnit 5, @DisplayName)
   - Use Case Test: `Create{Context}UseCaseTest.java` (Mockito)

## 체크리스트
- [ ] Domain 레이어에 Spring/JPA 임포트 없음
- [ ] Repository는 Aggregate Root 단위로만
- [ ] 모든 DTO는 record 사용
- [ ] 팩토리 메서드 패턴 (create, reconstruct)
- [ ] OpenAPI 문서화 완료
- [ ] 기본 테스트 작성

생성 후 사용자에게 생성된 파일 목록과 다음 단계를 안내하세요.
