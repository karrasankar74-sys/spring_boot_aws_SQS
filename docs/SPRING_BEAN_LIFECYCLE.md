# Spring Bean Lifecycle - Complete Flow

A comprehensive guide to the Spring Bean lifecycle with all the nuances and internal mechanisms that matter in production applications.

---

## Complete Lifecycle Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SPRING APPLICATION STARTUP                          │
└─────────────────────────────────────────────────────────────────────────┘
                                   ↓
         ┌─────────────────────────────────────────────────┐
         │  1. COMPONENT SCANNING & BEAN DEFINITION        │
         │  - Scan @Configuration, @Component, etc.       │
         │  - Create BeanDefinition (metadata only)        │
         │  - Register in BeanDefinitionRegistry           │
         └─────────────────────────────────────────────────┘
                                   ↓
         ┌─────────────────────────────────────────────────┐
         │  2. BEAN INSTANTIATION (Constructor Called)    │
         │  - Constructor injection params resolved        │
         │  - Object created in memory                     │
         │  - Dependencies NOT yet injected                │
         └─────────────────────────────────────────────────┘
                                   ↓
         ┌─────────────────────────────────────────────────┐
         │  3. POPULATE PROPERTIES (Dependency Injection)  │
         │  - @Autowired field injection                   │
         │  - @Resource injection                          │
         │  - Setter injection                             │
         │  - Circular dependency detection                │
         └─────────────────────────────────────────────────┘
                                   ↓
         ┌─────────────────────────────────────────────────┐
         │  4. AWARE INTERFACES (Spring-Specific Setup)    │
         │  - BeanNameAware.setBeanName()                  │
         │  - BeanClassLoaderAware.setBeanClassLoader()    │
         │  - BeanFactoryAware.setBeanFactory()            │
         │  - ApplicationContextAware.setApplicationContext()
         │  (Executed in this order)                       │
         └─────────────────────────────────────────────────┘
                                   ↓
         ┌─────────────────────────────────────────────────┐
         │  5. PRE-INITIALIZATION PROCESSING               │
         │  - BeanPostProcessor.postProcessBefore...()     │
         │  - @Autowired/@Value resolution (happens HERE)  │
         │  - Custom annotation processing                 │
         │  - AOP interceptor registration                 │
         └─────────────────────────────────────────────────┘
                                   ↓
         ┌─────────────────────────────────────────────────┐
         │  6. INITIALIZATION CALLBACKS                    │
         │  Priority Order:                                │
         │  a) @PostConstruct method (if present)          │
         │  b) InitializingBean.afterPropertiesSet()       │
         │  c) init-method (in bean definition)            │
         └─────────────────────────────────────────────────┘
                                   ↓
         ┌─────────────────────────────────────────────────┐
         │  7. POST-INITIALIZATION PROCESSING              │
         │  - BeanPostProcessor.postProcessAfter...()      │
         │  - AOP PROXY CREATION (happens HERE!)           │
         │  - @Transactional proxy wrapping                │
         │  - Custom bean wrapping/decoration              │
         └─────────────────────────────────────────────────┘
                                   ↓
         ┌─────────────────────────────────────────────────┐
         │  8. BEAN READY FOR USE                          │
         │  - Registered in singleton cache                │
         │  - Available for injection into other beans     │
         │  - Available via context.getBean()              │
         └─────────────────────────────────────────────────┘
                                   ↓
         ┌─────────────────────────────────────────────────┐
         │  9. APPLICATION RUNS                            │
         │  - Beans are used by application                │
         │  - Lifecycle continues until shutdown           │
         └─────────────────────────────────────────────────┘
                                   ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                    APPLICATION SHUTDOWN / GRACEFUL STOP                 │
└─────────────────────────────────────────────────────────────────────────┘
                                   ↓
         ┌─────────────────────────────────────────────────┐
         │  10. DESTRUCTION CALLBACKS                      │
         │  Priority Order (REVERSE of init):              │
         │  a) @PreDestroy method (if present)             │
         │  b) DisposableBean.destroy()                    │
         │  c) destroy-method (in bean definition)         │
         └─────────────────────────────────────────────────┘
                                   ↓
         ┌─────────────────────────────────────────────────┐
         │  11. BEAN DESTROYED                             │
         │  - Removed from singleton cache                 │
         │  - Resources cleaned up                         │
         │  - Eligible for garbage collection              │
         └─────────────────────────────────────────────────┘
```

---

## Detailed Stage Breakdown

### 1. Component Scanning & Bean Definition

```java
// Spring reads these annotations during classpath scanning
@Component
@Service      // Specialized @Component for business logic
@Repository   // Specialized @Component for data access
@Controller   // Specialized @Component for web layer
@Configuration // Holds @Bean methods

// Result: BeanDefinition created (NOT the actual bean yet)
// BeanDefinition contains:
// - Bean class name
// - Constructor argument values
// - Property values
// - Scope (singleton, prototype, request, etc.)
// - Lazy initialization flag
// - Init/destroy method names
```

**Key Point:** At this stage, no objects exist in memory. Spring just knows what beans *could* be created.

---

### 2. Bean Instantiation (Constructor)

```java
@Component
public class MyService {
    
    // Constructor injection happens BEFORE this method completes
    public MyService(DependencyA depA, DependencyB depB) {
        // At this point: 
        // ✓ Constructor parameters are resolved and passed
        // ✗ @Autowired fields are NOT yet injected
        // ✗ @PostConstruct has NOT run yet
    }
}

// Constructor injection resolution timeline:
// 1. Spring checks constructor parameters
// 2. Looks up beans matching those types
// 3. Injects them into constructor
// 4. Constructor executes
// 5. Object instance created
```

**Key Point:** Constructor injection happens *before* the object exists. Field/setter injection happens *after*.

---

### 3. Populate Properties (Dependency Injection)

```java
@Component
public class MyService {
    
    @Autowired
    private DependencyA depA;  // ← Injected HERE (step 3)
    
    @Autowired
    private DependencyB depB;  // ← Injected HERE (step 3)
    
    @Value("${app.name}")
    private String appName;    // ← Injected HERE (step 3)
    
    @Autowired
    public void setSomething(SomethingElse something) {
        // ← Setter injection also HERE (step 3)
    }
}
```

**Key Point:** This is where field and setter injection occur. Circular dependency detection happens at this stage.

---

### 4. Aware Interfaces (Spring Context Setup)

These interfaces allow beans to "become aware" of their context:

```java
@Component
public class MyBean implements 
    BeanNameAware,
    BeanFactoryAware,
    ApplicationContextAware {

    @Override
    public void setBeanName(String name) {
        // Called 1st - You get your bean name
        System.out.println("My bean name is: " + name);
    }

    @Override
    public void setBeanFactory(BeanFactory beanFactory) {
        // Called 2nd - You get access to BeanFactory
        this.beanFactory = beanFactory;
    }

    @Override
    public void setApplicationContext(ApplicationContext context) {
        // Called 3rd - You get full ApplicationContext
        this.context = context;  // ← Most commonly used
    }
}
```

**Execution Order:** BeanNameAware → BeanClassLoaderAware → BeanFactoryAware → ApplicationContextAware

**Real-World Use:** Typically used in frameworks/libraries, not business code.

---

### 5. BeanPostProcessor - Pre-Initialization

```java
@Component
public class CustomBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        // Called BEFORE @PostConstruct
        // Spring internals use this to:
        // - Resolve @Autowired annotations (AutowiredAnnotationBeanPostProcessor)
        // - Resolve @Value annotations (ConfigurationPropertiesBindingPostProcessor)
        // - Process @Resource, @Inject, etc.
        
        System.out.println("Before init: " + beanName);
        return bean;  // Return same or wrapped bean
    }
}
```

**Critical:** This is where Spring framework magic happens. Most annotation processing occurs here, not in `@PostConstruct`.

---

### 6. Initialization Callbacks

```java
@Component
public class MyService implements InitializingBean {

    @PostConstruct
    public void init1() {
        // Called 1st
        System.out.println("1. @PostConstruct");
    }

    @Override
    public void afterPropertiesSet() {
        // Called 2nd
        System.out.println("2. InitializingBean.afterPropertiesSet()");
    }

    // In XML: <bean init-method="init3"/>
    public void init3() {
        // Called 3rd
        System.out.println("3. init-method");
    }
}
```

**Execution Order:** 
1. `@PostConstruct` method
2. `InitializingBean.afterPropertiesSet()`
3. Custom `init-method` from bean definition

**Best Practice:** Use `@PostConstruct` in modern Spring (it's the standard).

---

### 7. BeanPostProcessor - Post-Initialization (AOP Proxies!)

```java
@Component
public class CustomBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        // Called AFTER initialization
        // Spring uses this for:
        // - AOP Proxy creation (happens HERE, not earlier!)
        // - @Transactional proxy wrapping
        // - Custom bean wrapping
        
        System.out.println("After init: " + beanName);
        
        // The bean you return might be different from input!
        // It might be a proxy wrapping the original
        return bean;
    }
}

// Example: @Transactional proxy creation
@Component
@Transactional
public class UserService {
    // What you registered in step 1: UserService
    // What's in the context after step 7: UserServiceProxy (wraps UserService)
}
```

**CRITICAL GOTCHA:** Your bean might be a proxy! If you cast or use `instanceof`, you're checking the proxy, not your bean.

```java
// Dangerous code:
@Component
public class Consumer {
    @Autowired
    private UserService userService;  // This is probably a proxy!
    
    public void test() {
        // This will be FALSE if @Transactional created a proxy
        if (userService instanceof UserService) {
            // userService is a proxy implementing UserService interface
        }
    }
}
```

---

### 8. Bean Ready for Use

```java
// After step 7:
// ✓ Object fully initialized
// ✓ All dependencies injected
// ✓ All post-processors executed
// ✓ Registered in singleton cache
// ✓ Available for:

@Component
public class Consumer {
    @Autowired
    private UserService userService;  // ← Can inject now
}

// Or retrieve manually:
UserService bean = context.getBean(UserService.class);
```

---

### 9. Bean Usage Phase

```java
// Application runs normally with fully initialized beans
// Bean lifecycle remains stable during normal operation
// (unless you manually destroy it)
```

---

### 10-11. Destruction (Shutdown)

```java
@Component
public class ResourceManager implements DisposableBean {

    @PreDestroy
    public void cleanup1() {
        // Called 1st when ApplicationContext closes
        System.out.println("1. @PreDestroy");
    }

    @Override
    public void destroy() {
        // Called 2nd
        System.out.println("2. DisposableBean.destroy()");
    }

    // In XML: <bean destroy-method="cleanup3"/>
    public void cleanup3() {
        // Called 3rd
        System.out.println("3. destroy-method");
    }
}

// Triggered by:
context.close();  // Graceful shutdown
```

**Important:** 
- `@PreDestroy` fires ONLY if context.close() is called
- Prototype beans do NOT trigger destruction callbacks
- Only singletons are tracked for destruction

---

## Critical Details & Edge Cases

### Scopes & Lifecycle Impact

| Scope | Lifecycle | When Destroyed |
|-------|-----------|---|
| **singleton** (default) | Created on startup, lives entire app | When context.close() |
| **prototype** | Created on each getBean() | Never (caller responsible) |
| **request** | Created per HTTP request | After request ends |
| **session** | Created per user session | After session ends |
| **application** | Created once per ServletContext | When context closes |

```java
@Component
@Scope("prototype")
public class MyPrototypeBean {
    
    @PreDestroy
    public void cleanup() {
        // ✗ This NEVER fires!
        // Prototype beans have no tracked lifecycle
        System.out.println("This won't print");
    }
}
```

### Constructor vs Field Injection Timing

```java
@Component
public class Example {
    
    // Constructor injection: Resolved in step 2 (before object exists)
    // ✓ Can't have circular dependencies
    // ✓ Better for testing (all deps in constructor)
    public Example(Dependency dep) {
        this.dep = dep;  // Guaranteed non-null
    }
    
    // Field injection: Resolved in step 3 (after object exists)
    // ✗ Can have circular dependencies (with setter)
    // ✗ Requires reflection
    @Autowired
    private AnotherDependency anotherDep;
}
```

### Circular Dependency Issues

```java
// ✗ FAILS - Constructor injection with circular dep
@Component
public class ServiceA {
    public ServiceA(ServiceB serviceB) { }
}

@Component
public class ServiceB {
    public ServiceB(ServiceA serviceA) { }
}
// Error: BeanCurrentlyInCreationException

// ✓ WORKS - Field injection with circular dep
@Component
public class ServiceA {
    @Autowired
    private ServiceB serviceB;
}

@Component
public class ServiceB {
    @Autowired
    private ServiceA serviceA;
}
// Spring handles this with proxies and lazy initialization
```

### @Lazy Initialization

```java
@Component
@Lazy
public class HeavyBean {
    // Not created during startup!
    // Created when first injected or context.getBean() called
}

@Component
public class Consumer {
    @Autowired
    @Lazy
    private HeavyBean heavyBean;  // Created on first use, not startup
}
```

**Impact:** Can hide initialization errors until beans are first used.

### BeanPostProcessor Execution Order

```java
@Component
@Order(1)
public class Processor1 implements BeanPostProcessor {
    // Executes 1st
}

@Component
@Order(2)
public class Processor2 implements BeanPostProcessor {
    // Executes 2nd
}

// Each processor sees the bean after previous processors modified it
// Processors can wrap beans (creating proxies)
```

### Proxy Creation Timing

```java
@Component
public class Service {
    @Transactional
    public void doSomething() { }
}

// Timeline:
// Step 1: Spring creates BeanDefinition for Service
// Step 2: Creates Service instance
// Step 3-6: Initializes Service instance
// Step 7: BeanPostProcessor creates ServiceProxy wrapping Service
// Step 8: ServiceProxy registered in context (not Service!)

// Later, when you inject:
@Autowired
private Service service;  // This is actually ServiceProxy!
```

---

## Production Checklist

- [ ] **Don't do heavy work in constructors** — Use `@PostConstruct`
- [ ] **Prefer constructor injection** — Better for testing, avoids circular deps
- [ ] **Use `@Transactional` judiciously** — Creates proxy overhead
- [ ] **Implement graceful shutdown** — Call `context.close()` or use Spring Boot defaults
- [ ] **Test initialization order** — Use `@Order` if multiple BeanPostProcessors
- [ ] **Watch out for proxies** — Don't rely on `instanceof` or casts
- [ ] **Avoid prototype scope in beans** — They don't get destroyed, can leak resources
- [ ] **Use `@Lazy` for expensive beans** — But watch for hidden init errors
- [ ] **Log in `@PostConstruct`** — Not in constructor, easier to debug
- [ ] **Implement `@PreDestroy`** — For cleanup (threads, connections, etc.)

---

## References

- [Spring Official Bean Lifecycle](https://docs.spring.io/spring-framework/reference/core/beans/factory-extension.html)
- [Spring AOP & Proxies](https://docs.spring.io/spring-framework/reference/core/aop.html)
- [Bean Scopes Documentation](https://docs.spring.io/spring-framework/reference/core/beans/factory-scopes.html)

