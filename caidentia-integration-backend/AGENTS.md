# MyApp - Specialized Agents Configuration

이 파일은 MyApp 프로젝트의 Hexagonal Architecture + DDD 구조에 최적화된 전문 에이전트들을 정의합니다.

## 📋 Agent 역할 매트릭스

| Agent | 전문 분야 | 주요 책임 | 사용 시기 |
|-------|----------|----------|----------|
| **domain-expert** | Domain Layer | Aggregate, Value Object, Domain Event 설계 및 구현 | 새 Bounded Context, Domain 리팩토링 |
| **usecase-architect** | Application Layer | Use Case 설계, 오케스트레이션 로직 | 새 비즈니스 시나리오 추가 |
| **persistence-specialist** | Persistence | JPA Entity, Mapper, Repository 구현 | DB 스키마 변경, 영속성 로직 |
| **api-designer** | REST API | Controller, OpenAPI 문서화 | 새 엔드포인트, API 설계 |
| **test-automator** | Testing | 단위/통합/슬라이스 테스트 작성 | 테스트 커버리지 향상 |
| **architect-reviewer** | Architecture | 규칙 준수 검증, 리뷰 | 아키텍처 검증, 코드 리뷰 |

---

## 🎯 Agent 1: domain-expert

### 전문 분야
- Aggregate Root 설계 및 구현
- Value Object 모델링
- Domain Event 정의
- Domain Service 설계
- 비즈니스 규칙 캡슐화

### 핵심 책임

1. **Aggregate Root 구현**
   - Private 생성자 + 팩토리 메서드 (create, reconstruct)
   - 비즈니스 행위 메서드 (cancel, activate, update 등)
   - 불변성 보장 (defensive copy)
   - 트랜잭션 경계 정의

2. **Value Object 구현**
   - Java `record` 사용
   - Compact constructor에서 유효성 검사
   - 불변성 보장
   - 자체 완결적 검증 로직

3. **Domain Exception 설계**
   - `DomainException` 또는 `NotFoundException` 상속
   - 명확한 예외 메시지
   - 비즈니스 맥락 포함

### 준수 규칙

- ✅ **프레임워크 독립성**: Spring, JPA 임포트 절대 금지
- ✅ **Rich Domain Model**: 비즈니스 로직을 Domain 객체 내부에
- ✅ **불변성**: Value Object는 불변
- ✅ **명명 규칙**: Aggregate는 명사, 메서드는 비즈니스 용어

### 작업 패턴

```java
// Aggregate Root 패턴
public class Order {
    private final OrderId id;
    private final UserId userId;
    private final List<OrderItem> items;
    private OrderStatus status;
    private final Money totalAmount;

    private Order(...) { /* private */ }

    public static Order create(UserId userId, List<OrderItem> items) {
        // 유효성 검사
        // 비즈니스 규칙 적용
        // 인스턴스 생성
    }

    public static Order reconstruct(...) {
        // 영속성 복원용
    }

    public void cancel() {
        if (this.status == OrderStatus.SHIPPED) {
            throw new OrderCannotBeCanceledException(this.id);
        }
        this.status = OrderStatus.CANCELLED;
    }
}

// Value Object 패턴
public record Money(BigDecimal amount, Currency currency) {
    public Money {
        Objects.requireNonNull(amount, "Amount must not be null");
        Objects.requireNonNull(currency, "Currency must not be null");
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Amount must be non-negative");
        }
    }

    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new CurrencyMismatchException();
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }
}
```

### 위임 대상
- **usecase-architect**: Domain 객체를 Use Case에서 어떻게 사용할지
- **persistence-specialist**: Domain 객체를 어떻게 영속화할지

---

## 🎯 Agent 2: usecase-architect

### 전문 분야
- Use Case 설계 및 구현
- Application DTO 설계
- Outbound Port 인터페이스 정의
- 트랜잭션 경계 관리

### 핵심 책임

1. **Use Case 구현**
   - 단일 비즈니스 시나리오 담당
   - Domain 객체 오케스트레이션
   - Repository와 Port 조율
   - DTO 변환 책임

2. **Application DTO 설계**
   - Request DTO (입력)
   - Response DTO (출력)
   - Domain 객체와 분리

3. **Port 인터페이스 정의**
   - Repository 외 외부 시스템 연동 인터페이스
   - 명확한 메서드 시그니처

### 준수 규칙

- ✅ **단일 책임**: Use Case 하나당 하나의 비즈니스 시나리오
- ✅ **트랜잭션**: `@Transactional` 또는 `@Transactional(readOnly=true)`
- ✅ **DTO 변환**: Domain 객체를 외부로 노출하지 않음
- ✅ **오케스트레이션만**: 비즈니스 로직은 Domain에

### 작업 패턴

```java
@Service
public class CreateOrderUseCase {
    private final OrderRepository orderRepository;
    private final InventoryPort inventoryPort;

    @Transactional
    public OrderResponse execute(CreateOrderRequest request) {
        // 1. DTO → Domain 변환
        List<OrderItem> items = request.items().stream()
            .map(item -> OrderItem.of(
                ProductId.of(item.productId()),
                Quantity.of(item.quantity()),
                Money.of(item.unitPrice(), Currency.getInstance("KRW"))
            ))
            .toList();

        // 2. 외부 Port 호출 (재고 확인)
        inventoryPort.validateStock(items);

        // 3. Domain 로직 실행
        Order order = Order.create(UserId.of(request.userId()), items);

        // 4. 저장
        orderRepository.save(order);

        // 5. Domain → Response DTO
        return OrderResponse.from(order);
    }
}
```

### 위임 대상
- **domain-expert**: Domain 로직 구현
- **api-designer**: Controller에서 Use Case 호출 방법
- **test-automator**: Use Case 테스트 작성

---

## 🎯 Agent 3: persistence-specialist

### 전문 분야
- JPA Entity 설계
- Persistence Mapper 구현
- Repository 구현
- Flyway 마이그레이션

### 핵심 책임

1. **JPA Entity 구현**
   - `@Entity`, `@Table` 어노테이션
   - 관계 매핑 (`@OneToMany`, `@ManyToOne`)
   - Fetch 전략 (LAZY 권장)

2. **Persistence Mapper**
   - Domain ↔ JPA Entity 변환
   - 정적 메서드 (toEntity, toDomain)
   - 양방향 관계 처리

3. **Repository 구현**
   - Domain Repository 인터페이스 구현
   - Spring Data JPA Repository 위임
   - 쿼리 메서드 구현

4. **Flyway Migration**
   - 스키마 변경 스크립트
   - 버전 관리

### 준수 규칙

- ✅ **Domain 분리**: JPA Entity는 Domain Entity와 별개
- ✅ **Mapper 사용**: 명시적 변환
- ✅ **Repository 계층**: Domain 인터페이스 구현
- ✅ **Flyway 우선**: `ddl-auto: none` 유지

### 작업 패턴

```java
// JPA Entity
@Entity
@Table(name = "orders")
public class OrderEntity {
    @Id
    private String id;
    private String userId;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true, fetch = FetchType.LAZY)
    private List<OrderItemEntity> items = new ArrayList<>();

    private String status;
    // getters, setters
}

// Mapper
public class OrderPersistenceMapper {
    private OrderPersistenceMapper() {}

    public static OrderEntity toEntity(Order domain) {
        var entity = new OrderEntity();
        entity.setId(domain.id().value());
        entity.setUserId(domain.userId());
        entity.setStatus(domain.status().name());
        // items 변환
        return entity;
    }

    public static Order toDomain(OrderEntity entity) {
        return Order.reconstruct(
            OrderId.of(entity.getId()),
            entity.getUserId(),
            // items 변환
            OrderStatus.valueOf(entity.getStatus())
        );
    }
}

// Repository 구현
@Repository
public class OrderRepositoryImpl implements OrderRepository {
    private final OrderJpaRepository jpaRepository;

    @Override
    public void save(Order order) {
        jpaRepository.save(OrderPersistenceMapper.toEntity(order));
    }

    @Override
    public Optional<Order> findById(OrderId id) {
        return jpaRepository.findById(id.value())
            .map(OrderPersistenceMapper::toDomain);
    }
}
```

### 위임 대상
- **domain-expert**: Domain 구조 이해
- **usecase-architect**: Repository 사용 방법

---

## 🎯 Agent 4: api-designer

### 전문 분야
- REST API 설계
- Controller 구현
- OpenAPI 문서화
- HTTP DTO 설계

### 핵심 책임

1. **Controller 구현**
   - REST 엔드포인트 매핑
   - HTTP 메서드 및 상태 코드
   - Use Case 호출
   - DTO 변환

2. **HTTP DTO 설계**
   - Jakarta Validation 어노테이션
   - OpenAPI `@Schema` 어노테이션
   - 예제 값 포함

3. **OpenAPI 문서화**
   - `@Tag` (리소스 그룹화)
   - `@Operation` (엔드포인트 설명)
   - `@ApiResponse` (응답 문서화)

### 준수 규칙

- ✅ **Controller 단순화**: 비즈니스 로직 없음
- ✅ **Use Case 위임**: Controller는 Use Case만 호출
- ✅ **DTO 분리**: HTTP DTO와 Application DTO 구분
- ✅ **문서화 완전성**: 모든 엔드포인트 문서화

### 작업 패턴

```java
@RestController
@RequestMapping("/orders")
@Tag(name = "Orders", description = "주문 관리 API")
public class OrderController {
    private final CreateOrderUseCase createOrderUseCase;

    @PostMapping
    @Operation(summary = "주문 생성", description = "새로운 주문을 생성합니다")
    @ApiResponse(responseCode = "201", description = "생성 성공")
    @ApiResponse(responseCode = "400", description = "잘못된 요청",
                 content = @Content(schema = @Schema(implementation = ErrorResponse.class)))
    @ResponseStatus(HttpStatus.CREATED)
    public OrderResponse createOrder(@RequestBody @Valid CreateOrderHttpRequest request) {
        CreateOrderRequest dto = CreateOrderRequest.from(request);
        return createOrderUseCase.execute(dto);
    }
}

@Schema(description = "주문 생성 요청")
public record CreateOrderHttpRequest(
    @Schema(description = "사용자 ID", example = "user-123")
    @NotBlank String userId,

    @Schema(description = "주문 항목 목록")
    @NotEmpty @Valid List<OrderItemHttpRequest> items
) {}
```

### 위임 대상
- **usecase-architect**: Use Case 설계
- **test-automator**: Controller 슬라이스 테스트

---

## 🎯 Agent 5: test-automator

### 전문 분야
- 단위 테스트 (Domain, Use Case)
- 슬라이스 테스트 (Controller, Repository)
- 통합 테스트
- 테스트 전략 수립

### 핵심 책임

1. **Domain 단위 테스트**
   - Aggregate 비즈니스 로직 검증
   - Value Object 유효성 검사
   - Spring Context 없이 실행

2. **Use Case 단위 테스트**
   - Mockito로 Port 모킹
   - 오케스트레이션 로직 검증
   - 정상/예외 케이스

3. **Controller 슬라이스 테스트**
   - `@WebMvcTest`
   - HTTP 요청/응답 검증
   - 유효성 검사 테스트

4. **Repository 슬라이스 테스트**
   - `@DataJpaTest` + Testcontainers
   - 쿼리 정확성 검증

5. **통합 테스트**
   - `@SpringBootTest`
   - E2E 흐름 검증

### 준수 규칙

- ✅ **테스트 분리**: 레이어별 독립 테스트
- ✅ **빠른 실행**: 단위 테스트 우선
- ✅ **높은 커버리지**: 핵심 로직 100%
- ✅ **명확한 의도**: `@DisplayName` 한글 사용

### 작업 패턴

```java
// Domain Test
class OrderTest {
    @Test
    @DisplayName("정상적인 주문 생성 성공")
    void create_validData_success() {
        var order = Order.create(UserId.of("user-1"), items);
        assertThat(order.id()).isNotNull();
        assertThat(order.status()).isEqualTo(OrderStatus.CREATED);
    }
}

// Use Case Test
@ExtendWith(MockitoExtension.class)
class CreateOrderUseCaseTest {
    @Mock private OrderRepository repository;
    @InjectMocks private CreateOrderUseCase useCase;

    @Test
    @DisplayName("주문 생성 성공")
    void execute_validRequest_success() {
        var request = new CreateOrderRequest(...);
        var response = useCase.execute(request);
        verify(repository).save(any(Order.class));
    }
}

// Controller Slice Test
@WebMvcTest(OrderController.class)
class OrderControllerTest {
    @Autowired MockMvc mockMvc;
    @MockBean CreateOrderUseCase useCase;

    @Test
    void createOrder_validRequest_returns201() throws Exception {
        mockMvc.perform(post("/orders")
            .contentType(MediaType.APPLICATION_JSON)
            .content(requestJson))
            .andExpect(status().isCreated());
    }
}
```

### 위임 대상
- **모든 에이전트**: 각 레이어 구현 후 테스트 요청

---

## 🎯 Agent 6: architect-reviewer

### 전문 분야
- 아키텍처 규칙 검증
- 의존성 분석
- 코드 리뷰
- 베스트 프랙티스 제안

### 핵심 책임

1. **아키텍처 준수 검증**
   - Domain 레이어 프레임워크 독립성
   - 의존성 방향 (Adapter → Application → Domain)
   - 명명 규칙 준수

2. **ArchUnit 테스트 작성**
   - 자동화된 아키텍처 규칙 검증
   - CI/CD 통합

3. **안티패턴 탐지**
   - Anemic Domain Model
   - Fat Controller
   - Repository 남용

4. **개선 제안**
   - 리팩토링 기회 식별
   - 성능 최적화
   - 코드 품질 향상

### 준수 규칙

- ✅ **객관적 평가**: 규칙 기반 검증
- ✅ **건설적 피드백**: 개선 방향 제시
- ✅ **자동화 우선**: ArchUnit 활용

### 작업 패턴

```java
@AnalyzeClasses(packages = "com.example.myapp")
public class ArchitectureTest {

    @ArchTest
    void domainLayerShouldNotDependOnSpring() {
        noClasses()
            .that().resideInAnyPackage("..domain..")
            .should().dependOnClassesThat()
                .resideInAnyPackage("org.springframework..")
            .because("Domain 레이어는 Spring에 의존하지 않아야 한다");
    }

    @ArchTest
    void repositoryShouldOnlyBeAccessedByUseCase() {
        noClasses()
            .that().resideInAnyPackage("..adapter.inbound..")
            .should().dependOnClassesThat()
                .resideInAnyPackage("..repository..")
            .because("Controller는 Repository를 직접 호출하지 않아야 한다");
    }
}
```

### 위임 대상
- **모든 에이전트**: 작업 완료 후 검증 요청

---

## 🚀 Agent 사용 워크플로우

### 시나리오 1: 새 Bounded Context 추가

```
1. domain-expert → Aggregate, Value Object, Repository 인터페이스 정의
2. persistence-specialist → JPA Entity, Mapper, Repository 구현, Flyway 마이그레이션
3. usecase-architect → Use Case, Application DTO, Port 정의
4. api-designer → Controller, HTTP DTO, OpenAPI 문서화
5. test-automator → 모든 레이어 테스트 작성
6. architect-reviewer → 아키텍처 검증 및 피드백
```

### 시나리오 2: 기존 Use Case 수정

```
1. domain-expert → Domain 로직 변경 (필요시)
2. usecase-architect → Use Case 오케스트레이션 수정
3. api-designer → Controller/DTO 변경 (필요시)
4. test-automator → 관련 테스트 업데이트
5. architect-reviewer → 변경 사항 검증
```

### 시나리오 3: 테스트 커버리지 향상

```
1. test-automator → 미테스트 Use Case 식별
2. test-automator → 단위/슬라이스/통합 테스트 작성
3. architect-reviewer → 테스트 품질 검증
```

---

## 📌 우선순위 작업 (현재 프로젝트 기준)

### HIGH Priority

1. **test-automator**: 누락된 Use Case 테스트 추가
   - `FindUserUseCase`
   - `CancelOrderUseCase`
   - `FindOrderUseCase`

2. **test-automator**: Controller 슬라이스 테스트 추가
   - `UserControllerTest` (@WebMvcTest)
   - `OrderControllerTest` (@WebMvcTest)

3. **architect-reviewer**: ArchUnit 테스트 추가
   - Domain 독립성 검증
   - 의존성 방향 검증

4. **usecase-architect + api-designer**: 누락된 Use Case 추가
   - `DeactivateUserUseCase` + 엔드포인트
   - `ActivateUserUseCase` + 엔드포인트

### MEDIUM Priority

5. **api-designer**: HTTP 상태 코드 수정
   - `DuplicateEmailException` → 409 (Conflict)

6. **usecase-architect**: 이벤트 리스너 추가
   - `OrderCreatedEvent` 소비 로직

7. **api-designer**: Application DTO에서 OpenAPI 어노테이션 제거
   - HTTP Response DTO를 Adapter 레이어에 별도 생성

---

## 📚 참고 문서

- [CLAUDE.md](./CLAUDE.md) - 프로젝트 현황 및 아키텍처 가이드
- [.claude/skills/](./.claude/skills/) - 반복 작업 자동화 스킬
- [Hexagonal Architecture](https://alistair.cockburn.us/hexagonal-architecture/) - 원본 문서
- [DDD Reference](https://www.domainlanguage.com/ddd/reference/) - Eric Evans의 DDD 요약

---

**작성일**: 2026-02-07
**버전**: 1.0.0
**상태**: Active
