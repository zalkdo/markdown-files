---
name: create-domain-test
description: Create unit tests for Domain layer (Aggregate, Value Object)
triggers:
  - "domain test"
  - "test domain"
  - "aggregate test"
---

# Domain Layer 테스트 생성

Aggregate Root, Entity, Value Object에 대한 단위 테스트를 생성합니다.

## 작업 순서

1. **테스트 대상 확인**
   - Aggregate Root의 비즈니스 메서드
   - Value Object의 유효성 검사
   - 팩토리 메서드 (create, reconstruct)
   - 불변성 규칙

2. **테스트 클래스 생성**
   ```java
   class {Aggregate}Test {

       @Test
       @DisplayName("정상적인 데이터로 생성 성공")
       void create_validData_success() {
           // given
           var field1 = ...;
           var field2 = ...;

           // when
           var aggregate = {Aggregate}.create(field1, field2);

           // then
           assertThat(aggregate.id()).isNotNull();
           assertThat(aggregate.field1()).isEqualTo(field1);
           assertThat(aggregate.status()).isEqualTo({Status}.CREATED);
       }

       @Test
       @DisplayName("null 값으로 생성 시 예외 발생")
       void create_nullValue_throwsException() {
           // when & then
           assertThatThrownBy(() -> {Aggregate}.create(null, ...))
               .isInstanceOf(NullPointerException.class)
               .hasMessageContaining("...");
       }

       @Test
       @DisplayName("비즈니스 행위 - 정상 케이스")
       void businessMethod_validState_success() {
           // given
           var aggregate = {Aggregate}.create(...);

           // when
           aggregate.businessMethod();

           // then
           assertThat(aggregate.status()).isEqualTo({Status}.EXPECTED);
       }

       @Test
       @DisplayName("비즈니스 행위 - 불가능한 상태에서 예외 발생")
       void businessMethod_invalidState_throwsException() {
           // given
           var aggregate = {Aggregate}.create(...);
           aggregate.changeToInvalidState();

           // when & then
           assertThatThrownBy(() -> aggregate.businessMethod())
               .isInstanceOf({Custom}Exception.class)
               .hasMessageContaining("...");
       }
   }
   ```

3. **Value Object 테스트**
   ```java
   class {ValueObject}Test {

       @Test
       @DisplayName("유효한 값으로 생성 성공")
       void of_validValue_success() {
           var value = "valid-value";
           var vo = {ValueObject}.of(value);
           assertThat(vo.value()).isEqualTo(value);
       }

       @Test
       @DisplayName("유효하지 않은 값으로 생성 시 예외")
       void of_invalidValue_throwsException() {
           assertThatThrownBy(() -> {ValueObject}.of("invalid"))
               .isInstanceOf(IllegalArgumentException.class);
       }
   }
   ```

4. **테스트 실행**
   - Domain 테스트는 Spring Context 없이 실행 (매우 빠름)
   - `./gradlew test --tests *{Aggregate}Test`

## 테스트 작성 체크리스트
- [ ] 팩토리 메서드 (create) 정상 케이스
- [ ] 팩토리 메서드 null/invalid 케이스
- [ ] 각 비즈니스 메서드 정상 케이스
- [ ] 각 비즈니스 메서드 예외 케이스
- [ ] Value Object 유효성 검사
- [ ] 불변성 확인 (필요시)

Domain 테스트는 가장 중요한 테스트입니다. 충분한 커버리지를 확보하세요.
