# Spring Boot Interview Guide - Complete Master Reference

> **For 20+ Year Java Developers** | Enterprise-Grade Coverage | Production Scenarios

This comprehensive guide covers the most critical Spring Boot concepts that experienced developers must master for technical interviews and production systems.

---

## 📚 Table of Contents

1. [Spring Bean Lifecycle](#1-spring-bean-lifecycle)
2. [@PostConstruct & Use Cases](#2-postconstruct--use-cases)
3. [@SpringBootApplication Internal Working](#3-springbootapplication-internal-working)
4. [How Spring Decides Which Beans to Create](#4-how-spring-decides-which-beans-to-create)
5. [Dependency Injection Internal Working](#5-dependency-injection-internal-working)
6. [Constructor vs Field Injection](#6-constructor-vs-field-injection)
7. [Production Patterns & Best Practices](#7-production-patterns--best-practices)
8. [Common Interview Scenarios](#8-common-interview-scenarios)

---

## 1. Spring Bean Lifecycle

### Complete 11-Step Lifecycle (Enterprise View)

```
STARTUP PHASE
├─ STEP 1: Component Scanning
│  └─ Spring scans classpath for @Component, @Service, @Repository, @Controller, @Configuration
│  └─ BeanDefinition created (metadata only, no objects yet)
│  └─ Registered in BeanDefinitionRegistry
│
├─ STEP 2: Bean Instantiation (Constructor Called)
│  └─ Constructor parameters resolved (Constructor Injection)
│  └─ New object created in memory
│  └─ Field @Autowired NOT yet injected
│
├─ STEP 3: Dependency Injection (Properties Populated)
│  └─ All @Autowired fields injected
│  └─ Setter injection executed
│  └─ @Value properties resolved from configuration
│
├─ STEP 4: Aware Interfaces Called
│  └─ BeanNameAware.setBeanName()
│  └─ BeanClassLoaderAware.setBeanClassLoader()
│  └─ BeanFactoryAware.setBeanFactory()
│  └─ ApplicationContextAware.setApplicationContext()
│
├─ STEP 5: BeanPostProcessor.postProcessBeforeInitialization()
│  └─ Spring framework internals process bean
│  └─ @Autowired/@Value resolution happens HERE
│  └─ Custom annotations processed
│  └─ AOP infrastructure setup
│
├─ STEP 6: Initialization Callbacks
│  └─ Priority 1: @PostConstruct method
│  └─ Priority 2: InitializingBean.afterPropertiesSet()
│  └─ Priority 3: init-method from bean definition
│
├─ STEP 7: BeanPostProcessor.postProcessAfterInitialization()
│  └─ AOP PROXY CREATION happens HERE (most critical!)
│  └─ @Transactional proxy wrapping
│  └─ Custom bean decoration/wrapping
│  └─ What you inject is potentially a proxy, NOT original bean
│
├─ STEP 8: Bean Ready for Use
│  └─ Registered in singleton cache
│  └─ Available for dependency injection
│  └─ Available via context.getBean()
│
└─ STEP 9: Application Running
   └─ Bean used by application
   └─ Normal business logic execution

SHUTDOWN PHASE
├─ STEP 10: Destruction Callbacks
│  └─ Priority 1: @PreDestroy method
│  └─ Priority 2: DisposableBean.destroy()
│  └─ Priority 3: destroy-method from bean definition
│
└─ STEP 11: Bean Destroyed
   └─ Removed from singleton cache
   └─ Eligible for garbage collection
```

### Lifecycle Code Example - All 11 Steps Visible

```java
@Component
public class BeanLifecycleDemo {
    
    private static final Logger log = LoggerFactory.getLogger(BeanLifecycleDemo.class);
    
    // ========== STEP 2: Constructor ==========
    public BeanLifecycleDemo() {
        log.info("STEP 2: Constructor called");
        log.info("  - Object being created in memory");
        log.info("  - @Autowired fields still NULL");
    }
    
    // ========== STEP 3: Dependency Injection ==========
    @Autowired
    private UserRepository userRepo;  // Injected here, not in constructor
    
    @Autowired
    private CacheService cache;       // Injected here
    
    @Value("${app.name}")
    private String appName;           // Injected here
    
    // ========== STEP 4: Aware Interface ==========
    @Override
    public void setApplicationContext(ApplicationContext context) throws BeansException {
        log.info("STEP 4: ApplicationContextAware.setApplicationContext() called");
        this.context = context;
    }
    
    // ========== STEP 5 (internal): BeanPostProcessor.postProcessBeforeInitialization ==========
    // This happens automatically - Spring calls BeanPostProcessors here
    
    // ========== STEP 6: Initialization ==========
    @PostConstruct
    public void init() {
        log.info("STEP 6: @PostConstruct called");
        log.info("  - userRepo is ready: " + (userRepo != null));
        log.info("  - cache is ready: " + (cache != null));
        log.info("  - appName is: " + appName);
        
        // Load initial data
        List<User> users = userRepo.findAll();
        cache.put("all_users", users);
        log.info("  - Loaded " + users.size() + " users into cache");
    }
    
    // ========== STEP 7 (internal): BeanPostProcessor.postProcessAfterInitialization ==========
    // If this bean has @Transactional, proxy is created here
    // This bean itself becomes a proxy
    
    // ========== STEP 8: Ready for use ==========
    public void doWork() {
        log.info("STEP 9: Bean being used");
        // Application logic
    }
    
    // ========== STEP 10: Destruction ==========
    @PreDestroy
    public void cleanup() {
        log.info("STEP 10: @PreDestroy called");
        log.info("  - Cleaning up resources");
        log.info("  - Closing connections");
        log.info("  - Flushing cache");
        cache.clear();
    }
    
    // ========== STEP 11: Destroyed ==========
    // Bean removed from memory automatically
}
```

### Key Interview Points

**Q: When exactly is @PostConstruct called?**
```
ANSWER: After STEP 5 (Bean completely constructed and all dependencies injected).
For singletons: At application startup
For prototypes: Every time context.getBean() is called
For @Lazy: When first injected into another bean
```

**Q: What happens in STEP 7 that's critical?**
```
ANSWER: AOP PROXY CREATION!
- If bean has @Transactional, a proxy is created wrapping the bean
- What you inject is NOT the original class instance
- This is why:
  * Bean might implement interface but proxy is different type
  * instanceof checks might behave unexpectedly
  * Stack traces show proxy class names ($$EnhancerBySpringCGLIB$$)
  * Private methods are NOT intercepted by proxies
```

**Q: Is @PreDestroy guaranteed to run?**
```
ANSWER: Only for singleton beans when context.close() is called!

Scenarios:
✓ Runs: Singleton bean + context.close() called
✗ Never runs: Prototype bean (Spring doesn't track them)
✗ Never runs: Singleton bean + context.close() NEVER called
✓ Runs: Spring Boot default (context.close() called automatically on shutdown)
```

---

## 2. @PostConstruct & Use Cases

### What is @PostConstruct?

`@PostConstruct` is a lifecycle hook called after:
1. Bean instantiation (constructor)
2. All dependencies injected (@Autowired fields)
3. All Aware interfaces called
4. BeanPostProcessor.postProcessBeforeInitialization() executed

**It's the safe place to do initialization logic.**

### Enterprise Use Cases with Examples

#### Use Case 1: Load Cache on Startup

```java
@Component
public class UserCacheService {
    
    @Autowired
    private UserRepository userRepository;
    
    private Map<Long, User> userCache;
    private boolean cacheLoaded = false;
    
    @PostConstruct
    public void loadCache() {
        long startTime = System.currentTimeMillis();
        
        try {
            // Load all users into cache at startup
            List<User> allUsers = userRepository.findAll();
            userCache = new ConcurrentHashMap<>();
            
            for (User user : allUsers) {
                userCache.put(user.getId(), user);
            }
            
            cacheLoaded = true;
            long duration = System.currentTimeMillis() - startTime;
            
            log.info("Cache loaded successfully: {} users in {} ms", 
                     allUsers.size(), duration);
            
        } catch (Exception e) {
            log.error("Failed to load cache at startup", e);
            throw new IllegalStateException("Cache initialization failed", e);
            // Application startup fails - this is intentional!
            // Better to fail fast than run with broken cache
        }
    }
    
    public User getUser(Long userId) {
        if (!cacheLoaded) {
            throw new IllegalStateException("Cache not initialized");
        }
        return userCache.get(userId);
    }
}
```

**Why @PostConstruct here?**
- All dependencies ready ✓
- Application startup waits for this ✓
- Fails fast if cache loading fails ✓

#### Use Case 2: Initialize Connection Pools

```java
@Component
public class DatabaseConnectionPool {
    
    @Value("${db.pool.size:10}")
    private int poolSize;
    
    @Value("${db.pool.timeout:30000}")
    private long connectionTimeout;
    
    private HikariDataSource dataSource;
    
    @PostConstruct
    public void initializePool() {
        log.info("Initializing connection pool: size={}, timeout={}ms", 
                 poolSize, connectionTimeout);
        
        try {
            HikariConfig config = new HikariConfig();
            config.setJdbcUrl("jdbc:mysql://localhost:3306/mydb");
            config.setUsername("user");
            config.setPassword("pass");
            config.setMaximumPoolSize(poolSize);
            config.setConnectionTimeout(connectionTimeout);
            
            dataSource = new HikariDataSource(config);
            
            // Test connection
            try (Connection conn = dataSource.getConnection()) {
                DatabaseMetaData metadata = conn.getMetaData();
                log.info("Connection pool initialized. Database: {}", 
                         metadata.getDatabaseProductName());
            }
            
        } catch (Exception e) {
            log.error("Failed to initialize connection pool", e);
            throw new IllegalStateException("Database pool initialization failed", e);
        }
    }
    
    @PreDestroy
    public void shutdownPool() {
        if (dataSource != null) {
            dataSource.close();
            log.info("Connection pool closed");
        }
    }
}
```

#### Use Case 3: Start Async Scheduled Tasks

```java
@Component
public class ScheduledTaskInitializer {
    
    @Autowired
    private TaskService taskService;
    
    @Value("${task.scheduler.enabled:true}")
    private boolean schedulerEnabled;
    
    private ScheduledExecutorService executor;
    
    @PostConstruct
    public void initializeScheduler() {
        if (!schedulerEnabled) {
            log.info("Task scheduler disabled");
            return;
        }
        
        executor = Executors.newScheduledThreadPool(5, r -> {
            Thread t = new Thread(r);
            t.setName("TaskScheduler-" + t.getId());
            t.setDaemon(false);  // Not daemon - should complete before shutdown
            return t;
        });
        
        // Schedule recurring tasks
        executor.scheduleAtFixedRate(
            this::processPendingTasks,
            0,           // initial delay
            60,          // period
            TimeUnit.SECONDS
        );
        
        log.info("Scheduled task executor initialized with 5 threads");
    }
    
    private void processPendingTasks() {
        try {
            taskService.processPending();
        } catch (Exception e) {
            log.error("Error processing tasks", e);
        }
    }
    
    @PreDestroy
    public void shutdownScheduler() {
        if (executor != null) {
            executor.shutdown();
            try {
                if (!executor.awaitTermination(30, TimeUnit.SECONDS)) {
                    executor.shutdownNow();
                    log.warn("Force shutdown scheduler after timeout");
                }
            } catch (InterruptedException e) {
                executor.shutdownNow();
                Thread.currentThread().interrupt();
            }
            log.info("Scheduled task executor shutdown");
        }
    }
}
```

#### Use Case 4: Validate Configuration at Startup

```java
@Component
public class ConfigurationValidator {
    
    @Value("${api.key:}")
    private String apiKey;
    
    @Value("${api.endpoint:}")
    private String apiEndpoint;
    
    @Value("${app.environment}")
    private String environment;
    
    @PostConstruct
    public void validateConfiguration() {
        log.info("Validating configuration for environment: {}", environment);
        
        List<String> errors = new ArrayList<>();
        
        if (StringUtils.isBlank(apiKey)) {
            errors.add("api.key is required");
        }
        
        if (StringUtils.isBlank(apiEndpoint)) {
            errors.add("api.endpoint is required");
        }
        
        if (!isValidEndpoint(apiEndpoint)) {
            errors.add("api.endpoint is not a valid URL");
        }
        
        if ("production".equals(environment) && !isSslEnabled()) {
            errors.add("SSL must be enabled in production");
        }
        
        if (!errors.isEmpty()) {
            log.error("Configuration validation failed:");
            errors.forEach(error -> log.error("  - {}", error));
            throw new IllegalStateException("Invalid configuration: " + errors);
        }
        
        log.info("Configuration validation passed");
    }
    
    private boolean isValidEndpoint(String endpoint) {
        try {
            new URI(endpoint);
            return endpoint.startsWith("https://") || endpoint.startsWith("http://");
        } catch (Exception e) {
            return false;
        }
    }
    
    private boolean isSslEnabled() {
        // Check if SSL is configured
        return true;
    }
}
```

#### Use Case 5: Initialize Kafka Producer/Consumer

```java
@Component
public class KafkaInitializer {
    
    @Value("${kafka.bootstrap.servers}")
    private String bootstrapServers;
    
    @Value("${kafka.consumer.group}")
    private String consumerGroup;
    
    private KafkaConsumer<String, String> consumer;
    private KafkaProducer<String, String> producer;
    
    @PostConstruct
    public void initializeKafka() {
        log.info("Initializing Kafka: servers={}", bootstrapServers);
        
        try {
            // Initialize Producer
            Properties producerProps = new Properties();
            producerProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
            producerProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
            producerProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
            
            producer = new KafkaProducer<>(producerProps);
            log.info("Kafka producer initialized");
            
            // Initialize Consumer
            Properties consumerProps = new Properties();
            consumerProps.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
            consumerProps.put(ConsumerConfig.GROUP_ID_CONFIG, consumerGroup);
            consumerProps.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
            consumerProps.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
            
            consumer = new KafkaConsumer<>(consumerProps);
            consumer.subscribe(Arrays.asList("my-topic"));
            log.info("Kafka consumer initialized for group: {}", consumerGroup);
            
        } catch (Exception e) {
            log.error("Failed to initialize Kafka", e);
            throw new IllegalStateException("Kafka initialization failed", e);
        }
    }
    
    @PreDestroy
    public void shutdownKafka() {
        if (consumer != null) {
            consumer.close();
            log.info("Kafka consumer closed");
        }
        if (producer != null) {
            producer.close();
            log.info("Kafka producer closed");
        }
    }
}
```

### @PostConstruct Best Practices for 20+ Year Developers

```java
@Component
public class BestPractices {
    
    private static final Logger log = LoggerFactory.getLogger(BestPractices.class);
    
    @PostConstruct
    public void demonstrateBestPractices() {
        
        // ✓ GOOD: Log initialization start
        log.info("Initializing critical resources");
        
        // ✓ GOOD: Measure timing for production monitoring
        long startTime = System.currentTimeMillis();
        
        try {
            // ✓ GOOD: Do initialization work here
            initializeCriticalResources();
            
            // ✓ GOOD: Validate state after initialization
            validateInitialization();
            
            // ✓ GOOD: Log success with timing
            long duration = System.currentTimeMillis() - startTime;
            log.info("Initialization completed in {} ms", duration);
            
        } catch (Exception e) {
            // ✓ GOOD: Fail fast with descriptive error
            log.error("Initialization failed - application will not start", e);
            throw new IllegalStateException("Failed to initialize", e);
        }
    }
    
    // ✗ BAD EXAMPLES (what to avoid):
    
    @PostConstruct
    public void badExample1() {
        // ✗ BAD: Doing heavy I/O synchronously (blocks startup)
        loadMillionsOfRecordsFromDatabase();  // Takes 2 minutes!
        
        // BETTER: Use @Lazy or load asynchronously
    }
    
    @PostConstruct
    public void badExample2() {
        // ✗ BAD: Catching all exceptions and ignoring
        try {
            criticalInitialization();
        } catch (Exception e) {
            e.printStackTrace();  // Silently fails!
        }
        
        // BETTER: Let exception propagate or handle explicitly
    }
    
    @PostConstruct
    public void badExample3() {
        // ✗ BAD: Creating threads without proper naming/management
        new Thread(() -> doSomething()).start();  // Anonymous thread, hard to debug
        
        // BETTER: Use ExecutorService with named thread pools
    }
    
    @PostConstruct
    public void badExample4() {
        // ✗ BAD: Assuming dependencies are available
        if (someService == null) {  // Should never happen if @Autowired
            // Defensive programming in @PostConstruct = design smell
        }
        
        // BETTER: Constructor injection ensures non-null
    }
}
```

---

## 3. @SpringBootApplication Internal Working

### @SpringBootApplication Composition

```java
@SpringBootApplication
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}

// @SpringBootApplication is a composite annotation consisting of:
// @Configuration
// @ComponentScan
// @EnableAutoConfiguration
```

### What @SpringBootApplication Does Internally (Step-by-Step)

```
1. @Configuration
   └─ Marks this class as a bean configuration source
   └─ Can define @Bean methods
   └─ Spring treats it like XML <beans> config
   └─ All @Bean methods executed to create beans

2. @ComponentScan
   └─ Scans classpath starting from the package containing @SpringBootApplication
   └─ Finds all @Component, @Service, @Repository, @Controller classes
   └─ Recursively scans subpackages
   └─ Creates BeanDefinitions for each
   └─ No scan of packages above the main class package unless explicitly configured

3. @EnableAutoConfiguration
   └─ The MAGIC annotation!
   └─ Enables Spring Boot's auto-configuration mechanism
   └─ Scans all JARs on classpath for META-INF/spring.factories
   └─ Reads configuration classes (e.g., DataSourceAutoConfiguration)
   └─ Each auto-configuration applies if:
      - @ConditionalOnClass - required class exists on classpath
      - @ConditionalOnMissingBean - bean not already defined
      - @ConditionalOnProperty - property exists in application.yml/properties
      - Other conditional annotations
   └─ Auto-configurations applied in specific order
```

### Code Deep-Dive: What Happens When You Run SpringApplication.run()

```java
public class SpringApplicationDemo {
    
    public static void main(String[] args) {
        // Step 1: SpringApplication instance created
        SpringApplication app = new SpringApplication(MyApplication.class);
        
        // Step 2: ApplicationContext created (usually AnnotationConfigApplicationContext)
        // Step 3: Bean factory initialized
        // Step 4: Component scanning happens
        // Step 5: Auto-configuration applied
        // Step 6: All beans instantiated and initialized
        // Step 7: ApplicationContext started
        // Step 8: Run methods executed
        
        ConfigurableApplicationContext context = app.run(args);
        
        // Now you can access beans:
        UserService userService = context.getBean(UserService.class);
    }
}
```

### Internal Execution Timeline

```
SpringApplication.run()
│
├─ Create ApplicationContext
│  └─ Based on environment (servlet, reactive, etc.)
│  └─ Usually: AnnotationConfigServletWebServerApplicationContext
│
├─ Prepare ApplicationContext
│  └─ Set active profiles
│  └─ Register property sources
│  └─ Register beans
│
├─ Apply AutoConfiguration
│  └─ Read spring.factories from all JARs
│  └─ Create list of auto-configuration classes
│  └─ Sort by @Order/@Priority
│  └─ Apply each auto-configuration:
│     ├─ Check @ConditionalOnClass (classpath scan)
│     ├─ Check @ConditionalOnMissingBean (registry scan)
│     ├─ Check @ConditionalOnProperty (config scan)
│     ├─ If all conditions met: Register beans
│     └─ Continue to next auto-configuration
│
├─ Refresh ApplicationContext (context.refresh())
│  └─ Invoke all BeanFactoryPostProcessors
│  └─ Register BeanPostProcessors
│  ├─ Instantiate all singleton beans
│  │  └─ Run through all 11 lifecycle steps for each bean
│  │  └─ STEP 2: Call constructors
│  │  └─ STEP 3: Inject dependencies
│  │  └─ STEP 6: Call @PostConstruct
│  │  └─ STEP 7: Create AOP proxies
│  ├─ Finalize bean factory
│  └─ Publish context refresh event
│
├─ Start Web Server (if Web application)
│  └─ Embedded Tomcat/Jetty/Undertow
│  └─ Listen on port 8080 (default)
│  └─ Application ready to handle requests
│
└─ Call ApplicationRunner / CommandLineRunner beans
   └─ If you have @Component class implementing ApplicationRunner
   └─ Execute custom startup logic

Application is now READY!
```

### Practical Example: Tracing All Steps

```java
@SpringBootApplication
@Slf4j
public class MyApplication {
    
    public static void main(String[] args) {
        log.info("=== APPLICATION STARTUP BEGINS ===");
        
        SpringApplication app = new SpringApplication(MyApplication.class);
        
        // Customize application before run
        app.setAdditionalProfiles("custom-profile");
        app.setLazyInitialization(false);  // Eager bean initialization
        
        ConfigurableApplicationContext context = app.run(args);
        
        log.info("=== APPLICATION STARTUP COMPLETE ===");
        log.info("Active profiles: {}", Arrays.toString(context.getEnvironment().getActiveProfiles()));
        log.info("Total beans: {}", context.getBeanDefinitionCount());
    }
}

@Configuration
@Slf4j
public class ApplicationStartupTracing {
    
    @Bean
    public ApplicationRunner applicationRunner() {
        return args -> {
            log.info("ApplicationRunner executed - all beans initialized");
        };
    }
}

@Component
@Slf4j
public class StartupComponent {
    
    @PostConstruct
    public void init() {
        log.info("StartupComponent initialized");
    }
}
```

### Auto-Configuration: Real Example

```java
// This is from spring-boot-starter-data-jpa:
// Inside spring-boot-autoconfigure-X.jar → META-INF/spring.factories

@Configuration
@ConditionalOnClass({Entity.class, EntityManager.class})  // Check if JPA is on classpath
@ConditionalOnBean(DataSource.class)  // Check if DataSource bean exists
@AutoConfigureAfter(DataSourceAutoConfiguration.class)  // Order matters!
public class HibernateJpaAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean  // Only if user didn't define JpaTransactionManager
    public JpaTransactionManager transactionManager(EntityManagerFactory emf) {
        return new JpaTransactionManager(emf);
    }
    
    // More bean definitions...
}

// In your application.yml:
spring:
  jpa:
    hibernate:
      ddl-auto: update
      show-sql: true

// Spring Boot auto-detects this property and configures Hibernate accordingly!
```

### Interview Questions on @SpringBootApplication

**Q: What's the difference between @SpringBootApplication and @Configuration + @ComponentScan?**
```
ANSWER: @SpringBootApplication = @Configuration + @ComponentScan + @EnableAutoConfiguration

Without @EnableAutoConfiguration:
- You manually configure everything
- Only scans @Component in your packages
- No auto-configuration from spring-boot-starter JARs

With @EnableAutoConfiguration:
- Auto-detects JARs on classpath
- Auto-configures beans based on presence
- Magical but can hide issues
```

**Q: If I exclude an auto-configuration, what happens?**
```
ANSWER: That auto-configuration class is not applied, so its beans not created.

@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)
public class App { }

Effects:
- No default DataSource created
- You must provide your own @Bean DataSource
- Or rely on external configuration
```

**Q: Why does @ComponentScan start from @SpringBootApplication package?**
```
ANSWER: By design. This encourages proper package structure.

Good practice:
com.company
├─ com.company.Application  (@SpringBootApplication here)
├─ com.company.service
├─ com.company.repository
├─ com.company.config

All subpackages automatically scanned.

If you put @SpringBootApplication in root (com.company), performance impact!
Scans entire classpath including third-party JARs unnecessarily.
```

---

## 4. How Spring Decides Which Beans to Create

### Bean Resolution Algorithm (Internal)

```
For each @Component found during component scanning:
│
├─ Check @Conditional annotations
│  ├─ @ConditionalOnClass
│  │  └─ Class on classpath? YES → continue, NO → skip
│  ├─ @ConditionalOnMissingClass
│  │  └─ Class on classpath? YES → skip, NO → continue
│  ├─ @ConditionalOnBean
│  │  └─ Bean exists in context? YES → continue, NO → skip
│  ├─ @ConditionalOnMissingBean
│  │  └─ Bean exists in context? YES → skip, NO → continue
│  ├─ @ConditionalOnProperty
│  │  └─ Property exists? YES → continue, NO → skip
│  ├─ @ConditionalOnExpression
│  │  └─ SpEL expression true? YES → continue, NO → skip
│  └─ @ConditionalOnWebApplication
│     └─ Web environment? YES → continue, NO → skip
│
├─ If ALL conditions passed:
│  └─ BeanDefinition created
│  └─ Registered in BeanDefinitionRegistry
│
└─ If ANY condition failed:
   └─ Bean is NOT created
   └─ As if @Component never existed
```

### Practical Examples

#### Example 1: Only Create if Class Exists

```java
@Component
@ConditionalOnClass(name = "org.springframework.data.redis.core.RedisTemplate")
public class RedisConfiguration {
    
    @Bean
    public RedisTemplate<String, String> redisTemplate() {
        // Only created if Redis library on classpath
        return new RedisTemplate<>();
    }
}

// If redis-spring-boot-starter in pom.xml: Bean created ✓
// If redis library NOT in pom.xml: Bean skipped ✓
```

#### Example 2: Only Create if Another Bean Doesn't Exist

```java
// In spring-boot-autoconfigure
@Configuration
public class DataSourceAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    public DataSource dataSource() {
        // Only if user didn't define their own DataSource
        return createDefaultDataSource();
    }
}

// In your application:

// Case 1: No custom DataSource
@SpringBootApplication
public class App { }
// Default DataSource is created ✓

// Case 2: Custom DataSource
@Configuration
public class CustomConfig {
    @Bean
    public DataSource dataSource() {
        return customDataSource();
    }
}
@SpringBootApplication
public class App { }
// Custom DataSource used, default skipped ✓
```

#### Example 3: Only Create if Property Exists

```java
@Component
@ConditionalOnProperty(
    name = "feature.advanced-cache.enabled",
    havingValue = "true",
    matchIfMissing = false  // If property missing: don't create
)
public class AdvancedCacheService {
    
    @PostConstruct
    public void init() {
        log.info("Advanced cache service enabled");
    }
}

// application.yml:
feature:
  advanced-cache:
    enabled: true
// AdvancedCacheService created ✓

// If property missing or false:
// AdvancedCacheService NOT created ✓
```

#### Example 4: Complex Conditions

```java
@Configuration
public class NotificationServiceConfig {
    
    @Bean
    @ConditionalOnProperty(name = "notifications.email.enabled")
    @ConditionalOnClass(name = "javax.mail.Session")  // Email library needed
    public EmailNotificationService emailService() {
        return new EmailNotificationService();
    }
    
    @Bean
    @ConditionalOnProperty(name = "notifications.slack.enabled")
    @ConditionalOnClass(name = "com.slack.api.Slack")  // Slack library needed
    public SlackNotificationService slackService() {
        return new SlackNotificationService();
    }
    
    @Bean
    @ConditionalOnMissingBean  // Create default if neither email nor slack
    public NoOpNotificationService noOpService() {
        return new NoOpNotificationService();
    }
}

// Scenarios:
// 1. Email enabled + Slack enabled → Use email + slack
// 2. Email enabled + Slack disabled → Use email only
// 3. Email disabled + Slack enabled → Use slack only
// 4. Both disabled → Use no-op (does nothing)
```

### Spring Boot Auto-Configuration Order

```java
// spring.factories contains auto-configuration classes in order:

// First (dependencies):
org.springframework.boot.autoconfigure.data.DataSourceAutoConfiguration

// Middle (depends on DataSource):
org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration
org.springframework.boot.autoconfigure.jdbc.DataSourceTransactionManagerAutoConfiguration

// Last (high-level features):
org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration
org.springframework.boot.autoconfigure.security.servlet.SecurityAutoConfiguration
```

```java
// Use @AutoConfigureAfter to ensure order:

@Configuration
@AutoConfigureAfter(DataSourceAutoConfiguration.class)
public class MyJpaConfiguration {
    // Runs AFTER DataSource is configured
}

@Configuration
@AutoConfigureBefore(WebMvcAutoConfiguration.class)
public class MyWebConfiguration {
    // Runs BEFORE WebMvc is configured
}
```

### How to Debug What Beans Are Created

```java
public class BeanDebugger {
    
    public static void main(String[] args) {
        ConfigurableApplicationContext context = 
            SpringApplication.run(MyApplication.class, args);
        
        // List all beans
        String[] beanNames = context.getBeanDefinitionNames();
        System.out.println("Total beans: " + beanNames.length);
        
        for (String name : beanNames) {
            BeanDefinition def = context.getBeanFactory()
                .getBeanDefinition(name);
            System.out.println("  - " + name + 
                              " (" + def.getBeanClassName() + ")");
        }
        
        // Enable debug logging
        // application.yml:
        // logging:
        //   level:
        //     org.springframework.boot.autoconfigure: DEBUG
        //     org.springframework: DEBUG
    }
}

// Output:
// Total beans: 127
//  - org.springframework.boot.autoconfigure.internalConfigurationPropertiesBinder
//  - configurationPropertiesBindingPostProcessor
//  - userService
//  - ... (125 more)
```

---

## 5. Dependency Injection Internal Working

### Dependency Injection Mechanism (Complete View)

```
When Spring needs to inject a dependency:

1. RESOLUTION PHASE
   ├─ Identify dependency type (field type or method parameter type)
   ├─ Check beans registry for matching bean
   ├─ Matching rules:
   │  ├─ By type: Find bean implementing interface/extending class
   │  ├─ By name: If multiple types, match by field/parameter name
   │  ├─ By qualifier: Use @Qualifier("specificBean")
   │  └─ By primary: Use @Primary if multiple candidates
   └─ Result: Single bean selected or exception thrown

2. INSTANTIATION PHASE (if needed)
   └─ If bean doesn't exist yet, create it first
   └─ Recursively resolve its dependencies
   └─ Go through entire 11-step lifecycle

3. INJECTION PHASE
   ├─ Constructor injection: Pass bean to constructor parameter
   ├─ Field injection: Use reflection to set field
   └─ Setter injection: Call setter method with bean

4. POST-INJECTION PHASE
   └─ @PostConstruct called (if present)
   └─ Dependent bean continues initialization
```

### Type Resolution (Autowiring Algorithm)

```java
@Service
public class UserService {
    
    // FIELD INJECTION (Autowiring by type)
    @Autowired
    private UserRepository userRepository;
    // Spring resolves: Type = UserRepository
    // Looks for: Bean that is instance of UserRepository
    // Finds: UserRepositoryImpl (implements UserRepository)
    // Injects: userRepository = UserRepositoryImpl bean instance
}

@Repository
public class UserRepositoryImpl implements UserRepository {
    // This bean implements UserRepository
    // So it can be injected as UserRepository
}
```

### What Happens When Multiple Beans Match

```java
// Multiple beans of same type
@Configuration
public class BeansConfig {
    
    @Bean
    public DataSource primaryDataSource() {
        return createDataSource("primary-db");
    }
    
    @Bean
    public DataSource secondaryDataSource() {
        return createDataSource("secondary-db");
    }
}

// In a service:
@Service
public class ReportService {
    
    @Autowired
    private DataSource dataSource;
    // ERROR! Two DataSource beans found:
    // - primaryDataSource
    // - secondaryDataSource
    // Which one to inject?
}

// SOLUTIONS:

// Solution 1: Use @Primary (for one bean)
@Bean
@Primary
public DataSource primaryDataSource() {
    return createDataSource("primary-db");
}

// Now: @Autowired private DataSource dataSource;
// Uses: primaryDataSource ✓

// Solution 2: Use @Qualifier
@Service
public class ReportService {
    
    @Autowired
    @Qualifier("primaryDataSource")
    private DataSource dataSource;
    // Explicitly says: Use this specific bean
}

// Solution 3: Inject list of all matching beans
@Service
public class ReportService {
    
    @Autowired
    private List<DataSource> dataSources;
    // Gets: List with both primaryDataSource and secondaryDataSource
}
```

### Circular Dependency Handling

```java
// CASE 1: Constructor Injection with Circular Dependency (FAILS)
@Component
public class ServiceA {
    public ServiceA(ServiceB b) { }  // Needs ServiceB
}

@Component
public class ServiceB {
    public ServiceB(ServiceA a) { }  // Needs ServiceA
}

// Startup error:
// BeanCurrentlyInCreationException
// Cannot instantiate: ServiceA needs ServiceB, but ServiceB needs ServiceA

// CASE 2: Field Injection with Circular Dependency (WORKS)
@Component
public class ServiceA {
    @Autowired
    private ServiceB serviceB;  // Injected AFTER construction
}

@Component
public class ServiceB {
    @Autowired
    private ServiceA serviceA;  // Injected AFTER construction
}

// How Spring handles this:
// 1. Create ServiceA instance (constructor doesn't need ServiceB)
// 2. Try to inject ServiceB
// 3. ServiceB needs ServiceA, but Spring has partial instance
// 4. Spring injects the partial ServiceA into ServiceB
// 5. ServiceB initialization completes
// 6. Complete ServiceB instance injected into ServiceA
// 7. Both fully initialized

// CASE 3: Setter Injection with Circular Dependency (WORKS)
@Component
public class ServiceA {
    private ServiceB serviceB;
    
    @Autowired
    public void setServiceB(ServiceB serviceB) {
        this.serviceB = serviceB;
    }
}

@Component
public class ServiceB {
    private ServiceA serviceA;
    
    @Autowired
    public void setServiceA(ServiceA serviceA) {
        this.serviceA = serviceA;
    }
}

// Same approach as field injection
```

### Lazy vs Eager Dependency Resolution

```java
// EAGER (Default)
@Service
public class EagerService {
    
    @Autowired
    private ExpensiveService expensive;
    // Created at startup immediately
    // If ExpensiveService fails to initialize, entire app fails
}

// LAZY
@Service
public class LazyService {
    
    @Autowired
    @Lazy
    private ExpensiveService expensive;
    // Created when first accessed
    // App startup faster
    // Error appears when method using expensive service is called
}

// LAZY with Provider
@Service
public class ProviderService {
    
    @Autowired
    private ObjectProvider<ExpensiveService> expensive;
    // Can check if bean exists before accessing
}

public void process() {
    // Check if exists (null-safe)
    expensive.ifAvailable(service -> service.doWork());
    
    // Get with fallback
    ExpensiveService service = expensive
        .getIfAvailable(() -> new DefaultService());
}
```

### Dependency Injection Trace Example

```java
@Slf4j
@Component
public class DependencyInjectionTrace {
    
    // Step 1: Spring analyzes this class
    // Sees: @Autowired private UserRepository repository
    // Determines: Need to find UserRepository bean
    
    @Autowired
    private UserRepository repository;
    
    @Autowired
    private NotificationService notificationService;
    
    // Step 2: Spring finds UserRepositoryImpl
    // (implements UserRepository interface)
    
    // Step 3: UserRepositoryImpl has dependencies too
    // @Autowired private DataSource dataSource
    // Spring recursively resolves DataSource
    
    // Step 4: All transitive dependencies resolved
    
    // Step 5: Create instances in correct order:
    // 1. DataSource created
    // 2. UserRepositoryImpl created (with DataSource injected)
    // 3. NotificationService created
    // 4. DependencyInjectionTrace created (with repos injected)
    
    @PostConstruct
    public void init() {
        log.info("Step 6: @PostConstruct called");
        log.info("  - repository: " + (repository != null ? "injected" : "null"));
        log.info("  - notification: " + (notificationService != null ? "injected" : "null"));
        
        // All dependencies guaranteed available here!
    }
}

@Repository
@Slf4j
public class UserRepositoryImpl implements UserRepository {
    
    @Autowired
    private DataSource dataSource;
    // DataSource needed before this repo can be used
    
    @PostConstruct
    public void init() {
        log.info("UserRepositoryImpl initialized");
    }
}

@Component
@Slf4j
public class NotificationService {
    
    @Autowired
    @Lazy
    private UserRepository repository;
    // Lazy: Not created until first accessed
    
    public void notifyUser(User user) {
        log.info("Notifying user: " + user.getName());
    }
}
```

---

## 6. Constructor vs Field Injection

### Detailed Comparison Table

| Aspect | Constructor Injection | Field Injection (@Autowired) |
|--------|---|---|
| **When Resolved** | During bean instantiation (STEP 2) | After instantiation (STEP 3) |
| **Circular Dependency** | ✗ Fails with BeanCurrentlyInCreationException | ✓ Handled via lazy initialization |
| **Testability** | ✓✓ Excellent (easy to mock) | ✗ Requires reflection or setter method |
| **Immutability** | ✓ Can use final fields | ✗ Fields must be mutable |
| **Completeness** | ✓ Fails fast if dependency missing | ✗ Error appears at first use |
| **Null Safety** | ✓ Non-null if required=true | ✗ Can be null if injection fails silently |
| **Code Clarity** | ✓✓ Obvious what's needed | ✗ Dependencies hidden in class |
| **Performance** | ✓ No reflection needed | ✗ Uses reflection for injection |
| **Valid with @Bean** | ✓ Supported | ✓ Supported |
| **Spring Recommendation** | ✓✓✓ Preferred for most cases | ⚠ Use for optional dependencies |

### Constructor Injection Deep-Dive

```java
@Service
@Slf4j
public class ConstructorInjectionExample {
    
    private final UserRepository userRepository;
    private final EmailService emailService;
    private final CacheService cacheService;
    
    // Constructor: Dependencies injected as parameters
    // These fields are FINAL - immutable after initialization
    // This is the recommended approach
    public ConstructorInjectionExample(
            UserRepository userRepository,
            EmailService emailService,
            CacheService cacheService) {
        
        log.info("Constructor called with dependencies");
        
        // Spring passes already-created bean instances
        this.userRepository = userRepository;
        this.emailService = emailService;
        this.cacheService = cacheService;
        
        // At this point: @Autowired fields NOT yet injected
        // But constructor parameters ARE available
    }
    
    public void processUser(Long userId) {
        // All dependencies guaranteed non-null (unless Optional)
        User user = userRepository.findById(userId);
        emailService.send(user.getEmail());
        cacheService.update(user);
    }
}

// Testing is simple:
class ConstructorInjectionTest {
    
    @Test
    public void testProcessUser() {
        // Easy to mock dependencies
        UserRepository mockRepo = mock(UserRepository.class);
        EmailService mockEmail = mock(EmailService.class);
        CacheService mockCache = mock(CacheService.class);
        
        // Pass mocks to constructor
        ConstructorInjectionExample service = 
            new ConstructorInjectionExample(mockRepo, mockEmail, mockCache);
        
        // Test normally
        service.processUser(1L);
        
        verify(mockRepo).findById(1L);
        verify(mockEmail).send(any());
    }
}
```

### Field Injection Deep-Dive

```java
@Service
@Slf4j
public class FieldInjectionExample {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private EmailService emailService;
    
    @Autowired
    @Lazy
    private ExpensiveService expensiveService;
    
    @Autowired(required = false)
    private Optional<CacheService> cacheService;
    
    // No constructor needed
    // Dependencies injected by Spring AFTER construction
    
    @PostConstruct
    public void init() {
        log.info("After construction:");
        log.info("  - userRepository: " + (userRepository != null ? "injected" : "null"));
        log.info("  - emailService: " + (emailService != null ? "injected" : "null"));
        log.info("  - expensiveService: " + (expensiveService != null ? "injected" : "null"));
        log.info("  - cacheService: " + (cacheService != null ? "present" : "absent"));
    }
    
    public void processUser(Long userId) {
        User user = userRepository.findById(userId);
        emailService.send(user.getEmail());
        
        // Lazy: only created here if accessed
        if (expensiveService != null) {
            expensiveService.analyze(user);
        }
        
        // Optional: handle absence safely
        cacheService.ifPresent(cache -> cache.update(user));
    }
}

// Testing is harder (requires reflection or setter):
class FieldInjectionTest {
    
    private FieldInjectionExample service;
    
    @Before
    public void setup() throws Exception {
        service = new FieldInjectionExample();
        
        // Method 1: Reflection (hacky)
        Field repoField = FieldInjectionExample.class.getDeclaredField("userRepository");
        repoField.setAccessible(true);
        repoField.set(service, mock(UserRepository.class));
        
        // Method 2: Use ReflectionTestUtils (better)
        ReflectionTestUtils.setField(service, "emailService", mock(EmailService.class));
    }
    
    @Test
    public void testProcessUser() {
        service.processUser(1L);
        // Tests pass, but setup is verbose
    }
}
```

### Setter Injection

```java
@Service
@Slf4j
public class SetterInjectionExample {
    
    private UserRepository userRepository;
    private EmailService emailService;
    
    // Spring calls this setter to inject dependencies
    @Autowired
    public void setUserRepository(UserRepository repo) {
        this.userRepository = repo;
    }
    
    @Autowired
    public void setEmailService(EmailService service) {
        this.emailService = service;
    }
    
    // Combined setter (multiple dependencies)
    @Autowired
    public void setupDependencies(UserRepository repo, EmailService email) {
        this.userRepository = repo;
        this.emailService = email;
    }
    
    public void processUser(Long userId) {
        User user = userRepository.findById(userId);
        emailService.send(user.getEmail());
    }
}

// Advantages:
// ✓ Easy to test (just call setters)
// ✓ Handles circular dependencies
// ✓ Can make fields mutable

// Disadvantages:
// ✗ Not obvious which setters are required
// ✗ Object can be in incomplete state
// ✗ Less common (older Spring style)

// Testing:
class SetterInjectionTest {
    
    @Test
    public void testProcessUser() {
        SetterInjectionExample service = new SetterInjectionExample();
        
        // Just call setters
        service.setUserRepository(mock(UserRepository.class));
        service.setEmailService(mock(EmailService.class));
        
        service.processUser(1L);
    }
}
```

### Which to Use? (Decision Tree)

```
Is dependency ALWAYS required?
├─ YES → Use Constructor Injection
│  └─ Force completion of initialization
│  └─ Fail fast if dependency missing
│  └─ Make fields final (immutable)
│  └─ Easy testing
│  └─ Clear code contract
│
└─ NO → Is it optional?
   ├─ YES → Use Field Injection with @Lazy or Optional
   │  └─ Mark @Autowired(required = false)
   │  └─ Or use ObjectProvider
   │  └─ Check if present before use
   │
   └─ SOMETIMES → Use Setter Injection
      └─ Rare case: Need circular dependencies
      └─ Or reconfigurability after initialization
      └─ (Usually bad design - reconsider!)
```

### Production Best Practices

```java
@Service
@Slf4j
public class BestPracticeExample {
    
    // ✓ BEST: Required dependencies via constructor
    private final UserRepository userRepository;
    private final EmailService emailService;
    private final SecurityContext securityContext;
    
    // ✓ GOOD: Optional dependencies via @Lazy or ObjectProvider
    private final ObjectProvider<CacheService> cacheProvider;
    
    public BestPracticeExample(
            UserRepository userRepository,
            EmailService emailService,
            SecurityContext securityContext,
            ObjectProvider<CacheService> cacheProvider) {
        
        this.userRepository = userRepository;
        this.emailService = emailService;
        this.securityContext = securityContext;
        this.cacheProvider = cacheProvider;
    }
    
    public void processUser(Long userId) {
        // All required dependencies available
        User user = userRepository.findById(userId);
        
        // Optional: use if available
        cacheProvider.ifAvailable(cache -> cache.invalidate(userId));
        
        emailService.send(user.getEmail());
    }
    
    // ✗ AVOID: Field injection (hidden dependencies)
    // @Autowired
    // private UserRepository repo;
    
    // ✗ AVOID: Setter injection (design smell)
    // @Autowired
    // public void setRepository(UserRepository repo) { }
}
```

---

## 7. Production Patterns & Best Practices

### Pattern 1: Graceful Initialization Failure Handling

```java
@Component
@Slf4j
public class CriticalResourceInitializer {
    
    private volatile boolean initialized = false;
    private volatile Exception initializationError;
    
    @PostConstruct
    public void init() {
        try {
            initializeCriticalResources();
            initialized = true;
            log.info("Critical resources initialized successfully");
        } catch (Exception e) {
            initializationError = e;
            log.error("Failed to initialize critical resources", e);
            // Don't rethrow - allow application to start
            // But mark as failed
        }
    }
    
    public void use() {
        if (!initialized) {
            throw new IllegalStateException(
                "Service not initialized: " + initializationError.getMessage(),
                initializationError);
        }
        // Safe to use
    }
    
    private void initializeCriticalResources() {
        // Initialization logic
    }
}
```

### Pattern 2: Feature Flags with Beans

```java
@Component
public class FeatureFlagService {
    
    @Value("${feature.advanced-analytics.enabled:false}")
    private boolean advancedAnalyticsEnabled;
    
    public boolean isEnabled(String feature) {
        // Check dynamically at runtime
        return true;
    }
}

@Component
@ConditionalOnProperty(
    name = "feature.advanced-analytics.enabled",
    havingValue = "true"
)
public class AdvancedAnalyticsService {
    // Only created if feature enabled in config
}

// In application:
@Component
@Slf4j
public class AnalyticsConsumer {
    
    @Autowired
    private Optional<AdvancedAnalyticsService> advancedAnalytics;
    
    public void analyze(Data data) {
        if (advancedAnalytics.isPresent()) {
            advancedAnalytics.get().analyze(data);
        } else {
            log.info("Advanced analytics not enabled");
        }
    }
}
```

### Pattern 3: Bean Initialization Ordering

```java
@Component
@Order(1)
@Slf4j
public class DatabaseInitializer {
    
    @PostConstruct
    public void init() {
        log.info("1. Initializing database");
    }
}

@Component
@Order(2)
@Slf4j
public class CacheInitializer {
    
    @PostConstruct
    public void init() {
        log.info("2. Initializing cache (depends on DB)");
    }
}

@Component
@Order(3)
@Slf4j
public class SchedulerInitializer {
    
    @PostConstruct
    public void init() {
        log.info("3. Starting scheduler (depends on cache)");
    }
}

// Order 1 → 2 → 3
```

### Pattern 4: Health Checks

```java
@Component
public class ApplicationHealthIndicator implements HealthIndicator {
    
    @Autowired
    private DatabaseService dbService;
    
    @Autowired
    private CacheService cacheService;
    
    @Override
    public Health health() {
        try {
            boolean dbOk = dbService.isHealthy();
            boolean cacheOk = cacheService.isHealthy();
            
            if (dbOk && cacheOk) {
                return Health.up()
                    .withDetail("database", "UP")
                    .withDetail("cache", "UP")
                    .build();
            } else {
                return Health.down()
                    .withDetail("database", dbOk ? "UP" : "DOWN")
                    .withDetail("cache", cacheOk ? "UP" : "DOWN")
                    .build();
            }
        } catch (Exception e) {
            return Health.down()
                .withException(e)
                .build();
        }
    }
}

// Endpoint: GET /actuator/health
// Returns health status of application
```

---

## 8. Common Interview Scenarios

### Scenario 1: What Happens When Bean Creation Fails?

```
@Component
public class BrokenBean {
    @PostConstruct
    public void init() {
        throw new RuntimeException("Init failed!");
    }
}

RESULT:
1. Spring attempts to create BrokenBean
2. Constructor succeeds
3. Dependencies injected
4. @PostConstruct called
5. Exception thrown from init()
6. BrokenBean NOT added to context
7. Other beans depending on BrokenBean also fail
8. ApplicationContext startup fails
9. Application doesn't start
10. Exception thrown to main()

KEY: Fail fast is GOOD for required initialization!
```

### Scenario 2: How to Share State Between Singleton Beans?

```
Singleton beans are:
✓ Created once at startup
✓ Shared across entire application
✓ Same instance injected everywhere
✓ Thread-safe design essential

@Service
public class SharedStateService {
    private final Map<String, Object> state = new ConcurrentHashMap<>();
    
    public void setState(String key, Object value) {
        state.put(key, value);  // Visible to all threads
    }
    
    public Object getState(String key) {
        return state.get(key);  // Get shared state
    }
}

All controllers/services get SAME instance:
@Controller
public class UserController {
    @Autowired
    private SharedStateService service;
}

@Service
public class AdminService {
    @Autowired
    private SharedStateService service;  // SAME instance
}

Caution: Thread-safe code required!
```

### Scenario 3: Prototype Beans - When Do They Get Created?

```
@Component
@Scope("prototype")
public class PrototypeBean {
    @PostConstruct
    public void init() {
        System.out.println("Prototype bean created");
    }
}

// Scenario 1: Injected into singleton
@Service
public class SingletonService {
    @Autowired
    private PrototypeBean proto;  // NEW instance every time singleton uses it
}

// Result:
// - Same SingletonService (singleton)
// - Different PrototypeBean (new each use)

// Scenario 2: Manual getBean()
ConfigurableApplicationContext context = ...;
PrototypeBean p1 = context.getBean(PrototypeBean.class);  // New instance
PrototypeBean p2 = context.getBean(PrototypeBean.class);  // Different new instance
p1 != p2  // TRUE

// Scenario 3: @Lazy prototype
@Service
public class LazyService {
    @Autowired
    @Lazy
    private PrototypeBean proto;
}

// Result:
// - proto created when first accessed
// - Each access might create new instance (depends on how used)
```

### Scenario 4: AOP Proxy Issue

```
@Service
@Transactional  // Creates proxy in STEP 7
public class UserService implements IUserService {
    
    private final UserRepository repo;
    
    public UserService(UserRepository repo) {
        this.repo = repo;
    }
    
    public void saveUser(User user) {
        // Proxy wraps this method with transaction logic
    }
}

// Problem 1: instanceof fails
@Controller
public class UserController {
    @Autowired
    private UserService service;  // Actually a proxy!
    
    public void test() {
        if (service instanceof UserService) {  // TRUE (proxy implements interface)
            // But you can't cast to concrete UserService
        }
    }
}

// Problem 2: Private methods not intercepted
@Service
@Transactional
public class PaymentService {
    
    public void processPayment(Payment p) {
        validatePayment(p);  // private - NO transaction wrapping!
    }
    
    private void validatePayment(Payment p) {
        // This is NOT wrapped by @Transactional proxy
        // Runs without transaction context
    }
}

// Solution: Use public methods or AspectJ weaving
```

### Scenario 5: Circular Dependency Resolution

```
@Component
public class ServiceA {
    @Autowired
    private ServiceB serviceB;  // Field injection handles circularity
}

@Component
public class ServiceB {
    @Autowired
    private ServiceA serviceA;
}

// Spring's handling:
// 1. Start creating ServiceA (instantiation succeeds)
// 2. Need to inject ServiceB
// 3. Start creating ServiceB
// 4. ServiceB needs ServiceA
// 5. Spring has partially initialized ServiceA - reuse it
// 6. ServiceB initialization completes
// 7. Complete ServiceB injected into ServiceA
// 8. ServiceA initialization completes

// Both beans initialized with circular dependency!
```

---

## 📋 Interview Checklist

### Must Know (Entry Level)
- [ ] 11-step bean lifecycle
- [ ] @PostConstruct and @PreDestroy purpose
- [ ] Constructor vs field injection
- [ ] Singleton vs prototype scope

### Should Know (Mid Level)
- [ ] How @SpringBootApplication works
- [ ] Auto-configuration mechanism
- [ ] @ConditionalOnX annotations
- [ ] Circular dependency handling
- [ ] AOP proxy creation

### Must Know (Senior Level - 20+ years)
- [ ] ObjectProvider patterns
- [ ] BeanPostProcessor chain
- [ ] Graceful shutdown configuration
- [ ] Bean initialization ordering
- [ ] Production deployment patterns
- [ ] Testing with @DirtiesContext costs
- [ ] AspectJ weaving vs runtime proxies

---

## 🔗 References

- [Spring Bean Lifecycle Documentation](https://docs.spring.io/spring-framework/reference/core/beans/factory-extension.html)
- [Spring Boot Auto-Configuration](https://spring.io/projects/spring-boot)
- [Spring Conditional Annotations](https://docs.spring.io/spring-boot/reference/using/conditional-on-annotations.html)
- [Spring Dependency Injection](https://docs.spring.io/spring-framework/reference/core/beans/dependencies.html)

---

**Last Updated:** July 2026
**Target Audience:** 20+ Year Java Developers
**Difficulty:** Advanced / Enterprise

