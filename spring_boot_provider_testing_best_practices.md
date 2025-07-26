# Spring Boot Provider Testing Best Practices with REST Assured

## Overview

This guide covers best practices for implementing provider-side contract testing in Spring Boot applications using REST Assured. It focuses on testing OpenAPI specifications against actual implementations to ensure your provider contract accurately represents your API's behavior.

## Setup and Dependencies

### Maven Dependencies
```xml
<dependencies>
    <!-- Spring Boot Test Starter -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    
    <!-- REST Assured -->
    <dependency>
        <groupId>io.rest-assured</groupId>
        <artifactId>rest-assured</artifactId>
        <scope>test</scope>
    </dependency>
    
    <!-- REST Assured Spring Mock MVC -->
    <dependency>
        <groupId>io.rest-assured</groupId>
        <artifactId>spring-mock-mvc</artifactId>
        <scope>test</scope>
    </dependency>
    
    <!-- OpenAPI Generator (optional for schema validation) -->
    <dependency>
        <groupId>com.atlassian.oai</groupId>
        <artifactId>swagger-request-validator-restassured</artifactId>
        <version>2.40.0</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```



## Test Structure and Organization

### Base Test Configuration
```java
@SpringBootTest(
    classes = Application.class,
    webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT,
    properties = {"server.max-http-request-header-size=300B"}
)
@ExtendWith({SpringExtension.class})
@Slf4j
public class UserControllerProviderPactTest {
    
    @LocalServerPort
    private int randomServerPort;
    
    private final String specPath = Paths.get(
        System.getProperty("user.dir"), 
        "src", "main", "resources", "spec", "provider"
    ).toAbsolutePath().toString();
    
    private final OpenApiValidationFilter defaultValidationFilter = 
        new OpenApiValidationFilter(specPath);
}
```

### Test Data Management
```java
@TestConfiguration
public class TestDataConfiguration {
    
    @Bean
    @Primary
    public UserRepository userRepository() {
        return Mockito.mock(UserRepository.class);
    }
    
    @PostConstruct
    public void setupTestData() {
        // Initialize test data that matches your OpenAPI examples
    }
}
```

## Core Testing Patterns

### 1. Happy Path Testing
```java
@Test
void shouldCreateUserSuccessfully() {
    JsonObject jsonObject = new JsonObject();
    jsonObject.addProperty("name", "John Doe");
    jsonObject.addProperty("age", 30);
    jsonObject.addProperty("email", "john.doe@example.com");
    
    Response response = given()
        .port(randomServerPort)
        .filter(defaultValidationFilter)
        .contentType(ContentType.JSON)
        .body(jsonObject.toString())
    .when()
        .post("/users");
    
    Assertions.assertEquals(201, response.getStatusCode());
    Assertions.assertNotNull(response.jsonPath().get("id"));
    Assertions.assertEquals("John Doe", response.jsonPath().getString("name"));
    Assertions.assertEquals("john.doe@example.com", response.jsonPath().getString("email"));
    Assertions.assertEquals(30, response.jsonPath().getInt("age"));
}
```

### 2. Validation Error Testing
```java
@Test
void shouldReturn400WhenAgeIsInvalid() {
    // Use custom validator for error scenarios to avoid response validation issues
    OpenApiInteractionValidator customValidator = createCustomValidator();
    
    JsonObject jsonObject = new JsonObject();
    jsonObject.addProperty("name", "John Doe");
    jsonObject.addProperty("age", -5); // Invalid age
    jsonObject.addProperty("email", "john.doe@example.com");
    
    Response response = given()
        .port(randomServerPort)
        .filter(new OpenApiValidationFilter(customValidator))
        .contentType(ContentType.JSON)
        .body(jsonObject.toString())
    .when()
        .post("/users");
    
    Assertions.assertEquals(400, response.getStatusCode());
    Assertions.assertNotNull(response.jsonPath().get("error"));
    Assertions.assertTrue(response.jsonPath().getString("message").contains("age"));
}

private OpenApiInteractionValidator createCustomValidator() {
    return OpenApiInteractionValidator.createFor(specPath)
        .withLevelResolver(
            LevelResolver.create()
                .withLevel("validation.request", Level.ERROR)
                .withLevel("validation.schema.required", Level.IGNORE)
                .withLevel("validation.response.body.missing", Level.IGNORE)
                .withLevel("validation.response.body.schema.additionalProperties", Level.IGNORE)
                .withLevel("validation.response.body.schema.required", Level.IGNORE)
                .withLevel("validation.response", Level.IGNORE)
                .build())
        .build();
}
```

### 3. Optional Fields Testing
```java
@Test
void shouldCreateUserWithoutAge() {
    JsonObject jsonObject = new JsonObject();
    jsonObject.addProperty("name", "Jane Smith");
    jsonObject.addProperty("email", "jane.smith@example.com");
    
    Response response = given()
        .port(randomServerPort)
        .filter(defaultValidationFilter)
        .contentType(ContentType.JSON)
        .body(jsonObject.toString())
    .when()
        .post("/users");
    
    Assertions.assertEquals(201, response.getStatusCode());
    Assertions.assertNotNull(response.jsonPath().get("id"));
    Assertions.assertEquals("Jane Smith", response.jsonPath().getString("name"));
    Assertions.assertEquals("jane.smith@example.com", response.jsonPath().getString("email"));
}
```

## Advanced Testing Patterns

### 1. Parameterized Testing for Multiple Scenarios
```java
@ParameterizedTest
@ValueSource(strings = {"Engineering", "Marketing", "Sales", "HR", "Finance", "Operations"})
void shouldAcceptAllValidDepartments(String department) {
    JsonObject jsonObject = new JsonObject();
    jsonObject.addProperty("name", "Test User");
    jsonObject.addProperty("age", 30);
    jsonObject.addProperty("email", "test@example.com");
    jsonObject.addProperty("department", department);
    
    Response response = given()
        .port(randomServerPort)
        .filter(defaultValidationFilter)
        .contentType(ContentType.JSON)
        .body(jsonObject.toString())
    .when()
        .post("/users");
    
    Assertions.assertEquals(201, response.getStatusCode());
    Assertions.assertEquals(department, response.jsonPath().getString("department"));
}
```

### 2. Edge Case Testing
```java
@Test
void shouldHandleBoundaryValues() {
    // Test minimum age
    JsonObject minAgeUser = new JsonObject();
    minAgeUser.addProperty("name", "Min Age User");
    minAgeUser.addProperty("age", 0);
    minAgeUser.addProperty("email", "min@example.com");
    
    Response response1 = given()
        .port(randomServerPort)
        .filter(defaultValidationFilter)
        .contentType(ContentType.JSON)
        .body(minAgeUser.toString())
    .when()
        .post("/users");
    
    Assertions.assertEquals(201, response1.getStatusCode());
    
    // Test maximum age
    JsonObject maxAgeUser = new JsonObject();
    maxAgeUser.addProperty("name", "Max Age User");
    maxAgeUser.addProperty("age", 110);
    maxAgeUser.addProperty("email", "max@example.com");
    
    Response response2 = given()
        .port(randomServerPort)
        .filter(defaultValidationFilter)
        .contentType(ContentType.JSON)
        .body(maxAgeUser.toString())
    .when()
        .post("/users");
    
    Assertions.assertEquals(201, response2.getStatusCode());
    
    // Test maximum name length
    String longName = "A".repeat(100);
    JsonObject longNameUser = new JsonObject();
    longNameUser.addProperty("name", longName);
    longNameUser.addProperty("age", 30);
    longNameUser.addProperty("email", "long@example.com");
    
    Response response3 = given()
        .port(randomServerPort)
        .filter(defaultValidationFilter)
        .contentType(ContentType.JSON)
        .body(longNameUser.toString())
    .when()
        .post("/users");
    
    Assertions.assertEquals(201, response3.getStatusCode());
}
```

### 3. Error Response Consistency
```java
@Test
void shouldReturnConsistentErrorFormat() {
    OpenApiInteractionValidator customValidator = createCustomValidator();
    
    // Test different validation failures
    List<JsonObject> invalidRequests = Arrays.asList(
        createInvalidUserRequest("", 30, "valid@example.com"), // Empty name
        createInvalidUserRequest("Valid Name", -1, "valid@example.com"), // Invalid age
        createInvalidUserRequest("Valid Name", 30, "invalid-email"), // Invalid email
        createInvalidUserRequest("Valid Name", 111, "valid@example.com") // Age too high
    );
    
    for (JsonObject request : invalidRequests) {
        Response response = given()
            .port(randomServerPort)
            .filter(new OpenApiValidationFilter(customValidator))
            .contentType(ContentType.JSON)
            .body(request.toString())
        .when()
            .post("/users");
        
        Assertions.assertEquals(400, response.getStatusCode());
        Assertions.assertNotNull(response.jsonPath().get("error"));
        Assertions.assertNotNull(response.jsonPath().get("message"));
        Assertions.assertNotNull(response.jsonPath().get("timestamp"));
    }
}

private JsonObject createInvalidUserRequest(String name, int age, String email) {
    JsonObject jsonObject = new JsonObject();
    jsonObject.addProperty("name", name);
    jsonObject.addProperty("age", age);
    jsonObject.addProperty("email", email);
    return jsonObject;
}
```

## Best Practices

### 1. Test Organization
```java
@Nested
@DisplayName("User Creation Tests")
class UserCreationTests {
    
    @Nested
    @DisplayName("Valid Requests")
    class ValidRequests {
        // Happy path tests
    }
    
    @Nested
    @DisplayName("Invalid Requests")
    class InvalidRequests {
        // Error scenario tests
    }
    
    @Nested
    @DisplayName("Edge Cases")
    class EdgeCases {
        // Boundary value tests
    }
}
```

### 2. Custom Matchers for Reusability
```java
public class UserMatchers {
    
    public static Matcher<String> validUserId() {
        return matchesPattern("^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$");
    }
    
    public static Matcher<String> validTimestamp() {
        return matchesPattern("\\d{4}-\\d{2}-\\d{2}T\\d{2}:\\d{2}:\\d{2}.*");
    }
    
    public static Matcher<String> validEmail() {
        return matchesPattern("^[A-Za-z0-9+_.-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$");
    }
}

// Usage
.body("id", validUserId())
.body("createdAt", validTimestamp())
.body("email", validEmail())
```

### 3. Test Data Builders
```java
public class UserTestDataBuilder {
    
    public static CreateUserRequest.CreateUserRequestBuilder validUser() {
        return CreateUserRequest.builder()
            .name("John Doe")
            .age(30)
            .email("john.doe@example.com")
            .department("Engineering");
    }
    
    public static CreateUserRequest.CreateUserRequestBuilder invalidAgeUser() {
        return validUser().age(-1);
    }
    
    public static CreateUserRequest.CreateUserRequestBuilder invalidEmailUser() {
        return validUser().email("invalid-email");
    }
}

// Usage
given()
    .contentType(ContentType.JSON)
    .body(UserTestDataBuilder.validUser().build())
.when()
    .post("/api/v1/users")
.then()
    .statusCode(201);
```

### 4. Environment-Specific Configuration
```java
@TestPropertySource(properties = {
    "app.feature.user-validation.strict=true",
    "app.database.cleanup.enabled=true",
    "logging.level.com.example.userservice=DEBUG"
})
class StrictValidationProviderTest extends BaseProviderTest {
    // Tests with strict validation enabled
}
```

### 5. Database State Management
```java
@Transactional
@Rollback
class UserProviderTest extends BaseProviderTest {
    
    @Autowired
    private TestEntityManager entityManager;
    
    @BeforeEach
    void setupDatabase() {
        // Create test data
        entityManager.persistAndFlush(new User("existing@example.com"));
    }
    
    @Test
    @DisplayName("Should return 409 when email already exists")
    void shouldReturn409WhenEmailExists() {
        given()
            .contentType(ContentType.JSON)
            .body(createUserRequest("New User", 25, "existing@example.com"))
        .when()
            .post("/api/v1/users")
        .then()
            .statusCode(409)
            .body("error", equalTo("Email already exists"));
    }
}
```

## Integration with Contract Testing

### 1. OpenAPI Validation Filter - Standard Usage
```java
private final String specPath = Paths.get(
    System.getProperty("user.dir"), 
    "src", "main", "resources", "spec", "provider"
).toAbsolutePath().toString();

private final OpenApiValidationFilter defaultValidationFilter = 
    new OpenApiValidationFilter(specPath);

@Test
void shouldComplyWithOpenAPIContract() {
    JsonObject jsonObject = new JsonObject();
    jsonObject.addProperty("name", "John Doe");
    jsonObject.addProperty("age", 30);
    jsonObject.addProperty("email", "john.doe@example.com");
    
    Response response = given()
        .port(randomServerPort)
        .filter(defaultValidationFilter)
        .contentType(ContentType.JSON)
        .body(jsonObject.toString())
    .when()
        .post("/users");
    
    Assertions.assertEquals(201, response.getStatusCode());
}
```

### 2. OpenAPI Validation Filter Limitations

**Important Note**: The OpenApiValidationFilter has limitations when dealing with error scenarios:

- **Single Response Body**: It cannot handle multiple response bodies for one request (e.g., 200 success response AND 400 error response)
- **Error Response Testing**: If you want to test error responses while still validating request path and body, you need to customize the filter

### 3. Customized OpenAPI Validation Filter for Error Scenarios
```java
private OpenApiInteractionValidator createCustomValidator() {
    return OpenApiInteractionValidator.createFor(specPath)
        .withLevelResolver(
            LevelResolver.create()
                .withLevel("validation.request", Level.ERROR)
                .withLevel("validation.schema.required", Level.IGNORE)
                .withLevel("validation.response.body.missing", Level.IGNORE)
                .withLevel("validation.response.body.schema.additionalProperties", Level.IGNORE)
                .withLevel("validation.response.body.schema.required", Level.IGNORE)
                .withLevel("validation.response", Level.IGNORE)
                .build())
        .build();
}

@Test
void shouldValidateRequestButIgnoreErrorResponse() {
    OpenApiInteractionValidator customValidator = createCustomValidator();
    
    JsonObject invalidUser = new JsonObject();
    invalidUser.addProperty("name", "John Doe");
    invalidUser.addProperty("age", -5); // Invalid age
    invalidUser.addProperty("email", "john.doe@example.com");
    
    Response response = given()
        .port(randomServerPort)
        .filter(new OpenApiValidationFilter(customValidator))
        .contentType(ContentType.JSON)
        .body(invalidUser.toString())
    .when()
        .post("/users");
    
    // Validates request path and body structure, but ignores response validation
    Assertions.assertEquals(400, response.getStatusCode());
    Assertions.assertTrue(response.jsonPath().getString("message").contains("age"));
}
```

### 4. Separate Tests for Different Response Types
```java
@Test
void shouldValidateSuccessResponseWithFullValidation() {
    JsonObject validUser = new JsonObject();
    validUser.addProperty("name", "John Doe");
    validUser.addProperty("age", 30);
    validUser.addProperty("email", "john.doe@example.com");
    
    Response response = given()
        .port(randomServerPort)
        .filter(defaultValidationFilter) // Full validation for success case
        .contentType(ContentType.JSON)
        .body(validUser.toString())
    .when()
        .post("/users");
    
    Assertions.assertEquals(201, response.getStatusCode());
}

@Test
void shouldValidateErrorResponseWithCustomValidation() {
    OpenApiInteractionValidator customValidator = createCustomValidator();
    
    JsonObject invalidUser = new JsonObject();
    invalidUser.addProperty("name", "John Doe");
    invalidUser.addProperty("age", 150); // Invalid age
    invalidUser.addProperty("email", "invalid-email");
    
    Response response = given()
        .port(randomServerPort)
        .filter(new OpenApiValidationFilter(customValidator))
        .contentType(ContentType.JSON)
        .body(invalidUser.toString())
    .when()
        .post("/users");
    
    // Request validation still works, response validation is ignored
    Assertions.assertEquals(400, response.getStatusCode());
}
```

### 2. Contract Test Suite
```java
@SpringBootTest
@TestMethodOrder(OrderAnnotation.class)
@DisplayName("Provider Contract Compliance Tests")
class ProviderContractTest extends BaseProviderTest {
    
    @Test
    @Order(1)
    @DisplayName("All OpenAPI examples should work")
    void shouldHandleAllOpenAPIExamples() throws Exception {
        // Parse OpenAPI specification
        OpenAPI openAPI = new OpenAPIV3Parser().read(specPath + "/openapi.yaml");
        
        // Test all request examples
        openAPI.getPaths().forEach((path, pathItem) -> {
            pathItem.readOperationsMap().forEach((httpMethod, operation) -> {
                if (operation.getRequestBody() != null) {
                    operation.getRequestBody().getContent().forEach((mediaType, mediaTypeObject) -> {
                        if (mediaTypeObject.getExamples() != null) {
                            mediaTypeObject.getExamples().forEach((exampleName, example) -> {
                                testRequestExample(path, httpMethod, mediaType, exampleName, example);
                            });
                        }
                    });
                }
            });
        });
    }
    
    private void testRequestExample(String path, PathItem.HttpMethod method, String mediaType, 
                                  String exampleName, Example example) {
        try {
            Response response = given()
                .port(randomServerPort)
                .filter(defaultValidationFilter)
                .contentType(mediaType)
                .body(example.getValue().toString())
            .when()
                .request(method.name(), path);
            
            // Verify response is successful (2xx) or expected error
            Assertions.assertTrue(response.getStatusCode() >= 200 && response.getStatusCode() < 500,
                String.format("Example '%s' for %s %s failed with status %d", 
                    exampleName, method, path, response.getStatusCode()));
                    
        } catch (Exception e) {
            Assertions.fail(String.format("Failed to test example '%s' for %s %s: %s", 
                exampleName, method, path, e.getMessage()));
        }
    }
    
    @Test
    @Order(2)
    @DisplayName("All error scenarios should match OpenAPI spec")
    void shouldMatchErrorScenariosInSpec() throws Exception {
        OpenAPI openAPI = new OpenAPIV3Parser().read(specPath + "/openapi.yaml");
        
        // Test all error response examples
        openAPI.getPaths().forEach((path, pathItem) -> {
            pathItem.readOperationsMap().forEach((httpMethod, operation) -> {
                operation.getResponses().forEach((statusCode, apiResponse) -> {
                    if (statusCode.startsWith("4") || statusCode.startsWith("5")) {
                        testErrorResponse(path, httpMethod, statusCode, apiResponse);
                    }
                });
            });
        });
    }
    
    private void testErrorResponse(String path, PathItem.HttpMethod method, 
                                 String statusCode, ApiResponse apiResponse) {
        // Create invalid request to trigger the error response
        JsonObject invalidRequest = createInvalidRequestForError(statusCode);
        
        Response response = given()
            .port(randomServerPort)
            .filter(createCustomValidator()) // Use custom validator for error scenarios
            .contentType(ContentType.JSON)
            .body(invalidRequest.toString())
        .when()
            .request(method.name(), path);
        
        int expectedStatus = Integer.parseInt(statusCode);
        Assertions.assertEquals(expectedStatus, response.getStatusCode(),
            String.format("Expected %s status for %s %s", statusCode, method, path));
    }
    
    private JsonObject createInvalidRequestForError(String statusCode) {
        JsonObject request = new JsonObject();
        switch (statusCode) {
            case "400":
                request.addProperty("age", -1); // Invalid age
                break;
            case "409":
                request.addProperty("email", "existing@example.com"); // Duplicate email
                break;
            default:
                request.addProperty("name", "Test User");
        }
        return request;
    }
}
```

## Performance and Load Testing

### 1. Basic Performance Testing
```java
@Test
@DisplayName("Should handle concurrent user creation")
void shouldHandleConcurrentRequests() throws InterruptedException {
    int numberOfThreads = 10;
    int requestsPerThread = 5;
    ExecutorService executor = Executors.newFixedThreadPool(numberOfThreads);
    CountDownLatch latch = new CountDownLatch(numberOfThreads * requestsPerThread);
    
    for (int i = 0; i < numberOfThreads * requestsPerThread; i++) {
        final int requestId = i;
        executor.submit(() -> {
            try {
                given()
                    .contentType(ContentType.JSON)
                    .body(createUserRequest("User " + requestId, 30, "user" + requestId + "@example.com"))
                .when()
                    .post("/api/v1/users")
                .then()
                    .statusCode(201);
            } finally {
                latch.countDown();
            }
        });
    }
    
    assertThat(latch.await(30, TimeUnit.SECONDS)).isTrue();
}
```

## Common Pitfalls and Solutions

### 1. Avoid Hard-coded Values
```java
// ❌ Bad
.body("createdAt", equalTo("2024-01-15T10:30:00Z"))

// ✅ Good
.body("createdAt", matchesPattern("\\d{4}-\\d{2}-\\d{2}T\\d{2}:\\d{2}:\\d{2}.*"))
```

### 2. Test Real Business Logic
```java
// ❌ Bad - Only testing serialization
@Test
void shouldReturnUser() {
    given()
        .contentType(ContentType.JSON)
        .body("{\"name\":\"John\"}")
    .when()
        .post("/users")
    .then()
        .statusCode(201);
}

// ✅ Good - Testing actual validation logic
@Test
void shouldValidateEmailFormat() {
    given()
        .contentType(ContentType.JSON)
        .body(createUserRequest("John", 30, "invalid-email"))
    .when()
        .post("/users")
    .then()
        .statusCode(400)
        .body("message", containsString("Invalid email format"));
}
```

### 3. Proper Error Message Testing
```java
// ❌ Bad - Too generic
.body("error", notNullValue())

// ✅ Good - Specific and helpful
.body("error", equalTo("Validation Failed"))
.body("message", containsString("age must be between 0 and 110"))
.body("field", equalTo("age"))
.body("rejectedValue", equalTo(-5))
```



## Conclusion

Following these best practices ensures your Spring Boot provider tests are:
- **Reliable**: Consistent and repeatable results
- **Maintainable**: Easy to update as your API evolves
- **Comprehensive**: Cover all scenarios defined in your OpenAPI spec
- **Fast**: Efficient execution in CI/CD pipelines
- **Clear**: Easy to understand and debug when they fail

Remember that provider tests should validate that your implementation matches your OpenAPI contract, ensuring consumers can rely on the documented behavior.