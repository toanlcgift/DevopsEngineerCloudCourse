# Lab 05 – NoSQL Database Management

Use DynamoDB Local with the AWS CLI to create a table, write/read/update/delete an item, and query a partition key. This is a local database, not the managed DynamoDB service. Complete the [WSL setup](../README.md#start-here).

## 1. Start DynamoDB Local

```bash
docker run -d --name lab05-dynamodb \
  -p 127.0.0.1:8000:8000 \
  amazon/dynamodb-local:latest -jar DynamoDBLocal.jar -sharedDb -inMemory
source ~/devops-labs/.venv/bin/activate
python -m pip install awscli
export AWS_ACCESS_KEY_ID=dummy
export AWS_SECRET_ACCESS_KEY=dummy
export AWS_DEFAULT_REGION=us-east-1
export DDB=http://127.0.0.1:8000
```

## 2. Create and inspect a table

```bash
aws --endpoint-url "$DDB" dynamodb create-table \
  --table-name LabOrders \
  --attribute-definitions AttributeName=customer_id,AttributeType=S AttributeName=order_id,AttributeType=S \
  --key-schema AttributeName=customer_id,KeyType=HASH AttributeName=order_id,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST
aws --endpoint-url "$DDB" dynamodb describe-table --table-name LabOrders
```

## 3. Perform CRUD operations

```bash
aws --endpoint-url "$DDB" dynamodb put-item --table-name LabOrders \
  --item '{"customer_id":{"S":"customer-101"},"order_id":{"S":"order-1"},"status":{"S":"created"},"total":{"N":"24.50"}}'
aws --endpoint-url "$DDB" dynamodb get-item --table-name LabOrders \
  --key '{"customer_id":{"S":"customer-101"},"order_id":{"S":"order-1"}}'
aws --endpoint-url "$DDB" dynamodb update-item --table-name LabOrders \
  --key '{"customer_id":{"S":"customer-101"},"order_id":{"S":"order-1"}}' \
  --update-expression 'SET #s = :s' \
  --expression-attribute-names '{"#s":"status"}' \
  --expression-attribute-values '{":s":{"S":"shipped"}}'
aws --endpoint-url "$DDB" dynamodb query --table-name LabOrders \
  --key-condition-expression 'customer_id = :customer' \
  --expression-attribute-values '{":customer":{"S":"customer-101"}}'
aws --endpoint-url "$DDB" dynamodb delete-item --table-name LabOrders \
  --key '{"customer_id":{"S":"customer-101"},"order_id":{"S":"order-1"}}'
```

Explain why the partition key supports the access pattern and what happens when a query does not specify it. Compare DynamoDB's key model with Cosmos DB partition keys and Firestore document paths/indexes; their APIs and consistency features are not interchangeable.

## 4. Stop the lab

```bash
docker rm -f lab05-dynamodb
unset DDB AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_DEFAULT_REGION
```
