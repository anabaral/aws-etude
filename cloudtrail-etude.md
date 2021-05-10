# Cloudtrail 기본 익히기

Azure에서 Activity log 가 리소스의 살고 죽는 동작을 기록하는 로그라면, AWS에서는 CloudTrail 이 그 역할을 합니다.

우리는 VM Instance가 죽고 살 때 그 로그 (혹은 이벤트) 를 기록하기 위해 CloudTrail을 이용할 것입니다.  
이 때 'CloudTrail 추적' 까지는 생성할 필요 없습니다. (뭣도 모르고 생성했다 10분만에 지웠네요.. S3 버킷까지 깨끗이 지워야 합니다)

대략 아래의 코드 정도만 실행해도 어느 인스턴스가 언제 죽고 살았는지 정도는 파악이 됩니다.
```
import boto3
from dateutil.tz import tzlocal

client = boto3.client('cloudtrail')

start_events = client.lookup_events(
    LookupAttributes=[
        {
            'AttributeKey': 'EventName',
            'AttributeValue': 'StartInstances'
        }
    ],
    StartTime = datetime(2021, 5, 1, tzinfo=tzlocal()),
    EndTime = datetime(2021, 5, 10, tzinfo=tzlocal())
)
stop_events = client.lookup_events(
    LookupAttributes=[
        {
            'AttributeKey': 'EventName',
            'AttributeValue': 'StopInstances'
        },
    ],
    StartTime = datetime(2021, 5, 1, tzinfo=tzlocal()),
    EndTime = datetime(2021, 5, 10, tzinfo=tzlocal())
)

events = start_events['Events'] + stop_events['Events']
for ev in events:
    print (f"{ev['EventName']} : {ev['Resources']}  at {ev['EventTime']}")

# 결과
StartInstances : [{'ResourceType': 'AWS::EC2::Instance', 'ResourceName': 'i-0b096a3e9ba3eb624'}]  at 2021-05-06 10:08:24+09:00
...
StopInstances : [{'ResourceType': 'AWS::EC2::Instance', 'ResourceName': 'i-02420ac5ff612cf71'}]  at 2021-05-03 00:19:34+09:00
```

EventName 에는 StartInstances, StopInstances 말고도 StartInstance, StopInstance 가 있던데, 이 역시 쓰이는 지는 모르겠습니다.


