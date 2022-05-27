### [kafka connect 사용법]
------------------------------

#### [주의사항!!! docker-compose 파일을 WLS2에서 실행 시 주요 이슈가 발생 부분]

##### 1. docker-compose 실행 path가 windows directory일 경우 volume mount(mysql/kafka/zookeeper) 문제가 발생한다
> 반드시 avoid!! <b>/mnt/c/ </b> 나 <b>/mnt/d/ </b> 에서 실행하지 않는다

##### 2. docker-compose 실행 후 생성되는 kafka/zooker의 권한을 꼭 확인한다
> docker container와 mount되는 path directory가 root권한으로 생성 되는 경우가 많다.
> 꼭 사전에 kafka/logs, zookeeper/data 폴더 생성 후 실행 user 권한으로 변경해준다
```
mkdir -p kafka/logs 
mkdir -p zookeeper/data
chown -R user:user zookeeper kafka
```

##### 3. /etc/hosts를 확인해서 사용할 도메인이 등록되어 있는지 확인한다
```
127.0.0.1 kafka zookeeper
```

##### 4. docker-compose 실행 후 docker container에 mount된 volume 권한을 변경하지 않는다
> docker-compose 재실행 시 권한 문제가 발생한다

![image](https://user-images.githubusercontent.com/30817824/170620369-16000fab-b9e1-47af-b95b-93e1cebf4282.png)


-----------------------



- ##### Kafka Connect : 별도의 개발 없이 Kafka를 통해 Data Source/Destination 간 메세지 송수신을 가능하도록 해주는 솔루션

- ##### Source Connector : Consumer 역할(ex: Debezium=MySQL의 bin log를 읽어서 Kafka로 전송한다)
- ##### Sink Connector : Producer 역할(ex: S3 Sync Connector)

---------------------------

* #### 사전확인사항 :
##### 1) Confluent Hub 접속(https://www.confluent.io/hub/)
##### 2) 화면 좌측 License에서 Free 선택
##### 3) Amazon S3 Sink Connector 설치 (docker-compose에 추가되어 있음)
##### 4) Datagen Source Connector 설치 (Sink Connector 테스트를 할 때 data를 generation 할 수 있음 - 테스트 시 유용)
##### 5) 화면 좌측 Plugin Type 설명 : <br/>
    - Source/Sink 타입을 선택해서 설치 할 수 있다. 
    - (JDBC Connector는 Source/Sink 둘 다 지원) 
    - Transform 타입은 Incoming data가 Kafka Broker에 저장되기 전에 Transform을 할 수 있는 플러그인 (or sink로 전달되기 전에 Transform) 
##### 6) Debezium MySQL CDC Source Connector 설치 (docker-compose에 추가되어 있음)


---------------------------

*  #### 실습진행 :
> 1. ##### <b> /etc/hosts </b> 설정
```
127.0.0.1   kafka1   kafka2  kafka3   connect1  zookeeper1  zookeeper2  zookeeper3  localstack1
```

> 2. ##### WSL2 환경에서 volume mount permission 때문에 container 생성 시 error가 발생하여 disable이 필요하며 아래 폴더를 매번 삭제해줘야 한다
```
rm -rf connectors/ kafka/ localstack/ zookeeper/
```

> 3. ##### docker-compose 파일을 실행한다. (localstack-1은 로컬환경에서 S3를 사용할 수 있게 해주며, 주석처리하고 실제 AWS S3 Bucket을 사용해도 된다)
```
docker-compose -f docker-compose-confluent-connect.yml up
```

> 4. ##### WSL2 환경에서 에러 발생 시 home path의 .docker/config.json 파일을 확인해본다(문제 발생 시 docker desktop 재기동 후 아래처럼 변경)
```  
  "credsStore": "desktop.exe"
  =>
  "credStore": "desktop.exe"

```
> 5. ##### Container를 띄웠으면 MySQL Client 및 AWS CLI를 설치한다 
```
# Toad For MySQL 설치
https://penguincloud.tistory.com/93

# aws cli 설치
https://docs.aws.amazon.com/ko_kr/cli/latest/userguide/install-cliv2-mac.html#cliv2-mac-install-cmd
```

> 6. ##### localstack(S3) 설정 
```
aws configure
aws_access_key_id:  test
aws_secret_access_key : test
region : ap-northeast-2
```

> 7. ##### AWS CLI로 bucket 생성 및 object 업로드/다운로드 
```
aws s3 --endpoint-url=http://localhost:4566 ls

# bucket 생성
aws s3api create-bucket --bucket repdpolex-kafka-2022 --endpoint-url=http://localhost:4566 --region ap-northeast-2 --create-bucket-configuration LocationConstraint=ap-northeast-2

# object 업로드
aws s3api put-object --bucket repdpolex-kafka-2022 --body hello.txt --key hello --endpoint-url=http://localhost:4566

# bucket 내 object list up
aws s3api list-objects --endpoint-url=http://localhost:4566 --bucket repdpolex-kafka-2022

# object 다운로드
aws s3api get-object --endpoint-url=http://localhost:4566 --bucket repdpolex-kafka-2022 --key hello output.txt
```

> 8. ##### mysql connector 등록 ({myip}를 로컬 IP로 변경)
```
curl -v -XPOST -H'Accept:application/json' -H'Content-Type:application/json' http://connect1:18083/connectors \
  -d '
{
    "name": "mysql-source-connector",
    "config": {
        "connector.class": "io.debezium.connector.mysql.MySqlConnector",
        "database.hostname": "{myip}",
        "database.port": "3306",
        "database.user": "root",
        "database.password": "passwd",
        "database.server.id": "1234",
        "database.server.name": "mysql-1",
        "database.include.list": "redpolex",
        "database.history.kafka.bootstrap.servers": "{myip}:19092, {myip}:29092, {myip}:39092",
        "database.history.kafka.topic": "kafka-hk-changes",
        "include.schema.changes": "true",
        "key.converter": "org.apache.kafka.connect.json.JsonConverter",
        "value.converter": "org.apache.kafka.connect.json.JsonConverter",
        "key.converter.schemas.enable": "false",
        "value.converter.schemas.enable": "false"

    }
}'
```

> 9. ##### s3 connector 등록 (s3.bucket.name, s3.region, {myip}, aws.access.key.id, aws.secret.access.key 등을 내 환경에 맞게 변경) <br> 실 AWS S3에 업로드를 원하면 <b>s3_sink_connector_real_s3.txt</b> 파일 참고!!
```
curl -v -XPOST -H'Accept:application/json' -H'Content-Type:application/json' http://connect1:18083/connectors \
  -d '{
    "name": "s3-sink-connector",
    "config": {
      "topics": "mysql-1.redpolex.kafka",
      "connector.class": "io.confluent.connect.s3.S3SinkConnector",
      "flush.size": 1,
      "s3.bucket.name": "repdpolex-kafka-2022",
      "s3.region": "ap-northeast-2",
      "s3.part.size": "5242880",
      "s3.proxy.url": "http://${myip}:4566",
      "format.class": "io.confluent.connect.s3.format.json.JsonFormat",
      "key.converter": "org.apache.kafka.connect.json.JsonConverter",
      "value.converter": "org.apache.kafka.connect.json.JsonConverter",
      "key.converter.schemas.enable": "false",
      "value.converter.schemas.enable": "false",
      "storage.class": "io.confluent.connect.s3.storage.S3Storage",
      "aws.access.key.id": "test",
      "aws.secret.access.key": "test",
      "topics.dir": "topicsdir"
    }
  }'
```

> 10. ##### connector 설정 및 상태 확인
```
# Cluster status
curl -v -XGET -H'Accept: application/json' http://connect1:18083

# ㅊonnectors
curl -v -XGET -H'Accept: application/json' http://connect1:18083/connectors
curl -v -XGET -H'Accept: application/json' 'http://connect1:18083/connectors?expand=status'
curl -v -XGET -H'Accept: application/json' http://connect1:18083/connectors/mysql-source-connector/config

# Connector 상태 확인
curl -v -XGET -H'Accept: application/json' http://connect1:18083/connectors/mysql-source-connector/status

# Pause 상태로 만들기
curl -v -XPUT -H'Accept: application/json' http://connect1:18083/connectors/mysql-source-connector/pause

# 다시 Resume 상태로 만들기
curl -v -XPUT -H'Accept: application/json' http://connect1:18083/connectors/mysql-source-connector/resume

```

> 11. ##### DB Client 툴에서 Table 및 Data를 생성한다.
```
# mysql queries
/* CREATE TABLE kafka (
    student_no int(10) NOT NULL AUTO_INCREMENT PRIMARY KEY,
    name char(10) NOT NULL,
    phone_no char(20)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

INSERT INTO kafka(name, phone_no) VALUES('Sam', '01012345768');
INSERT INTO kafka(name, phone_no) VALUES('Mary', '01022445768');
INSERT INTO kafka(name, phone_no) VALUES('Tom', '0212342132');
INSERT INTO kafka(name, phone_no) VALUES('Susan', '021234423');
INSERT INTO kafka(name, phone_no) VALUES('Joe', '01073219284');

SELECT * FROM kafka;
```

> 12. ##### S3 Bucket에 마지막으로 Insert 한 Joe 관련 Data가 들어가있는지 확인한다(Data의 Key 확인) :
```
aws s3api list-objects --endpoint-url=http://localhost:4566 --bucket repdpolex-kafka-2022
```

> 13. ##### 파일을 S3로부터 다운로드 받아서 내용을 output.json이라는 파일로 저장한다.
```
aws s3api get-object --endpoint-url=http://localhost:4566 --bucket repdpolex-kafka-2022 --key topicsdir/mysql-1.redpolex.kafka/partition=0/mysql-1.redpolex.kafka+0+0000000004.json output.json
```

> 14. ##### 내용을 확인해보면 해당 DB에서 Insert한 Data가 잘 들어가 있다. (op : c = create )
```
{
   "op":"c",
   "before":null,
   "after":{
      "phone_no":"01073219284",
      "student_no":5,
      "name":"Joe"
   },
   "source":{
      "query":null,
      "thread":null,
      "server_id":1234,
      "version":"1.7.0.Final",
      "sequence":null,
      "file":"bin-log.000003",
      "connector":"mysql",
      "pos":2245,
      "name":"mysql-1",
      "gtid":null,
      "row":0,
      "ts_ms":1653462962000,
      "snapshot":"false",
      "db":"redpolex",
      "table":"kafka"
   },
   "ts_ms":1653462962806,
   "transaction":null
}
```

> 15. ##### 이번에는 DB Data를 Update 해본다
```
UPDATE kafka SET phone_no='01077778888' where name='Sam';
```

> 16. ##### S3 Bucket을 확인하고 이전처럼 Key파일을 조회하여 다운받는다. 
```
#조회
aws s3api list-objects --endpoint-url=http://localhost:4566 --bucket repdpolex-kafka-2022

#다운로드
aws s3api get-object --endpoint-url=http://localhost:4566 --bucket repdpolex-kafka-2022 --key topicsdir/mysql-1.redpolex.kafka/partition=0/mysql-1.redpolex.kafka+0+0000000004.json output2.json
```

> 17. ##### output2.json 파일을 확인해본다 (op : u = update )
```
{
   "op":"u",
   "before":{
      "phone_no":"01012345768",
      "student_no":1,
      "name":"Sam"
   },
   "after":{
      "phone_no":"01077778888",
      "student_no":1,
      "name":"Sam"
   },
   "source":{
      "query":null,
      "thread":null,
      "server_id":1234,
      "version":"1.7.0.Final",
      "sequence":null,
      "file":"bin-log.000003",
      "connector":"mysql",
      "pos":2611,
      "name":"mysql-1",
      "gtid":null,
      "row":0,
      "ts_ms":1653463422000,
      "snapshot":"false",
      "db":"redpolex",
      "table":"kafka"
   },
   "ts_ms":1653463422578,
   "transaction":null
}
```
