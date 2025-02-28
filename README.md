# spring-boot-reference

# Spring Boot Reference Card

## Core Concepts

### Spring Boot Basics
- **Spring Boot**: Opinionated framework that simplifies Spring application development
- **Auto-configuration**: Automatically configures beans based on classpath, properties, etc.
- **Starter dependencies**: Curated dependencies that simplify build configuration
- **Standalone applications**: Built as executable JARs with embedded servers
- **Spring Boot Actuator**: Production-ready features for monitoring and management

### Application Setup
```java
@SpringBootApplication
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
```

### Main Annotations
- `@SpringBootApplication`: Combines `@Configuration`, `@EnableAutoConfiguration`, and `@ComponentScan`
- `@Configuration`: Marks a class as a source of bean definitions
- `@EnableAutoConfiguration`: Enables Spring Boot auto-configuration
- `@ComponentScan`: Scans for Spring components in the package and sub-packages
- `@ConfigurationProperties`: Binds external configuration to Java objects

## Configuration

### Property Sources (in order of precedence)
1. Command-line arguments
2. JNDI attributes
3. Java System properties
4. OS environment variables
5. Profile-specific properties (application-{profile}.properties)
6. Application properties (application.properties or application.yml)
7. Default properties

### Application Properties Example
```properties
# Server configuration
server.port=8080
server.servlet.context-path=/myapp

# Database configuration
spring.datasource.url=jdbc:mysql://localhost:3306/mydb
spring.datasource.username=user
spring.datasource.password=password
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

# JPA/Hibernate
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
```

### Configuration Classes
```java
@Configuration
@PropertySource("classpath:custom.properties")
public class AppConfig {
    @Bean
    public Service myService() {
        return new ServiceImpl();
    }
}
```

### Profiles
```java
@Configuration
@Profile("dev")
public class DevConfig {
    // Dev-specific beans
}

// Activation
// In properties file: spring.profiles.active=dev
// Or programmatically:
SpringApplication.setAdditionalProfiles("dev");
```

## Web Development

### REST Controller
```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    @Autowired
    private UserService userService;

    @GetMapping
    public List<User> getAllUsers() {
        return userService.findAll();
    }

    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("User not found"));
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public User createUser(@Valid @RequestBody User user) {
        return userService.save(user);
    }

    @PutMapping("/{id}")
    public User updateUser(@PathVariable Long id, @Valid @RequestBody User user) {
        return userService.update(id, user);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void deleteUser(@PathVariable Long id) {
        userService.deleteById(id);
    }
}
```

### Common Web Annotations
- `@Controller`: Marks a class as a web controller
- `@RestController`: Combines `@Controller` and `@ResponseBody`
- `@RequestMapping`: Maps web requests to methods
- HTTP method-specific: `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`, `@PatchMapping`
- `@PathVariable`: Extracts values from URI path
- `@RequestParam`: Extracts query parameters
- `@RequestBody`: Binds request body to an object
- `@Valid`: Triggers validation of an object
- `@ResponseStatus`: Specifies the HTTP status code to return

## Data Access

### JPA Entity
```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 50)
    @NotBlank(message = "Name is required")
    private String name;

    @Column(unique = true)
    @Email(message = "Email should be valid")
    private String email;

    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private List<Order> orders = new ArrayList<>();

    // Getters, setters, constructors, etc.
}
```

### Spring Data Repository
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    List<User> findByNameContaining(String name);
    
    Optional<User> findByEmail(String email);
    
    @Query("SELECT u FROM User u WHERE u.email LIKE %:domain%")
    List<User> findByEmailDomain(@Param("domain") String domain);
    
    @Modifying
    @Query("UPDATE User u SET u.status = :status WHERE u.lastLoginDate < :date")
    int updateStatusForInactiveUsers(@Param("status") String status, @Param("date") LocalDate date);
}
```

### Transactional Support
```java
@Service
public class UserServiceImpl implements UserService {
    @Autowired
    private UserRepository userRepository;
    
    @Override
    @Transactional(readOnly = true)
    public List<User> findAll() {
        return userRepository.findAll();
    }
    
    @Override
    @Transactional
    public User save(User user) {
        // Business logic
        return userRepository.save(user);
    }
    
    @Override
    @Transactional(propagation = Propagation.REQUIRED, isolation = Isolation.READ_COMMITTED)
    public void complexOperation() {
        // Complex operation with specific transaction attributes
    }
}
```

## Security

### Basic Configuration
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Autowired
    private UserDetailsService userDetailsService;
    
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .csrf().disable()
            .authorizeRequests()
                .antMatchers("/api/public/**").permitAll()
                .antMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            .and()
            .formLogin()
                .loginPage("/login")
                .permitAll()
            .and()
            .logout()
                .permitAll();
    }
    
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userDetailsService)
            .passwordEncoder(passwordEncoder());
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### JWT Authentication
```java
@Configuration
public class JwtConfig {
    @Value("${jwt.secret}")
    private String secret;
    
    @Value("${jwt.expiration}")
    private long expiration;
    
    @Bean
    public JwtTokenUtil jwtTokenUtil() {
        return new JwtTokenUtil(secret, expiration);
    }
    
    @Bean
    public JwtAuthenticationFilter jwtAuthenticationFilter() {
        return new JwtAuthenticationFilter(jwtTokenUtil, userDetailsService);
    }
}
```

## Testing

### Unit Testing
```java
@RunWith(MockitoJUnitRunner.class)
public class UserServiceTest {
    @Mock
    private UserRepository userRepository;
    
    @InjectMocks
    private UserServiceImpl userService;
    
    @Test
    public void whenFindById_thenReturnUser() {
        // Arrange
        User user = new User();
        user.setId(1L);
        user.setName("Test User");
        
        when(userRepository.findById(1L)).thenReturn(Optional.of(user));
        
        // Act
        Optional<User> found = userService.findById(1L);
        
        // Assert
        assertTrue(found.isPresent());
        assertEquals("Test User", found.get().getName());
        verify(userRepository).findById(1L);
    }
}
```

### Integration Testing
```java
@RunWith(SpringRunner.class)
@SpringBootTest
@AutoConfigureMockMvc
public class UserControllerIntegrationTest {
    @Autowired
    private MockMvc mockMvc;
    
    @Autowired
    private ObjectMapper objectMapper;
    
    @MockBean
    private UserService userService;
    
    @Test
    public void whenGetUser_thenReturnUser() throws Exception {
        User user = new User();
        user.setId(1L);
        user.setName("Test User");
        
        when(userService.findById(1L)).thenReturn(Optional.of(user));
        
        mockMvc.perform(get("/api/users/1")
                .contentType(MediaType.APPLICATION_JSON))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.name", is("Test User")));
    }
}
```

### Slice Testing
```java
@DataJpaTest
public class UserRepositoryTest {
    @Autowired
    private UserRepository userRepository;
    
    @Test
    public void whenFindByEmail_thenReturnUser() {
        // Arrange
        User user = new User();
        user.setEmail("test@example.com");
        user.setName("Test User");
        userRepository.save(user);
        
        // Act
        Optional<User> found = userRepository.findByEmail("test@example.com");
        
        // Assert
        assertTrue(found.isPresent());
        assertEquals("Test User", found.get().getName());
    }
}
```

## Advanced Features

### Caching
```java
@Configuration
@EnableCaching
public class CacheConfig {
    @Bean
    public CacheManager cacheManager() {
        return new ConcurrentMapCacheManager("users", "products");
    }
}

@Service
public class CachedUserService {
    @Cacheable(value = "users", key = "#id")
    public User getUserById(Long id) {
        // This will be cached
        return userRepository.findById(id).orElse(null);
    }
    
    @CachePut(value = "users", key = "#user.id")
    public User updateUser(User user) {
        // Updates the cache
        return userRepository.save(user);
    }
    
    @CacheEvict(value = "users", key = "#id")
    public void deleteUser(Long id) {
        // Removes from cache
        userRepository.deleteById(id);
    }
}
```

### Messaging with Spring AMQP (RabbitMQ)
```java
@Configuration
public class RabbitConfig {
    @Bean
    public Queue ordersQueue() {
        return new Queue("orders.queue", true);
    }
    
    @Bean
    public TopicExchange ordersExchange() {
        return new TopicExchange("orders.exchange");
    }
    
    @Bean
    public Binding binding(Queue queue, TopicExchange exchange) {
        return BindingBuilder.bind(queue).to(exchange).with("orders.#");
    }
}

// Producer
@Service
public class OrderProducer {
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    public void sendOrder(Order order) {
        rabbitTemplate.convertAndSend("orders.exchange", "orders.created", order);
    }
}

// Consumer
@Service
public class OrderConsumer {
    @RabbitListener(queues = "orders.queue")
    public void processOrder(Order order) {
        // Process the received order
    }
}
```

### Scheduling
```java
@Configuration
@EnableScheduling
public class SchedulingConfig {
}

@Service
public class ScheduledTasks {
    @Scheduled(fixedRate = 60000) // Every minute
    public void performTask() {
        // Task logic
    }
    
    @Scheduled(cron = "0 0 2 * * *") // 2 AM every day
    public void performDailyTask() {
        // Daily task logic
    }
}
```

### Actuator
```properties
# Enable all endpoints
management.endpoints.web.exposure.include=*

# Custom endpoints
management.endpoint.health.show-details=always
management.endpoint.health.group.custom.include=diskSpace,db
```

## Best Practices

### Application Structure
```
com.example.myapp/
├── MyAppApplication.java
├── config/
│   ├── SecurityConfig.java
│   ├── JpaConfig.java
│   └── ...
├── controller/
│   ├── UserController.java
│   └── ...
├── model/
│   ├── User.java
│   └── ...
├── repository/
│   ├── UserRepository.java
│   └── ...
├── service/
│   ├── UserService.java
│   ├── UserServiceImpl.java
│   └── ...
├── exception/
│   ├── ResourceNotFoundException.java
│   └── ...
└── dto/
    ├── UserDto.java
    └── ...
```

### Coding Best Practices
1. **Use DTOs for API responses/requests**
   - Separate domain models from API contracts
   - Use mappers like MapStruct

2. **Implement proper exception handling**
   ```java
   @ControllerAdvice
   public class GlobalExceptionHandler {
       @ExceptionHandler(ResourceNotFoundException.class)
       @ResponseStatus(HttpStatus.NOT_FOUND)
       public ErrorResponse handleResourceNotFound(ResourceNotFoundException ex) {
           return new ErrorResponse("NOT_FOUND", ex.getMessage());
       }
   }
   ```

3. **Follow interface-based programming**
   - Define service interfaces
   - Implement them in concrete classes

4. **Use constructor injection over field injection**
   ```java
   @Service
   public class UserServiceImpl implements UserService {
       private final UserRepository userRepository;
       
       @Autowired // Optional in newer Spring versions
       public UserServiceImpl(UserRepository userRepository) {
           this.userRepository = userRepository;
       }
   }
   ```

5. **Externalize configuration**
   - Use different property files for different environments
   - Use `@ConfigurationProperties` for typed configuration

6. **Implement proper validation**
   ```java
   @RestController
   public class UserController {
       @PostMapping("/users")
       public ResponseEntity<User> createUser(@Valid @RequestBody UserDto userDto) {
           // Validation errors will be caught automatically
       }
   }
   
   public class UserDto {
       @NotBlank(message = "Name is required")
       private String name;
       
       @Email(message = "Email should be valid")
       private String email;
   }
   ```

7. **Use Spring Boot Actuator for monitoring**
   - Enable health, metrics, and info endpoints
   - Customize health indicators for business-critical components

8. **Implement proper logging**
   ```java
   @Service
   public class UserServiceImpl implements UserService {
       private static final Logger log = LoggerFactory.getLogger(UserServiceImpl.class);
       
       public User save(User user) {
           log.debug("Saving user: {}", user.getEmail());
           // Business logic
           log.info("User saved successfully: {}", user.getId());
           return savedUser;
       }
   }
   ```

9. **Write comprehensive tests**
   - Unit tests for services and utilities
   - Integration tests for repositories
   - API tests for controllers

10. **Implement proper security**
    - Use HTTPS in production
    - Implement proper authentication and authorization
    - Secure sensitive data and properties
    - Regularly update dependencies for security patches

### Performance Optimization
1. Use connection pooling (HikariCP is default)
2. Implement caching for frequently accessed data
3. Optimize JPA queries and use projections for large datasets
4. Use pagination for large result sets
5. Consider asynchronous processing for long-running tasks
6. Profile and monitor your application regularly

