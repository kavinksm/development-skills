# Spring Boot Messaging Reference

## Kafka — Producer

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class OrderEventProducer {

    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;

    @Value("${app.kafka.topic.orders}")
    private String ordersTopic;

    public void publishOrderCreated(OrderEvent event) {
        var record = new ProducerRecord<>(ordersTopic, event.orderId().toString(), event);
        kafkaTemplate.send(record)
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    log.error("Failed to publish order event: {}", event.orderId(), ex);
                } else {
                    log.debug("Published order event: {} to partition {}",
                        event.orderId(),
                        result.getRecordMetadata().partition());
                }
            });
    }

    public void publishOrderCreatedSync(OrderEvent event) throws ExecutionException, InterruptedException {
        kafkaTemplate.send(ordersTopic, event.orderId().toString(), event).get();
    }
}
```

---

## Kafka — Consumer with DLQ

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class OrderEventConsumer {

    private final OrderService orderService;

    @KafkaListener(
        topics = "${app.kafka.topic.orders}",
        groupId = "${app.kafka.consumer.group-id}",
        containerFactory = "kafkaListenerContainerFactory"
    )
    public void handleOrderEvent(@Payload OrderEvent event,
                                  @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
                                  @Header(KafkaHeaders.OFFSET) long offset,
                                  Acknowledgment ack) {
        log.info("Received order event: {} from partition {} offset {}", event.orderId(), partition, offset);
        try {
            orderService.processOrderEvent(event);
            ack.acknowledge();
        } catch (Exception e) {
            log.error("Failed to process order event: {}", event.orderId(), e);
            // DLQ publishing handled by DeadLetterPublishingRecoverer in config
            throw e;
        }
    }

    // Batch listener
    @KafkaListener(
        topics = "${app.kafka.topic.notifications}",
        groupId = "${app.kafka.consumer.group-id}-batch",
        containerFactory = "batchKafkaListenerContainerFactory"
    )
    public void handleBatch(List<OrderEvent> events, Acknowledgment ack) {
        log.info("Processing batch of {} events", events.size());
        orderService.processBatch(events);
        ack.acknowledge();
    }
}
```

---

## Kafka Configuration

```java
@Configuration
@EnableKafka
public class KafkaConfig {

    @Value("${spring.kafka.bootstrap-servers}")
    private String bootstrapServers;

    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        config.put(ProducerConfig.ACKS_CONFIG, "all");
        config.put(ProducerConfig.RETRIES_CONFIG, 3);
        config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        return new DefaultKafkaProducerFactory<>(config);
    }

    @Bean
    public KafkaTemplate<String, Object> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }

    @Bean
    public ConsumerFactory<String, Object> consumerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        config.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        config.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);
        config.put(JsonDeserializer.TRUSTED_PACKAGES, "com.example.app.event");
        config.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        config.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        return new DefaultKafkaConsumerFactory<>(config);
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> kafkaListenerContainerFactory(
            ConsumerFactory<String, Object> consumerFactory,
            KafkaTemplate<String, Object> kafkaTemplate) {
        var factory = new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL);
        factory.setConcurrency(3);

        // Dead Letter Queue
        factory.setCommonErrorHandler(new DefaultErrorHandler(
            new DeadLetterPublishingRecoverer(kafkaTemplate,
                (record, ex) -> new TopicPartition(record.topic() + ".DLT", record.partition())),
            new FixedBackOff(1000L, 3)
        ));
        return factory;
    }

    @Bean
    public NewTopic ordersTopic(@Value("${app.kafka.topic.orders}") String topic) {
        return TopicBuilder.name(topic)
            .partitions(6)
            .replicas(1)
            .compact()
            .build();
    }
}
```

---

## Kafka application.yml

```yaml
spring:
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all
      retries: 3
    consumer:
      group-id: app-consumer-group
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      enable-auto-commit: false
      properties:
        spring.json.trusted.packages: com.example.app.event

app:
  kafka:
    topic:
      orders: orders-events
      notifications: notification-events
    consumer:
      group-id: app-consumer-group
```

---

## Event DTO

```java
public record OrderEvent(
    UUID orderId,
    String userId,
    OrderStatus status,
    BigDecimal total,
    LocalDateTime occurredAt
) {
    public static OrderEvent created(Order order) {
        return new OrderEvent(
            order.getId(),
            order.getUserId(),
            OrderStatus.CREATED,
            order.getTotal(),
            LocalDateTime.now()
        );
    }
}
```

---

## RabbitMQ — Producer

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class NotificationProducer {

    private final RabbitTemplate rabbitTemplate;

    @Value("${app.rabbitmq.exchange.notifications}")
    private String exchange;

    public void send(NotificationEvent event) {
        rabbitTemplate.convertAndSend(exchange, event.routingKey(), event);
        log.debug("Sent notification event: {}", event.id());
    }
}
```

---

## RabbitMQ — Consumer

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class NotificationConsumer {

    private final NotificationService notificationService;

    @RabbitListener(queues = "${app.rabbitmq.queue.email}")
    public void handleEmailNotification(NotificationEvent event) {
        log.info("Processing email notification: {}", event.id());
        notificationService.sendEmail(event);
    }

    @RabbitListener(queues = "${app.rabbitmq.queue.sms}")
    public void handleSmsNotification(NotificationEvent event) {
        notificationService.sendSms(event);
    }
}
```

---

## RabbitMQ Configuration

```java
@Configuration
@EnableRabbit
public class RabbitMQConfig {

    public static final String NOTIFICATIONS_EXCHANGE = "notifications.exchange";
    public static final String EMAIL_QUEUE = "notifications.email";
    public static final String SMS_QUEUE = "notifications.sms";
    public static final String DEAD_LETTER_EXCHANGE = "notifications.dlx";
    public static final String DEAD_LETTER_QUEUE = "notifications.dead-letter";

    @Bean
    public TopicExchange notificationsExchange() {
        return new TopicExchange(NOTIFICATIONS_EXCHANGE, true, false);
    }

    @Bean
    public Queue emailQueue() {
        return QueueBuilder.durable(EMAIL_QUEUE)
            .withArgument("x-dead-letter-exchange", DEAD_LETTER_EXCHANGE)
            .withArgument("x-message-ttl", 60000)
            .build();
    }

    @Bean
    public Queue smsQueue() {
        return QueueBuilder.durable(SMS_QUEUE)
            .withArgument("x-dead-letter-exchange", DEAD_LETTER_EXCHANGE)
            .build();
    }

    @Bean
    public Binding emailBinding() {
        return BindingBuilder.bind(emailQueue()).to(notificationsExchange()).with("notification.email.#");
    }

    @Bean
    public Binding smsBinding() {
        return BindingBuilder.bind(smsQueue()).to(notificationsExchange()).with("notification.sms.#");
    }

    @Bean
    public DirectExchange deadLetterExchange() {
        return new DirectExchange(DEAD_LETTER_EXCHANGE);
    }

    @Bean
    public Queue deadLetterQueue() {
        return QueueBuilder.durable(DEAD_LETTER_QUEUE).build();
    }

    @Bean
    public Binding deadLetterBinding() {
        return BindingBuilder.bind(deadLetterQueue())
            .to(deadLetterExchange()).with(DEAD_LETTER_QUEUE);
    }

    @Bean
    public Jackson2JsonMessageConverter messageConverter(ObjectMapper objectMapper) {
        return new Jackson2JsonMessageConverter(objectMapper);
    }

    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory,
                                          Jackson2JsonMessageConverter messageConverter) {
        var template = new RabbitTemplate(connectionFactory);
        template.setMessageConverter(messageConverter);
        template.setConfirmCallback((correlationData, ack, cause) -> {
            if (!ack) {
                log.warn("Message not confirmed: {}", cause);
            }
        });
        return template;
    }
}
```

---

## Kafka Testing

```java
@SpringBootTest
@EmbeddedKafka(partitions = 1, brokerProperties = {
    "listeners=PLAINTEXT://localhost:${kafka.port:9092}",
    "port=${kafka.port:9092}"
})
@TestPropertySource(properties = "spring.kafka.bootstrap-servers=${spring.embedded.kafka.brokers}")
class OrderEventProducerTest {

    @Autowired
    private OrderEventProducer producer;

    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;

    @Value("${app.kafka.topic.orders}")
    private String topic;

    @Test
    void publishOrderCreated_SendsToKafka() throws Exception {
        var consumer = buildConsumer();
        consumer.subscribe(Collections.singletonList(topic));

        var event = new OrderEvent(UUID.randomUUID(), "user-1", OrderStatus.CREATED,
            BigDecimal.TEN, LocalDateTime.now());
        producer.publishOrderCreated(event);

        var records = consumer.poll(Duration.ofSeconds(5));
        assertThat(records.count()).isEqualTo(1);
        consumer.close();
    }

    private KafkaConsumer<String, Object> buildConsumer() {
        var props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "test-consumer");
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);
        return new KafkaConsumer<>(props);
    }
}
```
