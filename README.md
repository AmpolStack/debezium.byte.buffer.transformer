## 🇬🇧 English

### Introduction

**Debezium** is a ***Change Data Capture (CDC)*** tool, commonly used to implement the **Outbox Pattern** in microservice or event-driven architectures.

It typically integrates with **Kafka** as a *Message Broker*, using ***JSON*** as the default message format.
However, many modern systems use binary serialization formats such as ***Avro***, ***Protobuf***, or any binary representation (`byte[]`).

Here lies a challenge: **Debezium** treats `BLOB`, `BINARY`, and `VARBINARY` fields as `ByteBuffer` objects.
This solution extends Debezium to natively convert those binary types, enabling seamless binary communication.

---

### 🚀 Implementation Steps

#### 1️⃣ Clone and build

```bash
./mvnw clean package
```

This command will generate a **`.jar-with-dependencies`** file containing the transformer that converts `ByteBuffer` to `byte[]`.

---

#### 2️⃣ Include the transformer in Debezium

If using Docker, create a custom image based on **Debezium Connect**, placing the generated JAR inside `kafka/connect`.

> 💡 You may also mount a local volume, but a custom image is preferred for production environments.

##### Example `Dockerfile`

```Dockerfile
FROM debezium/connect:2.7.3.Final

# ====== (Optional for future implementations) AVRO Converter Dependencies ======
RUN mkdir -p /kafka/connect/avro-converter

# Download the necessary jars for Avro Converter from Confluence
RUN cd /kafka/connect/avro-converter && \
    curl -fSL https://packages.confluent.io/maven/io/confluent/kafka-connect-avro-converter/8.0.0/kafka-connect-avro-converter-8.0.0.jar -o kafka-connect-avro-converter-8.0.0.jar && \
    curl -fSL https://packages.confluent.io/maven/io/confluent/kafka-connect-avro-data/8.0.0/kafka-connect-avro-data-8.0.0.jar -o kafka-connect-avro-data-8.0.0.jar && \
    curl -fSL https://packages.confluent.io/maven/io/confluent/kafka-avro-serializer/8.0.0/kafka-avro-serializer-8.0.0.jar -o kafka-avro-serializer-8.0.0.jar && \
    curl -fSL https://packages.confluent.io/maven/io/confluent/kafka-schema-serializer/8.0.0/kafka-schema-serializer-8.0.0.jar -o kafka-schema-serializer-8.0.0.jar && \
    curl -fSL https://packages.confluent.io/maven/io/confluent/kafka-schema-converter/8.0.0/kafka-schema-converter-8.0.0.jar -o kafka-schema-converter-8.0.0.jar && \
    curl -fSL https://packages.confluent.io/maven/io/confluent/kafka-schema-registry-client/8.0.0/kafka-schema-registry-client-8.0.0.jar -o kafka-schema-registry-client-8.0.0.jar && \
    curl -fSL https://packages.confluent.io/maven/io/confluent/common-config/8.0.0/common-config-8.0.0.jar -o common-config-8.0.0.jar && \
    curl -fSL https://packages.confluent.io/maven/io/confluent/common-utils/8.0.0/common-utils-8.0.0.jar -o common-utils-8.0.0.jar && \
    curl -fSL https://repo1.maven.org/maven2/com/google/guava/guava/32.1.2-jre/guava-32.1.2-jre.jar -o guava-32.1.2-jre.jar && \
    curl -fSL https://repo1.maven.org/maven2/org/apache/avro/avro/1.12.1/avro-1.12.1.jar -o avro-1.12.1.jar


RUN mkdir -p /kafka/connect/custom
COPY ByteBufferToBytesTransform-1.0-jar-with-dependencies.jar /kafka/connect/custom/

ENV CONNECT_PLUGIN_PATH="/kafka/connect,/kafka/connect/avro-converter,/kafka/connect/custom"
ENV CONNECT_REST_ADVERTISED_HOST_NAME=kafka-connect
```

---

#### 3️⃣ Configure the connector

```json
{
  "name": "mariadb-outbox-connector",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "tasks.max": "1",

    "database.hostname": "<<your-database-host>>",
    "database.port": "<<your-database-port>>",
    "database.user": "<<your-username>>",
    "database.password": "<<your-password>>",
    "database.server.name": "<<your-server-name>>",

    "database.include.list": "<<your-database-name>>",
    "table.include.list": "<<your-database-name>>.outbox_event",
    "topic.prefix": "<<your-topic-prefix>>",

    "schema.history.internal.kafka.bootstrap.servers": "<<your-brokers>>",
    "schema.history.internal.kafka.topic": "<<your-schema-history-topic>>",

    "snapshot.mode": "schema_only",

    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "value.converter": "org.apache.kafka.connect.converters.ByteArrayConverter",

    "transforms": "convertBytes,outbox",

    "transforms.convertBytes.type": "com.yourorg.ByteBufferToBytesTransform",
    "transforms.convertBytes.fields": "payload",

    "transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter",
    "transforms.outbox.route.by.field": "topic",
    "transforms.outbox.route.topic.replacement": "${routedByValue}"
  }
}
```

---

#### 4️⃣ Register connector

```bash
curl -X POST -H "Content-Type: application/json" \
     --data @connector-config.json \
     http://<<your-connect-host>>:8083/connectors
```

#### 5️⃣ Check status

```bash
curl -X GET http://<<your-connect-host>>:8083/connectors/mariadb-outbox-connector/status
```

---

## 🇪🇸 Español

### Introducción

**Debezium** es una herramienta de ***Change Data Capture (CDC)***, útil para implementar el **patrón Outbox** en arquitecturas de microservicios o sistemas orientados a eventos.

Usualmente se integra con **Kafka** como *Message Broker*, comunicándose por defecto mediante el formato ***JSON***.
Sin embargo, en muchos entornos modernos se emplean otros formatos de serialización como ***Avro***, ***Protobuf*** o representaciones binarias (`byte[]`).

Aquí surge un problema: **Debezium** interpreta los campos de tipo `BLOB`, `BINARY` y `VARBINARY` como objetos `ByteBuffer`.
Esta solución propone extender Debezium para manejar de forma nativa estos tipos binarios, facilitando una comunicación directa y transparente con formatos binarios.

---

### 🚀 Pasos para la implementación

#### 1️⃣ Clonar el repositorio y compilar

Ejecuta el empaquetado con el wrapper proporcionado en el proyecto:

```bash
./mvnw clean package
```

Esto generará un archivo **`.jar-with-dependencies`**, que contiene el transformador encargado de convertir los `ByteBuffer` en arreglos de bytes (`byte[]`).

---

#### 2️⃣ Incluir el transformador en Debezium

Si usas Docker, debes crear una imagen personalizada basada en **Debezium Connect**, incluyendo el `.jar` generado dentro de la carpeta `kafka/connect`.

> 💡 *También puedes montar un volumen local que contenga el JAR, pero se recomienda la construcción de una imagen personalizada para entornos productivos.*

##### Ejemplo de `Dockerfile`

```Dockerfile
# Imagen base oficial de Debezium Connect
FROM debezium/connect:2.7.3.Final

# ====== (Opcional para futuras implementaciones) Dependencias para AVRO Converter ======
RUN mkdir -p /kafka/connect/avro-converter

# Descargar los jars necesarios para Avro Converter en Concluence
RUN cd /kafka/connect/avro-converter && \
    curl -fSL https://packages.confluent.io/maven/io/confluent/kafka-connect-avro-converter/8.0.0/kafka-connect-avro-converter-8.0.0.jar -o kafka-connect-avro-converter-8.0.0.jar && \
    curl -fSL https://packages.confluent.io/maven/io/confluent/kafka-connect-avro-data/8.0.0/kafka-connect-avro-data-8.0.0.jar -o kafka-connect-avro-data-8.0.0.jar && \
    curl -fSL https://packages.confluent.io/maven/io/confluent/kafka-avro-serializer/8.0.0/kafka-avro-serializer-8.0.0.jar -o kafka-avro-serializer-8.0.0.jar && \
    curl -fSL https://packages.confluent.io/maven/io/confluent/kafka-schema-serializer/8.0.0/kafka-schema-serializer-8.0.0.jar -o kafka-schema-serializer-8.0.0.jar && \
    curl -fSL https://packages.confluent.io/maven/io/confluent/kafka-schema-converter/8.0.0/kafka-schema-converter-8.0.0.jar -o kafka-schema-converter-8.0.0.jar && \
    curl -fSL https://packages.confluent.io/maven/io/confluent/kafka-schema-registry-client/8.0.0/kafka-schema-registry-client-8.0.0.jar -o kafka-schema-registry-client-8.0.0.jar && \
    curl -fSL https://packages.confluent.io/maven/io/confluent/common-config/8.0.0/common-config-8.0.0.jar -o common-config-8.0.0.jar && \
    curl -fSL https://packages.confluent.io/maven/io/confluent/common-utils/8.0.0/common-utils-8.0.0.jar -o common-utils-8.0.0.jar && \
    curl -fSL https://repo1.maven.org/maven2/com/google/guava/guava/32.1.2-jre/guava-32.1.2-jre.jar -o guava-32.1.2-jre.jar && \
    curl -fSL https://repo1.maven.org/maven2/org/apache/avro/avro/1.12.1/avro-1.12.1.jar -o avro-1.12.1.jar

# ====== Añadir el transformador personalizado ======
RUN mkdir -p /kafka/connect/custom

# Copia el archivo JAR generado desde tu build local
COPY ByteBufferToBytesTransform-1.0-jar-with-dependencies.jar /kafka/connect/custom/

# ====== Configuración de los plugins ======
ENV CONNECT_PLUGIN_PATH="/kafka/connect,/kafka/connect/avro-converter,/kafka/connect/custom"

# (Opcional) Hostname del servicio REST de Kafka Connect
ENV CONNECT_REST_ADVERTISED_HOST_NAME=kafka-connect
```

---

#### 3️⃣ Configurar el conector Debezium

Ejemplo de configuración para **MariaDB + Outbox Pattern**.

> 🔧 Reemplaza los valores entre `<< >>` por tus propios datos.

```json
{
  "name": "mariadb-outbox-connector",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "tasks.max": "1",

    "database.hostname": "<<your-database-host>>",
    "database.port": "<<your-database-port>>",
    "database.user": "<<your-username>>",
    "database.password": "<<your-password>>",
    "database.server.id": "1",
    "database.server.name": "<<your-server-name>>",

    "database.include.list": "<<your-database-name>>",
    "table.include.list": "<<your-database-name>>.outbox_event",
    "topic.prefix": "<<your-topic-prefix>>",

    "schema.history.internal.kafka.bootstrap.servers": "<<your-broker-1>>:9092,<<your-broker-2>>:9092",
    "schema.history.internal.kafka.topic": "<<your-schema-history-topic>>",
    "include.schema.changes": "false",

    "snapshot.mode": "schema_only",

    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "key.converter.schemas.enable": "false",

    "value.converter": "org.apache.kafka.connect.converters.ByteArrayConverter",
    "value.converter.schemas.enable": "false",

    "transforms": "convertBytes,outbox",

    "transforms.convertBytes.type": "com.yourorg.ByteBufferToBytesTransform",
    "transforms.convertBytes.fields": "payload",

    "transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter",
    "transforms.outbox.route.by.field": "topic",
    "transforms.outbox.route.topic.replacement": "${routedByValue}",
    "transforms.outbox.add.headers": "eventType:event_type",
    "transforms.outbox.table.fields.additional.placement": "topic:header:eventTopic",
    "transforms.outbox.table.field.event.id": "id",
    "transforms.outbox.table.field.event.key": "aggregate_id",
    "transforms.outbox.table.field.event.type": "event_type",
    "transforms.outbox.table.field.event.payload": "payload",
    "transforms.outbox.table.expand.json.payload": "false"
  }
}
```

---

#### 4️⃣ Registrar el conector

Ejecuta el siguiente comando para registrar el conector en Kafka Connect:

```bash
curl -X POST -H "Content-Type: application/json" \
     --data @connector-config.json \
     http://<<your-connect-host>>:8083/connectors
```

---

#### 5️⃣ Verificar el estado del conector

```bash
curl -X GET http://<<your-connect-host>>:8083/connectors/mariadb-outbox-connector/status
```

Si todo está correcto, deberías ver un estado `"state": "RUNNING"`.