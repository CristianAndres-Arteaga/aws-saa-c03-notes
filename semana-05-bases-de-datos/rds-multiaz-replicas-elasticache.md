# Repaso: RDS Multi-AZ, Read Replicas, backups/cifrado y ElastiCache

**Dominio:** RES | PERF | SEC | COST
**Semana:** 05

## Qué es

- **RDS:** base relacional administrada (MySQL, MariaDB, PostgreSQL, Oracle, SQL Server, Db2). AWS gestiona el hardware, los parches, los backups y el failover; tú gestionas el esquema, las consultas y los parámetros.
- **Multi-AZ:** una **standby síncrona** en otra AZ para **alta disponibilidad**. No sirve para leer.
- **Read Replica:** una **copia asíncrona de solo lectura** para **escalar lecturas** (y para DR cross-region).
- **ElastiCache:** **caché en memoria** (Redis/Valkey o Memcached) delante de la base para bajar la latencia y la carga.

```
                 ┌──────── Región ────────────────────────────────┐
App ──endpoint──►│ Primaria (AZ-a) ══síncrona══► Standby (AZ-b)   │  Multi-AZ: HA, failover automático por DNS
     (escrituras)│      │                         (no se puede leer)│
                 │      └──asíncrona──► Read Replica (AZ-c/región)  │  Lecturas, endpoint propio
                 └─────────────────────────────────────────────────┘
App ──► ElastiCache (hit < 1 ms) ──miss──► RDS
```

## Cuándo usarlo

- **Multi-AZ:** producción que tiene que sobrevivir a la falla de una AZ o de una instancia **sin intervención manual y sin perder datos** (RPO ≈ 0, RTO de 1 a 2 min).
- **Read Replicas:** reportes, analítica y cargas con muchas lecturas que saturan la primaria; **DR entre regiones** (réplica cross-region que se promueve).
- **Multi-AZ + Read Replica juntas:** cuando piden **HA y además descargar lecturas**. Son funciones distintas que se combinan.
- **ElastiCache:** consultas repetidas y costosas; **sesiones** fuera de las EC2 (apps *stateless* en un ASG); rankings en tiempo real (Redis, sorted sets).

## Cuándo NO usarlo (y qué usar en su lugar)

- Para **escalar lecturas** no uses Multi-AZ (la standby no se puede leer) → **Read Replica** (o un **Multi-AZ DB cluster**, cuyos standbys sí se leen).
- Para **HA automática con cero pérdida** no uses Read Replicas (son asíncronas y se promueven a mano) → **Multi-AZ**.
- Para recuperarte de un **error humano** (un `UPDATE`/`DELETE` malo) no sirven Multi-AZ ni las réplicas, porque copian el error → **PITR o snapshot**.
- Para datos que deben leerse **siempre actualizados** al instante no leas de una réplica → **primaria**.
- Para una caché sin cambiar código no hay opción: ElastiCache **requiere cambios en la app**. Si piden "sin cambiar código" con DynamoDB → **DAX** (tema 3).

## Datos clave

**Multi-AZ (despliegue de instancia)**
- Replicación **síncrona**. La standby **no acepta conexiones**.
- **Un solo endpoint DNS (CNAME).** En el failover, RDS lo apunta a la standby promovida y la app no cambia el connection string.
- Failover en **60 a 120 s**. Se dispara por caída de la AZ, fallo de la instancia o del almacenamiento, cambio de tipo de instancia, parcheo del SO o **reboot with failover**.
- Pasar de Single-AZ a Multi-AZ **no tiene downtime** (RDS toma un snapshot, restaura la standby y sincroniza).
- Funciona **dentro de una región**.

**Multi-AZ DB cluster** (MySQL y PostgreSQL): 1 writer + **2 standbys legibles** en 3 AZs, replicación semi-síncrona, failover de unos **35 s**.

**Read Replicas**
- Replicación **asíncrona**, con *replication lag*: las lecturas son **eventualmente consistentes**.
- **Hasta 15** por primaria (MySQL/MariaDB/PostgreSQL).
- **Endpoint propio**: la app tiene que mandar las lecturas ahí.
- Misma AZ, otra AZ u **otra región**.
- **Promoción manual**: la réplica pasa a ser una base independiente con escritura y deja de replicar.
- Una réplica puede ser Multi-AZ.
- **Costo de red:** replicar dentro de la misma región es **gratis**; entre regiones **se paga** la transferencia.

**Backups y restauración**
- **Automáticos:** snapshot diario + transaction logs cada **5 min** → **PITR** hasta unos 5 min antes del momento actual. Retención de **0 a 35 días**. Se borran al eliminar la instancia (salvo que elijas conservarlos).
- **Snapshots manuales:** **no expiran**. Sirven antes de cambios riesgosos, para retención larga y para copiar o compartir con otra región o cuenta. Para cumplimiento centralizado → **AWS Backup**.
- **Restaurar (PITR o snapshot) crea una instancia NUEVA con endpoint NUEVO.** No sobrescribe la original.

**Cifrado**
- **En reposo (KMS):** se elige **al crear** la instancia. Cubre almacenamiento, backups, snapshots y réplicas.
- **No se puede cifrar una instancia existente.** El camino es: **snapshot → copiar con cifrado → restaurar → mover la app al nuevo endpoint.**
- Primaria sin cifrar → réplicas **sin cifrar**. Primaria cifrada → réplicas cifradas.
- **En tránsito:** SSL/TLS; para forzarlo en PostgreSQL, `rds.force_ssl=1`. Solo cubre el tránsito, no el reposo.

**ElastiCache**

| | Redis / Valkey | Memcached |
|---|---|---|
| Alta disponibilidad | **Multi-AZ con failover automático**, réplicas | No: si cae un nodo se pierden sus datos |
| Persistencia | Sí (snapshots, AOF) | No |
| Tipos de datos | Listas, sets, **sorted sets** (rankings), pub/sub | Clave-valor simple |
| Escalado | Réplicas + sharding (cluster mode) | **Multi-hilo** + sharding |
| Seguridad | **Redis AUTH**, cifrado en tránsito y en reposo | Muy limitada (SASL) |

**Estrategias de caché**
- **Lazy loading / cache-aside:** se busca en caché; si hay miss, se lee de RDS y se guarda en caché. Solo se cachea lo que se pide, pero el primer acceso es lento y los datos pueden quedar desactualizados (se mitiga con **TTL**).
- **Write-through:** cada escritura en RDS actualiza también la caché. Nunca hay datos desactualizados, pero cada escritura cuesta más y la caché se llena de datos que quizá nadie lea.
- En la práctica se combinan: **lazy loading + TTL** o **write-through + TTL**.

## Se confunde con…

| | Multi-AZ (instancia) | Read Replica |
|---|---|---|
| Para qué sirve | Alta disponibilidad | Escalar lecturas / DR cross-region |
| Replicación | **Síncrona** | **Asíncrona** |
| ¿Se puede leer? | No | Sí |
| Endpoint | Uno solo, failover por DNS | Uno propio por réplica |
| Failover | **Automático** | **Promoción manual** |
| Alcance | Una región | Misma región o cross-region |

## Trampas de examen

**Frases → respuesta**
- "si falla la AZ", "sin intervención manual", "sin pérdida de datos" → **Multi-AZ**
- "reportes/analítica ralentizan producción", "escalar lecturas" → **Read Replica** (y la app apunta a su endpoint)
- "HA **y** descargar lecturas" → **Multi-AZ + Read Replica**
- "DR en otra región" con RDS → **Read Replica cross-region + promoción**
- "volver al estado de hace 10 minutos", "borrado accidental" → **PITR** (instancia nueva)
- "retención de más de 35 días" → **snapshots manuales / AWS Backup**
- "cifrar una base existente sin cifrar" → **snapshot → copia cifrada → restore**
- "forzar conexiones cifradas" → **SSL/TLS (`rds.force_ssl`)**
- "sesiones", "app stateless" → **ElastiCache** (o DynamoDB)
- "ranking", "leaderboard" → **Redis (sorted sets)**
- "caché con HA, persistencia, backups o AUTH" → **Redis**; "simple, multi-hilo, sin persistencia" → **Memcached**

**Opciones trampa**
- **Leer desde la standby** de Multi-AZ (despliegue de instancia) → no tiene endpoint ni acepta conexiones.
- **"Activar el cifrado"** en una instancia existente → no existe esa opción.
- **Réplica cifrada** de una primaria sin cifrar → no se puede.
- **Read Replica que se promueve automáticamente** → eso es de **Aurora**, no de RDS.
- **Multi-AZ en otra región** → Multi-AZ es solo dentro de una región.
- **Multi-AZ para recuperarse de un borrado accidental** → la standby copió el borrado al instante.
- **Memcached** cuando se pide HA o persistencia → no tiene ninguna de las dos.

## Lo vi en mis proyectos

- **loan-collections (`02-rds.yaml`):** PostgreSQL 16 con `MultiAZ` parametrizable (`EnableMultiAz`), `StorageEncrypted: true` (KMS desde la creación, por la regla de "no se puede cifrar después") y `BackupRetentionPeriod: 1` (mínimo, para pruebas de bajo costo; en producción serían 7 a 35 días).
- **Prueba de failover real (`~/e2e-rds-failover.txt`):** con *reboot with failover* y la app consultando el ALB cada ~3,5 s, solo fallaron **2 peticiones** (un timeout a las 02:02:05 y un 500 a las 02:02:13). La app se recuperó sola sin cambiar el endpoint. El corte visible fue de **unos 10-15 s**, menos que los 60-120 s "de manual" porque la app reintentaba la conexión.
- **ElastiCache (`07-cache.yaml`, ADR-0011):** **Valkey** en `cache.t4g.micro` con **cache-aside** para `GET /installments/?overdue=true` y clave por fecha; cifrado en tránsito y en reposo activado. `MultiAZEnabled: false` y un solo nodo por costo: es una decisión de laboratorio, no de producción (en producción → réplica + Multi-AZ).
- No usé Read Replicas. El caso natural habría sido separar los reportes de cobranza (OLTP vs reportes), que estaba planeado en el módulo de HA/DR.
