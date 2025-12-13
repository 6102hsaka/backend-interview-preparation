### What are the pitfall of using prototype bean with singleton bean?

#### 1. Prototype Bean Lifecycle vs Singleton Bean Lifecycle
- **Singleton bean:** created once per Spring container, lives throughout the application context.
- **Prototype bean:** created every time it is requested from the container.

**Pitfall:** If a singleton bean injects a prototype bean directly (via @Autowired), Spring injects a single instance of the prototype at the time of singleton creation. This means the singleton reuses the same prototype instance, defeating the purpose of the prototype scope.
```java
@Component
public class SingletonBean {
    @Autowired
    private PrototypeBean prototypeBean; // injected only once at singleton creation

    public void doSomething() {
        prototypeBean.action();
    }
}
```
Here, even though PrototypeBean is a prototype, every call to doSomething() will use the same instance.

#### 2. Statefulness Issues
If the prototype bean is stateful, this problem becomes worse. Singleton will hold a reference to the same prototype bean, leading to unexpected behavior:
- Shared state between requests that are meant to be independent.
- Thread-safety issues if accessed concurrently.

#### 3. Dependency Resolution Timing
- Prototype beans are resolved at injection time if injected normally.
- Singleton beans are created once, so the prototype bean is resolved only once at singleton creation.
If you expect a fresh prototype bean each time a method is called, direct injection does not work.

#### ✅ How to Solve / Avoid These Pitfalls
#### Use ObjectFactory or Provider
```java
@Component
public class SingletonBean {
    @Autowired
    private ObjectProvider<PrototypeBean> prototypeBeanProvider;

    public void doSomething() {
        PrototypeBean prototypeBean = prototypeBeanProvider.getObject();
        prototypeBean.action();
    }
}
```
This ensures a new instance of the prototype bean is used each time.

#### Method Injection (`@Lookup`)
```java
@Component
public abstract class SingletonBean {
    public void doSomething() {
        getPrototypeBean().action();
    }

    @Lookup
    protected abstract PrototypeBean getPrototypeBean();
}
```
Spring overrides getPrototypeBean() to return a fresh prototype bean every call.

Summary Table of Pitfalls
| Pitfall |	Description |
| -- | -- |
| Singleton holds prototype reference |	Prototype bean injected once, losing prototype behavior |
| Shared state issues |	Stateful prototype bean behaves like singleton |
| Thread-safety problems | Multiple threads share the same prototype instance |
| Unexpected lifecycle |	Prototype destruction callbacks are not called automatically |
| Manual bean lookup |	Can solve the problem but increases coupling |
