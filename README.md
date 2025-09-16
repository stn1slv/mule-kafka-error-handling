# Kafka Error Handling POC

A comprehensive proof-of-concept demonstrating sophisticated error handling patterns for Kafka message processing in MuleSoft, featuring retry mechanisms, dead letter queues, and configurable error classification.

## Overview

This project showcases an enterprise-grade approach to handling errors in Kafka message processing workflows. It implements a multi-layered error handling strategy that distinguishes between retryable and non-retryable errors, provides automatic retry mechanisms with configurable limits, and ensures no message loss through proper dead letter queue (DLQ) handling.

## Architecture

### Core Components

1. **`main-flow`** - Primary message consumer from `main-topic`
2. **`retry-topic-processing-flow`** - Scheduled batch processor for retry messages
3. **`processing-logic`** - Business logic sub-flow with simulated error conditions
4. **`retry-logic-flow`** - Error classification and routing logic
5. **`classify-error-flow`** - Configurable error type classification

### Message Flow

```
main-topic → main-flow → processing-logic
                ↓ (on error)
         retry-logic-flow → classify-error-flow
                ↓
    ┌─────────────────┬─────────────────┐
    ▼                 ▼                 ▼
retry-topic      retry-topic        dlq-topic
(retryable)    (max retries)    (non-retryable)
    ↓
retry-topic-processing-flow
(scheduled batch processing)
```

## Key Features

### 1. Error Classification System

The system categorizes errors into two types based on configuration in [`application.yaml`](src/main/resources/application.yaml):

- **Retryable Errors**: Temporary failures that might succeed on retry
  - `HTTP:TIMEOUT`, `HTTP:CONNECTIVITY`, `HTTP:SERVICE_UNAVAILABLE`
  - `KAFKA:TIMEOUT`, `KAFKA:CONNECTIVITY`
  - `REDELIVERY_EXHAUSTED`

- **Non-Retryable Errors**: Permanent failures that will always fail
  - `HTTP:BAD_REQUEST`, `HTTP:UNAUTHORIZED`, `HTTP:FORBIDDEN`
  - `HTTP:NOT_FOUND`, `TRANSFORMATION`, `VALIDATION`
  - `EXPRESSION`, `SECURITY`

### 2. Multi-Level Retry Strategy

1. **Immediate Classification**: Errors are classified upon first occurrence
2. **Retryable Path**: Messages sent to retry topic with retry count headers
3. **Scheduled Processing**: Batch processing of retry messages every minute
4. **Max Retry Enforcement**: Messages exceeding max attempts go to DLQ
5. **Non-Retryable Path**: Direct routing to DLQ for permanent failures

### 3. Comprehensive Message Tracking

Each message carries metadata headers:
- `X-Retry-Count`: Current retry attempt number
- `X-Original-Error`: Truncated error description (300 chars)
- `X-Error-Type`: MuleSoft error type identifier
- `X-Error-Namespace`: Error namespace
- `X-Full-Error-Type`: Complete error type (namespace:identifier)
- `X-Failure-Reason`: Categorized failure reason
- `X-Timestamp`: Processing timestamp

## Configuration

### Kafka Settings

```yaml
kafka:
  bootstrap-servers: "127.0.0.1:9094"
  consumer:
    main:
      group-id: "consumer-group-1"
      topic: "main-topic"
    retry:
      group-id: "consumer-group-2"  
      topic: "retry-topic"
      poll-timeout: "10"
      recordLimit: "1"
  producer:
    retry-topic: "retry-topic"
    dlq-topic: "dlq-topic"
```

### Retry Configuration

```yaml
retry:
  max-attempts: "3"
  scheduler:
    frequency: "1" # minutes
  retryable-errors: "HTTP:TIMEOUT,HTTP:CONNECTIVITY,HTTP:SERVICE_UNAVAILABLE"
  non-retryable-errors: "HTTP:BAD_REQUEST,HTTP:UNAUTHORIZED,TRANSFORMATION"
```

### Processing Settings

```yaml
processing:
  batch-size: "1000"
  error-country: "XYZ"  # Triggers error condition for testing
```

## Technical Implementation Details

### Tricky Points and Solutions

#### 1. **Avoiding Infinite Retry Loops**

**Challenge**: Preventing newly produced retry messages from being immediately consumed in the same batch.

**Solution**: Timestamp-based filtering in `retry-topic-processing-flow`:
```xml
<when expression="#[attributes.creationTimestamp >= vars.executionTimestamp]">
    <raise-error type="ERROR-HANDLING:MESSAGE-SHOULD-BE-SKIPPED" />
</when>
```

**Why it works**: Messages created during or after the current execution are skipped, ensuring only previously failed messages are processed.

#### 2. **Dual Error Classification Logic**

**Challenge**: Supporting both full error types (`NAMESPACE:TYPE`) and simple types (`TYPE`) for maximum flexibility.

**Solution**: Two-stage DataWeave evaluation:
```dataweave
isNonRetryable: (vars.nonRetryableErrors contains vars.fullErrorType) 
                or (vars.nonRetryableErrors contains vars.errorType)
```

**Why it's needed**: Different error conditions may be referenced by simple type or full namespace, providing configuration flexibility.

#### 3. **Batch Processing with Early Termination**

**Challenge**: Efficiently processing retry messages while handling empty topics gracefully.

**Solution**: Controlled iteration with state tracking:
```xml
<foreach doc:name="For Each Consume Attempt">
    <choice doc:name="Check if topic is empty">
        <when expression="#[vars.topicEmpty == false]">
            <!-- Process message -->
        </when>
    </choice>
</foreach>
```

**Benefits**: Avoids unnecessary polling attempts once topic is empty, improving performance.

#### 4. **Error Header Truncation**

**Challenge**: Kafka header size limitations with large error messages.

**Solution**: Intelligent truncation with information preservation:
```dataweave
"X-Original-Error": substring(toString(error.description ++ "\n" ++ 
                   write(error.errorMessage.payload, "application/json")), 0, 300)
```

**Why 300 chars**: Balances information retention with header size constraints.

#### 5. **Separate Consumer Groups**

**Challenge**: Preventing main topic consumption from interfering with retry processing.

**Solution**: Dedicated consumer groups:
- `consumer-group-1`: Main topic processing
- `consumer-group-2`: Retry topic processing

**Benefits**: Independent offset management and processing isolation.

## Error Simulation

The application includes built-in error simulation for testing:

1. **Country-based Error Trigger**: Messages with `address.country = "XYZ"` trigger HTTP calls
2. **Configurable HTTP Endpoints**: Switch between retryable (503) and non-retryable (400) errors
3. **Mock Endpoints**: Pre-configured mockbin.io endpoints for different error scenarios

## Running the Application

### Prerequisites

- Mule Runtime 4.9+
- Docker
- Java 17

### Setup

1. **Start Kafka with Docker**:
   ```bash
   docker run --name kafka \
     --restart always \
     -p 9092:9092 \
     -p 9094:9094 \
     -e KAFKA_CFG_NODE_ID=0 \
     -e KAFKA_CFG_PROCESS_ROLES=broker,controller \
     -e KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093,EXTERNAL://:9094 \
     -e KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092,EXTERNAL://localhost:9094 \
     -e KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,EXTERNAL:PLAINTEXT,PLAINTEXT:PLAINTEXT \
     -e KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=0@kafka:9093 \
     -e KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER \
     bitnami/kafka:4.0
   ```

2. **Create Topics**:
   ```bash
   # Create topics using Docker exec
   docker exec -it kafka kafka-topics.sh --create --topic main-topic --bootstrap-server localhost:9094
   docker exec -it kafka kafka-topics.sh --create --topic retry-topic --bootstrap-server localhost:9094
   docker exec -it kafka kafka-topics.sh --create --topic dlq-topic --bootstrap-server localhost:9094
   ```

3. **Configure Bootstrap Server**: Update `kafka.bootstrap-servers` in `application.yaml`

4. **Deploy Application**: Run in Anypoint Studio or deploy to runtime

### Testing

1. **Send Test Message**:
   ```bash
   echo "$(date +%s%N):{\"id\": 1, \"name\": \"Test\", \"address\": {\"country\": \"XYZ\"}}" | \
   docker exec -i kafka kafka-console-producer.sh --topic main-topic --bootstrap-server localhost:9094 \
     --property parse.key=true --property key.separator=:
   ```

2. **Monitor Topics**:
   ```bash
   # Watch retry topic (clean formatted output with key)
   docker exec -it kafka kafka-console-consumer.sh --topic retry-topic --from-beginning --bootstrap-server localhost:9094 \
     --property print.headers=true --property print.timestamp=true --property print.key=true \
     --property key.separator=" | KEY: " --property headers.separator=" | "
   
   # Watch DLQ topic (clean formatted output with key)
   docker exec -it kafka kafka-console-consumer.sh --topic dlq-topic --from-beginning --bootstrap-server localhost:9094 \
     --property print.headers=true --property print.timestamp=true --property print.key=true \
     --property key.separator=" | KEY: " --property headers.separator=" | "
   
   # Watch retry topic (headers and key only, no payload)
   docker exec -it kafka kafka-console-consumer.sh --topic retry-topic --from-beginning --bootstrap-server localhost:9094 \
     --property print.headers=true --property print.value=false --property print.key=true \
     --property print.timestamp=true --property key.separator=" | KEY: " --property headers.separator=$'\n  '
   
   # Watch DLQ topic (headers and key only, no payload)
   docker exec -it kafka kafka-console-consumer.sh --topic dlq-topic --from-beginning --bootstrap-server localhost:9094 \
     --property print.headers=true --property print.value=false --property print.key=true \
     --property print.timestamp=true --property key.separator=" | KEY: " --property headers.separator=$'\n  '
   ```

### Header Monitoring

The commands above show your custom error tracking headers:
- `X-Retry-Count`: Current retry attempt number
- `X-Error-Type`: MuleSoft error type identifier  
- `X-Failure-Reason`: Categorized failure reason (RETRYABLE_ERROR, NON_RETRYABLE_ERROR, MAX_RETRIES_EXCEEDED)
- `X-Timestamp`: Processing timestamp
- `X-Original-Error`: Truncated error description

Example output:
```
CreateTime:1758033614894 | KEY: 1758033613841880000 | X-Retry-Count:1 | X-Error-Type:SERVICE_UNAVAILABLE | X-Failure-Reason:RETRYABLE_ERROR
{"id": 1, "name": "Test", "address": {"country": "XYZ"}}
```

The format shows: `CreateTime:<timestamp> | KEY: <message-key> | <headers> | <headers>...`

## Monitoring and Observability

### Log Levels

- **INFO**: Normal processing flow and retry statistics
- **WARN**: DLQ routing and max retry exceeded scenarios  
- **ERROR**: Processing failures and error details
- **DEBUG**: Detailed error classification information

### Key Metrics to Monitor

1. **Processing Counters**: Successfully processed, failed, DLQ messages
2. **Retry Patterns**: Retry count distribution and success rates
3. **Error Classifications**: Retryable vs non-retryable error ratios
4. **Batch Performance**: Messages processed per scheduler execution

## Best Practices Implemented

1. **Idempotent Processing**: Retry logic handles duplicate processing gracefully
2. **Message Preservation**: No message loss through comprehensive error handling
3. **Configurable Behavior**: All retry and error classification rules are externalized
4. **Resource Management**: Batch processing prevents resource exhaustion
5. **Observability**: Comprehensive logging and metric collection
6. **Graceful Degradation**: System continues processing even with partial failures

## Extension Points

1. **Custom Error Classification**: Add new error types to configuration
2. **Dynamic Retry Intervals**: Implement exponential backoff in scheduler
3. **Circuit Breaker**: Add circuit breaker pattern for downstream services
4. **Metrics Integration**: Connect to monitoring systems (Prometheus, Grafana)
5. **Alert Integration**: Add notifications for DLQ thresholds

## Dependencies

- **Mule Runtime**: 4.9.9
- **Kafka Connector**: 4.11.0
- **HTTP Connector**: 1.10.5
- **MUnit**: 3.4.0 (for testing)

## Contributing

When modifying the error handling logic:

1. Test both retryable and non-retryable error paths
2. Verify message headers are preserved correctly
3. Ensure batch processing handles edge cases
4. Update configuration documentation for new error types
5. Add appropriate logging for observability

## License

This is a proof-of-concept project for educational and demonstration purposes.