# debezium-test

DebeziumのTutorialとibm-messagingで提供されているJDBC Sink Connectorを使って、MySQL -> Db2のデータ連携を実装してみる

https://debezium.io/documentation/reference/1.8/tutorial.html

https://github.com/ibm-messaging/kafka-connect-jdbc-sink


# Source Connector

## SMTなし

curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d @source-connector.json

出力メッセージ
./kafka-console-consumer.sh --topic dbserver1.inventory.customers --from-beginning --bootstrap-server kafka:9092 --property print.key=true

Key
```json
{"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"}],"optional":false,"name":"dbserver1.inventory.customers.Key"},"payload":{"id":1001}}
```
Value
```json
{"schema":{"type":"struct","fields":[{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":false,"field":"first_name"},{"type":"string","optional":false,"field":"last_name"},{"type":"string","optional":false,"field":"email"}],"optional":true,"name":"dbserver1.inventory.customers.Value","field":"before"},{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":false,"field":"first_name"},{"type":"string","optional":false,"field":"last_name"},{"type":"string","optional":false,"field":"email"}],"optional":true,"name":"dbserver1.inventory.customers.Value","field":"after"},{"type":"struct","fields":[{"type":"string","optional":false,"field":"version"},{"type":"string","optional":false,"field":"connector"},{"type":"string","optional":false,"field":"name"},{"type":"int64","optional":false,"field":"ts_ms"},{"type":"string","optional":true,"name":"io.debezium.data.Enum","version":1,"parameters":{"allowed":"true,last,false,incremental"},"default":"false","field":"snapshot"},{"type":"string","optional":false,"field":"db"},{"type":"string","optional":true,"field":"sequence"},{"type":"string","optional":true,"field":"table"},{"type":"int64","optional":false,"field":"server_id"},{"type":"string","optional":true,"field":"gtid"},{"type":"string","optional":false,"field":"file"},{"type":"int64","optional":false,"field":"pos"},{"type":"int32","optional":false,"field":"row"},{"type":"int64","optional":true,"field":"thread"},{"type":"string","optional":true,"field":"query"}],"optional":false,"name":"io.debezium.connector.mysql.Source","field":"source"},{"type":"string","optional":false,"field":"op"},{"type":"int64","optional":true,"field":"ts_ms"},{"type":"struct","fields":[{"type":"string","optional":false,"field":"id"},{"type":"int64","optional":false,"field":"total_order"},{"type":"int64","optional":false,"field":"data_collection_order"}],"optional":true,"field":"transaction"}],"optional":false,"name":"dbserver1.inventory.customers.Envelope"},"payload":{"before":null,"after":{"id":1001,"first_name":"Sally","last_name":"Thomas","email":"sally.thomas@acme.com"},"source":{"version":"1.8.0.Final","connector":"mysql","name":"dbserver1","ts_ms":1643250617084,"snapshot":"true","db":"inventory","sequence":null,"table":"customers","server_id":0,"gtid":null,"file":"mysql-bin.000003","pos":156,"row":0,"thread":null,"query":null},"op":"r","ts_ms":1643250617086,"transaction":null}}
```

curl -X DELETE http://localhost:8083/connectors/inventory-connector

## SMTあり

DebeziumのNew Record State Extractionを利用

https://debezium.io/documentation/reference/stable/transformations/event-flattening.html

source-connector-smt.json
```json
{
    "name": "inventory-connector2",
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

curl -X DELETE http://localhost:8083/connectors/inventory-connector2



curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d @source-connector-smt2.json


{"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"}],"optional":false,"name":"dbserver1.inventory.customers.Key"},"payload":{"id":1001}}	{"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id_org"},{"type":"string","optional":false,"field":"first_name"},{"type":"string","optional":false,"field":"last_name"},{"type":"string","optional":false,"field":"email"}],"optional":false,"name":"dbserver1.inventory.customers.Value"},"payload":{"id_org":1001,"first_name":"Sally","last_name":"Thomas","email":"sally.thomas@acme.com"}}
{"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"}],"optional":false,"name":"dbserver1.inventory.customers.Key"},"payload":{"id":1002}}	{"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id_org"},{"type":"string","optional":false,"field":"first_name"},{"type":"string","optional":false,"field":"last_name"},{"type":"string","optional":false,"field":"email"}],"optional":false,"name":"dbserver1.inventory.customers.Value"},"payload":{"id_org":1002,"first_name":"George","last_name":"Bailey","email":"gbailey@foobar.com"}}
{"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"}],"optional":false,"name":"dbserver1.inventory.customers.Key"},"payload":{"id":1003}}	{"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id_org"},{"type":"string","optional":false,"field":"first_name"},{"type":"string","optional":false,"field":"last_name"},{"type":"string","optional":false,"field":"email"}],"optional":false,"name":"dbserver1.inventory.customers.Value"},"payload":{"id_org":1003,"first_name":"Edward","last_name":"Walker","email":"ed@walker.com"}}
{"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"}],"optional":false,"name":"dbserver1.inventory.customers.Key"},"payload":{"id":1004}}	{"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id_org"},{"type":"string","optional":false,"field":"first_name"},{"type":"string","optional":false,"field":"last_name"},{"type":"string","optional":false,"field":"email"}],"optional":false,"name":"dbserver1.inventory.customers.Value"},"payload":{"id_org":1004,"first_name":"Anne","last_name":"Kretchmar","email":"annek@noanswer.org"}}


# Sink Connector

登録
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d @jdbc-sink-connector.json






