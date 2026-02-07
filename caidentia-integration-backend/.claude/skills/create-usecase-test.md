---
name: create-usecase-test
description: Create unit tests for Use Case with Mockito
triggers:
  - "usecase test"
  - "test usecase"
  - "use case test"
---

# Use Case 테스트 생성

Use Case의 오케스트레이션 로직을 Mockito로 테스트합니다.

## 작업 순서

1. **테스트 클래스 생성**
   ```java
   @ExtendWith(MockitoExtension.class)
   class {Verb}{Noun}UseCaseTest {

       @Mock
       private {Noun}Repository repository;

       @Mock
       private {Other}Port otherPort;  // 필요시

       @InjectMocks
       private {Verb}{Noun}UseCase useCase;

       @Test
       @DisplayName("정상적인 요청으로 실행 성공")
       void execute_validRequest_success() {
           // given
           var request = new {Verb}{Noun}Request(...);
           var expectedDomain = {Noun}.create(...);

           given(repository.someMethod(...))
               .willReturn(Optional.of(expectedDomain));

           // when
           var response = useCase.execute(request);

           // then
           assertThat(response.id()).isEqualTo(expectedDomain.id().value());
           assertThat(response.field()).isEqualTo(expectedDomain.field());

           verify(repository).save(any({Noun}.class));
           verify(otherPort, never()).someMethod();
       }

       @Test
       @DisplayName("존재하지 않는 리소스 조회 시 예외 발생")
       void execute_notFound_throwsException() {
           // given
           var request = new {Verb}{Noun}Request(...);
           given(repository.findById(...))
               .willReturn(Optional.empty());

           // when & then
           assertThatThrownBy(() -> useCase.execute(request))
               .isInstanceOf({Noun}NotFoundException.class);
       }

       @Test
       @DisplayName("비즈니스 규칙 위반 시 예외 발생")
       void execute_businessRuleViolation_throwsException() {
           // given - 비즈니스 규칙을 위반하는 상태 설정

           // when & then
           assertThatThrownBy(() -> useCase.execute(request))
               .isInstanceOf({Custom}Exception.class);
       }
   }
   ```

2. **Mockito 사용 패턴**
   - `given(...).willReturn(...)` - Stub 설정
   - `given(...).willThrow(...)` - 예외 발생 설정
   - `verify(...)` - 메서드 호출 검증
   - `verify(..., never())` - 호출되지 않음 검증
   - `any()`, `eq()` - ArgumentMatcher

3. **테스트 케이스**
   - **정상 케이스**: 모든 조건이 만족되는 경우
   - **Not Found**: Repository가 empty 반환
   - **비즈니스 규칙 위반**: Domain Exception 발생
   - **외부 Port 실패**: Port에서 예외 발생 (있는 경우)

4. **테스트 실행**
   ```bash
   ./gradlew test --tests *{Verb}{Noun}UseCaseTest
   ```

## 주의사항
- Use Case 테스트는 오케스트레이션 검증에 집중
- 비즈니스 로직 검증은 Domain 테스트에서
- Repository는 항상 모킹 (실제 DB 사용 안 함)
- Spring Context 불필요 (빠른 실행)

생성 후 5개 Use Case 중 테스트가 없는 3개에 우선 적용하세요:
- FindUserUseCase
- CancelOrderUseCase
- FindOrderUseCase
