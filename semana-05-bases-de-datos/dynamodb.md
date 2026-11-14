# DynamoDB (claves, capacidad, GSI/LSI, DAX, Streams, TTL, Global Tables)

**Dominio:** PERF | RES | SEC | COST
**Semana:** 05

## Qué es

- **DynamoDB:** base de datos **NoSQL clave-valor y de documentos**, **serverless** y totalmente administrada. Latencia de **un solo dígito de ms a cualquier escala**. Replica sola en **3 AZ** de la región.
- Se diseña **según los patrones de acceso** (cómo vas a consultar), no según la forma "normalizada" de los datos. **No hay JOINs.**

```
                       hash(PK)
 Item {PK, SK, ...} ──────────────► Partición N  (items con la misma PK, ordenados por SK)
                                     límite ≈ 1.000 WCU / 3.000 RCU por partición

 Tabla ──► GSI (otra PK/SK, capacidad propia, solo eventual)
       ──► LSI (misma PK, otra SK, solo al crear la tabla)
       ──► Streams (cambios 24 h) ──► Lambda ──► SNS / S3 / otra tabla
       ──► TTL (borra gratis items caducados → aparecen en Streams)

 App ──► DAX (caché en µs, API compatible) ──► DynamoDB
 Global Table:  us-east-1 ⇄ eu-west-1 ⇄ ap-northeast-1   (todas leen y escriben)
 Lambda en subred privada ──► VPC Gateway Endpoint (gratis) ──► DynamoDB
```

## Cuándo usarlo

- Acceso **clave-valor** a escala masiva con latencia baja y predecible (sesiones, carritos, perfiles, IoT, gaming, leaderboards).
- Apps **serverless** con Lambda: es una API HTTP, **no hay conexiones persistentes** que agotar.
- Tráfico **impredecible** → modo **On-Demand**.
- Usuarios **globales que escriben en su propia región** → **Global Tables**.

## Cuándo NO usarlo (y qué usar en su lugar)

- Datos con **relaciones reales, JOINs, consultas ad-hoc complejas** o transacciones SQL clásicas → **RDS / Aurora**.
- **Analítica** (agregaciones sobre toda la tabla, BI) → exportar a S3 + **Athena**, o **Redshift**. No hagas Scan.
- Objetos de **más de 400 KB** (imágenes, PDFs) → guárdalos en **S3** y en DynamoDB solo el puntero.
- Búsqueda de texto libre → **OpenSearch**.

## Datos clave

**Modelo de datos**
- **Item máximo: 400 KB.**
- Clave primaria **simple** (solo PK) o **compuesta** (PK + SK; la combinación debe ser única).
- El **hash de la PK** decide la partición. Una PK con **alta cardinalidad** reparte la carga; una con pocos valores crea una **hot partition** y hay **throttling** aunque la tabla tenga capacidad de sobra.
- Solución a la hot partition: **write sharding** (sufijo aleatorio o calculado: `candidato#0..9`). Coste: para leer el total hay que hacer **una Query por sufijo** y sumar.
- **Query** = lee por PK (eficiente). **Scan** = lee **toda** la tabla (lento y caro).

**Consistencia y transacciones**
- **Eventual** (por defecto): puede devolver datos de ~1 s atrás. Cuesta **la mitad**.
- **Fuerte** (`ConsistentRead=true`): siempre la última escritura. Cuesta **el doble**. **No disponible en GSI.**
- **Transacciones** (`TransactWriteItems` / `TransactGetItems`): ACID, todo o nada, varios items o tablas. Cuestan **2×**.

**Unidades de capacidad**

| Unidad | Equivale a |
|---|---|
| **1 WCU** | 1 escritura/s de hasta **1 KB** |
| **1 RCU** | 1 lectura fuerte/s de hasta **4 KB**, o **2** eventuales/s |
| Transaccional | **2 WCU / 2 RCU** |

- Siempre se **redondea hacia arriba**. Ejemplo: 100 lecturas/s de 6 KB → 6/4 = 1,5 → **2 RCU** × 100 = **200 RCU** fuerte, **100 RCU** eventual. Escribir esos mismos items: 6 KB → **6 WCU** × 100 = **600 WCU**.

**Modos de capacidad**

| | Provisioned | On-Demand |
|---|---|---|
| Pagas | RCU/WCU reservadas por hora (las uses o no) | Por petición |
| Escalado | **Auto Scaling** (target tracking), reacciona en **minutos** | Instantáneo (hasta 2× el pico anterior) |
| Picos | **Burst capacity** de hasta 300 s | Absorbe picos bruscos |
| Mejor para | Tráfico **estable/predecible**, bajar costes (+ **Reserved Capacity**) | Tráfico **impredecible**, apps nuevas, poco uso |

- Exceso en Provisioned: `ProvisionedThroughputExceededException`. El SDK reintenta con **exponential backoff**.
- Cambio de modo: **1 vez cada 24 h**.

**Índices secundarios**

| | LSI | GSI |
|---|---|---|
| Clave | **Misma PK**, otra SK | **PK y SK distintas** |
| Cuándo se crea | **Solo al crear la tabla** | **En cualquier momento** |
| Máximo | 5 | 20 (por defecto) |
| Consistencia | Eventual **o fuerte** | **Solo eventual** |
| Capacidad | La de la tabla base | **Propia** (RCU/WCU) |
| Límite | 10 GB por valor de PK | — |

- ⚠️ Si un **GSI hace throttling de escrituras**, **la tabla base también hace throttling**. Dale al GSI capacidad de escritura suficiente.
- **Proyecciones:** `KEYS_ONLY`, `INCLUDE` o `ALL`. Más atributos proyectados = más almacenamiento y más WCU.

**DAX (DynamoDB Accelerator)**
- Caché **en memoria**, administrada, **solo para DynamoDB**. Lecturas en **microsegundos**.
- **API compatible**: cambias el cliente del SDK, **no la lógica**.
- Clúster **dentro de la VPC**, multi-AZ, se paga por nodo/hora.
- Cachea lecturas **eventuales**. Las **fuertes pasan directo** a DynamoDB.
- No arregla un mal patrón de acceso: un Scan con DAX sigue siendo un Scan.

**DynamoDB Streams**
- Registro **ordenado** de cada cambio a nivel de item, **retención 24 h**.
- Vistas: `KEYS_ONLY`, `NEW_IMAGE`, `OLD_IMAGE`, `NEW_AND_OLD_IMAGES`.
- **Trigger de Lambda** para arquitecturas orientadas a eventos (emails, notificaciones, replicar, auditar, agregados).
- Alternativa: **Kinesis Data Streams for DynamoDB** (retención de hasta **1 año**, más consumidores).

**TTL**
- Atributo con **timestamp Unix (segundos)**. Pasada esa fecha, DynamoDB **borra el item gratis** (**sin gastar WCU**).
- No es inmediato (normalmente en **48 h**). Filtra por fecha si los caducados no deben verse.
- Los borrados por TTL **aparecen en Streams** → patrón **TTL → Streams → Lambda → S3** para archivar.

**Global Tables**
- **Multi-región y multi-activa:** **todas las réplicas leen y escriben**.
- Replicación asíncrona, normalmente **< 1 s**. Requiere **Streams activado**.
- Conflictos: **last writer wins**.
- DR con RPO de segundos y RTO casi cero (las otras regiones ya están activas, no hay que promover nada).

**Backups**

| Tipo | Qué hace |
|---|---|
| **PITR** | Continuo; restaura a **cualquier segundo** de los últimos **1–35 días** |
| **On-demand** | Snapshot completo, **se guarda hasta que lo borras** |
| **AWS Backup** | Políticas centralizadas, copias **entre regiones/cuentas** |

- ⚠️ **Toda restauración crea una tabla nueva.**
- Los backups **no consumen RCU** ni afectan al rendimiento.
- **Export to S3** (requiere PITR) → analizar con **Athena** sin gastar RCU.

**Seguridad**
- **Cifrado en reposo siempre activo**: clave *AWS owned* (por defecto), *AWS managed* o **CMK en KMS** (control y auditoría de la clave).
- En tránsito: **TLS**.
- IAM fino: condición **`dynamodb:LeadingKeys`** → cada usuario solo accede a los items **con su propia PK** (típico con Cognito).
- **VPC Gateway Endpoint**: acceso privado desde subredes privadas, **sin NAT ni internet y gratis**. Solo existe para **S3 y DynamoDB**.
- **Interface Endpoint** (PrivateLink) para DynamoDB: también privado, pero **de pago**. Se usa cuando hay que llegar **desde on-premises** (VPN/Direct Connect).

## Se confunde con…

| | DAX | ElastiCache |
|---|---|---|
| Para qué sirve | Caché **solo de DynamoDB** | Caché genérica (RDS, sesiones, resultados calculados) |
| Cambios en la app | **Ninguno** (API compatible) | Programar cache-aside o write-through |
| Latencia | Microsegundos | Sub-milisegundo |

| | DynamoDB Global Tables | Aurora Global Database |
|---|---|---|
| Escrituras | **Todas las regiones** (active-active) | **Solo la región primaria** |
| Failover | No hace falta (todas activas) | Promover una secundaria (< 1 min) |
| Conflictos | Last writer wins | No hay (un solo writer) |

| | Streams | Kinesis Data Streams for DynamoDB |
|---|---|---|
| Retención | 24 h | Hasta 1 año |
| Consumidores | Pocos (Lambda típico) | Muchos, integración con Firehose/analítica |

## Trampas de examen

**Frases → respuesta**
- "throttling con capacidad de sobra", "PK con pocos valores" → **hot partition → mejor PK / write sharding**
- "tabla ya existe", "consultar por otro atributo", "usan Scan y es lento" → **GSI**
- "misma partition key, otro orden" + "strong consistency" + tabla nueva → **LSI**
- "throttling en la tabla tras crear un GSI" → **subir la WCU del GSI**
- "tráfico impredecible / picos en segundos", "sin planificar capacidad" → **On-Demand**
- "tráfico estable", "reducir costes" → **Provisioned + Auto Scaling (+ Reserved Capacity)**
- "microsegundos", "sin cambiar el código" → **DAX**
- "reaccionar a cada item nuevo/modificado", "menor esfuerzo operativo" → **Streams + Lambda**
- "datos que caducan", "borrar automáticamente sin coste" → **TTL** (+ Streams → Lambda → S3 para archivar)
- "escrituras locales en varias regiones", "sobrevivir a la caída de una región" → **Global Tables**
- "borrado o escritura accidental", "restaurar a un segundo concreto" → **PITR** (tabla nueva)
- "retención larga", "copias a otra cuenta/región" → **AWS Backup**
- "subred privada", "sin internet", "menor coste" → **VPC Gateway Endpoint**
- "cada usuario solo ve sus datos" → **IAM `dynamodb:LeadingKeys`**
- "objeto de varios MB" → **S3 + puntero en DynamoDB**

**Opciones trampa**
- **DAX** para un Scan lento → sigue siendo Scan; la respuesta es un **índice**.
- **On-Demand** para un Scan lento → cambia cómo pagas, no cómo consultas.
- **LSI** en una tabla existente → imposible.
- **Lectura fuerte en un GSI** → no existe.
- **TTL** para detectar items nuevos → TTL solo borra.
- **Cron/EC2 haciendo Scan** para detectar cambios → caro y operativo; usa **Streams**.
- **NAT Gateway** para que una Lambda privada llegue a DynamoDB → cuesta y sale a internet; usa **Gateway Endpoint**.
- **Interface Endpoint** cuando piden "menor coste" → funciona, pero el Gateway Endpoint es gratis.
- **Aurora Global Database** cuando piden escribir en todas las regiones → solo escribe la primaria.
- **Subir el target de Auto Scaling** para picos de segundos → reacciona demasiado lento; usa **On-Demand**.

## Lo vi en mis proyectos

- **TMS** usó DynamoDB con **single-table design** (`tms-table`, PK + SK compuestas como `String`), **`PAY_PER_REQUEST` (On-Demand)** y **sin GSI**. Fue una buena elección: tráfico mínimo e impredecible, y los patrones de acceso eran lecturas por clave.
- Mejoras que hoy sabría proponer para TMS: activar **PITR** (no estaba configurado) para recuperarse de un borrado accidental; un **GSI** si hiciera falta listar tareas por estado o fecha entre usuarios; **TTL** para tareas archivadas; **`dynamodb:LeadingKeys`** para reforzar a nivel IAM que cada usuario de Cognito solo toque su propia partición.
- **loan-collections** usó RDS PostgreSQL a propósito: había relaciones reales y dinero (ACID, JOINs, reportes). Es el contraste para una entrevista: elijo la base según la forma de los datos y los patrones de acceso, no por defecto.
