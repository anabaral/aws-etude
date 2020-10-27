# Troubleshootings

일반적인 관리상 마주쳤던 문제들을 다룹니다.  
(개발하면서 혹은 설치하면서 나왔던 문제들은 그 관련 문서에서 다룸)

## disk quota exceeded

이 연구 프로젝트는 2020년 5월에 시작해 10월 말에 끝나는데, 막판 10월에 종종 발생헀던 문제가 이것임.  
처음엔 elasticsearch 에서, 다음엔 mariadb에서 주로 이러한 로그가 떴었음.

```
2020-10-26 23:37:01 13 [ERROR] InnoDB: Unable to lock ./keycloakdb/REALM.ibd error: 122
2020-10-26 23:37:01 13 [ERROR] InnoDB: Operating system error number 122 in a file operation.
2020-10-26 23:37:01 13 [ERROR] InnoDB: Error number 122 means 'Disk quota exceeded'
2020-10-26 23:37:01 13 [Note] InnoDB: Some operating system error numbers are described at https://mariadb.com
/kb/en/library/operating-system-error-codes/
2020-10-26 23:37:01 13 [Warning] InnoDB: Cannot open './keycloakdb/REALM.ibd'. Have you deleted .ibd files und
er a running mysqld server?
2020-10-26 23:37:01 13 [ERROR] InnoDB: Unable to lock ./keycloakdb/REALM.ibd error: 122
2020-10-26 23:37:01 13 [ERROR] InnoDB: Operating system error number 122 in a file operation.
2020-10-26 23:37:01 13 [ERROR] InnoDB: Error number 122 means 'Disk quota exceeded'
```
AWS 문서를 보면 다음과 같은 문구가 있던데 ( https://docs.aws.amazon.com/ko_kr/efs/latest/ug/troubleshooting-efs-fileop-errors.html )
* Amazon EFS에서는 현재 사용자 디스크 할당량을 지원하지 않습니다. 이 오류는 다음 제한 중 하나를 초과한 경우 발생할 수 있습니다.
  - 인스턴스 하나에 대해 최대 128개의 활성 사용자 계정에서 동시에 파일을 열어 놓을 수 있습니다.
  - 인스턴스 하나에 대해 최대 32,768개의 파일을 동시에 열어 놓을 수 있습니다.
  - 고유한 인스턴스 탑재마다 256개의 고유한 파일-프로세스 페어에 대해 최대 총 8,192개의 잠금을 획득할 수 있습니다. 
    예를 들어, 한 프로세스에서 개별 파일 256개에 대해 잠금을 하나 이상 획득하거나 여덟 개의 프로세스에서 각각 32개 파일에 대해 하나 이상의 잠금을 획득할 수 있습니다.

이들 모두를 의심했었지만 이를 증명할 방법이 없었음.

나중에 하나씩 해결이 되었는데, 대략 다음과 같음:
- elasticsearch의 경우, 시스템 파라미터 ```sysctl -w vm.max_map_count=262144``` 설정을 해 줘야 함.
  설치하신 분이 재설치하면서 이걸 실행하는 initContainer 설정을 누락했던 게 원인.  
  (일반적으로 docker 에서 자기를 띄운 VM의 시스템 파라미터를 고칠 수 없는데, privileged:true 모드로 띄우면 가능함)
- mariadb의 경우, 파일 락을 많이 잡는 게 문제일 수 있음. 실제 ```lslocks``` 명령으로 확인해 보면 mariadb는 최소 100개 이상의 파일 락을 잡음.
  그러므로 비슷한 경쟁자와 같은 VM에 뜰 경우 문제를 일으키게 됨.



 
