# Description
This tutorial demonstrates how to set up Debezium with the Unwrap Single Message Transform (SMT) using Podman. The source database is MySQL, and the sink database is PostgreSQL. Any changes made to the MySQL database are replicated to the PostgreSQL database, including inserts, updates, and deletes.

# Prerequisites
- Podman
- Git

# Steps
## Setting up Debezium with Unwrap SMT
1. Clone the Debezium examples repository from GitHub:
```bash
git clone https://github.com/debezium/debezium-examples
```

2. Change the directory to the Unwrap SMT example:
```bash
cd debezium-examples/unwrap-smt
```

3. Set the Debezium version as an environment variable:
```bash
export DEBEZIUM_VERSION=2.1
```

4. Create a Pod for Zookeeper, Kafka, MySQL, and PostgreSQL:
```bash
podman pod create --name debezium-pod -p 8083:8083
```

5. Start Zookeeper:
```bash
podman run --rm -d --pod debezium-pod --name zookeeper quay.io/debezium/zookeeper:${DEBEZIUM_VERSION}
```

6. Start Kafka:
```bash
podman run --rm -d --pod debezium-pod --name kafka -e ZOOKEEPER_CONNECT=zookeeper:2181 -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092 -e KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092 -e ZOOKEEPER_CONNECT=zookeeper:2181 quay.io/debezium/kafka:${DEBEZIUM_VERSION}
```

7. Start MySQL:
```bash
podman run --rm -d --pod debezium-pod --name mysql -e MYSQL_ROOT_PASSWORD=debezium -e MYSQL_USER=mysqluser -e MYSQL_PASSWORD=mysqlpw quay.io/debezium/example-mysql:${DEBEZIUM_VERSION}
```

8. Start PostgreSQL:
```bash
podman run --rm -d --pod debezium-pod --name postgres -e POSTGRES_USER=postgresuser -e POSTGRES_PASSWORD=postgrespw -e POSTGRES_DB=inventory quay.io/debezium/postgres:9.6
```

9. Build the custom Kafka Connect image:
```bash
podman build -t debezium/connect-jdbc-es:${DEBEZIUM_VERSION} debezium-jdbc-es --build-arg DEBEZIUM_VERSION=${DEBEZIUM_VERSION}
```

9. Start Kafka Connect:
```bash
podman run -d --pod debezium-pod --name connect -e BOOTSTRAP_SERVERS=kafka:9092 -e GROUP_ID=1 -e CONFIG_STORAGE_TOPIC=my_connect_configs -e OFFSET_STORAGE_TOPIC=my_connect_offsets -e STATUS_STORAGE_TOPIC=my_source_connect_statuses debezium/connect-jdbc-es:${DEBEZIUM_VERSION}
```

10. Switch to the old version of the `jdbc-sink.json` file:
```bash
git checkout c94b3b1 -- jdbc-sink.json
```

10. Start PostgreSQL connector:
```bash
curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @jdbc-sink.json
```

11. Start MySQL connector:
```bash
curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @source.json
```
## Testing the Unwrap SMT
### Read the contents of the MySQL and PostgreSQL databases
Read the contents of the MySQL database:
```bash
podman exec mysql bash -c 'mysql -u $MYSQL_USER  -p$MYSQL_PASSWORD inventory -e "select * from customers"'
```
Read the contents of the PostgreSQL database:
```bash
podman exec postgres bash -c 'psql -U $POSTGRES_USER $POSTGRES_DB -c "select * from customers"'
```
### Create a new record in the MySQL database
1. Create a new record in the MySQL database:
```bash
podman exec mysql bash -c 'mysql -u $MYSQL_USER  -p$MYSQL_PASSWORD inventory -e "INSERT INTO customers VALUES (1005, \"John\", \"Doe\", \"john.doe@example.com\")"'
```
2. Verify that PostgreSQL contains the new record:
```bash
podman exec postgres bash -c 'psql -U $POSTGRES_USER $POSTGRES_DB -c "select * from customers"'
```

### Update a record in the MySQL database
1. Update a record in the MySQL database:
```bash
podman exec mysql bash -c 'mysql -u $MYSQL_USER  -p$MYSQL_PASSWORD inventory -e "UPDATE customers SET first_name = \"Jane\" WHERE id = 1005"'
```

2. Verify that PostgreSQL contains the updated record:
```bash
podman exec postgres bash -c 'psql -U $POSTGRES_USER $POSTGRES_DB -c "select * from customers"'
```

### Delete a record in the MySQL database
1. Delete a record in the MySQL database:
```bash
podman exec mysql bash -c 'mysql -u $MYSQL_USER  -p$MYSQL_PASSWORD inventory -e "DELETE FROM customers WHERE id = 1005"'
```

2. Verify that PostgreSQL does not contain the deleted record:
```bash
podman exec postgres bash -c 'psql -U $POSTGRES_USER $POSTGRES_DB -c "select * from customers"'
```

# Cleanup
To stop and remove the Pod:
```bash
podman pod kill debezium-pod
podman pod rm debezium-pod
```
