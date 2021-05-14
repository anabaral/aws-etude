# 람다함수 등록하고 테스트

2021-03-26 테스트하면서 남긴것들 끌어모음..


lambda 화면으로 이동
함수 블레이드 클릭
함수 생성 버튼 클릭
- 이름 ds04226-ec2-start-and-stop
- 코드 작성
  ```
  import boto3
  #import pprint
  import json
  
  
  def lambda_handler(event, context):
      #ec2_client = boto3.client('ec2', region_name='ap-northeast-2')
      # (copied from ds05599 lambda function)
      #filters_Name_ds05599=[{
      #            'Name':'tag:owner',
      #            'Values':['ds05599']
      #        }]
      #        
      #instances = ec2_client.describe_instances(Filters=filters_Name_ds05599)
      #inst_ids = instances['Reservations'][0]['Instances'][0]['InstanceId']
      #ec2_client.start_instances(InstanceIds=[inst_ids])
      #ec2_client.stop_instances(InstanceIds=[inst_ids])
      
      # 이 샘플 프로그램의 목적은 정해진 시각마다 실행되면서
      # - 실행되는 시각을 얻고 확인
      # - 추가로 그 시각에 하기로 한 무언가를 확인
      print(f"{event['time']} 에 시작 혹은 종료될 뭔가를 찾아요")
      print(f"""context:\n name={context.function_name} , version={context.function_version} , 
                mem_lim={context.memory_limit_in_mb} , context_id={context.identity.cognito_identity_id} ,
                context_pool_id={context.identity.cognito_identity_pool_id} , 
                client_context={context.client_context} , 
             """
          )
      #    f"str(context.cognito_identity_id) , {context.cognito_identity_pool_id} , " + 
      #    f"{context.client_context}"
      #     )
      #return {
      #    'statusCode': 200,
      #    'body': {"event": json.dumps(event), "context": json.dumps(context) }
      #}
  
  
  ```
- 트리거 추가
  * 타입: EventBridge(Cloudwatch Events)
  * 이름: every-minutes
  * cron expression: cron(* * * * ? *)
  * 이벤트 버스:default
- 권한
  * 적당히 람다함수 실행할 때 쓰는 거 하나 골랐음.
  * 이걸 잘 고르거나 아예 새로 선택해야 함. 이미 존재하는 거 하나 선택했다가 
    로그를 eu-central-1 으로 보내려고 해서 (당연히 거기 아무것도 없으니 하나도 남지 않음) 한참 원인 모르고 함
- 대상
  * 비워둠. (이건 람다 연쇄적 실행하려고 할 때 쓰이는 것 같음)
  

- 람다함수에 boto3 실행을 위한 레이어 추가하기
  * 일반적인 가상머신이나 컨테이너라면 그 안에 파이썬 라이브러리를 설치하면 될 것이지만, 람다는 그렇게 못하므로
    어떤 라이브러리를 사용하고 싶다면 두 가지 정도의 접근이 필요합니다.
    - 내가 만든 람다함수 코드를 라이브러리와 함께 말아서 올리는 식으로 디플로이
    - 람다함수 코드는 따로 디플로이하고, 여기서 쓸 layer를 따로 올림
    후자가 당연히 더 권고되는 패턴입니다. 다음과 같이 해보겠습니다.
  ```
  D:\>mkdir python_tmp
  D:\>cd python_tmp
  D:\python_tmp>mkdir python
  D:\python_tmp>pip3 install boto3 -t python
  D:\python_tmp>"c:\Program Files\7-Zip\7z.exe" a ..\boto3-layer.zip .
  D:\python_tmp>aws lambda publish-layer-version --layer-name boto3-layer --zip-file fileb://d:\boto3-layer.zip
  ...
      "LayerVersionArn": "arn:aws:lambda:ap-northeast-2:592806604814:layer:boto3-layer:1",
  ...
  ```
  

# 람다함수에 세부 기능 추가하면서 테스트

## 권한 조정

### DB 접근 관련 권한

serverless aurora db 에 붙이려니 lambda 함수를 VPC에 넣어야 가능함.  
(아니면 db가 public open 되거나...)

쉽게 가기 위해 같은 VPC에 넣으려는데 에러가 난다.  
`The provided execution role does not have permissions to call CreateNetworkInterface on EC2`
이걸 풀려면 다음 권한이 필요한데
- "ec2:DescribeNetworkInterfaces"
- "ec2:CreateNetworkInterface"
찾아보니 예전에 lambda를 위해 만든 역할(여기서는 arn:aws:iam::592806604814:role/service-role/ds04226-lambda-1)을 찾아 여기에 정책을 더하면 됨.
- AWSLambdaVPCAccessExecutionRole
이 정책에 위의 두 권한도 포함되어 있음.

### EC2 시작/중지 권한

이상하게 중지가 오래 걸리고 3분 이상 줘도 timeout 걸림. 늦어도 1분 안에 끝나야 하는데.

다음 권한을 만들어서 (주어진 정책에서 안찾아져서) 추가함.
- 이름: EC2StartStopRole
- 내용:
  ```
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ],
        "Resource": "arn:aws:logs:*:*:*"
      },
      {
        "Effect": "Allow",
        "Action": [
          "ec2:Start*",
          "ec2:Stop*"
        ],
        "Resource": "*"
      }
    ]
  }
  ```
- 전체 정책들은 (아마도 만들 때 주어졌던 정책까지 합쳐) 다음 4개임 (2021-05-14)
  * AWSLambdaVPCAccessExecutionRole
  * EC2StartStopRole
  * AWSLambdaTestHarnessExecutionRole-8d8313fb-e016-4b81-bd07-954751e891b5
  * AWSLambdaBasicExecutionRole-72e67c3a-85c3-4c3c-9f8c-558f00960905


## sts 문제: 아직 해결 못함

실행할 때 다음 코드에서 멈추다가 타임아웃 됨.
```sts_identity = boto3.client('sts').get_caller_identity()```
API 문서 찾아보니 다음 권한 얘기가 있는데, 한편으론 없어도 될 것처럼 나와서 고민중.
sts:GetCallerIdentity

일단은 그냥 상수값으로 부여함.


## timeout 문제 : 아직 해결 못함

```
2021-05-14T23:05:43.374+09:00	[ERROR] ConnectTimeoutError: Connect timeout on endpoint URL: "https://ec2.ap-northeast-2.amazonaws.com/"
```
권한 문제가 아니고 접속 문제였어..?





