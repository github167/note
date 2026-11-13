1. EC2
```
docker run --rm --name floci \
  -p 4566:4566 \
  -v ./abc:/app/data \
  -v /var/run/docker.sock:/var/run/docker.sock \
  floci/floci:latest

```

2. ECS
```
cat << EOF > task-def.json
{
  "family": "nginx-task",
  "containerDefinitions": [
    {
      "name": "web",
      "image": "nginx:latest",
      "cpu": 256,
      "memory": 512,
      "portMappings": [
        {
          "containerPort": 80,
          "hostPort": 8080
        }
      ]
    }
  ]
}  
EOF

aws ecs create-cluster --cluster-name my-local-cluster > /dev/null

aws ecs register-task-definition --cli-input-json file://task-def.json > /dev/null

aws ecs run-task --cluster my-local-cluster --task-definition nginx-task > /dev/null

curl http://localhost:8080

```
```
aws ecs stop-task \
    --cluster my-local-cluster \
    --task $(aws ecs list-tasks --cluster my-local-cluster --family nginx-task --query "taskArns[0]" --output text) > /dev/null

(aws ecs deregister-task-definition --task-definition nginx-task:1 && aws ecs delete-task-definitions --task-definitions nginx-task:1) > /dev/null

aws ecs delete-cluster --cluster my-local-cluster > /dev/null

# aws ecs list-tasks --cluster my-local-cluster --family nginx-task

```

3. Create IAM
```
cat << EOF > trust-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "://amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF


aws iam create-role --role-name MyLambdaExecutionRole --assume-role-policy-document file://trust-policy.json > /dev/null

```
```
aws iam delete-role --role-name MyLambdaExecutionRole > /dev/null
aws iam list-roles --query "Roles[*].RoleName" --output text

```

4. Serverless (Lambda)
```
cat << EOF >> lambda_function.py
import json

def lambda_handler(event, context):
    # 接收傳入的參數（預設從 event 讀取）
    name = event.get('name', '世界')
    message = f"你好，{name}！這是來自 AWS CLI 部署的 Lambda。"

    return {
        'statusCode': 200,
        'message': message
    }

EOF

rm -f function.zip && zip function.zip lambda_function.py

aws lambda create-function \
    --function-name MyLocalLambda \
    --runtime python3.12 \
    --zip-file fileb://function.zip \
    --handler lambda_function.lambda_handler \
    --role arn:aws:iam::000000000000:role/MyLambdaExecutionRole  > /dev/null



aws lambda invoke \
    --function-name MyLocalLambda \
	--payload '{"name": "UI 測試員"}' \
	--cli-binary-format raw-in-base64-out \
    response.json
	
cat response.json

```
```
aws lambda delete-function --function-name MyLocalLambda

# aws lambda update-function-code --function-name MyLocalLambda --zip-file fileb://function.zip > /dev/null

```

5. API Gateway
```
cat << EOF >> api_lambda_function.py
import json

def lambda_handler(event, context):
    # 接收傳入的參數（預設從 event 讀取）
    name = json.loads(event.get('body', '{}')).get('name', '世界')
    message = f"你好，{name}！這是來自 AWS CLI 部署的 Lambda。"

    return {
        'statusCode': 200,
        'body': json.dumps({'message': message}, ensure_ascii=False)
    }

EOF

rm -f function.zip && zip function.zip api_lambda_function.py

aws lambda create-function \
    --function-name apiMyLocalLambda \
    --runtime python3.12 \
    --zip-file fileb://function.zip \
    --handler api_lambda_function.lambda_handler \
    --role arn:aws:iam::000000000000:role/lambda-role >> /dev/null

aws lambda invoke \
	--function-name apiMyLocalLambda \
	--payload '{"body": "{\"name\": \"abc\"}"}' \
	--cli-binary-format raw-in-base64-out \
    response.json
	
cat response.json


LAMBDA_ARN=$(aws lambda get-function \
    --function-name apiMyLocalLambda \
    --query 'Configuration.FunctionArn' \
    --output text
)

API_ID=$(aws apigatewayv2 create-api \
    --name "floci-lambda-api" \
    --protocol-type HTTP \
    --query "ApiId" --output text)

INTEGRATION_ID=$(aws apigatewayv2 create-integration \
    --api-id "$API_ID" \
    --integration-type AWS_PROXY \
    --integration-method POST \
    --integration-uri "$LAMBDA_ARN" \
    --payload-format-version "2.0" \
    --query "IntegrationId" --output text)

aws apigatewayv2 create-route \
    --api-id "$API_ID" \
    --route-key "POST /test" \
    --target "integrations/$INTEGRATION_ID" > /dev/null

aws apigatewayv2 create-stage \
    --api-id "$API_ID" \
    --stage-name '$default' \
    --auto-deploy > /dev/null


curl -i -X POST "http://localhost:4566/_aws/execute-api/${API_ID}/\$default/test" \
     -H "Content-Type: application/json" \
     -d '{"name":"First"}'

```
```
aws lambda delete-function --function-name apiMyLocalLambda
aws apigatewayv2 delete-stage --api-id ${API_ID} --stage-name '$default'
aws apigatewayv2 delete-integration --api-id ${API_ID} --integration-id ${INTEGRATION_ID}
aws apigatewayv2 delete-route --api-id ${API_ID} --route-id $(aws apigatewayv2 get-routes --api-id ${API_ID} --output text --query "Items[?RouteKey=='POST /test'].RouteId")
aws apigatewayv2 delete-api --api-id ${API_ID}

```

6. lambda + dynamodb
```
aws dynamodb create-table \
    --table-name Staff \
    --attribute-definitions AttributeName=StaffID,AttributeType=S \
    --key-schema AttributeName=StaffID,KeyType=HASH \
    --billing-mode PAY_PER_REQUEST > /dev/null


aws dynamodb put-item \
    --table-name Staff \
    --item '{
        "StaffID": {"S": "EMP001"},
        "Name": {"S": "Alex Chan"},
        "Age": {"N": "29"}
    }'


aws dynamodb get-item \
    --table-name Staff \
    --key '{"StaffID": {"S": "EMP001"}}'

```
```
aws dynamodb describe-table --table-name Staff --query "Table.TableArn" --output text
aws dynamodb scan --table-name Staff
aws dynamodb delete-table --table-name Staff > /dev/null

```
```
cat << EOF > trust-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "dynamodb:GetItem",
            "Resource": "$(aws dynamodb describe-table --table-name Staff --query "Table.TableArn" --output text)"
        }
    ]
}
EOF

aws iam create-role --role-name MyLambdaExecutionRole --assume-role-policy-document file://trust-policy.json > /dev/null

```
```
cat << EOF > lambda_function.py
import json
from decimal import Decimal
import boto3
from botocore.exceptions import ClientError

# Initialize DynamoDB resource outside the handler for connection reuse
dynamodb = boto3.resource("dynamodb")
table = dynamodb.Table("Staff")


class DecimalEncoder(json.JSONEncoder):
    """Converts DynamoDB Decimal objects to int or float for JSON serialization."""

    def default(self, obj):
        if isinstance(obj, Decimal):
            return int(obj) if obj % 1 == 0 else float(obj)
        return super().default(obj)


def lambda_handler(event, context):
    # Support both direct invocation and API Gateway query parameters
    staff_id = event.get("StaffID")
    if not staff_id and "queryStringParameters" in event and event["queryStringParameters"]:
        staff_id = event["queryStringParameters"].get("StaffID")

    if not staff_id:
        return {
            "statusCode": 400,
            "headers": {"Content-Type": "application/json"},
            "body": json.dumps({"error": "Missing StaffID parameter"}),
        }

    try:
        response = table.get_item(Key={"StaffID": staff_id})
        item = response.get("Item")

        if not item:
            return {
                "statusCode": 404,
                "headers": {"Content-Type": "application/json"},
                "body": json.dumps({"error": f"StaffID '{staff_id}' not found"}),
            }

        return {
            "statusCode": 200,
            "headers": {"Content-Type": "application/json"},
            "body": json.dumps(item, cls=DecimalEncoder),
        }

    except ClientError as e:
        return {
            "statusCode": 500,
            "headers": {"Content-Type": "application/json"},
            "body": json.dumps({"error": e.response["Error"]["Message"]}),
        }
EOF

rm -f function.zip && zip function.zip lambda_function.py

aws lambda create-function \
    --function-name MyLocalLambda \
    --runtime python3.12 \
    --zip-file fileb://function.zip \
    --handler lambda_function.lambda_handler \
    --role arn:aws:iam::000000000000:role/MyLambdaExecutionRole  > /dev/null

aws lambda invoke \
    --function-name MyLocalLambda \
	--payload '{"StaffID": "EMP001"}' \
	--cli-binary-format raw-in-base64-out \
    response.json
	
cat response.json

```

```
aws lambda delete-function --function-name MyLocalLambda

```
```
cat << EOF > lambda_function.py
import json
from decimal import Decimal
import boto3
from botocore.exceptions import ClientError

# Initialize DynamoDB resource outside the handler for connection reuse
dynamodb = boto3.resource("dynamodb")
table = dynamodb.Table("Staff")

class DecimalEncoder(json.JSONEncoder):
    """Converts DynamoDB Decimal objects to int or float for JSON serialization."""

    def default(self, obj):
        if isinstance(obj, Decimal):
            return int(obj) if obj % 1 == 0 else float(obj)
        return super().default(obj)


def lambda_handler(event, context):
    # Support both direct invocation and API Gateway query parameters
    staff_id = json.loads(event.get('body', '{}')).get('StaffID', 'abc')
    if not staff_id and "queryStringParameters" in event and event["queryStringParameters"]:
        staff_id = event["queryStringParameters"].get("StaffID")

    if not staff_id:
        return {
            "statusCode": 400,
            "headers": {"Content-Type": "application/json"},
            "body": json.dumps({"error": "Missing StaffID parameter"}),
        }

    try:
        response = table.get_item(Key={"StaffID": staff_id})
        item = response.get("Item")

        if not item:
            return {
                "statusCode": 404,
                "headers": {"Content-Type": "application/json"},
                "body": json.dumps({"error": f"StaffID '{staff_id}' not found"}),
            }

        return {
            "statusCode": 200,
            "headers": {"Content-Type": "application/json"},
            "body": json.dumps(item, cls=DecimalEncoder),
        }

    except ClientError as e:
        return {
            "statusCode": 500,
            "headers": {"Content-Type": "application/json"},
            "body": json.dumps({"error": e.response["Error"]["Message"]}),
        }

EOF

rm -f function.zip && zip function.zip lambda_function.py

aws lambda create-function \
    --function-name MyLocalLambda \
    --runtime python3.12 \
    --zip-file fileb://function.zip \
    --handler lambda_function.lambda_handler \
    --role arn:aws:iam::000000000000:role/MyLambdaExecutionRole  > /dev/null

LAMBDA_ARN=$(aws lambda get-function \
    --function-name MyLocalLambda \
    --query 'Configuration.FunctionArn' \
    --output text
)

API_ID=$(aws apigatewayv2 create-api \
    --name "floci-lambda-api" \
    --protocol-type HTTP \
    --query "ApiId" --output text)

INTEGRATION_ID=$(aws apigatewayv2 create-integration \
    --api-id "$API_ID" \
    --integration-type AWS_PROXY \
    --integration-method POST \
    --integration-uri "$LAMBDA_ARN" \
    --payload-format-version "2.0" \
    --query "IntegrationId" --output text)

aws apigatewayv2 create-route \
    --api-id "$API_ID" \
    --route-key "POST /test" \
    --target "integrations/$INTEGRATION_ID" > /dev/null

aws apigatewayv2 create-stage \
    --api-id "$API_ID" \
    --stage-name '$default' \
    --auto-deploy > /dev/null

curl -i -X POST "http://localhost:4566/_aws/execute-api/${API_ID}/\$default/test" \
     -H "Content-Type: application/json" \
     -d '{"StaffID": "EMP001"}'
	 
```
```
aws apigatewayv2 delete-stage --api-id ${API_ID} --stage-name '$default'
aws apigatewayv2 delete-integration --api-id ${API_ID} --integration-id ${INTEGRATION_ID}
aws apigatewayv2 delete-route --api-id ${API_ID} --route-id $(aws apigatewayv2 get-routes --api-id ${API_ID} --output text --query "Items[?RouteKey=='POST /test'].RouteId")
aws apigatewayv2 delete-api --api-id ${API_ID}	
aws iam delete-role --role-name MyLambdaExecutionRole > /dev/null
aws lambda delete-function --function-name MyLocalLambda
aws dynamodb delete-table --table-name Staff > /dev/null

```
7. S3 + website
```
aws s3 mb s3://my-local-site
aws s3 cp - s3://my-local-site/index.html --content-type "text/html" << 'EOF'
<!DOCTYPE html>
<html>
  <head><title>Home</title></head>
  <body><h1>Hello from S3!</h1></body>
</html>
EOF

aws s3 website s3://my-local-site/ --index-document index.html
curl http://localhost:4566/my-local-site/index.html
aws s3api delete-bucket-website --bucket my-local-site
aws s3 rb s3://my-local-site --force

```
