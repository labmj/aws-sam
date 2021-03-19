# CI/CD pipeline to deploy functions

사전 준비
- git 설정


- 시작


![image](https://user-images.githubusercontent.com/79297534/111742359-63db7f00-88cb-11eb-91b7-149cad09e4e4.png)


![image](https://user-images.githubusercontent.com/79297534/111742458-91c0c380-88cb-11eb-82d1-8304904c61c7.png)


![image](https://user-images.githubusercontent.com/79297534/111744392-8327db80-88ce-11eb-9f04-be5988e66fd6.png)

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


![image](https://user-images.githubusercontent.com/79297534/111746136-f6325180-88d0-11eb-977f-c12d254fa272.png)


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


![image](https://user-images.githubusercontent.com/79297534/111746435-63de7d80-88d1-11eb-9a10-0195be44f7cb.png)


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

![image](https://user-images.githubusercontent.com/79297534/111746510-7c4e9800-88d1-11eb-81fb-3033a2be6de0.png)

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

![image](https://user-images.githubusercontent.com/79297534/111746728-c0419d00-88d1-11eb-97ea-eb1d8455c397.png)


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

![image](https://user-images.githubusercontent.com/79297534/111747059-2e865f80-88d2-11eb-9d1d-324407440a50.png)


- Cloud Formation에 필요한 역할 만들기 (사용 사례에서 CloudFormation 선택)

![image](https://user-images.githubusercontent.com/79297534/111748949-74dcbe00-88d4-11eb-8ebf-074240b796d9.png)


- AWSLambdaExecute 정책 추가

![image](https://user-images.githubusercontent.com/79297534/111749238-d866eb80-88d4-11eb-82e2-78505608c566.png)


- AWSLambdaExecute 정책 추가

![image](https://user-images.githubusercontent.com/79297534/111749368-02b8a900-88d5-11eb-9424-aa4b46cf32d1.png)

- 아래 그림과 같이 정책 생성후 추가

![image](https://user-images.githubusercontent.com/79297534/111749517-401d3680-88d5-11eb-99ae-8a5d1b955b4d.png)

![image](https://user-images.githubusercontent.com/79297534/111749926-d5202f80-88d5-11eb-90f2-c771e9d9d666.png)

![image](https://user-images.githubusercontent.com/79297534/111750148-32b47c00-88d6-11eb-99c3-bf5b35753787.png)



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
