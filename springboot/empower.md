### Q. How to dynamically update configuration without restarting microservices in Spring Boot?
To support dynamic configuration updates at runtime, Spring Cloud Config Server can be used along with Spring Boot Actuator. Configuration files are stored centrally in a Git repository and organized per environment and per microservice. A Spring Cloud Config Server is created to serve these configurations, and all microservices are configured to connect to it.

At application startup, microservices fetch configuration properties from the Config Server. If configuration changes are made later, calling the `POST /actuator/refresh` endpoint reloads the property sources. Only beans annotated with @RefreshScope are destroyed and recreated, allowing them to pick up the updated configuration values without restarting the application.Beans not annotated with @RefreshScope will not reflect updated values and continue using the initial configuration loaded at startup.

**(Optional, but recommended for large systems)**: Links multiple microservice instances with a message broker (like RabbitMQ or Kafka). Instead of manually triggering the `POST /actuator/refresh` endpoint for each service instance, you can use the `POST /actuator/busrefresh` endpoint on the Config Server, and the change will be broadcast to all linked clients automatically

```java
@Configuration
@ConfigurationProperties(prefix = "payment")
@RefreshScope
public class PaymentConfig {
    private int timeout;
}
```

When the same property is defined both locally and in Spring Cloud Config Server, the value from Config Server takes precedence because it is treated as a higher-priority external property source and is loaded earlier in the configuration phase.

### Q. How to serialize/deserialize an object in Spring Boot Kafka?

### Q. What are tracer and span IDs?

### Q. How to configure different DBS instances for reading and writing in Spring Boot?
