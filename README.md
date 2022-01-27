# debezium-test

DebeziumのTutorialとibm-messagingで提供されているJDBC Sink Connectorを使って、MySQL -> Db2のデータ連携を実装してみる

https://debezium.io/documentation/reference/1.8/tutorial.html

https://github.com/ibm-messaging/kafka-connect-jdbc-sink

## Usage

`docker-compose up` で起動して、Source ConnectorのSMTあり2のConnectorとSink Connectorを登録してあげればOK

※Db2がちゃんと上がりきるのを待って実行すること


## Source Connector

### SMTなし

登録

curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d @source-connector.json

出力メッセージの確認

./kafka-console-consumer.sh --topic dbserver1.inventory.customers --from-beginning --bootstrap-server kafka:9092 --property print.key=true

Key
```json
{"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"}],"optional":false,"name":"dbserver1.inventory.customers.Key"},"payload":{"id":1001}}
```

Value
```json
{"schema":{"type":"struct","fields":[{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":false,"field":"first_name"},{"type":"string","optional":false,"field":"last_name"},{"type":"string","optional":false,"field":"email"}],"optional":true,"name":"dbserver1.inventory.customers.Value","field":"before"},{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":false,"field":"first_name"},{"type":"string","optional":false,"field":"last_name"},{"type":"string","optional":false,"field":"email"}],"optional":true,"name":"dbserver1.inventory.customers.Value","field":"after"},{"type":"struct","fields":[{"type":"string","optional":false,"field":"version"},{"type":"string","optional":false,"field":"connector"},{"type":"string","optional":false,"field":"name"},{"type":"int64","optional":false,"field":"ts_ms"},{"type":"string","optional":true,"name":"io.debezium.data.Enum","version":1,"parameters":{"allowed":"true,last,false,incremental"},"default":"false","field":"snapshot"},{"type":"string","optional":false,"field":"db"},{"type":"string","optional":true,"field":"sequence"},{"type":"string","optional":true,"field":"table"},{"type":"int64","optional":false,"field":"server_id"},{"type":"string","optional":true,"field":"gtid"},{"type":"string","optional":false,"field":"file"},{"type":"int64","optional":false,"field":"pos"},{"type":"int32","optional":false,"field":"row"},{"type":"int64","optional":true,"field":"thread"},{"type":"string","optional":true,"field":"query"}],"optional":false,"name":"io.debezium.connector.mysql.Source","field":"source"},{"type":"string","optional":false,"field":"op"},{"type":"int64","optional":true,"field":"ts_ms"},{"type":"struct","fields":[{"type":"string","optional":false,"field":"id"},{"type":"int64","optional":false,"field":"total_order"},{"type":"int64","optional":false,"field":"data_collection_order"}],"optional":true,"field":"transaction"}],"optional":false,"name":"dbserver1.inventory.customers.Envelope"},"payload":{"before":null,"after":{"id":1001,"first_name":"Sally","last_name":"Thomas","email":"sally.thomas@acme.com"},"source":{"version":"1.8.0.Final","connector":"mysql","name":"dbserver1","ts_ms":1643250617084,"snapshot":"true","db":"inventory","sequence":null,"table":"customers","server_id":0,"gtid":null,"file":"mysql-bin.000003","pos":156,"row":0,"thread":null,"query":null},"op":"r","ts_ms":1643250617086,"transaction":null}}
```

このデータ・フォーマットだと、JDBC Sink Connector側が期待しているフォーマットと異なるためNG

登録削除

curl -X DELETE http://localhost:8083/connectors/source-connector

### SMTあり

DebeziumのNew Record State Extractionを利用

https://debezium.io/documentation/reference/stable/transformations/event-flattening.html

source-connector-smt.json
```json
{
    "name": "source-connector-smt",
    "config": {
        "connector.class": "io.debezium.connector.mysql.MySqlConnector",
        "tasks.max": "1",
        "database.hostname": "mysql",
        "database.port": "3306",
        "database.user": "debezium",
        "database.password": "dbz",
        "database.server.id": "184054",
        "database.server.name": "dbserver1",
        "database.include.list": "inventory",
        "database.history.kafka.bootstrap.servers": "kafka:9092",
        "database.history.kafka.topic": "dbhistory.inventory",
        "transforms": "unwrap",
        "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState"
    }
}
```

curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d @source-connector-smt.json

メッセージ確認

Key
```json
{"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"}],"optional":false,"name":"dbserver1.inventory.customers.Key"},"payload":{"id":1001}}
```

Value
```json
{
	"schema": {
		"type": "struct",
		"fields": [
			{
				"type": "int32",
				"optional": false,
				"field": "id"
			},
			{
				"type": "string",
				"optional": false,
				"field": "first_name"
			},
			{
				"type": "string",
				"optional": false,
				"field": "last_name"
			},
			{
				"type": "string",
				"optional": false,
				"field": "email"
			}
		],
		"optional": false,
		"name": "dbserver1.inventory.customers.Value"
	},
	"payload": {
		"id": 1001,
		"first_name": "Sally",
		"last_name": "Thomas",
		"email": "sally.thomas@acme.com"
	}
}
```

このフォーマットであればOKと思いきや、JDBC Sink Connectorは勝手にデータ項目「id」を追加する作りになってて、Source側の「id」と項目名が重複してしまうため、これもNG。。

curl -X DELETE http://localhost:8083/connectors/source-connector-smt


### SMTあり2

Kafka Connectで提供されているReplaceFieldも併用

https://kafka.apache.org/documentation/#connect_transforms

```json
{
    "name": "source-connector-smt2",
    "config": {
        "connector.class": "io.debezium.connector.mysql.MySqlConnector",
        "tasks.max": "1",
        "database.hostname": "mysql",
        "database.port": "3306",
        "database.user": "debezium",
        "database.password": "dbz",
        "database.server.id": "184054",
        "database.server.name": "dbserver1",
        "database.include.list": "inventory",
        "database.history.kafka.bootstrap.servers": "kafka:9092",
        "database.history.kafka.topic": "dbhistory.inventory",
        "transforms": "unwrap,replace",
        "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
        "transforms.replace.type": "org.apache.kafka.connect.transforms.ReplaceField$Value",
        "transforms.replace.renames": "id:id_org"
    }
}
```

Source側のidをid_orgに変換する処理を追加


curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d @source-connector-smt2.json

出力メッセージ
```json
{"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"}],"optional":false,"name":"dbserver1.inventory.customers.Key"},"payload":{"id":1001}}	{"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id_org"},{"type":"string","optional":false,"field":"first_name"},{"type":"string","optional":false,"field":"last_name"},{"type":"string","optional":false,"field":"email"}],"optional":false,"name":"dbserver1.inventory.customers.Value"},"payload":{"id_org":1001,"first_name":"Sally","last_name":"Thomas","email":"sally.thomas@acme.com"}}
{"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"}],"optional":false,"name":"dbserver1.inventory.customers.Key"},"payload":{"id":1002}}	{"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id_org"},{"type":"string","optional":false,"field":"first_name"},{"type":"string","optional":false,"field":"last_name"},{"type":"string","optional":false,"field":"email"}],"optional":false,"name":"dbserver1.inventory.customers.Value"},"payload":{"id_org":1002,"first_name":"George","last_name":"Bailey","email":"gbailey@foobar.com"}}
{"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"}],"optional":false,"name":"dbserver1.inventory.customers.Key"},"payload":{"id":1003}}	{"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id_org"},{"type":"string","optional":false,"field":"first_name"},{"type":"string","optional":false,"field":"last_name"},{"type":"string","optional":false,"field":"email"}],"optional":false,"name":"dbserver1.inventory.customers.Value"},"payload":{"id_org":1003,"first_name":"Edward","last_name":"Walker","email":"ed@walker.com"}}
{"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"}],"optional":false,"name":"dbserver1.inventory.customers.Key"},"payload":{"id":1004}}	{"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id_org"},{"type":"string","optional":false,"field":"first_name"},{"type":"string","optional":false,"field":"last_name"},{"type":"string","optional":false,"field":"email"}],"optional":false,"name":"dbserver1.inventory.customers.Value"},"payload":{"id_org":1004,"first_name":"Anne","last_name":"Kretchmar","email":"annek@noanswer.org"}}
```

最後のメッセージのValueを整形
```json
{
	"schema": {
		"type": "struct",
		"fields": [
			{
				"type": "int32",
				"optional": false,
				"field": "id_org"
			},
			{
				"type": "string",
				"optional": false,
				"field": "first_name"
			},
			{
				"type": "string",
				"optional": false,
				"field": "last_name"
			},
			{
				"type": "string",
				"optional": false,
				"field": "email"
			}
		],
		"optional": false,
		"name": "dbserver1.inventory.customers.Value"
	},
	"payload": {
		"id_org": 1004,
		"first_name": "Anne",
		"last_name": "Kretchmar",
		"email": "annek@noanswer.org"
	}
}
```

このフォーマットであればJDBC Sink Connectorは処理できる

## Sink Connector

登録

curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d @jdbc-sink-connector.json

## Kafka

```
docker exec -it kafka bash
```

```
$ cd bin
```

トピック確認

```
$ ./kafka-topics.sh --bootstrap-server kafka:9092 --list
__consumer_offsets
dbhistory.inventory
dbserver1
dbserver1.inventory.addresses
dbserver1.inventory.customers
dbserver1.inventory.geom
dbserver1.inventory.orders
dbserver1.inventory.products
dbserver1.inventory.products_on_hand
my_connect_configs
my_connect_offsets
my_connect_statuses
```

メッセージ確認

```
$ ./kafka-console-consumer.sh --topic dbserver1.inventory.customers --from-beginning --bootstrap-server kafka:9092 --property print.key=true
```


## Db2

docker exec -it db2 bash

su - db2inst1

db2 connect to MYDB

```
db2 list tables

Table/View                      Schema          Type  Creation time             
------------------------------- --------------- ----- --------------------------
CUSTOMERS                       DB2INST1        T     2022-01-27-05.45.14.055391
```

```
db2 describe table customers

                                Data type                     Column
Column name                     schema    Data type name      Length     Scale Nulls
------------------------------- --------- ------------------- ---------- ----- ------
ID                              SYSIBM    INTEGER                      4     0 No    
ID_ORG                          SYSIBM    INTEGER                      4     0 No    
FIRST_NAME                      SYSIBM    VARCHAR                    255     0 No    
LAST_NAME                       SYSIBM    VARCHAR                    255     0 No    
EMAIL                           SYSIBM    VARCHAR                    255     0 No    

  5 record(s) selected.
```

```
db2 "select * from customers"

ID          ID_ORG      FIRST_NAME                                                                                                                                                                                                                                                      LAST_NAME                                                                                                                                                                                                                                                       EMAIL                                                                                                                                                                                                                                                          
----------- ----------- --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
          1        1001 Sally                                                                                                                                                                                                                                                           Thomas                                                                                                                                                                                                                                                          sally.thomas@acme.com                                                                                                                                                                                                                                          
          2        1002 George                                                                                                                                                                                                                                                          Bailey                                                                                                                                                                                                                                                          gbailey@foobar.com                                                                                                                                                                                                                                             
          3        1003 Edward                                                                                                                                                                                                                                                          Walker                                                                                                                                                                                                                                                          ed@walker.com                                                                                                                                                                                                                                                  
          4        1004 Anne                                                                                                                                                                                                                                                            Kretchmar                                                                                                                                                                                                                                                       annek@noanswer.org                                                                                                                                                                                                                                             

  4 record(s) selected.
```


