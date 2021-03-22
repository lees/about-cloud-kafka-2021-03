# about:cloud - Зачем нужны «неуправляемые» топики в Managed Service for Apache Kafka®
В этом репозитории содержатся примеры кода из доклада.

## Полезные ссылки
- [Managed Service for Apache Kafka®](https://cloud.yandex.ru/services/managed-kafka)
- [Kafka Streams Into](https://kafka.apache.org/documentation/streams/)
- [ksqlDB Quickstart](https://ksqldb.io/quickstart.html)
- [Раздел идей](https://cloud.yandex.ru/features)

## Пример разворачивание Apache Kafka® кластера с ksqlDB
### Создаем кластер
- Создание кластера
```bash
yc managed-kafka cluster create --zone-ids ru-central1-b --brokers-count 1 \
--unmanaged-topics --network-name main \
test_cluster
```
- Добавляем админа
```bash
yc managed-kafka user create --cluster-name test_cluster --password=AdminPassword --permission topic="*",role=ACCESS_ROLE_ADMIN admin
```
### Добавляем машину с ksqlDB
- Создаем виртуальную машину в той же сети что и кластер (например на базе Ubuntu)
- Устанавливаем на нее ksqDB
```bash
wget -qO - https://packages.confluent.io/deb/6.0/archive.key | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://packages.confluent.io/deb/6.0 stable main"
sudo apt-get update && sudo apt-get install confluent-community-2.13
```
- Устанавливаем на машину сертификат удостоверяющего центра
```
mkdir -p /usr/local/share/ca-certificates/Yandex
wget "https://storage.yandexcloud.net/cloud-certs/CA.pem" -O /usr/local/share/ca-certificates/Yandex/YandexCA.crt
# Здесь потребуется указать пароль хранилища ключей, которое понадобится потом
keytool -keystore client.truststore.jks -alias CARoot -import -file /usr/local/share/ca-certificates/Yandex/YandexCA.crt
cp client.truststore.jks /etc/ksqldb
```
- Заполняем настройки подключения к кластеру в `/etc/ksqldb/ksql-server.properties`
```
bootstrap.servers=<broker>:9091
sasl.mechanism=SCRAM-SHA-512
security.protocol=SASL_SSL
ssl.truststore.location=/etc/ksqldb/client.truststore.jks
ssl.truststore.password=<пароль хранилища>
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="admin" password="AdminPassword";
```
### Работа в ksqlDB
- Запускаем клиент ksql (с виртуальной машины ksqlDB сервера) и в нем создаем подключение к существующему топику
```sql
CREATE STREAM riderLocations (profileId VARCHAR, latitude DOUBLE, longitude DOUBLE) WITH (kafka_topic='locations', value_format='json', partitions=3);
```
- И создаем запрос к данным
```sql
SELECT * FROM riderLocations WHERE GEO_DISTANCE(latitude, longitude, 37.4133, -122.1162) <= 5 EMIT CHANGES;
```
- Предыдущий запрос показывает данные по мере появления, поэтому мы его не закрываем, а открываем еще одну вкладку терминала и в ней запускаем еще один ksql. В нем мы будем добавлять тестовые данные
```sql
INSERT INTO riderLocations (profileId, latitude, longitude) VALUES ('c2309eec', 37.7877, -122.4205);
INSERT INTO riderLocations (profileId, latitude, longitude) VALUES ('18f4ea86', 37.3903, -122.0643);
INSERT INTO riderLocations (profileId, latitude, longitude) VALUES ('4ab5cbad', 37.3952, -122.0813);
INSERT INTO riderLocations (profileId, latitude, longitude) VALUES ('8b6eae59', 37.3944, -122.0813);
INSERT INTO riderLocations (profileId, latitude, longitude) VALUES ('4a7c7b41', 37.4049, -122.0822);
INSERT INTO riderLocations (profileId, latitude, longitude) VALUES ('4ddad000', 37.7857, -122.4011);
```
- И можем убедиться что данные появились
```bash
root@rc1b-******** ~ # kafkacat -C -b **************:9091 -t locations \
>         -X security.protocol=SASL_SSL -X ssl.ca.location=/opt/yandex/allCAs.pem \
>         -X sasl.mechanisms=SCRAM-SHA-512 \
>         -X sasl.username=admin -X sasl.password=TestPassword
{"PROFILEID":"c2309eec","LATITUDE":37.7877,"LONGITUDE":-122.4205}
{"PROFILEID":"18f4ea86","LATITUDE":37.3903,"LONGITUDE":-122.0643}
{"PROFILEID":"8b6eae59","LATITUDE":37.3944,"LONGITUDE":-122.0813}
{"PROFILEID":"4ddad000","LATITUDE":37.7857,"LONGITUDE":-122.4011}
{"PROFILEID":"4ab5cbad","LATITUDE":37.3952,"LONGITUDE":-122.0813}
{"PROFILEID":"4a7c7b41","LATITUDE":37.4049,"LONGITUDE":-122.0822}
```
- И в первой вкладке видим появившиеся строчки, удовлетворившие условию
```
+--------------------------+--------------------------+---------------------------+
|PROFILEID                 |LATITUDE                  |LONGITUDE                  |
+--------------------------+--------------------------+---------------------------+
|4ab5cbad                  |37.3952                   |-122.0813                  |
|8b6eae59                  |37.3944                   |-122.0813                  |
|4a7c7b41                  |37.4049                   |-122.0822                  |
```
