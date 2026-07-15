# Spring Bean Lifecycle - Complete Guide for Beginners & Interviews

A comprehensive, beginner-friendly guide covering every aspect of Spring Bean lifecycle with real-world examples and interview Q&A.

---

## 🎯 What is Spring Bean Lifecycle?

**Definition:** The Spring Bean Lifecycle is the sequence of stages a Spring bean goes through from the moment it's created until it's destroyed.

Think of it like a **person's life journey**:
- **Birth** = Bean Creation
- **Growing Up** = Dependency Injection & Initialization
- **Living Life** = Bean Ready for Use
- **Death** = Bean Destruction

---

## 📊 Visual Flow Diagram

```
┌──────────────────────────────────────────────────────────────┐
│           SPRING APPLICATION STARTS                          │
└──────────────────────────────────────────────────────────────┘
                            ↓
      ┌────────────────────────────────────────────────┐
      │  STEP 1: COMPONENT SCANNING                    │
      │  Spring scans for @Component, @Service, etc.   │
      │  Creates BeanDefinition (blueprint only)       │
      └────────────────────────────────────────────────┘
                            ↓
      ┌────────────────────────────────────────────────┐
      │  STEP 2: BEAN INSTANTIATION                    │
      │  Constructor is called                         │
      │  Object created in memory                      │
      │  Dependencies NOT yet injected                 │
      └────────────────────────────────────────────────┘
                            ↓
      ┌────────────────────────────────────────────────┐
      │  STEP 3: DEPENDENCY INJECTION                  │
      │  @Autowired fields are injected                │
      │  Setter injection happens                      │
      │  All dependencies now available                │
      └────────────────────────────────────────────────┘
                            ↓
      ┌────────────────────────────────────────────────┐
      │  STEP 4: AWARE INTERFACES (Optional)           │
      │  BeanNameAware.setBeanName()                   │
      │  ApplicationContextAware.setApplicationContext()
      └────────────────────────────────────────────────┘
                            ↓
      ┌────────────────────────────────────────────────┐
      │  STEP 5: PRE-INITIALIZATION PROCESSING         │
      │  BeanPostProcessor.postProcessBefore...()      │
      │  Spring magic happens here!                    │
      └────────────────────────────────────────────────┘
                            ↓
      ┌────────────────────────────────────────────────┐
      │  STEP 6: @PostConstruct / init-method          │
      │  Your initialization code runs here            │
      │  Perfect for setup tasks                       │
      └────────────────────────────────────────────────┘
                            ↓
      ┌────────────────────────────────────────────────┐
      │  STEP 7: POST-INITIALIZATION PROCESSING        │
      │  BeanPostProcessor.postProcessAfter...()       │
      │  AOP Proxy creation happens HERE               │
      └────────────────────────────────────────────────┘
                            ↓
      ┌────────────────────────────────────────────────┐
      │  STEP 8: BEAN READY FOR USE ✓                  │
      │  Registered in singleton cache                 │
      │  Can be injected into other beans              │
      │  Available via context.getBean()               │
      └────────────────────────────────────────────────┘
                            ↓
      ┌────────────────────────────────────────────────┐
      │  STEP 9: APPLICATION RUNNING                   │
      │  Your code uses the bean                       │
      └────────────────────────────────────────────────┘
                            ↓
┌──────────────────────────────────────────────────────────────┐
│           APPLICATION SHUTDOWN TRIGGERED                     │
└──────────────────────────────────────────────────────────────┘
                            ↓
      ┌────────────────────────────────────────────────┐
      │  STEP 10: @PreDestroy / destroy-method         │
      │  Cleanup code runs                             │
      │  Close connections, threads, etc.              │
      └────────────────────────────────────────────────┘
                            ↓
      ┌────────────────────────────────────────────────┐
      │  STEP 11: BEAN DESTROYED                       │
      │  Removed from memory                           │
      │  Eligible for garbage collection               │
      └────────────────────────────────────────────────┘
```

---

## 📝 Step-by-Step Explanation for Beginners

### STEP 1: Component Scanning

**What Happens:**
- Spring scans your classpath for classes annotated with `@Component`, `@Service`, `@Repository`, `@Controller`, `@Configuration`
- Spring doesn't create objects yet—just collects information about what beans exist
- This information is stored in something called `BeanDefinition` (think of it as a blueprint)

**Example:**
```java
@Component  // Spring finds this!
public class UserService {
    // Blueprint created, but object not created yet
}
```

**Interview Tip:** "Component scanning creates metadata about beans, not the actual beans themselves."

---

### STEP 2: Bean Instantiation (Constructor Called)

**What Happens:**
- Spring calls the constructor of the bean class
- A new object is created in memory
- Dependencies are **NOT** yet injected (this is important!)
- If you have constructor injection, parameters are injected here

**Example:**
```java
@Component
public class UserService {
    
    // Constructor injection: Dependencies passed as parameters
    public UserService(UserRepository repository) {
        System.out.println("Constructor called!");
        this.repository = repository;  // ✓ Dependencies injected here
    }
    
    // This field is NOT yet injected at this point
    @Autowired
    private EmailService emailService;  // ✗ Still NULL here!
}
```

**Timeline:**
```
Before constructor: ✗ emailService = null
Constructor runs:   ✓ repository injected
After constructor:  ✗ emailService = null (injected in STEP 3)
```

**Interview Tip:** "Constructor injection happens during instantiation, but field injection happens later."

---

### STEP 3: Dependency Injection (Properties Populated)

**What Happens:**
- Spring injects all `@Autowired` fields
- Setter injection happens
- All dependencies are now resolved
- Your bean has access to all its dependencies

**Example:**
```java
@Component
public class UserService {
    
    @Autowired
    private UserRepository repository;  // ← Injected HERE
    
    @Autowired
    private EmailService emailService;  // ← Injected HERE
    
    @Value("${app.name}")
    private String appName;  // ← Property injected HERE
    
    @Autowired
    public void setNotificationService(NotificationService service) {
        // ← Setter injection happens HERE
        this.notificationService = service;
    }
}
```

**Interview Tip:** "Field injection (@Autowired) happens in STEP 3, not in the constructor."

---

### STEP 4: Aware Interfaces (Optional)

**What Happens:**
- If your bean implements special Spring interfaces, Spring calls specific methods
- This gives your bean access to Spring framework objects

**Common Aware Interfaces:**

```java
@Component
public class MyBean implements 
    BeanNameAware,
    ApplicationContextAware {

    @Override
    public void setBeanName(String name) {
        // Called 1st - You learn what your bean name is
        System.out.println("My bean name: " + name);
    }

    @Override
    public void setApplicationContext(ApplicationContext context) {
        // Called 2nd - You get the entire ApplicationContext
        this.context = context;
    }
}
```

**When is it used?** Rarely in business code. Mostly in frameworks and libraries.

---

### STEP 5: Pre-Initialization Processing (BeanPostProcessor)

**What Happens:**
- Spring calls `postProcessBeforeInitialization()` on all BeanPostProcessors
- This is where Spring does internal magic
- Your code usually doesn't see this

**Advanced:**
```java
@Component
public class MyBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        System.out.println("Before init: " + beanName);
        return bean;  // Return same or modified bean
    }
}
```

**Interview Tip:** "BeanPostProcessor is where Spring framework magic happens internally."

---

### STEP 6: Initialization - @PostConstruct ⭐

**What Happens:**
- `@PostConstruct` method is automatically called
- Perfect place for your initialization logic
- Runs **only once** when the bean is created
- Dependencies are **guaranteed** to be injected

**Example:**
```java
@Component
public class CacheService {

    @Autowired
    private EmployeeRepository repository;

    private List<Employee> cachedEmployees;

    @PostConstruct
    public void loadCache() {
        System.out.println("Loading cache...");
        
        // All dependencies are ready!
        cachedEmployees = repository.findAll();
        
        System.out.println("Loaded: " + cachedEmployees.size() + " employees");
    }
}
```

**Common Use Cases:**
- ✓ Load data into cache
- ✓ Initialize database connections
- ✓ Start background tasks
- ✓ Initialize Kafka producers/consumers
- ✓ Load configuration files
- ✓ Set up scheduled tasks

**Important:** 
- `@PostConstruct` is called **AFTER** all dependencies are injected
- It's the safest place to do initialization

---

### STEP 7: Post-Initialization Processing (AOP Proxies Created!)

**What Happens:**
- Spring calls `postProcessAfterInitialization()` on all BeanPostProcessors
- **Most Important:** AOP proxies are created here (if needed)
- `@Transactional` creates proxies in this step
- What you get in your code might be a **proxy**, not your original bean

**Advanced Example - @Transactional Proxy:**
```java
@Component
@Transactional  // ← This creates a proxy!
public class UserService {
    
    public void saveUser(User user) {
        // Transaction magic handled by proxy
    }
}

// What actually happens:
// 1. UserService bean created
// 2. During STEP 7: UserServiceProxy created (wraps UserService)
// 3. UserServiceProxy registered in context (not UserService!)
// 4. When you inject: you get UserServiceProxy
```

**Interview Tip:** "Your bean might be wrapped in a proxy. Don't use `instanceof` to check the actual type."

---

### STEP 8: Bean Ready for Use ✓

**What Happens:**
- Bean is fully initialized
- All dependencies injected
- Bean registered in singleton cache
- Ready to be injected into other beans
- Available via `context.getBean()`

**Example:**
```java
@Component
public class UserController {
    
    @Autowired
    private UserService userService;  // ← Can inject here!
}

// Or retrieve manually:
UserService service = context.getBean(UserService.class);
```

---

### STEP 9: Application Running

**What Happens:**
- Your Spring Boot application runs normally
- Beans are used by controllers, services, etc.
- Nothing special happens unless you do something

---

### STEP 10: Destruction - @PreDestroy ⭐

**What Happens:**
- Called just before bean is destroyed
- Perfect place for cleanup logic
- Runs **only once** when application shuts down
- Should close resources, connections, threads

**Example:**
```java
@Component
public class DatabaseConnection {

    private Connection connection;

    @PostConstruct
    public void init() {
        System.out.println("Opening database connection...");
        connection = DriverManager.getConnection("jdbc:mysql://localhost/db");
    }

    @PreDestroy
    public void cleanup() {
        System.out.println("Closing database connection...");
        try {
            if (connection != null) {
                connection.close();
            }
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
}
```

**Common Use Cases:**
- ✓ Close database connections
- ✓ Shutdown thread pools
- ✓ Close Kafka consumers/producers
- ✓ Flush logs
- ✓ Clean up temporary files
- ✓ Release external resources

**Important:**
- `@PreDestroy` is called only for singleton beans (default)
- It's called **only** when `context.close()` is called
- Prototype beans don't trigger `@PreDestroy`

---

### STEP 11: Bean Destroyed

**What Happens:**
- Bean is removed from memory
- Eligible for garbage collection
- No longer accessible

---

## 🎓 Real-World Complete Example

```java
@Component
public class EmployeeService {

    @Autowired
    private EmployeeRepository repository;  // Injected in STEP 3

    @Autowired
    private EmailService emailService;      // Injected in STEP 3

    @Value("${cache.enabled}")
    private boolean cacheEnabled;            // Injected in STEP 3

    private List<Employee> cache;

    // ========== INITIALIZATION ==========
    
    public EmployeeService() {
        System.out.println("1. Constructor called - STEP 2");
    }

    @PostConstruct
    public void init() {
        System.out.println("2. @PostConstruct called - STEP 6");
        System.out.println("   Repository is ready: " + (repository != null));
        System.out.println("   Cache enabled: " + cacheEnabled);
        
        if (cacheEnabled) {
            cache = repository.findAll();
            System.out.println("   Cache loaded with " + cache.size() + " employees");
        }
    }

    // ========== BUSINESS LOGIC ==========
    
    public List<Employee> getAllEmployees() {
        System.out.println("3. getAllEmployees() called - STEP 9 (Using bean)");
        return cache != null ? cache : repository.findAll();
    }

    public void saveEmployee(Employee emp) {
        System.out.println("4. saveEmployee() called - STEP 9 (Using bean)");
        Employee saved = repository.save(emp);
        emailService.sendConfirmation(saved);
    }

    // ========== CLEANUP ==========
    
    @PreDestroy
    public void cleanup() {
        System.out.println("5. @PreDestroy called - STEP 10");
        if (cache != null) {
            cache.clear();
            System.out.println("   Cache cleared");
        }
        System.out.println("   Database connections closed");
    }
}
```

**Output when application starts and shuts down:**
```
1. Constructor called - STEP 2
2. @PostConstruct called - STEP 6
   Repository is ready: true
   Cache enabled: true
   Cache loaded with 50 employees
3. getAllEmployees() called - STEP 9 (Using bean)
4. saveEmployee() called - STEP 9 (Using bean)
5. @PreDestroy called - STEP 10
   Cache cleared
   Database connections closed
```

---

## ⭐ Key Differences Between Initialization & Destruction

| Aspect | Initialization | Destruction |
|--------|---|---|
| **Annotation** | `@PostConstruct` | `@PreDestroy` |
| **When Called** | After dependency injection | Before bean is destroyed |
| **How Many Times** | Once per bean creation | Once per bean destruction |
| **Purpose** | Setup, initialize resources | Cleanup, release resources |
| **Dependencies Available** | ✓ Yes, all injected | ✓ Yes, still available |
| **When Executed** | During startup (for singletons) | During shutdown (if context.close() called) |

---

## 🎯 Important Interview Questions & Answers

### Q1: What is Spring Bean Lifecycle?

**Answer:**
"Spring Bean Lifecycle is the series of stages a Spring bean goes through from creation to destruction. It includes:
1. Component scanning
2. Bean instantiation (constructor)
3. Dependency injection
4. Initialization (@PostConstruct)
5. Bean usage
6. Destruction (@PreDestroy)

For singleton beans, steps 1-4 happen during application startup. For lazy or prototype beans, they happen when the bean is requested."

---

### Q2: What is the difference between Constructor and @PostConstruct?

**Answer:**
```java
// Constructor: Called during bean instantiation (STEP 2)
// At this point: @Autowired fields are NOT yet injected
public MyService(Dependency dep) {
    this.dep = dep;  // ✓ Parameter injected
    this.field = null;  // ✗ Field still null!
}

// @PostConstruct: Called after dependency injection (STEP 6)
// At this point: ALL dependencies are injected
@PostConstruct
public void init() {
    // ✓ Constructor parameter is available
    // ✓ @Autowired field is now available
    // ✓ Perfect place for initialization
}
```

**Interview Answer:** "Constructor is called during instantiation before dependencies are injected. @PostConstruct is called after all dependencies are injected, making it the safe place for initialization logic."

---

### Q3: When is @PostConstruct called? Only at startup?

**Answer:**
"@PostConstruct is called after every bean is created and dependency injection is complete, not just at startup.

For **singleton** beans (default):
- Created at startup → @PostConstruct runs at startup

For **prototype** beans:
- Created on every context.getBean() call → @PostConstruct runs every time

For **@Lazy** beans:
- Created on first use → @PostConstruct runs when first injected

Example:"

```java
@Component
@Lazy
public class LazyService {
    @PostConstruct
    public void init() {
        System.out.println("Not at startup, but when first used!");
    }
}
```

**Interview Answer:** "@PostConstruct is tied to bean creation, not application startup. It runs after every bean is created and dependency injection is done."

---

### Q4: Is @PreDestroy guaranteed to run?

**Answer:**
```java
// ✓ Runs: For singleton beans when context.close() is called
@Component
public class SingletonService {
    @PreDestroy
    public void cleanup() {
        System.out.println("This will run!");
    }
}

// ✗ Never runs: For prototype beans (Spring doesn't track them)
@Component
@Scope("prototype")
public class PrototypeService {
    @PreDestroy
    public void cleanup() {
        System.out.println("This NEVER runs!");
    }
}

// ✗ Won't run: If context.close() is never called
@Component
public class ServiceWithoutClose {
    @PreDestroy
    public void cleanup() {
        System.out.println("Won't run if context.close() is not called!");
    }
}
```

**Interview Answer:** "@PreDestroy is only guaranteed to run for singleton beans when ApplicationContext.close() is called. It doesn't run for prototype beans because Spring doesn't track them."

---

### Q5: What are common use cases for @PostConstruct?

**Answer:**
"@PostConstruct is used for initialization logic that requires all dependencies to be injected:

1. **Load Cache:**
```java
@PostConstruct
public void loadCache() {
    cachedData = repository.findAll();
}
```

2. **Initialize Kafka Consumer:**
```java
@PostConstruct
public void initKafka() {
    kafkaConsumer = new KafkaConsumer(props);
    kafkaConsumer.subscribe(topics);
}
```

3. **Start Background Tasks:**
```java
@PostConstruct
public void startTasks() {
    scheduler.scheduleAtFixedRate(this::processQueue, 0, 5, TimeUnit.MINUTES);
}
```

4. **Validate Configuration:**
```java
@PostConstruct
public void validateConfig() {
    if (apiKey == null || apiKey.isEmpty()) {
        throw new IllegalStateException("API key not configured!");
    }
}
```

5. **Initialize Database Schema:**
```java
@PostConstruct
public void initDb() {
    dbService.createTablesIfNotExist();
}
```"

---

### Q6: What are common use cases for @PreDestroy?

**Answer:**
"@PreDestroy is used for cleanup logic before the bean is destroyed:

1. **Close Database Connections:**
```java
@PreDestroy
public void closeDb() {
    if (connection != null) {
        connection.close();
    }
}
```

2. **Stop Kafka Consumer:**
```java
@PreDestroy
public void stopKafka() {
    if (kafkaConsumer != null) {
        kafkaConsumer.close();
    }
}
```

3. **Shutdown Thread Pool:**
```java
@PreDestroy
public void shutdownThreadPool() {
    executor.shutdown();
}
```

4. **Flush Logs:**
```java
@PreDestroy
public void flushLogs() {
    logger.flush();
}
```

5. **Release External Resources:**
```java
@PreDestroy
public void releaseResources() {
    cache.clear();
    fileHandle.close();
    socketConnection.disconnect();
}
```"

---

### Q7: What happens if @PostConstruct throws an exception?

**Answer:**
```java
@Component
public class BrokenService {
    @PostConstruct
    public void init() {
        throw new RuntimeException("Initialization failed!");
    }
}
```

**Result:**
- The bean is NOT added to the Spring context
- Application startup fails
- Error is logged
- Other beans that depend on BrokenService cannot be created

**Interview Answer:** "If @PostConstruct throws an exception, the bean is not initialized, and if other beans depend on it, they also fail to create. The application startup fails."

---

### Q8: What's the difference between @PostConstruct and InitializingBean.afterPropertiesSet()?

**Answer:**
```java
// Both do the same thing, but @PostConstruct is preferred:

// Method 1: @PostConstruct (Modern, recommended)
@Component
public class ModernService {
    @PostConstruct
    public void init() {
        System.out.println("Initialization");
    }
}

// Method 2: InitializingBean (Old, less common)
@Component
public class LegacyService implements InitializingBean {
    @Override
    public void afterPropertiesSet() {
        System.out.println("Initialization");
    }
}

// Method 3: init-method in XML (Very old)
// <bean id="service" class="Service" init-method="init"/>
```

**Order of Execution (if multiple are present):**
1. @PostConstruct
2. InitializingBean.afterPropertiesSet()
3. init-method

**Interview Answer:** "@PostConstruct is the modern standard and preferred way. InitializingBean is legacy and less commonly used."

---

### Q9: Do prototype beans execute @PostConstruct?

**Answer:**
```java
// Yes! @PostConstruct runs for prototype beans
@Component
@Scope("prototype")
public class PrototypeService {
    @PostConstruct
    public void init() {
        System.out.println("Called every time a new instance is created!");
    }
}

// But @PreDestroy does NOT run for prototype beans
@Component
@Scope("prototype")
public class PrototypeService {
    @PreDestroy
    public void cleanup() {
        System.out.println("This NEVER runs!");  // Not managed by Spring
    }
}
```

**Why?** Spring creates prototype beans but doesn't manage their lifecycle. The caller is responsible for cleanup.

**Interview Answer:** "Prototype beans execute @PostConstruct every time a new instance is created, but @PreDestroy never runs because Spring doesn't track prototype bean lifecycle."

---

### Q10: Can you do heavy operations in @PostConstruct?

**Answer:**
```java
// ✓ Yes, but be careful about performance

@Component
public class DataService {
    @PostConstruct
    public void loadData() {
        // Loads 1 million records from database
        // Takes 10 seconds
        // Application startup is delayed by 10 seconds!
        allRecords = repository.findAll();
    }
}

// Better approach: Use @Lazy or load asynchronously
@Component
@Lazy
public class DataService {
    @PostConstruct
    public void initAsync() {
        // Load data in background thread
        executor.submit(() -> loadData());
    }
}
```

**Interview Answer:** "You can do heavy operations in @PostConstruct, but remember it delays application startup. For heavy initialization, consider using @Lazy or loading asynchronously."

---

### Q11: What is the difference between Singleton and Prototype scope?

**Answer:**
```java
// SINGLETON (Default) - One instance for entire app
@Component  // Default scope is singleton
public class SingletonService {
    @PostConstruct
    public void init() {
        System.out.println("Created once at startup");
    }
    
    @PreDestroy
    public void cleanup() {
        System.out.println("Destroyed once at shutdown");  // ✓ Runs
    }
}

// PROTOTYPE - New instance every time you request it
@Component
@Scope("prototype")
public class PrototypeService {
    @PostConstruct
    public void init() {
        System.out.println("Created every time you call getBean()");
    }
    
    @PreDestroy
    public void cleanup() {
        System.out.println("Never called!");  // ✗ Never runs
    }
}
```

**Comparison:**

| Aspect | Singleton | Prototype |
|--------|-----------|-----------|
| **Instances Created** | 1 per application | 1 per request |
| **When Created** | At startup | On demand |
| **@PostConstruct** | Runs at startup | Runs on each creation |
| **@PreDestroy** | ✓ Runs at shutdown | ✗ Never runs |
| **Memory Usage** | Low (one instance) | High (many instances) |
| **Default** | Yes | No |

**Interview Answer:** "Singleton creates one bean per application lifetime (default). Prototype creates new beans on every request. @PreDestroy only runs for singletons."

---

### Q12: What is @Lazy annotation?

**Answer:**
```java
// Without @Lazy - Created at startup
@Component
public class EagerService {
    @PostConstruct
    public void init() {
        System.out.println("Created at startup");  // Runs immediately
    }
}

// With @Lazy - Created on first use
@Component
@Lazy
public class LazyService {
    @PostConstruct
    public void init() {
        System.out.println("Created on first use!");  // Runs when first injected
    }
}

// In another component:
@Component
public class Consumer {
    @Autowired
    @Lazy
    private LazyService lazyService;  // Not created until this line executes
}
```

**Benefits:**
- ✓ Faster startup time
- ✓ Less memory usage initially
- ✓ Good for expensive beans

**Drawbacks:**
- ✗ Initialization errors hidden until first use
- ✗ First request might be slow

**Interview Answer:** "@Lazy delays bean creation until first use. Useful for expensive beans to speed up startup, but initialization errors appear later."

---

### Q13: Explain the complete lifecycle with all 11 steps

**Answer:**
"The complete Spring Bean Lifecycle has 11 steps:

1. **Component Scanning** - Spring finds @Component classes and creates BeanDefinitions
2. **Bean Instantiation** - Constructor is called, object created
3. **Dependency Injection** - @Autowired fields injected, setter methods called
4. **Aware Interfaces** - BeanNameAware, ApplicationContextAware methods called
5. **Pre-Initialization Processing** - BeanPostProcessor.postProcessBeforeInitialization()
6. **Initialization** - @PostConstruct, InitializingBean.afterPropertiesSet(), init-method
7. **Post-Initialization Processing** - BeanPostProcessor.postProcessAfterInitialization(), AOP proxies created
8. **Bean Ready** - Registered in singleton cache, available for injection
9. **Application Running** - Bean is used by application
10. **Destruction Callbacks** - @PreDestroy, DisposableBean.destroy(), destroy-method
11. **Bean Destroyed** - Removed from memory, eligible for garbage collection"

---

## 🏢 Enterprise Features for Experienced Developers

### ObjectProvider & Lazy Resolution

**Modern Alternative to @Autowired:**

```java
@Component
public class AdvancedService {
    
    // Traditional @Autowired - Fails if bean doesn't exist
    @Autowired
    private OptionalDependency dep;  // Throws exception if missing
    
    // ObjectProvider - Handles missing beans gracefully
    @Autowired
    private ObjectProvider<ExpensiveService> expensiveServiceProvider;
    
    public void process() {
        // Get bean if it exists, fallback to default
        ExpensiveService service = expensiveServiceProvider.getIfAvailable(
            () -> new ExpensiveService("default")
        );
        
        // Or check existence
        expensiveServiceProvider.ifAvailable(s -> {
            System.out.println("Service is available: " + s);
        });
    }
}
```

**Key Differences:**
- `@Autowired` - Eager resolution, fails fast if missing
- `ObjectProvider` - Lazy resolution, null-safe, supports fallbacks
- `ObjectProvider<T>` with `@Lazy` - Most powerful for optional dependencies

**Use Case:** Framework code, plugin systems, optional integrations (Redis cache, external APIs).

---

### BeanFactory vs ApplicationContext Initialization

**Lazy vs Eager Bean Creation:**

```java
// BeanFactory: Lazy initialization (default)
BeanFactory factory = new DefaultListableBeanFactory();
// Beans NOT created yet

Bean bean1 = (Bean) factory.getBean("myBean");  // Created on demand
Bean bean2 = (Bean) factory.getBean("myBean");  // Same instance retrieved

// ApplicationContext: Eager initialization (most common)
ApplicationContext context = new ClassPathXmlApplicationContext("config.xml");
// All singleton beans created immediately during startup

UserService service = context.getBean(UserService.class);  // Already created
```

**Implications for Lifecycle:**

```java
// BeanFactory timeline:
// getBean() call → STEP 1-8 entire lifecycle

// ApplicationContext timeline:
// Constructor call → STEP 1-8 entire lifecycle for all singletons
// getBean() call → Bean already ready
```

**Production Impact:**
- `ApplicationContext` - Higher startup time, better error detection
- `BeanFactory` - Lower startup time, errors may appear later
- **Recommendation:** Use `ApplicationContext` for applications, `BeanFactory` for containers.

---

### Scoped Proxies for Prototype Bean Lifecycle

**Inverse the Prototype Lifecycle Rule:**

```java
// Without proxy: @PreDestroy never fires
@Component
@Scope("prototype")
public class PrototypeBean {
    @PreDestroy
    public void cleanup() {
        System.out.println("Never called!");  // ✗ Not managed
    }
}

// With proxy: @PreDestroy fires when proxy is destroyed
@Component
@Scope(value = "prototype", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class ManagedPrototypeBean {
    @PreDestroy
    public void cleanup() {
        System.out.println("Now this fires!");  // ✓ Proxy manages lifecycle
    }
}

// Usage:
@Component
public class Consumer {
    @Autowired
    private ManagedPrototypeBean bean;  // Injected by proxy
    
    public void test() {
        // Each call to bean getters creates new instance internally
        // But proxy cleans up old instances
    }
}
```

**When to Use:**
- Prototype beans with thread-local state
- Request-scoped beans needing cleanup
- Stateful beans in multi-threaded environments

**Performance Note:** Scoped proxies add minimal overhead (~5-10% per call).

---

### AOP: Runtime Proxies vs Compile-Time Weaving

**Spring Default: Runtime JDK/CGLIB Proxies**

```java
@Component
@Transactional  // Creates proxy in STEP 7
public class UserService {
    public void saveUser(User u) { }
}

// Proxy chain: UserServiceProxy → UserService
// Created at runtime using reflection
// Visible in stack traces as UserService$$EnhancerBySpringCGLIB$$xyz
```

**Alternative: AspectJ Compile-Time Weaving**

```java
// In pom.xml:
<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>aspectj-maven-plugin</artifactId>
</plugin>

@Component
@Transactional
public class UserService {
    public void saveUser(User u) { }
}

// Bytecode modified at compile-time
// No proxy overhead at runtime
// No visibility in code (pure bytecode)
```

**Comparison:**

| Aspect | JDK/CGLIB Proxy | AspectJ Weaving |
|--------|---|---|
| **When Created** | Runtime (STEP 7) | Compile-time |
| **Overhead** | 10-15% per call | 0-2% per call |
| **Stack Traces** | Shows proxy class | Clean stack traces |
| **Private Methods** | ✗ Not intercepted | ✓ Intercepted |
| **Constructor Calls** | ✗ Not intercepted | ✓ Intercepted |
| **Setup Complexity** | Simple | Requires build plugin |
| **Performance** | Good for most cases | Better for high-throughput |

**Recommendation:** Use default proxies for most applications. Use AspectJ weaving only if profiling shows AOP overhead > 5%.

---

### BeanPostProcessor Chain & Order Gotchas

**Multiple Processors with @Order:**

```java
@Component
@Order(1)
public class FirstProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        System.out.println("First processor wrapping: " + beanName);
        return new FirstProxy(bean);  // Wraps bean
    }
}

@Component
@Order(2)
public class SecondProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        System.out.println("Second processor wrapping: " + beanName);
        return new SecondProxy(bean);  // Wraps result of first
    }
}

// Result: SecondProxy(FirstProxy(OriginalBean))
// Nested proxies - each wraps the result of previous!
```

**Common Gotcha: Proxy Chain Depth**

```java
// With 5 BeanPostProcessors creating proxies:
// MyClass$$EnhancerBySpringCGLIB$$xxx$$EnhancerBySpringCGLIB$$yyy...
// Stack depth increases with each processor
// Can cause performance issues with deep chains

// Solution: Minimize processors or use @ConditionalOnBean
@Component
@ConditionalOnBean(name = "mySpecificBean")
public class SelectiveProcessor implements BeanPostProcessor {
    // Only processes when specific bean exists
}
```

**Execution Order is NOT Same as Creation:**
- Created in declared order
- Each processor sees ALL previous processors' modifications
- Unknown processor order = unpredictable behavior

---

### Event Listeners During Lifecycle

**When Do Events Fire?**

```java
@Component
public class LifecycleEventListener {
    
    @EventListener
    public void onContextStarted(ContextStartedEvent event) {
        System.out.println("Context started");
        // Fires AFTER all singleton beans initialized
        // STEP 9 equivalent
    }
    
    @EventListener
    public void onContextRefreshed(ContextRefreshedEvent event) {
        System.out.println("Context refreshed");
        // Fires after all beans created and initialized
        // SAFE place to access all beans
    }
}

@Component
public class Service {
    @PostConstruct
    public void init() {
        System.out.println("Service initialized");
        // Fires in STEP 6
        // Event listeners NOT yet fired!
    }
}

// Order: Service.init() → Then event listeners fire
```

**Production Pattern:**

```java
@Component
public class ApplicationStartupService {
    
    @Autowired
    private List<InitializationHandler> handlers;
    
    @EventListener
    public void onStartup(ContextRefreshedEvent event) {
        // ALL beans are initialized here
        // Safe to initialize cross-bean dependencies
        handlers.forEach(InitializationHandler::afterAllBeansReady);
    }
}
```

---

### Graceful Shutdown (Spring Boot 2.3+)

**Configuration:**

```yaml
# application.yml
server:
  shutdown: graceful  # Was 'immediate' before 2.3
  
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

**Implementation:**

```java
@Component
public class GracefulShutdownService implements ApplicationListener<ContextClosedEvent> {
    
    private static final Logger log = LoggerFactory.getLogger(GracefulShutdownService.class);
    
    @Override
    public void onApplicationEvent(ContextClosedEvent event) {
        log.info("Application shutting down gracefully...");
        // Triggers @PreDestroy on all singletons
        // Context.close() is called automatically by Spring Boot
    }
}

@Component
public class ResourceManager {
    
    @PreDestroy
    public void shutdown() {
        log.info("Cleaning up resources...");
        // Called by graceful shutdown handler
        // 30 seconds available to complete
    }
}
```

**Timeline:**
```
1. Kill signal (SIGTERM) received
2. Graceful shutdown initiated
3. No new requests accepted
4. Existing requests allowed to complete (30s timeout)
5. @PreDestroy methods fire
6. Application terminates
```

**vs Old Behavior:**
- Before: Immediate termination on SIGTERM
- Now: Graceful with timeout (configurable)

---

### Testing Lifecycle with @DirtiesContext

**Lifecycle Impact on Tests:**

```java
@SpringBootTest
public class IntegrationTests {
    
    @Autowired
    private UserService userService;
    
    @Test
    public void test1() {
        // Uses shared context with all singleton beans
    }
    
    @Test
    @DirtiesContext(classMode = ClassMode.AFTER_EACH_TEST_METHOD)
    public void test2() {
        // After this test: context.close() called
        // ALL @PreDestroy methods fire
        // Expensive: new context created for next test
    }
    
    @Test
    public void test3() {
        // Uses new context (if test2 already ran)
    }
}
```

**Cost Implications:**

```
3 tests without @DirtiesContext:
- Context creation: 1x (shared)
- @PostConstruct calls: 1x each bean
- Total time: ~1 second

3 tests with @DirtiesContext after each:
- Context creation: 3x (fresh each test)
- @PostConstruct calls: 3x each bean
- Total time: ~10 seconds (10x slower!)
```

**Best Practices:**

```java
@SpringBootTest
public class Tests {
    
    @Test
    @DirtiesContext  // Only when absolutely necessary
    public void testRequiringFreshContext() {
        // Modify singletons or static state
    }
    
    @Test
    public void normalTest() {
        // Don't use @DirtiesContext
        // Use @MockBean for dependencies instead
    }
}
```

---

### Production Checklist (Enhanced)

**Initialization:**
- [ ] **Don't do heavy work in constructors** — Use `@PostConstruct`
- [ ] **Prefer constructor injection** — Better for testing, avoids circular deps
- [ ] **Use `ObjectProvider<T>`** — For optional dependencies in frameworks
- [ ] **Consider `@Lazy`** — For expensive beans, watch for hidden errors
- [ ] **Validate configuration in `@PostConstruct`** — Fail fast on startup
- [ ] **Use `ContextRefreshedEvent`** — For cross-bean initialization logic
- [ ] **Log in `@PostConstruct`** — Not in constructor, easier to debug

**Proxies & AOP:**
- [ ] **Use `@Transactional` judiciously** — Creates proxy overhead (10-15%)
- [ ] **Don't rely on `instanceof`** — Bean might be a proxy
- [ ] **Be aware of proxy chain depth** — Multiple processors = nested proxies
- [ ] **Test proxy behavior** — Use `@ConditionalOnBean` for processors

**Lifecycle Management:**
- [ ] **Implement graceful shutdown** — Configure `server.shutdown=graceful`
- [ ] **Implement `@PreDestroy`** — For cleanup (threads, connections, etc.)
- [ ] **Use scoped proxies cautiously** — For prototype beans needing cleanup
- [ ] **Avoid prototype scope** — They don't get destroyed, can leak resources
- [ ] **Track singleton count** — Beans accumulate over app lifetime

**Testing:**
- [ ] **Use `@DirtiesContext` sparingly** — 10x performance penalty per test
- [ ] **Use `@MockBean` instead** — Reset dependencies without fresh context
- [ ] **Test initialization order** — Use `@Order` if multiple BeanPostProcessors
- [ ] **Watch for lazy initialization errors** — Appear in first use, not startup

---

## 📋 Spring Bean Lifecycle Checklist

**For Beginners:**
- [ ] Understand the 11-step lifecycle
- [ ] Know what @PostConstruct does
- [ ] Know what @PreDestroy does
- [ ] Know the difference between constructor and @PostConstruct
- [ ] Remember: dependencies injected in STEP 3
- [ ] Remember: @PostConstruct runs in STEP 6
- [ ] Remember: @PreDestroy runs in STEP 10

**For Interviews:**
- [ ] Explain the complete lifecycle flow
- [ ] Explain when @PostConstruct is called
- [ ] Explain @PostConstruct vs constructor
- [ ] Know that @PreDestroy only runs for singletons
- [ ] Know that prototype beans don't trigger @PreDestroy
- [ ] Understand the difference between singleton and prototype
- [ ] Understand @Lazy initialization
- [ ] Know common use cases for @PostConstruct/@PreDestroy
- [ ] Understand AOP proxy creation in STEP 7
- [ ] Know what happens if @PostConstruct throws exception

**For Experienced Developers:**
- [ ] Understand `ObjectProvider<T>` vs `@Autowired`
- [ ] Know `BeanFactory` vs `ApplicationContext` lifecycle differences
- [ ] Understand scoped proxies for prototype lifecycle
- [ ] Know when to use AspectJ weaving vs runtime proxies
- [ ] Understand BeanPostProcessor ordering and nesting
- [ ] Know event listener timing in lifecycle
- [ ] Configure graceful shutdown properly
- [ ] Use `@DirtiesContext` wisely in tests
- [ ] Monitor bean creation count in production
- [ ] Profile AOP proxy overhead if needed

---

## 🔗 References

- [Spring Official Documentation - Bean Lifecycle](https://docs.spring.io/spring-framework/reference/core/beans/factory-extension.html)
- [Spring AOP & Proxies](https://docs.spring.io/spring-framework/reference/core/aop.html)
- [Bean Scopes](https://docs.spring.io/spring-framework/reference/core/beans/factory-scopes.html)
- [ObjectProvider Documentation](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/beans/factory/ObjectProvider.html)
- [Graceful Shutdown (Spring Boot 2.3+)](https://spring.io/blog/2020/03/27/graceful-shutdown-now-available-in-spring-boot-2-3-0-m1)
- [AspectJ Weaving Guide](https://docs.spring.io/spring-framework/reference/core/aop-api.html)

---

## 💡 Quick Memory Tips

| Concept | Memory Tip |
|---------|-----------|
| **@PostConstruct** | "After Construction" = After bean is created and all dependencies injected |
| **@PreDestroy** | "Pre Destruction" = Before bean is destroyed, do cleanup |
| **Constructor** | Called BEFORE dependency injection (STEP 2) |
| **@Autowired** | Set AFTER constructor (STEP 3) |
| **Singleton (default)** | One instance for entire application lifetime |
| **Prototype** | New instance created each time you request it |
| **@Transactional** | Creates a proxy wrapping your bean in STEP 7 |
| **@Lazy** | Bean created on first use, not at startup |
| **ObjectProvider<T>** | Lazy resolution with null-safety and fallbacks |
| **Graceful Shutdown** | Set `server.shutdown=graceful` for production |
| **Scoped Proxy** | Allows @PreDestroy to fire for prototype beans |
| **@DirtiesContext** | 10x slower - use `@MockBean` instead |
