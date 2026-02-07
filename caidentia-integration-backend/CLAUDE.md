# MyApp - Spring Boot Hexagonal Architecture 프로젝트

## 📋 프로젝트 현황

### 개요
- **프로젝트명**: MyApp
- **아키텍처**: Hexagonal Architecture (Ports & Adapters) + DDD
- **개발 단계**: 초중기 (Foundation Complete, Features In Progress)
- **아키텍처 준수도**: ✅ 100% (Domain 레이어 프레임워크 독립성, 의존성 방향, 명명 규칙 모두 준수)


### 기술 스택

| 영역 | 기술 | 버전 |
|------|------|------|
| Framework | Spring Boot | 3.3.7 |
| Language | Java | 21 (LTS) |
| Build | Gradle | 8.12 |
| Database | H2 (development) | in-memory |
| Migration | Flyway | (Spring Boot 관리) |
| ORM | Spring Data JPA + Hibernate | (Spring Boot 관리) |
| Validation | Jakarta Validation | (Spring Boot 관리) |
| API Docs | Springdoc OpenAPI | 2.3.0 |
| Testing | JUnit 5 + Mockito + AssertJ | (Spring Boot 관리) |


### 📚 현재 사용 중인 패턴

#### 1. ID Value Object 패턴
```java
public record UserId(String value) {
    public static UserId generate() { return new UserId(UUID.randomUUID().toString()); }
    public static UserId of(String value) { return new UserId(value); }
}
```
**사용**: `UserId`, `OrderId`, `ProductId`

#### 2. Aggregate Root 팩토리 패턴
```java
private Order(...) { /* private constructor */ }
public static Order create(...) { /* 생성 */ }
public static Order reconstruct(...) { /* 복원 */ }
```
**사용**: `User`, `Order`

#### 3. Use Case 패턴
```java
@Service
@Transactional
public class VerbNounUseCase {
    public XxxResponse execute(XxxRequest request) { ... }
}
```
**사용**: 모든 Use Case

#### 4. Persistence Adapter 패턴
- `XxxEntity` (JPA)
- `XxxJpaRepository` (Spring Data)
- `XxxPersistenceMapper` (정적 변환 메서드)
- `XxxRepositoryImpl` (Domain 인터페이스 구현)

#### 5. 예외 계층
```
DomainException (shared)
  ├── NotFoundException
  │     ├── UserNotFoundException
  │     └── OrderNotFoundException
  ├── DuplicateEmailException
  └── OrderCannotBeCanceledException
```

---

## 아키텍처 가이드

### 아키텍처 개요

```
┌─────────────────────────────────────────────────────────┐
│                    Infrastructure                        │
│  ┌─────────────┐                      ┌───────────────┐ │
│  │  Inbound    │                      │   Outbound    │ │
│  │  Adapters   │  ┌─────────────────┐ │   Adapters    │ │
│  │             │  │  Application    │ │               │ │
│  │ Controller  ├─►│  (Use Cases)    │ │  Repository   │ │
│  │ EventListener│ │                 │◄┤  Impl        │ │
│  │ CLI         │  │  ┌───────────┐  │ │  API Client   │ │
│  └─────────────┘  │  │  Domain   │  │ │  Publisher    │ │
│                   │  │  (Core)   │  │ └───────────────┘ │
│                   │  └───────────┘  │                   │
│                   └─────────────────┘                   │
└─────────────────────────────────────────────────────────┘

의존성 방향: Adapter → Application → Domain
Domain은 어떤 외부 레이어에도 의존하지 않음
```

---

## 패키지 구조

**Bounded Context를 최상위 단위로 사용합니다.** 각 컨텍스트 내부에서 Hexagonal 레이어를 구성합니다.

```
src/main/java/com/example/{service-name}/
│
├── {bounded-context-name}/                 # 예: order, user, product
│   │
│   ├── domain/                            # 내부(Inside) — 순수 비즈니스 로직
│   │   ├── model/                         # Aggregate Root, Entity, Value Object
│   │   ├── event/                         # Domain Event
│   │   ├── exception/                     # Domain 예외
│   │   ├── service/                       # Domain Service (단일 Aggregate에 속하지 못하는 로직만)
│   │   └── repository/                    # Repository 인터페이스 (Outbound Port)
│   │
│   ├── application/                       # Use Case 오케스트레이션 레이어
│   │   ├── usecase/                       # Use Case 클래스 (Input Port 역할)
│   │   ├── port/
│   │   │   └── out/                       # 외부 서비스 Port 인터페이스 (Repository 제외)
│   │   └── dto/                           # Use Case 입출력 DTO
│   │
│   └── adapter/                           # 외부(Outside) — 프레임워크와의 연결
│       ├── inbound/
│       │   └── rest/                      # Controller, Request/Response DTO
│       └── outbound/
│           ├── persistence/               # JPA Entity, Repository 구현, Mapper
│           └── messaging/                 # Event Publisher 구현, Kafka Producer 등
│
├── shared/                                # Shared Kernel — 컨텍스트 간 공유되는 개념
│   ├── domain/                            # 공유 Value Object, 기본 타입
│   └── exception/                         # 공통 예외
│
└── infrastructure/                        # Cross-cutting Concerns
    ├── config/                            # Spring 설정, Bean 정의
    ├── security/                          # 보안 설정
    ├── persistence/                       # Datasource, JPA 글로벌 설정, Flyway
    └── web/                               # GlobalExceptionHandler, WebMvcConfigurer
```

### 핵심 원칙
- 새 기능 추가 시 **해당 Bounded Context 폴더 안에서만** 작업
- `shared/`에 추가할 것은 반드시 여러 컨텍스트에서 실제로 사용되는 경우에만
- `infrastructure/`는 전체 프로젝트 공통 설정에만 사용

---

## Domain Layer 규칙

**Domain 레이어는 프레임워크와 완전히 분리된 순수 Java 코드입니다.**

### ✅ 반드시 준수
- 모든 클래스는 **Spring 어노테이션 없음** (`@Entity`, `@Service`, `@Component` 금지)
- **외부 라이브러리 임포트 금지** (`javax.persistence`, `org.springframework` 등)
- **Rich Domain Model**: 비즈니스 로직을 Aggregate/Entity 내부에 배치
- Value Object는 **불변(immutable)** — `final` 필드, setter 없음, `equals`/`hashCode` 구현
- Aggregate Root는 **트랜잭션과 일관성 경계**를 정의
- Repository 인터페이스는 **Aggregate Root 단위로 하나씩** 정의

### ❌ 금지 사항
- Domain Service에 비즈니스 규칙 과도 집중 (Anemic Model 방지)
- Aggregate Root가 아닌 단일 Entity용 Repository 생성
- 도메인 객체에서 외부 API 호출 또는 DB 접근

### 예시: Aggregate Root + Value Object

```java
// domain/model/Order.java — Aggregate Root
public class Order {
    private final OrderId id;                  // Value Object (Identity)
    private final UserId userId;
    private final List<OrderItem> items;       // Entity (Aggregate 내 관리)
    private OrderStatus status;
    private final Money totalAmount;           // Value Object

    // 팩토리 메서드로 생성
    public static Order create(UserId userId, List<OrderItem> items) {
        Money total = items.stream()
            .map(OrderItem::subtotal)
            .reduce(Money.ZERO, Money::add);
        return new Order(OrderId.generate(), userId, items, OrderStatus.CREATED, total);
    }

    // 비즈니스 행위는 여기에
    public void cancel() {
        if (this.status == OrderStatus.SHIPPED) {
            throw new OrderCannotBeCanceledException(this.id);
        }
        this.status = OrderStatus.CANCELLED;
        // Domain Event 등록 가능
    }
}

// domain/model/Money.java — Value Object (불변)
public record Money(BigDecimal amount, Currency currency) {
    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new CurrencyMismatchException();
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }
}
```

### 예시: Repository 인터페이스 (Outbound Port)

```java
// domain/repository/OrderRepository.java
public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(OrderId id);
    List<Order> findByUserId(UserId userId);
    void delete(OrderId id);
}
```

---

## Application Layer 규칙

**Use Case는 Domain 로직의 오케스트레이션만 담당합니다. 비즈니스 규칙은 여기에 없습니다.**

### ✅ 반드시 준수
- Use Case 하나당 **단일 비즈니스 시나리오** 담당
- `@Transactional`은 Use Case 메서드 단위에서 관리
- 입출력은 반드시 **DTO**로 변환 (Domain 객체를 밖으로 노출하지 않음)
- Repository와 외부 Port는 **인터페이스로 주입**

### ❌ 금지 사항
- 비즈니스 규칙, 유효성 검사 로직 배치 (→ Domain로)
- 직접 JPA/Infrastructure 클래스 임포트
- 여러 Use Case 로직을 하나의 클래스에 집중

### 예시

```java
// application/usecase/CreateOrderUseCase.java
@Service
public class CreateOrderUseCase {

    private final OrderRepository orderRepository;   // Domain의 Port
    private final InventoryPort inventoryPort;       // application/port/out/

    public CreateOrderUseCase(OrderRepository orderRepository, InventoryPort inventoryPort) {
        this.orderRepository = orderRepository;
        this.inventoryPort = inventoryPort;
    }

    @Transactional
    public OrderResponse execute(CreateOrderRequest request) {
        // 1. DTO → Domain 객체 변환
        List<OrderItem> items = request.items().stream()
            .map(item -> OrderItem.of(ProductId.of(item.productId()), item.quantity()))
            .toList();

        // 2. 재고 확인 (외부 Port 호출)
        inventoryPort.validateStock(items);

        // 3. Domain 로직 실행
        Order order = Order.create(UserId.of(request.userId()), items);

        // 4. 저장
        orderRepository.save(order);

        // 5. Domain 객체 → Response DTO 변환
        return OrderResponse.from(order);
    }
}

// application/port/out/InventoryPort.java
public interface InventoryPort {
    void validateStock(List<OrderItem> items);
}
```

---

## Adapter Layer 규칙

### Inbound Adapter (REST Controller)

**Controller는 요청 수신과 Use Case 호출만 담당합니다.**

### ✅ 반드시 준수
- Controller 하나당 리소스(Bounded Context) 하나
- 비즈니스 로직 없음 — Use Case에 위임
- Request/Response DTO를 별도로 정의 (Application DTO와 분리 가능)
- `@RestControllerAdvice`는 `infrastructure/web/`에 중앙 관리

```java
// adapter/inbound/rest/OrderController.java
@RestController
@RequestMapping("/orders")
public class OrderController {

    private final CreateOrderUseCase createOrderUseCase;

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public OrderResponse createOrder(@RequestBody @Validated CreateOrderHttpRequest request) {
        CreateOrderRequest dto = CreateOrderRequest.from(request);
        return createOrderUseCase.execute(dto);
    }
}
```

### Outbound Adapter (Persistence)

**JPA Entity는 Domain Entity와 별개의 클래스입니다. Mapper로 변환합니다.**

### ✅ 반드시 준수
- JPA `@Entity`는 `adapter/outbound/persistence/` 에만 존재
- Domain 객체 ↔ JPA Entity 변환은 **Mapper** 클래스가 전담
- Repository 구현 클래스는 Domain의 Repository 인터페이스를 `implements`
- Spring Data JPA Repository는 내부 구현 세부사항으로 사용

```java
// adapter/outbound/persistence/OrderEntity.java
@Entity
@Table(name = "orders")
public class OrderEntity {
    @Id
    private String id;
    private String userId;
    private String status;
    @OneToMany
    private List<OrderItemEntity> items;
    // JPA 관련 어노테이션은 여기만
}

// adapter/outbound/persistence/OrderPersistenceMapper.java
public class OrderPersistenceMapper {
    public static OrderEntity toEntity(Order order) { ... }
    public static Order toDomain(OrderEntity entity) { ... }
}

// adapter/outbound/persistence/OrderRepositoryImpl.java
@Repository
public class OrderRepositoryImpl implements OrderRepository {

    private final OrderJpaRepository jpaRepository;   // Spring Data JPA

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

// adapter/outbound/persistence/OrderJpaRepository.java — Spring Data 내부용
public interface OrderJpaRepository extends JpaRepository<OrderEntity, String> { }
```

---

## Infrastructure 규칙

| 폴더 | 역할 |
|---|---|
| `config/` | Bean 정의, 프로파일 설정, 앱 초기화 |
| `security/` | Spring Security 설정, JWT 처리 |
| `persistence/` | Datasource, JPA 글로벌 설정, Flyway 마이그레이션 |
| `web/` | `@RestControllerAdvice` (글로벌 예외 핸들러), CORS 등 |

- `@SpringBootApplication`은 프로젝트 루트 패키지에만
- Bean 수동 등록이 필요하면 `config/`에 `@Configuration` 클래스 생성

---

## 명명 규칙

| 구성 요소 | 접미사/패턴 | 예시 |
|---|---|---|
| Aggregate Root | 명사 | `Order`, `User`, `Product` |
| Entity (Aggregate 내) | 명사 | `OrderItem`, `Address` |
| Value Object | 명사 | `Money`, `Email`, `OrderId` |
| Repository 인터페이스 | `{Aggregate}Repository` | `OrderRepository` |
| Repository 구현 | `{Aggregate}RepositoryImpl` | `OrderRepositoryImpl` |
| JPA Entity | `{Aggregate}Entity` | `OrderEntity` |
| Spring Data JPA | `{Aggregate}JpaRepository` | `OrderJpaRepository` |
| Mapper | `{Aggregate}PersistenceMapper` | `OrderPersistenceMapper` |
| Use Case | `{Verb}{Noun}UseCase` | `CreateOrderUseCase` |
| Domain Service | `{Noun}DomainService` | `PricingDomainService` |
| Domain Event | `{Noun}{Past}Event` | `OrderCreatedEvent` |
| Domain Exception | `{Noun}{Reason}Exception` | `OrderNotFoundException` |
| Controller | `{Noun}Controller` | `OrderController` |
| HTTP Request DTO | `{Verb}{Noun}HttpRequest` | `CreateOrderHttpRequest` |
| HTTP Response DTO | `{Noun}HttpResponse` | `OrderHttpResponse` |
| Application DTO | `{Verb}{Noun}Request` / `{Noun}Response` | `CreateOrderRequest`, `OrderResponse` |
| Outbound Port | `{Noun}Port` | `InventoryPort` |

---

## 의존성 규칙 (Dependency Rule)

```
Controller  →  Use Case  →  Domain (Aggregate, Repository 인터페이스)
                  ↓
            Outbound Port 인터페이스
                  ↑
         Repository Impl (JPA)

✅ 허용: 아래 레이어 → 위 레이어 (안쪽 방향)
❌ 금지: Domain → Application, Domain → Adapter, Application → Adapter
```

**실제 import 검증 기준:**
- `domain/` 폴더의 클래스: `import org.springframework.*` 없음
- `application/` 폴더의 클래스: `import javax.persistence.*` 없음
- `adapter/inbound/` 클래스: `domain/` 직접 사용 금지 (Use Case를 거침)

---

## 테스트 전략

| 레이어 | 테스트 유형 | Spring Context | 주 초점 |
|---|---|---|---|
| Domain | Unit Test | ❌ | Aggregate 행위, Value Object 규칙, Domain Service |
| Application (Use Case) | Unit Test | ❌ | 오케스트레이션 흐름, Port 모킹 (`@ExtendWith(MockitoExtension.class)`) |
| Inbound Adapter (Controller) | Slice Test | `@WebMvcTest` | HTTP 요청/응답, 유효성 검사, 상태 코드 |
| Outbound Adapter (Repository) | Slice Test | `@DataJpaTest` + Testcontainers | DB 연동, 쿼리 정확성 |
| Integration | Full Test | `@SpringBootTest` | E2E 흐름 검증 |

### 테스트 패키지 구조
테스트 폴더 구조는 `src/main/java`와 **동일하게 미러링**합니다.

```
src/test/java/com/example/{service-name}/
├── {bounded-context}/
│   ├── domain/
│   │   └── model/
│   │       └── OrderTest.java
│   ├── application/
│   │   └── usecase/
│   │       └── CreateOrderUseCaseTest.java
│   └── adapter/
│       ├── inbound/rest/
│       │   └── OrderControllerTest.java
│       └── outbound/persistence/
│           └── OrderRepositoryImplTest.java
└── integration/
    └── OrderIntegrationTest.java
```

---

## Anti-patterns (피해야 할 패턴)

| Anti-pattern | 문제 | 해결 |
|---|---|---|
| Anemic Domain Model | 로직이 Service에만, Domain은 데이터 컨테이너 | Rich Domain Model — 로직을 Aggregate 안에 배치 |
| Domain에 Spring 어노테이션 | Domain과 프레임워크 결합 | Domain은 순수 POJO, 어노테이션은 Adapter에만 |
| Controller → Repository 직접 호출 | Application 레이어 우회 | Controller는 Use Case만 호출 |
| Entity를 Controller까지 전달 | 내부 구현 노출 | DTO로 변환 후 응답 |
| Fat Controller / Fat Service | 단일 책임 위반 | Use Case 분리, 로직을 Domain으로 |
| Repository가 단일 Entity 관리 | Aggregate 경계 위반 | Repository는 Aggregate Root 단위로만 |
| Application Service에 비즈니스 규칙 | Domain 빈곤화 | 규칙은 Domain, Use Case는 오케스트레이션만 |

---

## 기술 스택 & 설정 (현재 프로젝트 기준)

| 영역 | 현재 사용 중 | 권장 사항 |
|---|---|---|
| ORM | Spring Data JPA + Hibernate | JPA Entity는 Adapter에만 (✅ 준수 중) |
| DB (개발) | H2 in-memory | 개발용으로 적합, 프로덕션은 PostgreSQL/MySQL |
| DB 마이그레이션 | Flyway (2개 마이그레이션) | ✅ 사용 중, `ddl-auto: none` 설정됨 |
| API 문서화 | Springdoc OpenAPI 2.3.0 | ✅ Swagger UI 구성 완료 (`/swagger-ui/index.html`) |
| 직렬화 | Jackson (Spring Boot 기본) | `application.yml`에 전역 설정 가능 |
| 프로파일 관리 | `application.yml` (단일) | ⚠️ `application-dev.yml`, `application-prod.yml` 추가 필요 |
| 예외 핸들링 | GlobalExceptionHandler (ErrorResponse) | ✅ 구조화된 에러 응답 구현됨 |
| 이벤트 | ApplicationEventPublisher (동기) | ✅ 사용 중 (`OrderCreatedEvent`), 리스너 추가 필요 |
| 테스트 DB | H2 in-memory | ⚠️ Testcontainers 도입 권장 |
| DTO 불변 | Java `record` | ✅ 모든 DTO에 사용 중 |
| ID 전략 | UUID (Value Object 래핑) | ✅ `UserId`, `OrderId` 사용 중 |
| 의존성 검증 | ❌ 없음 | ⚠️ ArchUnit 테스트 추가 필요 (HIGH Priority) |

### ArchUnit 의존성 규칙 검증 예시
```java
@AnalyzeClasses(packages = "com.example.myservice")
public class ArchitectureTest {

    @ArchTest
    void domainLayerShouldNotDependOnSpring(ArchStore store) {
        allClasses()
            .that().resideInAnyPackage("..domain..")
            .should().notResideInAnyPackage("org.springframework..")
            .because("Domain 레이어는 Spring에 의존하지 않아야 한다");
    }

    @ArchTest
    void controllerShouldNotDependOnRepository(ArchStore store) {
        allClasses()
            .that().resideInAnyPackage("..adapter.inbound..")
            .should().onlyDependOnClassesThat()
                .resideInAnyPackage("..application..", "..adapter.inbound..", "org.springframework..")
            .because("Controller는 Use Case만 호출해야 한다");
    }
}
```
