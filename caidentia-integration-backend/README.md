# Spring Boot Hexagonal DDD Architecture - myapp

## 프로젝트 개요

Spring Boot 3.3.7 + Java 21 기반으로 **Hexagonal Architecture (Ports & Adapters)** 와 **Domain-Driven Design (DDD)** 원칙을 적용한 Clean Architecture 예제 프로젝트입니다.

### 기술 스택

- **Java**: 21 (Temurin JDK)
- **Spring Boot**: 3.3.7
- **Build Tool**: Gradle 8.12
- **Database**: H2 (In-Memory)
- **ORM**: Spring Data JPA + Hibernate
- **Migration**: Flyway
- **Test**: JUnit 5, Mockito, Spring Boot Test

---

## 아키텍처

### Bounded Contexts

프로젝트는 2개의 Bounded Context로 구성됩니다:

1. **User Context** - 사용자 등록, 조회
2. **Order Context** - 주문 생성, 취소, 조회

### 레이어 구조

```
com.example.myapp/
├── {bounded-context}/          # user, order
│   ├── domain/                 # 핵심 비즈니스 로직 (프레임워크 독립)
│   │   ├── model/              # Aggregate Root, Entity, Value Object
│   │   ├── event/              # Domain Event
│   │   ├── exception/          # Domain 예외
│   │   └── repository/         # Repository 인터페이스 (Port)
│   ├── application/            # Use Case 오케스트레이션
│   │   ├── usecase/            # Use Case (Application Service)
│   │   └── dto/                # Request/Response DTO
│   └── adapter/                # 외부 세계와의 연결
│       ├── inbound/rest/       # REST Controller
│       └── outbound/persistence/ # JPA Entity, Repository 구현
├── shared/                     # 공유 모듈
│   ├── domain/vo/              # 공유 Value Object (Money 등)
│   └── exception/              # 공유 예외
└── infrastructure/             # Cross-cutting Concerns
    └── web/                    # GlobalExceptionHandler
```

---

## 주요 구현 패턴

### 1. Domain Layer (Pure Java)

- ✅ **Spring 어노테이션 없음** - 프레임워크 독립적
- ✅ **Rich Domain Model** - 비즈니스 로직이 Domain 객체 내부에 위치
- ✅ **Value Object 불변성** - Java `record` 활용
- ✅ **Aggregate Root** - 트랜잭션 경계 관리

**예시: User Aggregate**
```java
public class User {
    private final UserId id;
    private final Email email;
    private UserStatus status;

    public static User create(Email email, UserName name) {
        return new User(UserId.generate(), email, name, UserStatus.ACTIVE);
    }

    public void deactivate() {
        if (this.status == UserStatus.DELETED) {
            throw new DomainException("...");
        }
        this.status = UserStatus.INACTIVE;
    }
}
```

### 2. Application Layer (Use Cases)

- ✅ **단일 책임** - Use Case 하나당 하나의 비즈니스 시나리오
- ✅ **DTO 변환** - Domain 객체를 외부에 직접 노출하지 않음
- ✅ **트랜잭션 관리** - `@Transactional` at Use Case level

**예시: RegisterUserUseCase**
```java
@Service
public class RegisterUserUseCase {
    private final UserRepository userRepository;

    @Transactional
    public UserResponse execute(RegisterUserRequest request) {
        Email email = Email.of(request.email());
        if (userRepository.existsByEmail(email)) {
            throw new DuplicateEmailException(email);
        }
        User user = User.create(email, UserName.of(request.name()));
        userRepository.save(user);
        return UserResponse.from(user);
    }
}
```

### 3. Adapter Layer (Inbound/Outbound)

#### Inbound (REST Controller)
```java
@RestController
@RequestMapping("/users")
public class UserController {
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public UserResponse registerUser(@RequestBody @Valid RegisterUserHttpRequest request) {
        return registerUserUseCase.execute(...);
    }
}
```

#### Outbound (Persistence)
```java
@Repository
public class UserRepositoryImpl implements UserRepository {
    private final UserJpaRepository jpaRepository;

    @Override
    public void save(User user) {
        jpaRepository.save(UserPersistenceMapper.toEntity(user));
    }
}
```

**핵심 패턴:**
- JPA Entity와 Domain Entity 분리
- Mapper 클래스로 변환 (`UserPersistenceMapper`)
- Domain Repository 인터페이스 구현

---

## 빌드 및 실행

### 환경 설정

```bash
# JDK 21 설치 확인
java -version  # openjdk version "21.0.4"

# Gradle 설치 확인
gradle --version  # Gradle 8.12
```

### 빌드

```bash
cd F:\workplace\claude
gradle clean build
```

**빌드 결과:**
- JAR: `build/libs/myapp-0.0.1-SNAPSHOT.jar` (48MB)
- 테스트: 20개 테스트 모두 통과 ✅

### 실행

```bash
gradle bootRun
```

또는 JAR 직접 실행:
```bash
java -jar build/libs/myapp-0.0.1-SNAPSHOT.jar
```

서버 시작 후 접속: `http://localhost:8080`

---

## API 엔드포인트

### User API

| Method | Endpoint | 설명 |
|--------|----------|------|
| POST | `/users` | 유저 등록 |
| GET | `/users/{userId}` | 유저 조회 |

**유저 등록 예시:**
```bash
curl -X POST http://localhost:8080/users \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "name": "홍길동"
  }'
```

**응답:**
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "email": "user@example.com",
  "name": "홍길동",
  "status": "ACTIVE"
}
```

### Order API

| Method | Endpoint | 설명 |
|--------|----------|------|
| POST | `/orders` | 주문 생성 |
| GET | `/orders/{orderId}` | 주문 조회 |
| GET | `/orders?userId={userId}` | 유저별 주문 목록 조회 |
| DELETE | `/orders/{orderId}` | 주문 취소 |

**주문 생성 예시:**
```bash
curl -X POST http://localhost:8080/orders \
  -H "Content-Type: application/json" \
  -d '{
    "userId": "user-123",
    "items": [
      {
        "productId": "prod-001",
        "quantity": 2,
        "unitPrice": 10000
      }
    ]
  }'
```

**응답:**
```json
{
  "id": "order-550e8400",
  "userId": "user-123",
  "status": "CREATED",
  "items": [
    {
      "productId": "prod-001",
      "quantity": 2,
      "unitPrice": "10000",
      "subtotal": "20000"
    }
  ],
  "totalAmount": "20000"
}
```

---

## 테스트

### 테스트 실행

```bash
gradle test
```

### 테스트 구조

| 레이어 | 테스트 유형 | 예시 |
|--------|------------|------|
| Domain | Unit Test | `UserTest`, `OrderTest` |
| Application | Unit Test (Mocked Ports) | `RegisterUserUseCaseTest`, `CreateOrderUseCaseTest` |
| Integration | `@SpringBootTest` | `UserIntegrationTest` |

**총 20개 테스트 케이스:**
- User Domain: 6개 테스트
- Order Domain: 8개 테스트
- User Application: 2개 테스트
- Order Application: 2개 테스트
- Integration: 2개 테스트

---

## 데이터베이스

### H2 Console 접속

서버 실행 후: `http://localhost:8080/h2-console`

**연결 정보:**
- JDBC URL: `jdbc:h2:mem:myapp`
- User: `sa`
- Password: (비워둠)

### 테이블 스키마

**users 테이블:**
```sql
CREATE TABLE users (
    id VARCHAR(36) PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(50) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE'
);
```

**orders / order_items 테이블:**
```sql
CREATE TABLE orders (
    id VARCHAR(36) PRIMARY KEY,
    user_id VARCHAR(36) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'CREATED'
);

CREATE TABLE order_items (
    id BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    order_id VARCHAR(36) NOT NULL,
    product_id VARCHAR(36) NOT NULL,
    quantity INT NOT NULL,
    unit_price DECIMAL(15,2) NOT NULL,
    currency VARCHAR(10) NOT NULL DEFAULT 'KRW',
    FOREIGN KEY (order_id) REFERENCES orders(id)
);
```

---

## 프로젝트 통계

| 항목 | 수치 |
|------|------|
| 총 파일 수 | 57개 |
| Java 소스 파일 | 47개 |
| 테스트 파일 | 5개 |
| Bounded Context | 2개 (user, order) |
| Aggregate Root | 2개 (User, Order) |
| Value Object | 8개 (UserId, Email, UserName, OrderId, ProductId, Quantity, Money 등) |
| Use Case | 6개 |
| REST Endpoint | 6개 |
| 빌드 JAR 크기 | 48MB |

---

## 참고 자료

이 프로젝트는 다음 Best Practice를 참조하여 구성되었습니다:

- [Hexagonal Architecture, DDD, and Spring - Baeldung](https://www.baeldung.com/hexagonal-architecture-ddd-spring)
- [Clean DDD: Project Structure and Naming Conventions](https://medium.com/unil-ci-software-engineering/clean-ddd-lessons-project-structure-and-naming-conventions-00d0b9c57610)
- [Hexagonal Architecture in Spring Boot Microservices](https://dev.to/rock_win_c053fa5fb2399067/hexagonal-architecture-in-spring-boot-microservices-a-complete-guide-with-folder-structure-1jld)

상세한 아키텍처 가이드라인은 `CLAUDE.md` 참조.

---

**프로젝트 생성일:** 2026-02-06
**빌드 상태:** ✅ BUILD SUCCESSFUL (52초)
**테스트 상태:** ✅ 20/20 테스트 통과
