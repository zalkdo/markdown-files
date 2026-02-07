---
name: add-rest-endpoint
description: Add a new REST API endpoint to existing Controller
triggers:
  - "add endpoint"
  - "new endpoint"
  - "add api"
---

# REST Endpoint 추가

기존 Controller에 새로운 API 엔드포인트를 추가합니다.

## 작업 순서

1. **엔드포인트 정보 확인**
   - HTTP 메서드 (GET, POST, PUT, DELETE)
   - 경로 (path)
   - Path Variable / Query Parameter
   - HTTP 상태 코드 (200, 201, 204 등)

2. **HTTP Request DTO 생성 (POST/PUT인 경우)**
   ```java
   @Schema(description = "...")
   public record XxxHttpRequest(
       @Schema(description = "...", example = "...")
       @NotBlank String field1,

       @Schema(description = "...", example = "...")
       @Valid List<XxxItem> items
   ) {}
   ```

3. **Controller 메서드 추가**
   ```java
   @{HttpMethod}("{path}")
   @Operation(summary = "...", description = "...")
   @ApiResponse(responseCode = "200", description = "성공")
   @ApiResponse(responseCode = "404", description = "...",
                content = @Content(schema = @Schema(implementation = ErrorResponse.class)))
   @ResponseStatus(HttpStatus.OK)
   public {Response}Dto methodName(
       @PathVariable String id,
       @RequestParam String queryParam,
       @RequestBody @Valid XxxHttpRequest request
   ) {
       // HTTP DTO → Application DTO 변환
       // Use Case 호출
       // 결과 반환
   }
   ```

4. **OpenAPI 문서 확인**
   - Swagger UI에서 새 엔드포인트 확인 (`/swagger-ui/index.html`)
   - Request/Response 스키마 확인
   - "Try it out" 테스트

5. **Controller Slice Test 추가 (권장)**
   ```java
   @WebMvcTest(XxxController.class)
   class XxxControllerTest {
       @Autowired MockMvc mockMvc;
       @MockBean XxxUseCase useCase;

       @Test void endpoint_validRequest_success() { ... }
   }
   ```

## HTTP 메서드별 패턴

| 메서드 | 용도 | 상태 코드 | Request Body | Response Body |
|--------|------|-----------|--------------|---------------|
| GET | 조회 | 200 | ❌ | ✅ |
| POST | 생성 | 201 | ✅ | ✅ |
| PUT | 전체 수정 | 200 | ✅ | ✅ |
| PATCH | 부분 수정 | 200 | ✅ | ✅ |
| DELETE | 삭제 | 204 | ❌ | ❌ |

생성 후 Swagger UI와 통합 테스트로 검증하세요.
