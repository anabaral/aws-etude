# AWS EKS 인스턴스 내렸다 올리기
학습용으로 EKS를 사용한다면 새벽시간에 인스턴스를 계속 살려두는 것은 비용부담이 심합니다.
그래서 안쓰는 시간에는 과감히 인스턴스를 내리는 식으로 절감을 시도해 봅시다. 과연 잘 될까?

현재 우리의 클러스터는 클러스터 관리용 노드그룹이 만들어져 있지 않고 따로 EC2 스케일링 그룹으로 지정해 두었습니다.
EC2 스케일링 그룹 설정에도 자동으로 그룹 크기를 조절하는 방법이 있습니다만,
여기서는 쉘과 cron 으로 해 보겠습니다.

## AWS EKS 인스턴스 내리기

미리 AWS EKS에 해당하는 스케일그룹의 상태를 알아둡니다.

다음과 같이 수행합니다 (stop-ek-instances.sh) :
<pre><code>#!/bin/sh

WORKER_NG_NAME=eksctl-mta-cluster-nodegroup-mta-worker-ng
AG_NAME_WORKER=$(aws autoscaling  describe-auto-scaling-groups | \
    jq '.AutoScalingGroups[] | [ .AutoScalingGroupName , .LaunchTemplate.LaunchTemplateName , .MinSize , .MaxSize , .DesiredCapacity ] ' | \
    jq -r '.|select(.[1] == "'${WORKER_NG_NAME}'" )[0]')
aws autoscaling  update-auto-scaling-group --auto-scaling-group-name ${AG_NAME_WORKER} \
    --min-size 0 --max-size 0 --desired-capacity 0

MGMT_NG_NAME=eksctl-mta-cluster-nodegroup-mta-mgmt-ng
AG_NAME_MGMT=$(aws autoscaling  describe-auto-scaling-groups | \
    jq '.AutoScalingGroups[] | [ .AutoScalingGroupName , .LaunchTemplate.LaunchTemplateName , .MinSize , .MaxSize , .DesiredCapacity ] ' | \
    jq -r '.|select(.[1] == "'${MGMT_NG_NAME}'" )[0]')
aws autoscaling  update-auto-scaling-group --auto-scaling-group-name ${AG_NAME_MGMT} \
    --min-size 0 --max-size 0 --desired-capacity 0
</code></pre>

## AWS EKS 인스턴스 올리기
다음과 같이 수행합니다 (start-eks-instances.sh) :
<pre><code>#!/bin/sh

WORKER_NG_NAME=eksctl-mta-cluster-nodegroup-mta-worker-ng
AG_NAME_WORKER=$(aws autoscaling  describe-auto-scaling-groups | \
    jq '.AutoScalingGroups[] | [ .AutoScalingGroupName , .LaunchTemplate.LaunchTemplateName , .MinSize , .MaxSize , .DesiredCapacity ] ' | \
    jq -r '.|select(.[1] == "'${WORKER_NG_NAME}'" )[0]')
aws autoscaling  update-auto-scaling-group --auto-scaling-group-name ${AG_NAME_WORKER} \
    --min-size 2 --max-size 5 --desired-capacity 2

MGMT_NG_NAME=eksctl-mta-cluster-nodegroup-mta-mgmt-ng
AG_NAME_MGMT=$(aws autoscaling  describe-auto-scaling-groups | \
    jq '.AutoScalingGroups[] | [ .AutoScalingGroupName , .LaunchTemplate.LaunchTemplateName , .MinSize , .MaxSize , .DesiredCapacity ] ' | \
    jq -r '.|select(.[1] == "'${MGMT_NG_NAME}'" )[0]')
aws autoscaling  update-auto-scaling-group --auto-scaling-group-name ${AG_NAME_MGMT} \
    --min-size 1 --max-size 3 --desired-capacity 2
</code></pre>

※ 위에서 <code>--min-size</code>, <code>--max-size</code>, <code>--desired-capacity</code> 값들은 기존에 있는 값들을 기억해서 적어 주어야 합니다.

※ 한편, 배스천으로 쓰는 EC2에 jq 정도는 기본으로 깔려 있더군요. 아니었으면 따로 깔아줘야 합니다.

## cron에 등록하기

이제 이를 cron으로 등록해 보았습니다:
```
[selee@ip-192-168-2-94 ~]$ crontab -l
0 12 * * * /home/selee/instances_manage/stop-eks-instances.sh   2>&1 >> /home/selee/instances_manage/manage.log
30 23 * * 0-4 /home/selee/instances_manage/start-eks-instances.sh   2>&1 >> /home/selee/instances_manage/manage.log
```
배스천 서버의 타임존이 UTC 기준인 것을 바꾸지 않아서 좀 오해하기 쉽게 설정하게 되었는데, 골자는 이렇습니다:
* 매일 오후 9시에 모든 떠 있는 EKS 인스턴스를 종료합니다. (종료 = 인스턴스를 없앰)
* 평일 아침 8시 30분에 모든 EKS 인스턴스를 본래의 설정으로 띄웁니다.

## 이벤트 알림(slack) 설정

여기에 더하여 다음과 같이 이벤트 수행마다 slack notification 하도록 설정하였습니다.

slack notification url 은 따로 얻었다고 가정합니다:
```
[selee@ip-192-168-2-94 instances_manage]$ cat alert.sh
#!/bin/sh
CATEGORY=$1
if [ "$CATEGORY" == "" ]; then
  MESSAGE="hello"
elif [ $CATEGORY -eq 1 ]; then
  MESSAGE="instances on"
elif [ $CATEGORY -eq 0 ]; then
  MESSAGE="instances off"
else
  MESSAGE="hello"
fi

curl -XPOST --data-urlencode "payload={\"channel\":\"#mta\", \"username\":\"webhookbot\", \"text\":\"${MESSAGE}\n\", \"icon_emoji\":\":ghost:\"}" \
    https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX
```

이 쉘은 위의 start-eks-instances.sh 및 stop-eks-instances.sh 에 다음과 같이 추가해 줍니다:
```
# start-eks-instances.sh 에는 다음 줄 추가
/home/selee/instances_manage/alert.sh 1

# stop-eks-instances.sh 에는 다음 줄 
/home/selee/instances_manage/alert.sh 0
```

## 비고

사실 EKS를 위해 만들어진 노드 스케일링 그룹 인스턴스 수 조정은 쉘을 통해 할 필요가 없습니다. 
[서비스 - EC2 - 오토 스케일링 - 오토 스케일링 그룹] 에서 자동 조정을 위한 예약 작업을 추가할 수 있습니다. 
여기서는 스케일링 그룹이 2개이고 각기 켜고 꺼야 하니 4개의 예약 작업을 등록하면 되겠지요.

처음엔 이게 있는 줄 몰라서 위와 같이 쉘과 cron 으로 작업했는데 예약 작업 등록이 가능하다는 걸 안 후에
등록을 시도했습니다만.. 2020년 8월 4일 현재 어쩐 일인지 등록한 게 실행이 되지 않아 부득이 CLI로 작업한 기존 것을 유지했습니다.

뭐.. CLI 작업도 장점이 있습니다.
* infrastructure as code 개념으로는 더 좋을 수 있고
* 현재 저는 쉘에 한 줄 추가해서 실행할 때마다 slack으로 알림을 보내도록 만들었는데 
  위에처럼 제공되는 기능일 경우 이런 개선이 쉽지 않죠. 

