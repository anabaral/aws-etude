# AWS EKS 인스턴스 내렸다 올리기
학습용으로 EKS를 사용한다면 새벽시간에 인스턴스를 계속 살려두는 것은 비용부담이 심합니다.
그래서 안쓰는 시간에는 과감히 인스턴스를 내리는 식으로 절감을 시도해 봅시다. 과연 잘 될까?

현재 우리의 클러스터는 클러스터 관리용 노드그룹이 만들어져 있지 않고 따로 EC2 스케일링 그룹으로 지정해 두었습니다.
그래서 이걸 가지고 해봅시다.

# AWS EKS 인스턴스 내리기

미리 AWS EKS에 해당하는 스케일그룹의 상태를 알아둡니다.

다음과 같이 수행합니다 (stop-elk-instances.sh) :
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

# AWS EKS 인스턴스 올리기
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

※ 한편, 배스티온으로 쓰는 EC2에 jq 정도는 기본으로 깔려 있더군요. 아니었으면 따로 깔아줘야 합니다.






