# ⚙️ Spring Boot Microservices — Kafka, Testcontainers & Distributed Tracing

Architecture microservices complète avec Spring Boot 3 : Order Service,
Inventory Service et Notification Service communiquant via Kafka.
Distributed tracing avec Zipkin pour suivre les requêtes à travers les
services, et Testcontainers pour des tests d'intégration réalistes.

---

## Architecture et flux de données

```
  POST /api/order
          │
          ▼
  ┌─────────────────┐
  │  Order Service  │──────── REST (Feign) ──────► Inventory Service
  │  (Port 8081)    │◄─────── "IN_STOCK: true" ────
  │                 │
  │  Saves order    │
  │  to PostgreSQL  │──────── Kafka Topic ─────────► Notification Service
  └─────────────────┘   "order-placed"              │ (Port 8083)
                                                    │ Sends email
                                                    │ Logs to MongoDB
                                                    ▼

  Chaque requête est tracée de bout en bout avec Zipkin :
  Order Service → Inventory Service → Notification Service
  → dashboard Zipkin : temps par service, erreurs, dépendances
```

---

## Order Service — placement de commande

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class OrderService {

    private final OrderRepository     orderRepository;
    private final InventoryServiceClient inventoryClient;
    private final KafkaTemplate<String, OrderPlacedEvent> kafkaTemplate;

    @Transactional
    public OrderResponse placeOrder(OrderRequest orderRequest) {

        // 1. Vérifier le stock via appel synchrone Feign
        InventoryResponse[] inventoryResponses = inventoryClient
            .isInStock(
                orderRequest.getOrderLineItemsList()
                    .stream()
                    .map(OrderLineItemsDto::getSkuCode)
                    .collect(Collectors.toList())
            );

        boolean allProductsInStock = Arrays.stream(inventoryResponses)
            .allMatch(InventoryResponse::isInStock);

        if (!allProductsInStock) {
            throw new IllegalArgumentException(
                "Product is not in stock, please try again later"
            );
        }

        // 2. Construire et sauvegarder la commande
        Order order = new Order();
        order.setOrderNumber(UUID.randomUUID().toString());
        order.setOrderLineItemsList(
            orderRequest.getOrderLineItemsList()
                .stream()
                .map(this::mapToDto)
                .collect(Collectors.toList())
        );
        orderRepository.save(order);

        // 3. Publier l'événement Kafka (async — ne bloque pas la réponse)
        OrderPlacedEvent event = new OrderPlacedEvent(order.getOrderNumber());
        log.info("Sending OrderPlacedEvent: {}", event);
        kafkaTemplate.send("notificationTopic", event);

        return new OrderResponse("Order placed successfully");
    }
}
```

---

## Testcontainers — tests d'intégration réalistes

```java
// Tests qui démarrent de vrais containers Docker pendant les tests
// Plus fiable qu'un H2 in-memory — on teste avec la vraie BDD

@SpringBootTest
@Testcontainers
class OrderServiceApplicationTests {

    // Démarre un vrai container PostgreSQL pour les tests
    @Container
    static PostgreSQLContainer<?> postgreSQLContainer =
        new PostgreSQLContainer<>("postgres:15-alpine")
            .withDatabaseName("order-service")
            .withUsername("test")
            .withPassword("test");

    // Démarre un vrai container Kafka
    @Container
    static KafkaContainer kafkaContainer =
        new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.4.0"));

    @DynamicPropertySource
    static void setProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url",
            postgreSQLContainer::getJdbcUrl);
        registry.add("spring.datasource.username",
            postgreSQLContainer::getUsername);
        registry.add("spring.datasource.password",
            postgreSQLContainer::getPassword);
        registry.add("spring.kafka.bootstrap-servers",
            kafkaContainer::getBootstrapServers);
    }

    @Autowired MockMvc mockMvc;
    @Autowired OrderRepository orderRepository;
    @Autowired ObjectMapper objectMapper;

    @Test
    void shouldPlaceOrder() throws Exception {
        OrderRequest orderRequest = getOrderRequest();
        String       requestBody  = objectMapper.writeValueAsString(orderRequest);

        // Mock Inventory Service (Wiremock) — répond "in stock"
        stubFor(get(urlEqualTo("/api/inventory?skuCode=iphone-13"))
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("[{\"skuCode\":\"iphone-13\",\"isInStock\":true}]")));

        mockMvc.perform(post("/api/order")
                .contentType(MediaType.APPLICATION_JSON)
                .content(requestBody))
            .andExpect(status().isCreated());

        // Vérifier que la commande est bien en base
        assertEquals(1, orderRepository.findAll().size());
    }

    private OrderRequest getOrderRequest() {
        return OrderRequest.builder()
            .orderLineItemsList(List.of(
                OrderLineItemsDto.builder()
                    .skuCode("iphone-13")
                    .price(new BigDecimal("1200"))
                    .quantity(1)
                    .build()
            ))
            .build();
    }
}
```

---

## Distributed Tracing avec Zipkin

```yaml
# application.yml — chaque service
spring:
  zipkin:
    base-url: http://zipkin:9411
  sleuth:
    sampler:
      probability: 1.0   # tracer 100% des requêtes (0.1 en prod)
```

```
  Dashboard Zipkin — trace d'une commande :
  ─────────────────────────────────────────
  POST /api/order                 150ms total
  ├── order-service               120ms
  │   ├── DB save (PostgreSQL)     45ms
  │   ├── Feign → inventory         35ms
  │   │   └── inventory-service     30ms
  │   └── Kafka publish             5ms
  └── notification-service (async)  25ms
      └── email send                20ms

  → Immédiatement visible : le DB save est le goulot d'étranglement
  → Action : ajouter un index sur order_number
```

---

## Docker Compose — stack complète en une commande

```yaml
# docker-compose.yml
services:
  order-service:
    image: order-service:latest
    ports: ["8081:8081"]
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/order_service
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      ZIPKIN_BASE_URL: http://zipkin:9411
    depends_on: [postgres, kafka, zipkin]

  inventory-service:
    image: inventory-service:latest
    ports: ["8082:8082"]
    depends_on: [postgres]

  notification-service:
    image: notification-service:latest
    ports: ["8083:8083"]
    depends_on: [kafka, mongodb]

  kafka:
    image: confluentinc/cp-kafka:7.4.0
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092

  zipkin:
    image: openzipkin/zipkin
    ports: ["9411:9411"]

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: secret

  mongodb:
    image: mongo:6.0

# Démarrer tout :
# docker-compose up -d
# → Order Service : http://localhost:8081
# → Zipkin UI    : http://localhost:9411
```

---

## Ce que j'ai appris

**Testcontainers** a changé ma façon d'écrire les tests d'intégration.
Avant, j'utilisais H2 in-memory — rapide, mais les tests passaient sur
ma machine et échouaient en CI à cause des différences de comportement
SQL entre H2 et PostgreSQL. Avec Testcontainers, on teste exactement
ce qui tourne en production. Le test est plus lent (30s au lieu de 5s)
mais fiable — et en CI/CD, la fiabilité prime sur la vitesse.

---

*Projet réalisé dans le cadre de ma formation ingénieur — ENSET Mohammedia*
*Par **Abderrahmane Elouafi** · [LinkedIn](https://www.linkedin.com/in/abderrahmane-elouafi-43226736b/) · [Portfolio](https://my-first-porfolio-six.vercel.app/)*
