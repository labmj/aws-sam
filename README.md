# CI/CD pipeline to deploy functions

![image](https://user-images.githubusercontent.com/79297534/111967487-7360ff00-8b3b-11eb-9d85-ebc94ec8ad36.png)



사전 준비
- git 설정


- 시작 (TAG 명명 규칙 참조) 

![image](https://user-images.githubusercontent.com/79297534/111940583-798eb580-8b12-11eb-9974-df2f128da167.png)


![image](https://user-images.githubusercontent.com/79297534/111930460-5d801980-8afc-11eb-8989-340c22daed02.png)

![image](https://user-images.githubusercontent.com/79297534/111930930-67eee300-8afd-11eb-9de4-4aec97c93a3a.png)


구조

```
Repository
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
```


위와 같은 구조 만든 후 아래와 같은 방법으로 로컬에서 작업한 코드 push

![image](https://user-images.githubusercontent.com/79297534/111939253-20715280-8b0f-11eb-8b6f-54a3dc1bdd0b.png)



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
   - export BUCKET=aire-aico-cicdpipelinetutorial-s3
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


- 새로운 정책 추가

![image](https://user-images.githubusercontent.com/79297534/111749368-02b8a900-88d5-11eb-9424-aa4b46cf32d1.png)

- 아래 그림과 같이 정책 생성후 추가하여 역할 생성 (현재는 명명을 AIRE-AICO-lambda-cicd-Role -> AIRE-AICO-cicd-cloudformation-Role로 변경함)

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




![image](https://user-images.githubusercontent.com/79297534/111932365-991ce280-8b00-11eb-8063-859cd4907f60.png)


- CodeBuild 정책 생성 ( 역할은 만들어 놓은것 위주로 재사용을 지향) !!추가로 CodeCommit에 대한 정책 추가 시켜주기

![image](https://user-images.githubusercontent.com/79297534/111946662-3ab32c80-8b1f-11eb-8754-fb039283ec50.png)



- 파이프라인 설정

![image](https://user-images.githubusercontent.com/79297534/111945938-f1aea880-8b1d-11eb-9045-8487d2bafab1.png)


![image](https://user-images.githubusercontent.com/79297534/111944681-8ebc1200-8b1b-11eb-882c-a5c79b8e55b0.png)



![image](https://user-images.githubusercontent.com/79297534/111926072-ad57e400-8aee-11eb-8a17-ca17472cdb9b.png)

![image](https://user-images.githubusercontent.com/79297534/111926141-01fb5f00-8aef-11eb-8721-e442e3c55704.png)

- 역할은 기존에 만든것이 있을 경우, 기존 것을 사용하는것을 지향함

![image](https://user-images.githubusercontent.com/79297534/111947211-54a13f00-8b20-11eb-94d2-db1cea25722d.png)


![image](https://user-images.githubusercontent.com/79297534/111926227-54d51680-8aef-11eb-8b48-181e73cc7242.png)


- Deploy 


![image](https://user-images.githubusercontent.com/79297534/112080496-73a5dc80-8bc5-11eb-8a60-68df24ebef4d.png)


- Deploy 추가작업 아래 그림처럼 만든후 저장하고 변경 사항 릴리스 선택

![image](https://user-images.githubusercontent.com/79297534/111926677-29532b80-8af1-11eb-88d2-37593ee6473d.png)


![image](https://user-images.githubusercontent.com/79297534/112080599-a2bc4e00-8bc5-11eb-96fd-63ba1f3ca71e.png)


- AWS Lambda 서비스에서 아래 그림같이 생성 확인

![image](https://user-images.githubusercontent.com/79297534/112081172-9258a300-8bc6-11eb-9ae9-6e63234b8ec1.png)


- 실행 확인

![image](https://user-images.githubusercontent.com/79297534/112081369-e499c400-8bc6-11eb-88b4-837fea9a2485.png)

![image](https://user-images.githubusercontent.com/79297534/112081442-02672900-8bc7-11eb-991b-378615df78ca.png)

![image](https://user-images.githubusercontent.com/79297534/112081458-0a26cd80-8bc7-11eb-94f8-241a0c833239.png)


- 번외


sam init tutorial

![image](https://user-images.githubusercontent.com/79297534/111422235-aff2bc00-8731-11eb-9d68-e2f9b4f85c04.png)



