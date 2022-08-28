# Howto

Just run like below :
```sh
docker-compose up -d
```

# How to test ::

Grant privileges to related user that use from source side
```sql
mysql> grant RELOAD, FLUSH_TABLES, SUPER, REPLICATION CLIENT, REPLICATION SLAVE  on *.* to mysqluser ; commit;
```

Create connector to MySQL source
```sh
curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @source.mysql.orders.json
curl -X GET http://localhost:8083/connectors | jq
```

Check connector status 
```sh
curl -s "http://localhost:8083/connectors"| jq '.[]'| xargs -I{connector_name} curl -s "http://localhost:8083/connectors/"{connector_name}"/status"| jq -c -M '[.name,.connector.state,.tasks[].state]|join(":|:")'
```

List establish topics
```sh
docker exec -i demo-kafka1 /kafka/bin/kafka-topics.sh --bootstrap-server broker:9092 --list
```

Try list existing stream from source into the topic
```sh
docker exec -i demo-kafka1 /kafka/bin/kafka-console-consumer.sh --bootstrap-server broker:9092 --topic orders --property schema.registry.url=http://localhost:8081  --from-beginning 
```


Try list existing stream from source into the topic
```sh
docker exec -i demo-kafka1 /kafka/bin/kafka-console-consumer.sh --bootstrap-server broker:9092 --topic orders --property schema.registry.url=http://localhost:8081  --from-beginning 
```

Do some kind of DML statement, above should show sumthing
```sh
mysql -u root -phahalala -h 127.0.0.1 -P 3306 inventory -e 'insert into orders values (default, current_timestamp(), 1001, 1,102);'
```


# Validating on KSQL ::

Log into KSQL service
```sh
docker exec -ti demo-ksqldb-cli ksql http://ksqldb-server:8088
```

Play around
```sql
ksql> show topics; -- Show list of topics

ksql> set 'auto.offset.reset' = 'earliest'; -- Set da ta streaming viewable since day 1

ksql> create stream raw_orders with (kafka_topic='orders', partitions=1, value_format='avro');  -- Replicate streaming topic into a stream event

ksql> select * from raw_orders emit changes limit 3 ;

ksql> create stream raw_orders2 as select order_number oid, AS_VALUE(order_number) order_number,  order_date, purchaser, quantity, product_id from raw_orders a emit changes ;

ksql> create stream latest_ord as
select oid,
LATEST_BY_OFFSET(order_number) order_number,  LATEST_BY_OFFSET(order_date) order_date, LATEST_BY_OFFSET(purchaser) purchaser, LATEST_BY_OFFSET(quantity) quantity, LATEST_BY_OFFSET(product_id) product_id
from raw_orders2 group by oid emit changes ;  -- Create stream for latest value changes

ksql> create table latest_ord_output with (kafka_topic='latest_ord_output', partitions=1, value_format='json') as
select * from latest_ord emit changes ;  -- Expose the result to another topic for consumer side

```

Test pull a consumer, should hav data changes realtime
```sh
docker exec -i demo-kafka1 /kafka/bin/kafka-console-consumer.sh --bootstrap-server broker:9092     --topic latest_ord_output --property schema.registry.url=http://localhost:8081  --from-beginning
```
