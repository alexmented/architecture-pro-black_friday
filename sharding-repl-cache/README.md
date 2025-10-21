### 1. Запуск контейнеров

```shell
docker compose up -d
```

### 2. Инициализация сервера конфигурации

```shell
docker compose exec -T configSrv mongosh --port 27017 --quiet <<EOF
rs.initiate(
  {
    _id : "config_server",
    configsvr: true,
    members: [
      { _id : 0, host : "configSrv:27017" }
    ]
  }
);
EOF
```

### 3. Инициализация набора реплик для Шарда 1

```shell
docker compose exec -T shard1-replica1 mongosh --port 27018 --quiet <<EOF
rs.initiate(
  {
    _id : "shard1",
    members: [
      { _id : 0, host : "shard1-replica1:27018" },
      { _id : 1, host : "shard1-replica2:27021" },
      { _id : 2, host : "shard1-replica3:27022" }
    ]
  }
);
EOF
```

### 4. Инициализация набора реплик для Шарда 2

```shell
docker compose exec -T shard2-replica1 mongosh --port 27019 --quiet <<EOF
rs.initiate(
  {
    _id : "shard2",
    members: [
      { _id : 0, host : "shard2-replica1:27019" },
      { _id : 1, host : "shard2-replica2:27023" },
      { _id : 2, host : "shard2-replica3:27024" }
    ]
  }
);
EOF
```

### 5. Инициализация роутера и настройка шардирования


```shell
docker compose exec -T mongos_router mongosh --port 27020 --quiet <<EOF
sh.addShard("shard1/shard1-replica1:27018,shard1-replica2:27021,shard1-replica3:27022");
sh.addShard("shard2/shard2-replica1:27019,shard2-replica2:27023,shard2-replica3:27024");
sh.enableSharding("somedb");
sh.shardCollection("somedb.helloDoc", { "name" : "hashed" });
EOF
```

### 6. Заполнение тестовыми данными

```shell
docker compose exec -T mongos_router mongosh --port 27020 --quiet <<EOF
use somedb
for(var i = 0; i < 1000; i++) db.helloDoc.insert({age:i, name:"ly"+i})
db.helloDoc.countDocuments()
EOF
```

### Проверка работы кеша

```bash
curl -w "\nTime: %{time_total}s\n" http://localhost:8080/helloDoc/users
```


