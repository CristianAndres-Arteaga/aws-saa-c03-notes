# Kinesis Data Streams, Amazon Data Firehose, Amazon MSK y Amazon MQ

**Dominio:** RES | PERF | COST
**Semana:** 04

## Qué es

- **Kinesis Data Streams (KDS)** es una **grabación en vivo**. Los eventos se graban **en orden** y la grabación **se guarda un tiempo** (retención). Varios consumidores ven **la misma grabación**, cada uno a su ritmo, y pueden **rebobinar** (*replay*). Leer no borra nada.
- **Amazon Data Firehose** (antes *Kinesis Data Firehose*) es una **manguera**. La conectas a un destino y los datos **llegan solos**: no hay shards ni consumidores que programar. No guarda nada, así que **no se puede rebobinar**.
- **Amazon MSK** es **Apache Kafka administrado**: el mismo Kafka que la empresa ya usa, pero AWS cuida los brokers.
- **Amazon MQ** es **ActiveMQ o RabbitMQ administrado**: el broker "clásico" que hablan las apps on-premise (JMS, AMQP, MQTT…).

Analogía de la mudanza: MSK y MQ sirven para **traer tus propios muebles** (el mismo software, sin reescribir código). SQS, SNS y KDS son **muebles nuevos**: nativos de AWS, pero hay que adaptar la app.

```
 Productores                 Kinesis Data Streams                 Consumidores
 App móvil ───┐      ┌──────────────────────────────┐      ┌──► Lambda (tiempo real)
 IoT ─────────┼────► │ Shard 1  ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓  │ ─────┼──► App KCL / Flink
 Logs ────────┘      │ Shard 2  ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓  │      └──► Firehose ──► S3 (Parquet) ──► Athena
 (partition key)     └──── retención 24 h – 365 d ──┘
```

**Combinación típica:** `KDS → Firehose → S3`. El stream alimenta a las apps en tiempo real y, a la vez, Firehose lo archiva.

## Cuándo usarlo

**Kinesis Data Streams**
- **Tiempo real** (~200 ms) con **varios consumidores independientes** sobre los mismos datos.
- **Orden** por entidad (usuario, dispositivo, repartidor) → misma partition key.
- **Reprocesar** datos pasados → retención ampliada + *replay*.
- Clickstream, telemetría IoT, GPS, métricas, logs en tiempo real.

**Data Firehose**
- **Cargar** datos de streaming en **S3, Redshift, OpenSearch, Splunk, endpoints HTTP**, Snowflake o Iceberg.
- **Menor esfuerzo operacional**: serverless, escala solo.
- Una latencia de segundos o minutos es aceptable (**near real-time**).
- Transformar con **Lambda** o convertir JSON a **Parquet/ORC** para consultar con Athena.

**Amazon MSK**
- La empresa **ya usa Apache Kafka** o necesita la **API de Kafka**.
- Mensajes **mayores de 1 MB** (configurable), o retención muy larga (tiered storage).

**Amazon MQ**
- Migrar apps on-premise que usan **ActiveMQ/RabbitMQ** o los protocolos **JMS, AMQP, MQTT, STOMP, OpenWire**, **sin reescribir código** (solo cambia la URL del broker).

## Cuándo NO usarlo (y qué usar en su lugar)

- Desacoplar, cada mensaje se procesa **una vez**, workers con reintentos y DLQ → **SQS**.
- Fan-out simple, email, SMS o push → **SNS**.
- Solo hay que **dejar datos en S3/Redshift** sin consumidores propios → **Firehose**, no KDS + código.
- Necesitas *replay* o consumidores propios en tiempo real → **KDS**, no Firehose solo.
- App **nueva** nativa de la nube → **SQS/SNS/KDS**, no MQ ni MSK (que corren sobre brokers dimensionados y cuestan más).

## Datos clave

**Kinesis Data Streams**

| Concepto | Dato |
|---|---|
| **Shard** | Entrada **1 MB/s o 1.000 registros/s**; salida **2 MB/s** (compartida entre consumidores) |
| **Partition key** | Mismo key → mismo shard → **orden garantizado dentro del shard** |
| **Retención** | **24 h** por defecto, hasta **365 días** → *replay* |
| **Registro** | Máximo **1 MB**. No se borran uno por uno; caducan con la retención |
| **Enhanced fan-out** | **2 MB/s por consumidor y por shard** (no se reparten); latencia ~70 ms |
| **Modos** | **Provisioned** (eliges shards) u **On-demand** (escala solo; tráfico impredecible) |
| **Consumidores** | Lambda, apps **KCL**, **Managed Service for Apache Flink** (antes Kinesis Data Analytics), Firehose |
| **Seguridad** | Cifrado con **KMS**, acceso con IAM, VPC endpoint |

**Data Firehose**
- **Serverless**, sin shards. Se paga **por GB** ingerido.
- **Buffer** por tamaño (MB) o por tiempo (segundos) → **near real-time**.
- Transformación con **Lambda**; **conversión de formato** a Parquet/ORC; compresión.
- **Redshift**: Firehose escribe en S3 y luego ejecuta `COPY`.
- Registros que fallan → **bucket S3 de respaldo**.
- Fuentes: Direct PUT, KDS, MSK, CloudWatch Logs, entre otras.
- **Sin retención ni replay.**

**Amazon MSK**
- Kafka: *topics* y *partitions* (equivalen a streams y shards de KDS).
- Brokers en **2 o 3 AZ**, datos en **EBS**. Retención configurable; **tiered storage** para guardar mucho más.
- **MSK Serverless** (sin dimensionar brokers) y **MSK Connect** (conectores).

**Amazon MQ**
- **ActiveMQ** o **RabbitMQ**; tiene **colas y tópicos** en el mismo servicio.
- Alta disponibilidad: ActiveMQ **active/standby en 2 AZ** (almacenamiento compartido en EFS); RabbitMQ en **clúster de 3 nodos**.
- **No escala infinito como SQS**: depende del tamaño del broker.

## Se confunde con…

### KDS vs Firehose vs SQS ⭐
| Si el enunciado dice… | Respuesta |
|---|---|
| tiempo real, **varios consumidores**, **orden**, **replay / reprocesar** | **Kinesis Data Streams** |
| "load / deliver into S3, Redshift, OpenSearch, Splunk", "**least operational overhead**", "near real-time", Parquet | **Data Firehose** |
| desacoplar, cada mensaje **una vez**, workers, reintentos/DLQ | **SQS** |

| | KDS | Firehose | SQS |
|---|---|---|---|
| Leer borra el dato | No | — (no guarda) | **Sí** (al borrarlo el consumidor) |
| Varios consumidores del mismo dato | **Sí** | No (un destino por stream) | No (cada mensaje a uno) |
| Replay | **Sí** (hasta 365 d) | **No** | No |
| Orden | Por shard | — | Solo FIFO |
| Administración | Shards o on-demand | **Ninguna** | Ninguna |
| Latencia | ~200 ms | Segundos–minutos (buffer) | Milisegundos |

### KDS vs MSK
| | KDS | MSK |
|---|---|---|
| Origen | Nativo de AWS, más simple | Apache Kafka (open source) |
| Elegir cuando… | App nueva, integración AWS | **Ya usan Kafka**, API de Kafka, mensajes > 1 MB |

### Amazon MQ vs SQS/SNS
| | Amazon MQ | SQS / SNS |
|---|---|---|
| Protocolos | **JMS, AMQP, MQTT, STOMP, OpenWire** | API propietaria de AWS |
| Escala | Según el broker | Prácticamente ilimitada |
| Elegir cuando… | **Migrar sin reescribir** | App nueva nativa de la nube |

## Trampas de examen

**Frases → respuesta**
- "multiple applications consume the same stream", "reprocess the last N days" → **KDS** con retención ≥ N días
- "ordering per device/user" → **KDS** con partition key = id del dispositivo/usuario
- "deliver to S3 / Redshift / OpenSearch / Splunk", "least operational overhead", "near real-time" → **Firehose**
- "convert to Parquet", "query with Athena" → **Firehose** con conversión de formato + S3
- "many consumers, throughput shared" → **enhanced fan-out**
- "unpredictable traffic" en KDS → **on-demand**
- "already uses Apache Kafka" → **MSK**
- "JMS / AMQP / MQTT", "ActiveMQ / RabbitMQ", "without rewriting code" → **Amazon MQ**

**Opciones trampa**
- **SQS para varios consumidores del mismo dato** → cada mensaje lo recibe uno solo.
- **SNS → varias SQS cuando piden replay** → el fan-out cumple, pero lo consumido se borra.
- **Firehose cuando piden reprocesar** → no retiene datos.
- **KDS + app KCL en EC2 solo para escribir en S3** → funciona, pero es mucha operación: usa Firehose.
- **SQS + Lambda escribiendo un objeto por mensaje** → millones de archivos pequeños; Athena se vuelve lento y caro.
- **Athena consultando un stream** → Athena consulta **S3**, no KDS.
- **MSK para una app JMS** → Kafka no habla JMS: es **Amazon MQ**.
- **Broker en una sola EC2** → sin alta disponibilidad y con toda la operación.

**Método (Día 13)**
1. ¿Hay **varios consumidores** o **replay**? → KDS.
2. ¿Solo hay que **entregar** datos a un destino con poco esfuerzo? → Firehose.
3. ¿Se menciona un **software o protocolo existente** (Kafka, JMS, AMQP, MQTT) y "sin reescribir"? → MSK o MQ.
4. En el por qué, nombra la **frase exacta** del enunciado ("menor esfuerzo operacional", "latencia de minutos aceptable", "3 días").

## Lo vi en mis proyectos

- **TMS:** no usó streaming. Si los eventos de la app (cambios de estado, logins) se enviaran a **Firehose → S3 en Parquet**, se podrían analizar con Athena sin servidores y sin tocar la base de datos principal.
- **loan-collections:** los eventos de pago podrían ir a un **Kinesis Data Streams** con `partition key = clienteId` (orden de los pagos de cada cliente). Una Lambda actualizaría las cuotas en tiempo real y Firehose archivaría todo en S3 para auditoría.
