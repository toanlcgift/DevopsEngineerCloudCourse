# Lab 06 – Messaging and Event-Driven Architecture

Create a local SQS-compatible queue, publish and consume an order event, and observe visibility timeout and retry behavior. The emulator does not guarantee AWS delivery semantics. Complete the [WSL setup](../README.md#start-here).

## 1. Start Floci and configure a local-only CLI

```bash
docker run -d --name lab06-floci -p 127.0.0.1:4566:4566 floci/floci:latest
source ~/devops-labs/.venv/bin/activate
python -m pip install awscli
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=us-east-1
export SQS_ENDPOINT=http://127.0.0.1:4566
```

## 2. Create queues and publish an order event

```bash
aws --endpoint-url "$SQS_ENDPOINT" sqs create-queue \
  --queue-name lab06-orders-dlq
aws --endpoint-url "$SQS_ENDPOINT" sqs create-queue \
  --queue-name lab06-orders --attributes VisibilityTimeout=5
DLQ_URL=$(aws --endpoint-url "$SQS_ENDPOINT" sqs get-queue-url \
  --queue-name lab06-orders-dlq --query QueueUrl --output text)
QUEUE_URL=$(aws --endpoint-url "$SQS_ENDPOINT" sqs get-queue-url \
  --queue-name lab06-orders --query QueueUrl --output text)
DLQ_ARN=$(aws --endpoint-url "$SQS_ENDPOINT" sqs get-queue-attributes \
  --queue-url "$DLQ_URL" --attribute-names QueueArn \
  --query 'Attributes.QueueArn' --output text)
aws --endpoint-url "$SQS_ENDPOINT" sqs set-queue-attributes \
  --queue-url "$QUEUE_URL" \
  --attributes "$(jq -nc --arg arn "$DLQ_ARN" \
    '{RedrivePolicy: ({deadLetterTargetArn:$arn,maxReceiveCount:"2"}|tojson)}')"
aws --endpoint-url "$SQS_ENDPOINT" sqs send-message \
  --queue-url "$QUEUE_URL" \
  --message-body '{"event":"OrderCreated","orderId":"order-1001","total":42.00}'
```

## 3. Receive, retry, acknowledge, and use the dead-letter queue

The first message is received and acknowledged below. Then a poison message is received twice without deletion. After each visibility timeout, it is retried and ultimately moved to the DLQ. The simulator may not implement redrive; if setting the attribute is unsupported, explain the policy that a managed queue would apply and skip the poison-message part.

```bash
MESSAGE=$(aws --endpoint-url "$SQS_ENDPOINT" sqs receive-message \
  --queue-url "$QUEUE_URL" --wait-time-seconds 1 --query 'Messages[0]' --output json)
printf '%s\n' "$MESSAGE"
sleep 6
MESSAGE=$(aws --endpoint-url "$SQS_ENDPOINT" sqs receive-message \
  --queue-url "$QUEUE_URL" --wait-time-seconds 1 --query 'Messages[0]' --output json)
printf '%s\n' "$MESSAGE"
RECEIPT=$(printf '%s\n' "$MESSAGE" | jq -r '.ReceiptHandle // empty')
if [ -n "$RECEIPT" ]; then
  aws --endpoint-url "$SQS_ENDPOINT" sqs delete-message \
    --queue-url "$QUEUE_URL" --receipt-handle "$RECEIPT"
fi
aws --endpoint-url "$SQS_ENDPOINT" sqs send-message \
  --queue-url "$QUEUE_URL" --message-body '{"event":"OrderCreated","orderId":"poison"}'
for attempt in 1 2; do
  MESSAGE=$(aws --endpoint-url "$SQS_ENDPOINT" sqs receive-message \
    --queue-url "$QUEUE_URL" --wait-time-seconds 1 --query 'Messages[0]' --output json)
  printf 'Failed processing attempt %s: %s\n' "$attempt" "$MESSAGE"
  sleep 6
done
aws --endpoint-url "$SQS_ENDPOINT" sqs receive-message \
  --queue-url "$DLQ_URL" --wait-time-seconds 1
```

Discuss duplicate delivery, idempotent handlers, visibility extension, retry limits, and dead-letter queues. A production DLQ redrive policy should be configured and tested in the provider environment; support differs between emulators.

## 4. Clean up

```bash
docker rm -f lab06-floci
unset SQS_ENDPOINT QUEUE_URL AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_DEFAULT_REGION
unset DLQ_URL DLQ_ARN MESSAGE RECEIPT
```
