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

  # 이것저것 테스트하고 최종적으로 만들었던 것. cron실행할 거라 리턴할 게 없음.
  def lambda_handler(event, context):
      print(f"{event['time']} 에 시작 혹은 종료될 뭔가를 찾아요")
      print(f"""context:\n name={context.function_name} , version={context.function_version} , 
                mem_lim={context.memory_limit_in_mb} , context_id={context.identity.cognito_identity_id} ,
                context_pool_id={context.identity.cognito_identity_pool_id} , 
                client_context={context.client_context} , 
             """
          )
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
  