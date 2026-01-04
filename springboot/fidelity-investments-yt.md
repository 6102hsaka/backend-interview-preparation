### Q. How does HashMap handle a null key?
HashMap allows one null key. It does not call hashCode() on a null key (as it will result in `NullPointerException`); instead, it assigns a hash value of 0, which results in the null key being stored in bucket index 0

### Q. Why does ConcurrentHashMap not support null keys?
ConcurrentHashMap does not allow null keys or values because, in a concurrent environment, null would make it impossible to distinguish between “key absent” and “key present with null value,” breaking correctness and atomicity.

### Q. How do you fix tightly coupled code in Spring Boot?
Tightly coupled code in Spring Boot is fixed by using dependency injection, programming to interfaces, constructor injection, and separating responsibilities so that components depend on abstractions rather than concrete implementations.
1. Instead of instantiating the implementation class directly using the new keyword, pass it as a dependency.
2. Autowire interfaces instead of classes, so dependent classes do not need to be changed if implementation changes.
3. Using environment-specific config and avoiding hardcoding 
```
@Profile("prod")
@Service
public class ProdPaymentService { }
```

### Q. Explain the Spring bean lifecycle end-to-end.
The lifecycle of a Spring bean consists of the following phases, which are listed below
- Container Started: The Spring IoC container is initialized.
- Bean Instantiated: The container creates an instance of the bean.
- Dependencies Injected: The container injects the dependencies into the bean.
- Custom init() method: If the bean implements InitializingBean or has a custom initialization method specified via @PostConstruct or init-method.
- Bean is Ready: The bean is now fully initialized and ready to be used.
- Custom utility method: This could be any custom method you have defined in your bean.
- Custom destroy() method: If the bean implements DisposableBean or has a custom destruction method specified via @PreDestroy or destroy-method, it is called when the container is shutting down.

### Q. What will happen when two starters provide the same bean?
When two Spring Boot starters provide the same bean, what happens depends on how the beans are defined and configured
1. If both starters define a bean with the same name and same type, and bean overriding is disabled (default since Spring Boot 2.x)
```
BeanDefinitionOverrideException:
Invalid bean definition with name 'dataSource'
```

2. Case By Case Senario <br />
**Case 1: Same bean name + same type** ➡ Application startup fails <br />
**Case 2: Same type, different bean names** ➡ Application starts but Injection by type fails unless resolved `NoUniqueBeanDefinitionException` <br />
**Case 3: @ConditionalOnMissingBean** ➡ Bean is created only when it is not available, User-defined bean wins

3. What Spring Boot starters actually do
Spring Boot starters are designed to be safe and non-intrusive:
They almost always use:
- @ConditionalOnMissingBean
- @ConditionalOnClass
- @ConditionalOnProperty
➡ If another starter or your application provides the same bean, auto-configuration backs off

4. Enabling bean overriding (not recommended)
```
spring.main.allow-bean-definition-overriding=true
```
➡ Last bean definition wins
➡ Order depends on classpath loading
⚠️ Dangerous and discouraged in production.

### Q. How to create a custom starter in Spring Boot?

### Q. If service A calls service B, how would you optimize API calls?
1. Reduce the Number of calls
  - API Aggregation (Club multiple requests into a single request)
```
/users/profile
/users/orders

/users/profile-with-orders
```
  - Batch Request
```
POST /users/batch
{
  "ids": [1,2,3,4,5]
}
```
2. Parallel and async call
3. Use message queues if immediate responses are not required.
4. Cache response in Service A
5. Use gRPC instead of REST for interservice communication.
6. HTTP Connection Pooling: Reuse HTTP connections, avoiding TCP + TLS handshake costs.
