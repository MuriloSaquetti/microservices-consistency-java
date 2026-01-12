# Microservices Consistency Java

Este repositório demonstra, em **Java (Spring Boot/Kafka)**, padrões essenciais para **consistência de dados** e **resiliência** em **microservices**:

- **Transactional Outbox + CDC** (evita “duas escritas” entre DB e mensageria).
- **Saga** (transações locais + **compensações**) — orquestração simples.
- **Idempotência** em APIs (`Idempotency-Key`) para *retries* seguros.
- **Produtor Kafka transacional** (base para *exactly-once*).
- **Resilience4j** (timeouts, retry com **exponential backoff + jitter**, circuit breaker).
- **Infra local** com Docker Compose (**Kafka + Zookeeper + Redis**).
- **CI (GitHub Actions)** para build dos módulos.

## 1) Problema complexo: Consistência de dados em microservices

Ao migrar de um monólito para microservices, perdemos a “transação única” ACID entre serviços/bancos. Precisamos lidar com **falhas de rede**, **partições** e o **trade‑off CAP** (consistência vs disponibilidade em presença de partições). Protocolos como 2PC são pouco práticos/bloqueantes; soluções modernas usam **Sagas, Outbox, Idempotência** e garantias de entrega/processamento (ex.: *exactly‑once* em pipelines).  
**Leituras de base:**  
- **Microservices e Sagas**: Martin Fowler / microservices.io; Microsoft Learn (Saga)  
- **CAP**: Gilbert & Lynch (prova teórica), Eric Brewer (12 anos depois)  
- **Outbox**: microservices.io; AWS Prescriptive Guidance; Confluent (overview)  
- **Exactly‑once (Flink/Kafka)**: Confluent (material didático)  
- **Idempotência**: Stripe API (Idempotency-Key)  
- **SRE (confiabilidade)**: *Site Reliability Workbook* (Google)  
- **Falácias de sistemas distribuídos**: Sun/Deutsch (rede não é perfeita)

---

## 2) Dificuldades principais no projeto

1. **Transações distribuídas** sem 2PC (bloqueante) → **Sagas** e **compensações**.  
2. **Dual write** (DB + broker) → **Transactional Outbox** para atomicidade local.  
3. **Duplicidades e ordenação** → **consumidores idempotentes** e ordenação por agregado.  
4. **Garantias de entrega/processamento** → *exactly-once* em pipelines críticos (Kafka/Flink/KStreams).  
5. **Idempotência em POST** com **Idempotency-Key** para *retries* seguros.  
6. **Resiliência operacional** → timeouts, *retry backoff + jitter*, circuit breakers, *bulkheads*, SLO/SLI, *error budgets*.  
7. **Consenso (Raft)** quando **consistência forte** é obrigatória em metadados críticos.

---

## 3) Trechos-chave de código (Java)

### 3.1) Transactional Outbox (Spring Boot + JPA + Kafka)
**Problema:** evitar inconsistência entre “salvar no banco” e “publicar evento”.  
**Solução:** salvar **entidade** e **evento (outbox)** na **mesma transação**; um **dispatcher** publica depois no Kafka.

**Entidades:**
```java
@Entity @Table(name = "orders")
public class Order { @Id @GeneratedValue private Long id; private String customerId; private String status; /*get/set*/ }

@Entity @Table(name = "outbox_events")
public class OutboxEvent { @Id @GeneratedValue private Long id; private String aggregateType, aggregateId, type;
  @Column(columnDefinition = "TEXT") private String payloadJson; private Instant createdAt; /*get/set*/ }

// **Transação única:**
```java
@Service
public class OrderService {
  @Transactional
  public Order createOrder(String customerId) throws JsonProcessingException {
    Order order = new Order(); order.setCustomerId(customerId); order.setStatus("PENDING"); orderRepo.save(order);

    Map<String,Object> payload = Map.of("orderId", order.getId(), "customerId", customerId);
    OutboxEvent evt = new OutboxEvent();
    evt.setAggregateType("Order"); evt.setAggregateId(order.getId().toString());
    evt.setType("OrderCreated"); evt.setPayloadJson(mapper.writeValueAsString(payload)); evt.setCreatedAt(Instant.now());
    outboxRepo.save(evt);
    return order;
  }
}

// **Dispatcher (publica e confirma)**
```java
@Component
public class OutboxDispatcher {
  @Scheduled(fixedDelay = 1000)
  public void dispatch() {
    List<OutboxEvent> batch = outboxRepo.findTop100ByOrderByCreatedAtAsc();
    for (OutboxEvent e : batch) {
      kafkaTemplate.send("orders", e.getAggregateId(), e.getPayloadJson());
      outboxRepo.delete(e); // produção: marcar "SENT" e só deletar após confirmação
    }
  }
}

---

### 3.2) Saga (orquestração com compensações)
**Problema:** 2PC é bloqueante; falhas intermediárias geram estado parcial.
**Solução:** Sagas coordenam transações locais e executam compensações em caso de falha.

@Service
public class CreateOrderSaga {
  public void execute(String customerId, String sku) {
    Order order = orderService.create(customerId);
    try {
      payment.authorize(order.getId(), BigDecimal.valueOf(100));
      inventory.reserve(order.getId(), sku);
      orderService.approve(order.getId());
    } catch (Exception ex) {
      try { inventory.release(order.getId(), sku); } catch (Exception ignored) {}
      try { payment.refund(order.getId()); } catch (Exception ignored) {}
      orderService.reject(order.getId(), ex.getMessage());
      throw ex;
    }
  }
}

---

### 3.3) Idempotency-Key em APIs (Idempotency‑Key)
**Problema:** retries em POST podem criar duplicidades (cobrança/ordem em dobro).
**Solução:** Idempotency‑Key — múltiplas tentativas retornam o mesmo resultado.

@RestController
@RequestMapping("/payments")
public class PaymentController {
  @PostMapping
  public ResponseEntity<?> pay(@RequestHeader(value="Idempotency-Key", required=false) String idemKey,
                               @RequestBody PaymentRequest req) {
    String key = (idemKey != null) ? idemKey : UUID.randomUUID().toString();
    Optional<StoredResponse> cached = store.lookup(key, req.hash());
    if (cached.isPresent()) return ResponseEntity.status(cached.get().getStatus()).body(cached.get().getBody());
    PaymentResult result = svc.process(req);
    store.save(key, req.hash(), result.httpStatus(), result.body());
    return ResponseEntity.status(result.httpStatus()).body(result.body());
  }
}

---

### 3.4) Kafka “exactly‑once” (produtor transacional)

Properties p = new Properties();
p.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
p.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, "true");
p.put(ProducerConfig.ACKS_CONFIG, "all");
p.put(ProducerConfig.RETRIES_CONFIG, Integer.MAX_VALUE);
p.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 5);
p.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "orders-tx-producer-1");

try (KafkaProducer<String,String> prod = new KafkaProducer<>(p, new StringSerializer(), new StringSerializer())) {
  prod.initTransactions();
  prod.beginTransaction();
  try {
    prod.send(new ProducerRecord<>("orders", "order-123", "{\"event\":\"OrderCreated\"}"));
    prod.send(new ProducerRecord<>("orders", "order-123", "{\"event\":\"ReserveCredit\"}"));
    prod.commitTransaction();
  } catch (Exception e) { prod.abortTransaction(); throw e; }
}

---

### 3.5) Resiliência operacional (Resilience4j)

RetryConfig rc = RetryConfig.custom()
  .maxAttempts(5).waitDuration(Duration.ofMillis(200))
  .intervalFunction(IntervalFunction.ofExponentialBackoff(200, 2.0, 0.5)) // jitter 50%
  .build();
Retry retry = Retry.of("payment-retry", rc);

CircuitBreakerConfig cbc = CircuitBreakerConfig.custom()
  .failureRateThreshold(50).waitDurationInOpenState(Duration.ofSeconds(30)).build();
CircuitBreaker cb = CircuitBreaker.of("payment-cb", cbc);

Supplier<Response> s = CircuitBreaker.decorateSupplier(cb, () -> paymentClient.call());
s = Retry.decorateSupplier(retry, s);
Response r = Try.ofSupplier(s).recover(ex -> Response.failed(ex.getMessage())).get();
``
---

### 5) Infra local com Docker Compose

docker compose up -d
# Kafka em localhost:9092, Zookeeper em localhost:2181, Redis em localhost:6379
