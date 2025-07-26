# Spring Boot Consumer Contract Testing Best Practices - Bidirectional Approach

## Overview

This guide covers best practices for implementing consumer-side contract testing in Spring Boot applications using the bidirectional contract testing approach. Unlike traditional Pact testing where consumers drive the contract, bidirectional testing allows consumers to generate contracts from their actual usage patterns while validating against provider OpenAPI specifications.

## Key Concepts

### Bidirectional Consumer Testing Flow
1. **Mock-based Testing**: Test consumer behavior against Pact mock servers
2. **Contract Generation**: Generate consumer contracts (Pact files) from actual interactions
3. **Contract Validation**: Contracts are validated against provider OpenAPI specifications in PactFlow

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
    
    <!-- Spring Boot Web Starter -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    
    <!-- Pact Consumer JUnit 5 -->
    <dependency>
        <groupId>au.com.dius.pact.consumer</groupId>
        <artifactId>junit5</artifactId>
        <version>4.6.2</version>
        <scope>test</scope>
    </dependency>
    

    
    <!-- REST Assured for HTTP testing -->
    <dependency>
        <groupId>io.rest-assured</groupId>
        <artifactId>rest-assured</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### Pact Configuration
```java
@TestPropertySource(properties = {
    "pact.consumer.name=user-consumer",
    "pact.provider.name=user-provider",
    "pact.pactDirectory=target/pacts"
})
public abstract class BaseConsumerTest {
    
    protected static final String CONSUMER_NAME = "user-consumer";
    protected static final String PROVIDER_NAME = "user-provider";
    
    @MockBean
    protected UserService userService;
    
    @Autowired
    protected TestRestTemplate restTemplate;
}
```

### Pact Version Specification
Always specify the `pactVersion` in `@PactTestFor` for better compatibility and explicit version control:

```java
@PactTestFor(providerName = PROVIDER_NAME, pactVersion = PactSpecVersion.V3)
```

**Why specify Pact version?**
- **Compatibility**: Ensures consistent behavior across different Pact library versions
- **Features**: V3 supports additional features like multiple content types, generators, and matchers
- **Explicit Control**: Makes version requirements clear and prevents unexpected behavior
- **Future-proofing**: Protects against breaking changes in newer Pact versions

### Important System Properties Configuration

Pact-JVM provides many system properties for fine-tuning behavior. Here are the most important ones for consumer testing:

```java
@TestPropertySource(properties = {
    // Core Pact configuration
    "pact.consumer.name=user-consumer",
    "pact.provider.name=user-provider",
    "pact.pactDirectory=target/pacts",
    
    // Mock server configuration
    "pact.mockserver.addCloseHeader=true",  // Adds Connection: close header to responses
    
    // Pact file handling
    "pact.writer.overwrite=false",  // Merge with existing Pact files instead of overwriting
    
    // Default Pact version
    "pact.defaultVersion=V3",  // Set default Pact specification version
    
    // Analytics (optional)
    "pact_do_not_track=true"  // Disable anonymous metrics collection
})
public abstract class BaseConsumerTest {
    
    @BeforeAll
    static void setupSystemProperties() {
        // Alternative: Set system properties programmatically
        System.setProperty("pact.mockserver.addCloseHeader", "true");
        System.setProperty("pact.writer.overwrite", "false");
    }
}
```

### Work-in-Progress (WIP) Pacts

During active development, you may want to publish new or failing contracts without immediately breaking the build. PactFlow supports "Work-in-Progress" (WIP) mode for this purpose.

#### When to Use WIP Pacts
- **Feature Development**: When developing new consumer features that don't yet have provider support
- **Experimental Changes**: Testing contract changes before provider implementation
- **Gradual Migration**: Moving from one API version to another
- **Cross-team Coordination**: Allowing consumer teams to work ahead of provider teams

#### Publishing WIP Pacts
```bash
# Publish a WIP pact (won't fail provider builds)
pact-broker publish ./pacts \
  --consumer-app-version 1.1.0-feature-branch \
  --branch feature/new-user-fields \
  --tag wip

# Or using environment variables
export PACT_BROKER_PUBLISH_VERIFICATION_RESULTS=true
export PACT_BROKER_TAG=wip
pact-broker publish ./pacts --consumer-app-version 1.1.0-wip
```

#### WIP Configuration in Tests
```java
@TestPropertySource(properties = {
    "pact.consumer.name=user-consumer",
    "pact.provider.name=user-provider",
    "pact.consumer.tags=wip,feature-branch",  // Tag as WIP
    "pact.publish.results=true"
})
class WipConsumerTest extends BaseConsumerTest {
    
    @Test
    @PactTestFor(pactMethod = "createUserWithNewFieldsPact", pactVersion = PactSpecVersion.V3)
    void shouldCreateUserWithNewFields(MockServer mockServer) {
        // Test new functionality that provider doesn't support yet
        UserServiceClient client = createTestClient(mockServer.getUrl());
        
        CreateUserRequest request = new CreateUserRequest(
            "John Doe", 30, "john@example.com", 
            "Engineering", "Senior Developer"  // New fields not yet supported
        );
        
        // This test may fail against current provider, but won't break provider builds
        assertThatThrownBy(() -> client.createUser(request))
            .isInstanceOf(UserServiceException.class);
    }
}
```

#### WIP Best Practices
- **Clear Naming**: Use descriptive branch names and tags for WIP pacts
- **Communication**: Coordinate with provider teams about upcoming changes
- **Time-boxing**: Don't leave WIP pacts indefinitely - either implement or remove
- **Documentation**: Document what the WIP pact represents and when it should be implemented

### Key System Properties for Consumer Testing

| Property | Purpose | Common Values | When to Use |
|----------|---------|---------------|-------------|
| `pact.mockserver.addCloseHeader` | Adds Connection: close header | `true`, `false` | When experiencing connection issues with mock server |
| `pact.writer.overwrite` | Controls Pact file merging | `true`, `false` | Set to `false` to merge multiple test interactions |
| `pact.rootDir` | Override Pact files directory | Directory path | When using custom build structures |
| `pact.defaultVersion` | Default Pact specification version | `V1`, `V2`, `V3`, `V4` | Ensure consistent version across tests |
| `pact_do_not_track` | Disable analytics | `true`, `false` | For privacy or corporate environments |

### Environment Variable Configuration

You can also use environment variables (useful for CI/CD):

```bash
# Environment variables (can use screaming snake case)
export PACT_MOCKSERVER_ADDCLOSEHEADER=true
export PACT_WRITER_OVERWRITE=false
export PACT_DEFAULTVERSION=V3
export PACT_DO_NOT_TRACK=true
```

### Programmatic Configuration for Specific Issues

```java
@BeforeEach
void setupPactConfiguration() {
    // Fix for connection issues with mock server
    System.setProperty("pact.mockserver.addCloseHeader", "true");
    
    // Ensure Pact files are merged, not overwritten
    System.setProperty("pact.writer.overwrite", "false");
    
    // Set explicit Pact version
    System.setProperty("pact.defaultVersion", "V3");
}
```
```

## Core Consumer Testing Patterns

### 1. Basic Pact Consumer Test

```java
@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = PROVIDER_NAME, hostInterface = "localhost", pactVersion = PactSpecVersion.V3)
class UserConsumerPactTest extends BaseConsumerTest {sys
    
    @Pact(consumer = CONSUMER_NAME)
    public RequestResponsePact createUserPact(PactDslWithProvider builder) {
        return builder
            .given("user creation is available")
            .uponReceiving("a request to create a user")
            .path("/users")
            .method("POST")
            .headers(Map.of("Content-Type", "application/json"))
            .body(new PactDslJsonBody()
                .stringType("name", "John Doe")
                .integerType("age", 30)
                .stringType("email", "john.doe@example.com"))
            .willRespondWith()
            .status(201)
            .headers(Map.of("Content-Type", "application/json"))
            .body(new PactDslJsonBody()
                .uuid("id")
                .stringType("name", "John Doe")
                .integerType("age", 30)
                .stringType("email", "john.doe@example.com")
                .datetime("createdAt", "yyyy-MM-dd'T'HH:mm:ss.SSSX"))
            .toPact();
    }
    
    @Test
    @PactTestFor(pactMethod = "createUserPact")
    void shouldCreateUserSuccessfully(MockServer mockServer) {
        // Arrange
        UserClient userClient = new UserClient(mockServer.getUrl());
        CreateUserRequest request = new CreateUserRequest("John Doe", 30, "john.doe@example.com");
        
        // Act
        UserResponse response = userClient.createUser(request);
        
        // Assert
        assertThat(response).isNotNull();
        assertThat(response.getId()).isNotNull();
        assertThat(response.getName()).isEqualTo("John Doe");
        assertThat(response.getAge()).isEqualTo(30);
        assertThat(response.getEmail()).isEqualTo("john.doe@example.com");
        assertThat(response.getCreatedAt()).isNotNull();
    }
}
```

### 2. Error Scenario Testing
```java
@Pact(consumer = CONSUMER_NAME)
public RequestResponsePact createUserValidationErrorPact(PactDslWithProvider builder) {
    return builder
        .given("user creation validation is enabled")
        .uponReceiving("a request to create user with invalid age")
        .path("/users")
        .method("POST")
        .headers(Map.of("Content-Type", "application/json"))
        .body(new PactDslJsonBody()
            .stringType("name", "John Doe")
            .integerType("age", -5) // Invalid age
            .stringType("email", "john.doe@example.com"))
        .willRespondWith()
        .status(400)
        .headers(Map.of("Content-Type", "application/json"))
        .body(new PactDslJsonBody()
            .stringType("error", "Validation Failed")
            .stringType("message", "Age must be between 0 and 110")
            .stringType("field", "age")
            .integerType("rejectedValue", -5))
        .toPact();
}

@Test
@PactTestFor(pactMethod = "createUserValidationErrorPact", pactVersion = PactSpecVersion.V3)
void shouldHandleValidationErrors(MockServer mockServer) {
    // Arrange
    UserClient userClient = new UserClient(mockServer.getUrl());
    CreateUserRequest request = new CreateUserRequest("John Doe", -5, "john.doe@example.com");
    
    // Act & Assert
    assertThatThrownBy(() -> userClient.createUser(request))
        .isInstanceOf(UserValidationException.class)
        .hasMessageContaining("Age must be between 0 and 110");
}
```

### 3. Optional Fields Testing
```java
@Pact(consumer = CONSUMER_NAME)
public RequestResponsePact createUserWithoutOptionalFieldsPact(PactDslWithProvider builder) {
    return builder
        .given("user creation accepts optional fields")
        .uponReceiving("a request to create user without age")
        .path("/users")
        .method("POST")
        .headers(Map.of("Content-Type", "application/json"))
        .body(new PactDslJsonBody()
            .stringType("name", "Jane Smith")
            .stringType("email", "jane.smith@example.com"))
        .willRespondWith()
        .status(201)
        .headers(Map.of("Content-Type", "application/json"))
        .body(new PactDslJsonBody()
            .uuid("id")
            .stringType("name", "Jane Smith")
            .stringType("email", "jane.smith@example.com")
            .datetime("createdAt", "yyyy-MM-dd'T'HH:mm:ss.SSSX"))
        .toPact();
}

@Test
@PactTestFor(pactMethod = "createUserWithoutOptionalFieldsPact")
void shouldCreateUserWithoutOptionalFields(MockServer mockServer) {
    // Arrange
    UserClient userClient = new UserClient(mockServer.getUrl());
    CreateUserRequest request = new CreateUserRequest("Jane Smith", null, "jane.smith@example.com");
    
    // Act
    UserResponse response = userClient.createUser(request);
    
    // Assert
    assertThat(response).isNotNull();
    assertThat(response.getName()).isEqualTo("Jane Smith");
    assertThat(response.getEmail()).isEqualTo("jane.smith@example.com");
    assertThat(response.getAge()).isNull();
}
```

## Advanced Consumer Testing Patterns

### 1. Multiple Interactions in Single Test
```java
@Pact(consumer = CONSUMER_NAME)
public RequestResponsePact userLifecyclePact(PactDslWithProvider builder) {
    return builder
        .given("user management is available")
        .uponReceiving("a request to create a user")
        .path("/users")
        .method("POST")
        .headers(Map.of("Content-Type", "application/json"))
        .body(new PactDslJsonBody()
            .stringType("name", "John Doe")
            .integerType("age", 30)
            .stringType("email", "john.doe@example.com"))
        .willRespondWith()
        .status(201)
        .headers(Map.of("Content-Type", "application/json"))
        .body(new PactDslJsonBody()
            .uuid("id", "550e8400-e29b-41d4-a716-446655440000")
            .stringType("name", "John Doe")
            .integerType("age", 30)
            .stringType("email", "john.doe@example.com"))
        
        .uponReceiving("a request to get the created user")
        .path("/users/550e8400-e29b-41d4-a716-446655440000")
        .method("GET")
        .willRespondWith()
        .status(200)
        .headers(Map.of("Content-Type", "application/json"))
        .body(new PactDslJsonBody()
            .uuid("id", "550e8400-e29b-41d4-a716-446655440000")
            .stringType("name", "John Doe")
            .integerType("age", 30)
            .stringType("email", "john.doe@example.com"))
        .toPact();
}

@Test
@PactTestFor(pactMethod = "userLifecyclePact")
void shouldHandleUserLifecycle(MockServer mockServer) {
    UserClient userClient = new UserClient(mockServer.getUrl());
    
    // Create user
    CreateUserRequest createRequest = new CreateUserRequest("John Doe", 30, "john.doe@example.com");
    UserResponse createdUser = userClient.createUser(createRequest);
    
    // Get user
    UserResponse retrievedUser = userClient.getUser(createdUser.getId());
    
    // Verify both operations
    assertThat(createdUser.getId()).isEqualTo(retrievedUser.getId());
    assertThat(retrievedUser.getName()).isEqualTo("John Doe");
}
```

### 2. State-based Testing
```java
@Pact(consumer = CONSUMER_NAME)
public RequestResponsePact getUserWithExistingUserPact(PactDslWithProvider builder) {
    return builder
        .given("user with id 123 exists")
        .uponReceiving("a request to get user by id")
        .path("/users/123")
        .method("GET")
        .willRespondWith()
        .status(200)
        .headers(Map.of("Content-Type", "application/json"))
        .body(new PactDslJsonBody()
            .stringType("id", "123")
            .stringType("name", "Existing User")
            .integerType("age", 25)
            .stringType("email", "existing@example.com"))
        .toPact();
}

@Pact(consumer = CONSUMER_NAME)
public RequestResponsePact getUserNotFoundPact(PactDslWithProvider builder) {
    return builder
        .given("user with id 999 does not exist")
        .uponReceiving("a request to get non-existent user")
        .path("/users/999")
        .method("GET")
        .willRespondWith()
        .status(404)
        .headers(Map.of("Content-Type", "application/json"))
        .body(new PactDslJsonBody()
            .stringType("error", "User not found")
            .stringType("message", "User with id 999 does not exist"))
        .toPact();
}
```

### 3. Query Parameters and Headers
```java
@Pact(consumer = CONSUMER_NAME)
public RequestResponsePact searchUsersPact(PactDslWithProvider builder) {
    return builder
        .given("users exist in the system")
        .uponReceiving("a request to search users with filters")
        .path("/users")
        .method("GET")
        .query("department=Engineering&minAge=25&maxAge=40")
        .headers(Map.of(
            "Accept", "application/json",
            "Authorization", "Bearer token123"
        ))
        .willRespondWith()
        .status(200)
        .headers(Map.of("Content-Type", "application/json"))
        .body(new PactDslJsonArray()
            .object()
                .uuid("id")
                .stringType("name", "John Engineer")
                .integerType("age", 30)
                .stringType("email", "john@example.com")
                .stringType("department", "Engineering")
            .closeObject()
            .object()
                .uuid("id")
                .stringType("name", "Jane Developer")
                .integerType("age", 28)
                .stringType("email", "jane@example.com")
                .stringType("department", "Engineering")
            .closeObject())
        .toPact();
}

@Test
@PactTestFor(pactMethod = "searchUsersPact")
void shouldSearchUsersWithFilters(MockServer mockServer) {
    UserClient userClient = new UserClient(mockServer.getUrl());
    
    SearchCriteria criteria = SearchCriteria.builder()
        .department("Engineering")
        .minAge(25)
        .maxAge(40)
        .build();
    
    List<UserResponse> users = userClient.searchUsers(criteria, "Bearer token123");
    
    assertThat(users).hasSize(2);
    assertThat(users).allMatch(user -> user.getDepartment().equals("Engineering"));
    assertThat(users).allMatch(user -> user.getAge() >= 25 && user.getAge() <= 40);
}
```

## Consumer Client Implementation

### 1. HTTP Interface Approach (Recommended)
```java
// Define the HTTP Interface
public interface UserServiceClient {
    
    @PostExchange("/users")
    UserResponse createUser(@RequestBody CreateUserRequest request);
    
    @GetExchange("/users/{userId}")
    UserResponse getUser(@PathVariable String userId);
    
    @GetExchange("/users")
    List<UserResponse> searchUsers(
        @RequestParam(required = false) String department,
        @RequestParam(required = false) Integer minAge,
        @RequestParam(required = false) Integer maxAge,
        @RequestHeader("Authorization") String authToken
    );
}

// Configuration for HTTP Interface with JDK HttpClient
@Configuration
public class HttpClientConfiguration {
    
    @Bean
    public UserServiceClient userServiceClient(@Value("${user.service.url}") String baseUrl) {
        RestClient restClient = RestClient.builder()
            .baseUrl(baseUrl)
            .requestFactory(new JdkClientHttpRequestFactory())
            .defaultStatusHandler(HttpStatusCode::is4xxClientError, this::handleClientError)
            .defaultStatusHandler(HttpStatusCode::is5xxServerError, this::handleServerError)
            .build();
        
        HttpServiceProxyFactory factory = HttpServiceProxyFactory
            .builderFor(RestClientAdapter.create(restClient))
            .build();
        
        return factory.createClient(UserServiceClient.class);
    }
    
    // Constructor for testing with custom base URL
    public UserServiceClient createTestClient(String baseUrl) {
        RestClient restClient = RestClient.builder()
            .baseUrl(baseUrl)
            .requestFactory(new JdkClientHttpRequestFactory())
            .defaultStatusHandler(HttpStatusCode::is4xxClientError, this::handleClientError)
            .defaultStatusHandler(HttpStatusCode::is5xxServerError, this::handleServerError)
            .build();
        
        HttpServiceProxyFactory factory = HttpServiceProxyFactory
            .builderFor(RestClientAdapter.create(restClient))
            .build();
        
        return factory.createClient(UserServiceClient.class);
    }
    
    private void handleClientError(HttpRequest request, ClientHttpResponse response) throws IOException {
        String responseBody = StreamUtils.copyToString(response.getBody(), StandardCharsets.UTF_8);
        
        switch (response.getStatusCode().value()) {
            case 400:
                ErrorResponse error = parseErrorResponse(responseBody);
                throw new UserValidationException(error.getMessage());
            case 404:
                throw new UserNotFoundException("User not found");
            case 409:
                throw new UserConflictException("User already exists");
            default:
                throw new UserServiceException("Client error: " + response.getStatusCode());
        }
    }
    
    private void handleServerError(HttpRequest request, ClientHttpResponse response) throws IOException {
        throw new UserServiceException("Server error: " + response.getStatusCode());
    }
    
    private ErrorResponse parseErrorResponse(String responseBody) {
        try {
            ObjectMapper mapper = new ObjectMapper();
            return mapper.readValue(responseBody, ErrorResponse.class);
        } catch (Exception e) {
            return new ErrorResponse("Unknown error", responseBody);
        }
    }
}

// Service layer that uses the HTTP Interface
@Service
public class UserService {
    
    private final UserServiceClient userServiceClient;
    
    public UserService(UserServiceClient userServiceClient) {
        this.userServiceClient = userServiceClient;
    }
    
    public UserResponse createUser(CreateUserRequest request) {
        return userServiceClient.createUser(request);
    }
    
    public UserResponse getUser(String userId) {
        return userServiceClient.getUser(userId);
    }
    
    public List<UserResponse> searchUsers(SearchCriteria criteria, String authToken) {
        return userServiceClient.searchUsers(
            criteria.getDepartment(),
            criteria.getMinAge(),
            criteria.getMaxAge(),
            authToken
        );
    }
}
```

### 2. RestClient with Apache HttpClient
```java
@Configuration
public class HttpClientConfiguration {
    
    @Bean
    public UserServiceClient userServiceClientWithApache(@Value("${user.service.url}") String baseUrl) {
        // Configure Apache HttpClient
        CloseableHttpClient httpClient = HttpClients.custom()
            .setConnectionTimeToLive(30, TimeUnit.SECONDS)
            .setMaxConnTotal(100)
            .setMaxConnPerRoute(20)
            .build();
        
        RestClient restClient = RestClient.builder()
            .baseUrl(baseUrl)
            .requestFactory(new HttpComponentsClientHttpRequestFactory(httpClient))
            .defaultStatusHandler(HttpStatusCode::is4xxClientError, this::handleClientError)
            .defaultStatusHandler(HttpStatusCode::is5xxServerError, this::handleServerError)
            .build();
        
        HttpServiceProxyFactory factory = HttpServiceProxyFactory
            .builderFor(RestClientAdapter.create(restClient))
            .build();
        
        return factory.createClient(UserServiceClient.class);
    }
    
    // Error handling methods same as above...
}
```

### 3. Direct RestClient Implementation (Alternative)
```java
@Component
public class UserClient {
    
    private final RestClient restClient;
    
    public UserClient(@Value("${user.service.url}") String baseUrl) {
        this.restClient = RestClient.builder()
            .baseUrl(baseUrl)
            .requestFactory(new JdkClientHttpRequestFactory())
            .defaultStatusHandler(HttpStatusCode::is4xxClientError, this::handleClientError)
            .defaultStatusHandler(HttpStatusCode::is5xxServerError, this::handleServerError)
            .build();
    }
    
    // Constructor for testing
    public UserClient(String baseUrl) {
        this.restClient = RestClient.builder()
            .baseUrl(baseUrl)
            .requestFactory(new JdkClientHttpRequestFactory())
            .defaultStatusHandler(HttpStatusCode::is4xxClientError, this::handleClientError)
            .defaultStatusHandler(HttpStatusCode::is5xxServerError, this::handleServerError)
            .build();
    }
    
    public UserResponse createUser(CreateUserRequest request) {
        return restClient.post()
            .uri("/users")
            .contentType(MediaType.APPLICATION_JSON)
            .body(request)
            .retrieve()
            .body(UserResponse.class);
    }
    
    public UserResponse getUser(String userId) {
        return restClient.get()
            .uri("/users/{userId}", userId)
            .accept(MediaType.APPLICATION_JSON)
            .retrieve()
            .body(UserResponse.class);
    }
    
    public List<UserResponse> searchUsers(SearchCriteria criteria, String authToken) {
        return restClient.get()
            .uri(uriBuilder -> uriBuilder
                .path("/users")
                .queryParamIfPresent("department", Optional.ofNullable(criteria.getDepartment()))
                .queryParamIfPresent("minAge", Optional.ofNullable(criteria.getMinAge()))
                .queryParamIfPresent("maxAge", Optional.ofNullable(criteria.getMaxAge()))
                .build())
            .header("Authorization", authToken)
            .accept(MediaType.APPLICATION_JSON)
            .retrieve()
            .body(new ParameterizedTypeReference<List<UserResponse>>() {});
    }
    
    private void handleClientError(HttpRequest request, ClientHttpResponse response) throws IOException {
        String responseBody = StreamUtils.copyToString(response.getBody(), StandardCharsets.UTF_8);
        
        switch (response.getStatusCode().value()) {
            case 400:
                ErrorResponse error = parseErrorResponse(responseBody);
                throw new UserValidationException(error.getMessage());
            case 404:
                throw new UserNotFoundException("User not found");
            case 409:
                throw new UserConflictException("User already exists");
            default:
                throw new UserServiceException("Client error: " + response.getStatusCode());
        }
    }
    
    private void handleServerError(HttpRequest request, ClientHttpResponse response) throws IOException {
        throw new UserServiceException("Server error: " + response.getStatusCode());
    }
    
    private ErrorResponse parseErrorResponse(String responseBody) {
        try {
            ObjectMapper mapper = new ObjectMapper();
            return mapper.readValue(responseBody, ErrorResponse.class);
        } catch (Exception e) {
            return new ErrorResponse("Unknown error", responseBody);
        }
    }
}
```

## Best Practices for Consumer Testing

### 1. Test Organization
```java
@Nested
@DisplayName("User Creation Tests")
class UserCreationTests {
    
    @Nested
    @DisplayName("Successful Creation")
    class SuccessfulCreation {
        
        @Test
        @PactTestFor(pactMethod = "createUserPact")
        void shouldCreateUserWithAllFields(MockServer mockServer) {
            // Test implementation
        }
        
        @Test
        @PactTestFor(pactMethod = "createUserWithoutOptionalFieldsPact")
        void shouldCreateUserWithRequiredFieldsOnly(MockServer mockServer) {
            // Test implementation
        }
    }
    
    @Nested
    @DisplayName("Validation Errors")
    class ValidationErrors {
        
        @Test
        @PactTestFor(pactMethod = "createUserValidationErrorPact")
        void shouldHandleInvalidAge(MockServer mockServer) {
            // Test implementation
        }
    }
}
```

### 2. Reusable Pact Builders
```java
public class UserPactBuilders {
    
    public static PactDslJsonBody validUserRequest() {
        return new PactDslJsonBody()
            .stringType("name", "John Doe")
            .integerType("age", 30)
            .stringType("email", "john.doe@example.com");
    }
    
    public static PactDslJsonBody validUserResponse() {
        return new PactDslJsonBody()
            .uuid("id")
            .stringType("name", "John Doe")
            .integerType("age", 30)
            .stringType("email", "john.doe@example.com")
            .datetime("createdAt", "yyyy-MM-dd'T'HH:mm:ss.SSSX");
    }
    
    public static PactDslJsonBody validationErrorResponse(String field, String message) {
        return new PactDslJsonBody()
            .stringType("error", "Validation Failed")
            .stringType("message", message)
            .stringType("field", field);
    }
}

// Usage in tests
@Pact(consumer = CONSUMER_NAME)
public RequestResponsePact createUserPact(PactDslWithProvider builder) {
    return builder
        .given("user creation is available")
        .uponReceiving("a request to create a user")
        .path("/users")
        .method("POST")
        .headers(Map.of("Content-Type", "application/json"))
        .body(UserPactBuilders.validUserRequest())
        .willRespondWith()
        .status(201)
        .headers(Map.of("Content-Type", "application/json"))
        .body(UserPactBuilders.validUserResponse())
        .toPact();
}
```

### 3. Environment-Specific Configuration
```java
@TestPropertySource(properties = {
    "user.service.url=http://localhost:8080",
    "pact.consumer.name=user-consumer-test",
    "pact.provider.name=user-provider-test",
    "logging.level.au.com.dius.pact=DEBUG"
})
class UserConsumerIntegrationTest extends BaseConsumerTest {
    // Integration tests with real service
}

@TestPropertySource(properties = {
    "user.service.url=http://localhost:${pact.mock.port}",
    "pact.consumer.name=user-consumer",
    "pact.provider.name=user-provider"
})
class UserConsumerPactTest extends BaseConsumerTest {
    // Pact contract tests
}
```

### 4. Contract Validation Helpers
```java
public class ContractValidationHelper {
    
    public static void validateUserResponse(UserResponse response) {
        assertThat(response).isNotNull();
        assertThat(response.getId()).isNotNull().matches(UUID_PATTERN);
        assertThat(response.getName()).isNotBlank();
        assertThat(response.getEmail()).isNotBlank().contains("@");
        assertThat(response.getCreatedAt()).isNotNull();
    }
    
    public static void validateErrorResponse(Exception exception, String expectedMessage) {
        assertThat(exception).isNotNull();
        assertThat(exception.getMessage()).contains(expectedMessage);
    }
    
    private static final String UUID_PATTERN = 
        "^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$";
}
```

## Contract Generation

Consumer tests automatically generate Pact files in the `target/pacts` directory when they run. These files contain the contract specifications based on your actual test interactions with the mock server.

### Single Provider Scenario
```java
@Test
void shouldGenerateContractFiles() {
    // When Pact tests run, they automatically generate contract files
    // The files are saved to the directory specified in pact.pactDirectory
    
    // Verify contracts were generated after running tests
    Path pactDirectory = Paths.get("target/pacts");
    assertThat(pactDirectory).exists();
    assertThat(pactDirectory.resolve("user-consumer-user-provider.json")).exists();
}
```

### Multiple Provider Scenario

When your consumer application depends on multiple upstream services, you will have **multiple contract files** in the pacts folder. Each consumer-provider pair generates its own contract file.

#### Example: E-commerce Consumer with Multiple Providers

```
target/pacts/
├── ecommerce-frontend-user-service.json          # User management contract
├── ecommerce-frontend-product-service.json       # Product catalog contract
├── ecommerce-frontend-payment-service.json       # Payment processing contract
├── ecommerce-frontend-inventory-service.json     # Inventory management contract
└── ecommerce-frontend-notification-service.json  # Email/SMS notifications contract
```

#### Organizing Tests for Multiple Providers

```java
// Base configuration for all provider tests
@TestPropertySource(properties = {
    "pact.consumer.name=ecommerce-frontend",
    "pact.pactDirectory=target/pacts"
})
public abstract class BaseConsumerTest {
    // Common test configuration
}

// User Service Contract Tests
@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "user-service", hostInterface = "localhost", pactVersion = PactSpecVersion.V3)
class UserServiceConsumerTest extends BaseConsumerTest {
    
    private static final String PROVIDER_NAME = "user-service";
    
    @Pact(consumer = "ecommerce-frontend")
    public RequestResponsePact createUserPact(PactDslWithProvider builder) {
        return builder
            .given("user creation is available")
            .uponReceiving("a request to create a user")
            .path("/users")
            .method("POST")
            .headers(Map.of("Content-Type", "application/json"))
            .body(new PactDslJsonBody()
                .stringType("name", "John Doe")
                .stringType("email", "john@example.com"))
            .willRespondWith()
            .status(201)
            .body(new PactDslJsonBody()
                .uuid("id")
                .stringType("name", "John Doe")
                .stringType("email", "john@example.com"))
            .toPact();
    }
    
    @Test
    @PactTestFor(pactMethod = "createUserPact", pactVersion = PactSpecVersion.V3)
    void shouldCreateUser(MockServer mockServer) {
        UserServiceClient client = new UserServiceClient(mockServer.getUrl());
        UserResponse response = client.createUser(new CreateUserRequest("John Doe", "john@example.com"));
        assertThat(response.getName()).isEqualTo("John Doe");
    }
}

// Product Service Contract Tests
@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "product-service", hostInterface = "localhost", pactVersion = PactSpecVersion.V3)
class ProductServiceConsumerTest extends BaseConsumerTest {
    
    private static final String PROVIDER_NAME = "product-service";
    
    @Pact(consumer = "ecommerce-frontend")
    public RequestResponsePact getProductsPact(PactDslWithProvider builder) {
        return builder
            .given("products exist in catalog")
            .uponReceiving("a request to get products")
            .path("/products")
            .method("GET")
            .query("category=electronics&limit=10")
            .willRespondWith()
            .status(200)
            .body(new PactDslJsonArray()
                .object()
                    .uuid("id")
                    .stringType("name", "Laptop")
                    .numberType("price", 999.99)
                    .stringType("category", "electronics")
                .closeObject())
            .toPact();
    }
    
    @Test
    @PactTestFor(pactMethod = "getProductsPact", pactVersion = PactSpecVersion.V3)
    void shouldGetProducts(MockServer mockServer) {
        ProductServiceClient client = new ProductServiceClient(mockServer.getUrl());
        List<Product> products = client.getProducts("electronics", 10);
        assertThat(products).hasSize(1);
        assertThat(products.get(0).getCategory()).isEqualTo("electronics");
    }
}

// Payment Service Contract Tests
@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "payment-service", hostInterface = "localhost", pactVersion = PactSpecVersion.V3)
class PaymentServiceConsumerTest extends BaseConsumerTest {
    
    private static final String PROVIDER_NAME = "payment-service";
    
    @Pact(consumer = "ecommerce-frontend")
    public RequestResponsePact processPaymentPact(PactDslWithProvider builder) {
        return builder
            .given("payment processing is available")
            .uponReceiving("a request to process payment")
            .path("/payments")
            .method("POST")
            .headers(Map.of("Content-Type", "application/json"))
            .body(new PactDslJsonBody()
                .numberType("amount", 99.99)
                .stringType("currency", "USD")
                .stringType("cardToken", "tok_visa"))
            .willRespondWith()
            .status(200)
            .body(new PactDslJsonBody()
                .uuid("transactionId")
                .stringType("status", "completed")
                .numberType("amount", 99.99))
            .toPact();
    }
    
    @Test
    @PactTestFor(pactMethod = "processPaymentPact", pactVersion = PactSpecVersion.V3)
    void shouldProcessPayment(MockServer mockServer) {
        PaymentServiceClient client = new PaymentServiceClient(mockServer.getUrl());
        PaymentResponse response = client.processPayment(
            new PaymentRequest(99.99, "USD", "tok_visa"));
        assertThat(response.getStatus()).isEqualTo("completed");
    }
}
```

#### Contract File Naming Convention

Pact automatically generates contract files using the pattern:
```
{consumer-name}-{provider-name}.json
```

**Examples:**
- `ecommerce-frontend-user-service.json`
- `ecommerce-frontend-product-service.json`
- `ecommerce-frontend-payment-service.json`

#### Managing Multiple Contracts

```java
@Test
void shouldGenerateAllProviderContracts() {
    Path pactDirectory = Paths.get("target/pacts");
    
    // Verify all expected contract files are generated
    assertThat(pactDirectory.resolve("ecommerce-frontend-user-service.json")).exists();
    assertThat(pactDirectory.resolve("ecommerce-frontend-product-service.json")).exists();
    assertThat(pactDirectory.resolve("ecommerce-frontend-payment-service.json")).exists();
    assertThat(pactDirectory.resolve("ecommerce-frontend-inventory-service.json")).exists();
    assertThat(pactDirectory.resolve("ecommerce-frontend-notification-service.json")).exists();
}
```

#### Publishing Multiple Contracts

When publishing to PactFlow, all contract files in the pacts directory are published:

```bash
# Publishes ALL contract files in the pacts directory
pact-broker publish ./target/pacts \
  --consumer-app-version 1.0.0 \
  --branch main \
  --broker-base-url https://your-pactflow-broker.com
```

This will publish:
- `ecommerce-frontend` ↔ `user-service` contract
- `ecommerce-frontend` ↔ `product-service` contract  
- `ecommerce-frontend` ↔ `payment-service` contract
- And so on...

#### Best Practices for Multiple Providers

1. **Separate Test Classes**: Create separate test classes for each provider
2. **Consistent Naming**: Use clear, descriptive provider names
3. **Shared Configuration**: Use base test classes for common configuration
4. **Independent Testing**: Each provider contract should be testable independently
5. **Clear Organization**: Group related interactions within each provider test class

## Common Pitfalls and Solutions

### 1. Avoid Over-Specification
```java
// ❌ Bad - Too specific, brittle
.body(new PactDslJsonBody()
    .stringValue("name", "John Doe") // Exact match
    .integerValue("age", 30)         // Exact match
    .stringValue("email", "john.doe@example.com"))

// ✅ Good - Type matching, flexible
.body(new PactDslJsonBody()
    .stringType("name", "John Doe")    // Any string
    .integerType("age", 30)            // Any integer
    .stringType("email", "john.doe@example.com")) // Any string
```

### 2. Handle Optional Fields Correctly
```java
// ❌ Bad - Assumes field is always present
.body(new PactDslJsonBody()
    .stringType("name")
    .integerType("age")  // This makes age mandatory
    .stringType("email"))

// ✅ Good - Makes age optional
.body(new PactDslJsonBody()
    .stringType("name")
    .optionalIntegerType("age")  // Optional field
    .stringType("email"))
```

### 3. Test Real Consumer Behavior
```java
// ❌ Bad - Testing mock behavior
@Test
void shouldReturnUser() {
    UserResponse response = mockUserService.getUser("123");
    assertThat(response).isNotNull();
}

// ✅ Good - Testing actual consumer logic
@Test
@PactTestFor(pactMethod = "getUserPact")
void shouldProcessUserDataCorrectly(MockServer mockServer) {
    UserClient client = new UserClient(mockServer.getUrl());
    UserService service = new UserService(client);
    
    // Test actual business logic that uses the client
    UserProfile profile = service.createUserProfile("123");
    
    assertThat(profile.getDisplayName()).isEqualTo("John Doe");
    assertThat(profile.isAdult()).isTrue();
}
```

### 4. Proper Error Handling Testing
```java
// ❌ Bad - Not testing error handling
@Test
void shouldCreateUser() {
    UserResponse response = userClient.createUser(validRequest);
    assertThat(response).isNotNull();
}

// ✅ Good - Testing error scenarios
@Test
@PactTestFor(pactMethod = "createUserValidationErrorPact")
void shouldHandleValidationErrorsGracefully(MockServer mockServer) {
    UserClient client = new UserClient(mockServer.getUrl());
    CreateUserRequest invalidRequest = new CreateUserRequest("", -5, "invalid");
    
    assertThatThrownBy(() -> client.createUser(invalidRequest))
        .isInstanceOf(UserValidationException.class)
        .hasMessageContaining("validation failed");
}
```

## Conclusion

Following these best practices ensures your Spring Boot consumer tests are:

- **Contract-Driven**: Generate contracts from actual consumer usage patterns
- **Realistic**: Test real consumer behavior, not just mock responses
- **Maintainable**: Use reusable builders and helpers for consistency
- **Comprehensive**: Cover success scenarios, error cases, and edge conditions
- **Integration-Ready**: Seamlessly integrate with bidirectional contract testing workflows

Remember that consumer tests should focus on how your application actually uses the provider's API, ensuring the generated contracts represent real usage patterns that can be validated against provider OpenAPI specifications.