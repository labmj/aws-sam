# CI/CD pipeline to deploy functions

사전 준비
- git 설정


- 시작 (TAG 명명 규칙 참조) 

![image](https://user-images.githubusercontent.com/79297534/111930333-011cfa00-8afc-11eb-8ded-79d66e1beb45.png)

![image](https://user-images.githubusercontent.com/79297534/111930460-5d801980-8afc-11eb-8989-340c22daed02.png)

![image](https://user-images.githubusercontent.com/79297534/111930930-67eee300-8afd-11eb-9de4-4aec97c93a3a.png)


구조

![image](https://user-images.githubusercontent.com/79297534/111744156-1b719080-88ce-11eb-9015-b44a6ffa4207.png)

위와 같은 구조 만든 후

![image](https://user-images.githubusercontent.com/79297534/111744906-47414600-88cf-11eb-8271-c1ba9d7fa4c8.png)


- functions/stock_checker/app.py

```python
from random import randint


def lambda_handler(event, context):

    stock_price=event['stock_price']
    
    return {"stock_price": stock_price}

```



- functions/stock_buyer/app.py

```python
from datetime import datetime
from random import randint
from uuid import uuid4


def lambda_handler(event, context):
    # Get the price of the stock provided as input
    stock_price = event["stock_price"]
    # Mocked result of a stock buying transaction
    transaction_result = {
        # "id": str(uuid4()),  # Unique ID for the transaction
        "price": str(stock_price),  # Price of each share
        "type": "buy",  # Type of transaction (buy/sell)
        "qty": str(
            randint(1, 10)
        ),  # Number of shares bought/sold (We are mocking this as a random integer between 1 and 10)
        "timestamp": datetime.now().isoformat(),  # Timestamp of the when the transaction was completed
    }
    return transaction_result


```


- stock_trader.asl.json


```json
{
    "Comment": "A state machine that does mock stock trading.",
    "StartAt": "Check Stock Value",
    "States": {
        "Check Stock Value": {
            "Type": "Task",
            "Resource": "${StockCheckerFunctionArn}",
            "Retry": [
                {
                    "ErrorEquals": [
                        "States.TaskFailed"
                    ],
                    "IntervalSeconds": 15,
                    "MaxAttempts": 5,
                    "BackoffRate": 1.5
                }
            ],
            "Next": "Buy Stock"
        },
        "Buy Stock": {
            "Type": "Task",
            "Resource": "${StockBuyerFunctionArn}",
            "Retry": [
                {
                    "ErrorEquals": [
                        "States.TaskFailed"
                    ],
                    "IntervalSeconds": 2,
                    "MaxAttempts": 3,
                    "BackoffRate": 1
                }
            ],
            "End": true
        }
        
    }
}

```


- buildspec.yml
```yaml
version: 0.2
phases:
 install:
  runtime-versions:
   python: 3.8
 build: 
  commands:
   - sam build
   - export BUCKET=test-pipeline-output-bucket1
   - sam package --s3-bucket $BUCKET --output-template-file outputtemplate.yml
artifacts:
 type: zip
 files:
  - template.yml
  - outputtemplate.yml
```



- template.yaml


```yml
AWSTemplateFormatVersion: "2010-09-09"
Transform: AWS::Serverless-2016-10-31
Description: >
  TEST_SECOND

  Sample SAM Template for TEST_SECOND

Resources:
  StockTradingStateMachine:
    Type: AWS::Serverless::StateMachine # More info about State Machine Resource: https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-resource-statemachine.html
    Properties:
      DefinitionUri: statemachine/stock_trader.asl.json
      DefinitionSubstitutions:
        StockCheckerFunctionArn: !GetAtt StockCheckerFunction.Arn
        StockBuyerFunctionArn: !GetAtt StockBuyerFunction.Arn
      
      Policies: # Find out more about SAM policy templates: https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-policy-templates.html
        - LambdaInvokePolicy:
            FunctionName: !Ref StockCheckerFunction
      
        - LambdaInvokePolicy:
            FunctionName: !Ref StockBuyerFunction
    

  StockCheckerFunction:
    Type: AWS::Serverless::Function # More info about Function Resource: https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-resource-function.html
    Properties:
      CodeUri: functions/stock_checker/
      Handler: app.lambda_handler
      Runtime: python3.8

  StockBuyerFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: functions/stock_buyer/
      Handler: app.lambda_handler
      Runtime: python3.8

```


- Cloud Formation에 필요한 역할 만들기 (사용 사례에서 CloudFormation 선택)

![image](https://user-images.githubusercontent.com/79297534/111748949-74dcbe00-88d4-11eb-8ebf-074240b796d9.png)


- 새로운 정책 추가 (명명법 수정 필요)

![image](https://user-images.githubusercontent.com/79297534/111749368-02b8a900-88d5-11eb-9424-aa4b46cf32d1.png)

- 아래 그림과 같이 정책 생성후 추가하여 역할 생성

![image](https://user-images.githubusercontent.com/79297534/111749517-401d3680-88d5-11eb-99ae-8a5d1b955b4d.png)

```json
{
    "Statement": [
        {
            "Action": [
                "apigateway:*",
                "codedeploy:*",
                "lambda:*",
                "cloudformation:CreateChangeSet",
                "iam:GetRole",
                "iam:CreateRole",
                "iam:DeleteRole",
                "iam:PutRolePolicy",
                "iam:AttachRolePolicy",
                "iam:DeleteRolePolicy",
                "iam:DetachRolePolicy",
                "iam:PassRole",
                "s3:GetObject",
                "s3:GetObjectVersion",
                "s3:GetBucketVersioning"
            ],
            "Resource": "*",
            "Effect": "Allow"
        }
    ],
    "Version": "2012-10-17"
}
```

![image](https://user-images.githubusercontent.com/79297534/111931625-ebf59a80-8afe-11eb-89c3-c0d21fb94832.png)

![image](https://user-images.githubusercontent.com/79297534/111931807-658d8880-8aff-11eb-983b-b51a34869e38.png)


![image](https://user-images.githubusercontent.com/79297534/111750903-2977df00-88d7-11eb-86ee-7fa8f27d5085.png)


![image](https://user-images.githubusercontent.com/79297534/111932365-991ce280-8b00-11eb-8063-859cd4907f60.png)





- 파이프라인 설정

![image](https://user-images.githubusercontent.com/79297534/111926034-84375380-8aee-11eb-96a0-510da0efe859.png)

![image](https://user-images.githubusercontent.com/79297534/111926055-9addaa80-8aee-11eb-981a-378a2962e133.png)


![image](https://user-images.githubusercontent.com/79297534/111926072-ad57e400-8aee-11eb-8a17-ca17472cdb9b.png)

![image](https://user-images.githubusercontent.com/79297534/111926141-01fb5f00-8aef-11eb-8721-e442e3c55704.png)

- 역할은 기존에 만든것이 있을 경우, 기존 것을 사용하는것을 지향함

![image](https://user-images.githubusercontent.com/79297534/111926165-12abd500-8aef-11eb-8db9-d842569ec08d.png)


![image](https://user-images.githubusercontent.com/79297534/111926227-54d51680-8aef-11eb-8b48-181e73cc7242.png)

- CodeBuild에서 만든 정책에 AmazonS3FullAccess 넣어주기 (새창 - IAM 서비스 선택)

![image](https://user-images.githubusercontent.com/79297534/111926243-73d3a880-8aef-11eb-9d7c-5b3f57d54764.png)

![image](https://user-images.githubusercontent.com/79297534/111926273-8e0d8680-8aef-11eb-871a-f340f2d0bad1.png)


- Deploy

![image](https://user-images.githubusercontent.com/79297534/111926397-06744780-8af0-11eb-96c8-71d402d43582.png)

- 추가작업

![image](https://user-images.githubusercontent.com/79297534/111926677-29532b80-8af1-11eb-88d2-37593ee6473d.png)


![image](https://user-images.githubusercontent.com/79297534/111926663-1a6c7900-8af1-11eb-96b0-aa34fb08cff3.png)



데이터 만들기



우선 python 가상환경 만듦 

git 연결

후에 sam init 과정

![image](https://user-images.githubusercontent.com/79297534/111422235-aff2bc00-8731-11eb-9d68-e2f9b4f85c04.png)

구조

![image](https://user-images.githubusercontent.com/79297534/111422171-981b3800-8731-11eb-843f-988c6f6b32a6.png)


TestRepository
├── buildspec.yml
├── functions
│   ├── stock_buyer
│   │   ├── app.py
│   │   └── requirements.txt
│   └── stock_checker
│       ├── app.py
│       └── requirements.txt
├── statemachine
│   └── stock_trader.asl.json
└── template.yml
