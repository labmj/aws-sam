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

![image](https://user-images.githubusercontent.com/79297534/111746136-f6325180-88d0-11eb-977f-c12d254fa272.png)


- functions/stock_buyer/app.py

![image](https://user-images.githubusercontent.com/79297534/111746435-63de7d80-88d1-11eb-9a10-0195be44f7cb.png)


- stock_trader.asl.json

![image](https://user-images.githubusercontent.com/79297534/111746510-7c4e9800-88d1-11eb-81fb-3033a2be6de0.png)

- buildspec.yml

![image](https://user-images.githubusercontent.com/79297534/111746728-c0419d00-88d1-11eb-97ea-eb1d8455c397.png)


- template.yaml

![image](https://user-images.githubusercontent.com/79297534/111747059-2e865f80-88d2-11eb-9d1d-324407440a50.png)




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
