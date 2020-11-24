# nodejs-db

## mta-admin mariadb scheme

```
MariaDB [my_database]> desc alert_mappings;
+--------------+-------------+------+-----+---------+-------+
| Field        | Type        | Null | Key | Default | Extra |
+--------------+-------------+------+-----+---------+-------+
| cluster      | varchar(50) | NO   | PRI | NULL    |       |
| namespace    | varchar(50) | NO   | PRI | NULL    |       |
| category     | varchar(20) | NO   | PRI | NULL    |       |
| channel_name | varchar(50) | NO   | PRI | NULL    |       |
| channel_type | varchar(30) | NO   | PRI | NULL    |       |
| level        | varchar(8)  | NO   | PRI | NULL    |       |
+--------------+-------------+------+-----+---------+-------+

MariaDB [my_database]> desc channel;
+--------------+-------------+------+-----+---------+-------+
| Field        | Type        | Null | Key | Default | Extra |
+--------------+-------------+------+-----+---------+-------+
| channel_type | varchar(30) | NO   | PRI | NULL    |       |
| channel_name | varchar(50) | NO   | PRI | NULL    |       |
| user_id      | varchar(10) | NO   | PRI | NULL    |       |
| level        | varchar(5)  | YES  |     | NULL    |       |
+--------------+-------------+------+-----+---------+-------+

MariaDB [my_database]> desc namespace;
+-----------+-------------+------+-----+---------+-------+
| Field     | Type        | Null | Key | Default | Extra |
+-----------+-------------+------+-----+---------+-------+
| cluster   | varchar(50) | NO   | PRI | NULL    |       |
| namespace | varchar(50) | NO   | PRI | NULL    |       |
| user_id   | varchar(10) | NO   | PRI | NULL    |       |
+-----------+-------------+------+-----+---------+-------+

MariaDB [my_database]> desc user;
+-----------+-------------+------+-----+---------+-------+
| Field     | Type        | Null | Key | Default | Extra |
+-----------+-------------+------+-----+---------+-------+
| user_id   | varchar(10) | NO   | PRI | NULL    |       |
| user_name | varchar(30) | NO   |     | NULL    |       |
+-----------+-------------+------+-----+---------+-------+

MariaDB [my_database]> desc alert_summary;
+--------------+--------------+------+-----+---------+-------+
| Field        | Type         | Null | Key | Default | Extra |
+--------------+--------------+------+-----+---------+-------+
| date         | varchar(10)  | NO   | PRI | NULL    |       |
| alert_name   | varchar(100) | NO   | PRI | NULL    |       |
| cluster      | varchar(50)  | NO   | PRI | NULL    |       |
| namespace    | varchar(50)  | NO   | PRI | NULL    |       |
| channel_name | varchar(50)  | YES  |     | NULL    |       |
| count        | int(11)      | YES  |     | NULL    |       |
+--------------+--------------+------+-----+---------+-------+
```
create table 스크립트로는 다음과 같음 (예전에 만든 걸 백업하는 거라, 만약 위와 아래가 내용이 다르다면 위의 것이 맞을 것임.)
```

create table alert_mappings(
  cluster       varchar(50) ,
  namespace     varchar(50) ,
  category      varchar(20) ,
  channel_name  varchar(30) ,
  channel_type  varchar(30)	,
  level         varchar(5),
  primary key(cluster, namespace, category, channel_name)
);

create table namespace(
  cluster    varchar(50) ,
  namespace  varchar(50) ,
  user_id    varchar(10) ,
  primary key (cluster, namespace, user_id)
);

create table channel (
  channel_type  varchar(30),
  channel_name  varchar(30),
  user_id       varchar(10),
  level         varchar(5),
  primary key (channel_type, channel_name, user_id)
);

create table user(
  user_id    varchar(10),
  user_name  varchar(30),
  primary key (user_id)
);

create table alert_summary (
    date            varchar (10), 
    alert_name      varchar (100), 
    cluster         varchar (50), 
    namespace       varchar (50), 
    channel_name    varchar (30), 
    count    integer, 
  primary key ( date, alert_name, cluster, namespace ) 
);
```

## mta-admin mongodb scheme
```
module.exports = mongoose.model('Alert', {
    date: {
        type: String
    },
    cluster: {
        type: String,
        default: ''
    },
    namespace: {
        type: String,
        default: ''
    },
    msg: {
        type: String,
        default: ''
    },
    alert_name: {
        type: String,
        default: ''
    },
    channel: {
        type: String,
        default: ''
    }
});

예:
{
alert_name: ""
alertname: "EKS-MonitoringHeartbeat"
channel: "default"
cluster: "cluster.local"
createdAt: "2020-10-13T08:38:54.596Z"
date: "2020-10-13 17:38:54"
msg: "{"username":"[발생] *EKS-MonitoringHeartbeat* - 5:38:54 PM\n","channel":"default","icon_emoji":":earth_asia:","attachments":[{"color":"#cccccc","fallback":"A-TCL: <https://www.skmta.net|Multi-Tenancy Alert>","pretext":"A-TCL: <https://www.skmta.net|Multi-Tenancy Alert>","fields":[{"title":"*알림명:* `EKS-MonitoringHeartbeat`\n","value":"*시간:* _2020-10-12T23:38:41.379910692Z_\n*심각도:* `none`\n*요약:* Alerting Heartbeat\n*내용:* This is a Hearbeat event meant to ensure that the entire Alerting pipeline is functional.\n","short":false}]}]}"
namespace: "default"
severity: "none"
updatedAt: "2020-10-13T08:38:54.596Z"
value: "{"status":"firing","labels":{"alertname":"EKS-MonitoringHeartbeat","severity":"none"},"annotations":{"description":"This is a Hearbeat event meant to ensure that the entire Alerting pipeline is functional.","summary":"Alerting Heartbeat"},"startsAt":"2020-10-12T23:38:41.379910692Z","endsAt":"0001-01-01T00:00:00Z","generatorURL":"http://prometheus-server-cf45f7746-sxcm9:9090/graph?g0.expr=vector%281%29&g0.tab=1","fingerprint":"3d3224a282c856a7"}"
_id: "5f85679ef8b72f0014a0c396"
}
```
date 필드가 그냥 ```yyyy-MM-dd HH:mm:ss``` 형태의 문자열임.
