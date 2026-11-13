# Aurora (réplicas, Serverless v2, Global Database) y RDS Proxy

**Dominio:** RES | PERF | SEC | COST
**Semana:** 05

## Qué es

- **Aurora:** motor relacional de AWS **compatible con MySQL y PostgreSQL** (mismo driver y mismo SQL). **Separa el cómputo del almacenamiento**: todas las instancias del clúster leen y escriben sobre **un único volumen compartido**.
- **Aurora Serverless v2:** instancias Aurora sin tamaño fijo. La capacidad se mide en **ACUs** y sube o baja sola en segundos.
- **Aurora Global Database:** un clúster Aurora que abarca **varias regiones**, con replicación a nivel de almacenamiento en **menos de 1 s**.
- **RDS Proxy:** proxy administrado entre la app y RDS/Aurora que **reutiliza conexiones** (pooling), acelera el failover y permite **autenticación IAM**.

```
                 App
          escrituras │ lecturas
                ▼          ▼
     [cluster endpoint]  [reader endpoint]
           │              │      │      │
        Writer        Réplica Réplica Réplica   (hasta 15, lag < 100 ms)
           └──────────────┴──────┴──────┘
                         │
     ┌─────── Volumen de clúster compartido ───────┐
     │  AZ-a: ▣▣     AZ-b: ▣▣     AZ-c: ▣▣         │  6 copias, auto-grow hasta 128 TiB
     └─────────────────────────────────────────────┘

 Lambda ×miles ──► [ RDS Proxy ] ══ pool pequeño ══► RDS / Aurora
                   (IAM auth, Secrets Manager, solo dentro de la VPC)
```

## Cuándo usarlo

- **Aurora:** muchas lecturas que necesitan **muchas réplicas con lag mínimo**; **failover en segundos**; almacenamiento que **crece de forma impredecible**; migrar desde MySQL/PostgreSQL sin reescribir la app.
- **Serverless v2:** carga **impredecible, intermitente o con picos**; entornos dev/test; apps nuevas sin historial de uso; "pagar solo lo que uso sin gestionar capacidad".
- **Global Database:** **DR entre regiones** con **RPO ≈ 1 s y RTO < 1 min**; **lecturas de baja latencia** para usuarios en otros continentes.
- **RDS Proxy:** **Lambda + RDS/Aurora** con errores **"too many connections"**; muchas conexiones cortas; reducir el tiempo de failover que ve la app; **forzar auth IAM** y sacar las credenciales del código.

## Cuándo NO usarlo (y qué usar en su lugar)

- Carga **alta y estable 24/7** → Aurora **provisionado** (+ Reserved Instances), no Serverless v2.
- Solo necesitas HA dentro de **una región** → Aurora con réplicas en varias AZ (o RDS Multi-AZ). Global Database es para **multi-región**.
- Datos que no son relacionales, o escala masiva clave-valor → **DynamoDB** (implica reescribir la app).
- Las **consultas** son lentas (no las conexiones) → índices, réplicas o caché (ElastiCache). RDS Proxy no acelera las consultas.
- Recuperarte de un `DELETE` malo en Aurora **PostgreSQL** → no hay Backtrack, usa **PITR** (instancia nueva).

## Datos clave

**Arquitectura de Aurora**
- Clúster = **1 writer + hasta 15 Aurora Replicas**.
- Volumen de clúster: **6 copias en 3 AZ** (2 por AZ). Quórum: **escritura 4/6, lectura 3/6** → aguanta perder 2 copias sin afectar escrituras y 3 sin afectar lecturas. **Self-healing.**
- **Auto-grow** en bloques de 10 GB **hasta 128 TiB**. No se aprovisiona espacio.
- Lag de réplicas **< 100 ms** (comparten disco). En RDS cada réplica tiene su propio almacenamiento y el lag es mayor.
- **Failover automático < 30 s** promoviendo una réplica. El orden se define con **priority tiers 0–15** (gana el número más bajo).
- **Aurora Replica Auto Scaling** (por CPU o conexiones). Como la app usa el reader endpoint, no se entera.
- Cuesta ~20 % más por instancia que RDS.

**Endpoints**

| Endpoint | Para qué |
|---|---|
| **Cluster (writer)** | Siempre apunta al writer → escrituras |
| **Reader** | Reparte las conexiones entre todas las réplicas → lecturas |
| **Custom** | Un subconjunto de réplicas (p. ej. réplicas grandes para BI) |
| **Instance** | Una instancia concreta (debug) |

**Aurora Serverless v2**
- Capacidad en **ACUs** (~2 GiB de RAM + CPU/red cada una). Escala en pasos de **0.5 ACU**, en segundos, **sin cortar conexiones**.
- Rango mínimo–máximo configurable. Puede bajar a **0 ACU (auto-pause)**.
- Soporta Multi-AZ, réplicas y Global Database. Se pueden **mezclar** instancias provisionadas y Serverless v2 en el mismo clúster.
- Se paga por **ACU-hora**.

**Aurora Global Database**
- **1 región primaria (lectura/escritura) + hasta 5 regiones secundarias (solo lectura)**.
- Replicación **a nivel de almacenamiento**, lag típico **< 1 s**.
- Hasta **16 réplicas** por región secundaria.
- **RPO ≈ 1 s, RTO < 1 min.** **Switchover** = cambio planificado sin pérdida de datos; **failover** = desastre, gestionado.
- **Write forwarding:** las secundarias reenvían las escrituras a la primaria.

**Extras de Aurora**
- **Cloning:** copia en minutos con **copy-on-write** (solo pagas lo que cambia). Para probar con datos de producción.
- **Backtrack (solo Aurora MySQL):** "rebobina" el **mismo clúster** hasta 72 h atrás, sin restaurar. PITR, en cambio, crea una base **nueva**.

**RDS Proxy**
- Para RDS (MySQL, PostgreSQL, MariaDB, SQL Server) y Aurora.
- **Connection pooling y multiplexing:** miles de conexiones de clientes → pocas conexiones reales reutilizadas. Baja la CPU y la RAM de la base.
- **Failover hasta un 66 % más rápido**: mantiene las conexiones de la app y redirige al nuevo primario sin depender del DNS.
- **Auth IAM** obligatoria (opcional), credenciales en **Secrets Manager**.
- **No es accesible públicamente**: solo desde la VPC.
- Serverless y Multi-AZ. Endpoints de solo lectura hacia réplicas de Aurora.
- Para la app **solo cambia el endpoint**.
- Precio por **vCPU** de la instancia de base de datos (o ACU) por hora.

## Se confunde con…

| | Aurora Replicas | RDS Read Replicas |
|---|---|---|
| Almacenamiento | **Compartido** (volumen de clúster) | Propio en cada réplica |
| Lag | **< 100 ms** | Asíncrono, puede llegar a segundos |
| Máximo | 15 | 15 (MySQL/MariaDB/PostgreSQL) |
| Endpoint para lecturas | **Reader endpoint** (uno solo, reparte) | Uno por réplica (la app reparte) |
| Failover | **Automático** a una réplica (< 30 s) | **Promoción manual** |

| | Aurora Global Database | RDS Read Replica cross-region |
|---|---|---|
| Replicación | Nivel de almacenamiento, < 1 s | Del motor, asíncrona |
| RPO / RTO | ~1 s / < 1 min | Mayores; promoción manual |
| Regiones | Hasta 5 secundarias | Una réplica por región que crees |

| | Aurora Serverless v2 | Aurora provisionado |
|---|---|---|
| Capacidad | Rango de ACUs, escala sola | Tamaño fijo de instancia |
| Mejor para | Carga impredecible o intermitente | Carga estable 24/7 (+ RIs) |

## Trampas de examen

**Frases → respuesta**
- "muchas réplicas con lag mínimo", "failover en segundos", "almacenamiento que crece solo" → **Aurora + reader endpoint**
- "no cambiar la conexión de la app al agregar réplicas" → **reader endpoint + Replica Auto Scaling**
- "réplicas grandes solo para reportes/BI" → **custom endpoint**
- "carga impredecible / intermitente", "pagar solo lo que uso", "sin gestionar capacidad" (relacional) → **Aurora Serverless v2**
- "caída de una región", "RPO de segundos", "RTO < 1 min", "lecturas de baja latencia en otro continente" → **Aurora Global Database**
- "copia rápida de producción para pruebas" → **Aurora Cloning**
- "deshacer un error reciente sin restaurar" (MySQL) → **Backtrack**
- "Lambda + RDS + too many connections" → **RDS Proxy**
- "reducir el tiempo de failover para la app" → **RDS Proxy**
- "forzar IAM / sin credenciales en el código" con RDS → **RDS Proxy + Secrets Manager** (o IAM DB auth)

**Opciones trampa**
- **Snapshots copiados a otra región** para un RPO de segundos → el RPO es el intervalo entre snapshots (horas).
- **Multi-AZ** para sobrevivir a la caída de una **región** → solo protege AZs.
- **DMS** para DR con RPO ≈ 1 s y mínimo esfuerzo → más operación, sin garantías. La respuesta es Global Database.
- **Instancia más grande / subir `max_connections`** para "too many connections" con Lambda → parche vertical y caro.
- **Read Replicas** para "too many connections" → no hacen pooling y las escrituras siguen yendo a la primaria.
- **Reserved concurrency baja en Lambda** → limita las conexiones, pero estrangula la app (throttling).
- **Backtrack en Aurora PostgreSQL** → no existe, solo MySQL.
- **RDS Proxy público** → no se puede, solo dentro de la VPC.

## Lo vi en mis proyectos

- No usé Aurora ni RDS Proxy. **loan-collections** usó RDS PostgreSQL (`02-rds.yaml`, Single-AZ por Free Tier según ADR-0003, con Multi-AZ parametrizable) porque el costo de laboratorio pesaba más que el failover más rápido de Aurora.
- Donde encajaría: si la API de loan-collections migrara de EC2 a **Lambda**, **RDS Proxy** sería la pieza para evitar "too many connections". Ojo: las credenciales estaban en **SSM Parameter Store `SecureString`** (ADR-0009, gratis), y RDS Proxy las lee de **Secrets Manager**. Habría que moverlas ahí (~$0.40/mes por secreto), con la ventaja de tener rotación automática.
- En **TMS** (Lambda + DynamoDB) el problema de las conexiones no existía: DynamoDB es una API HTTP y no usa conexiones persistentes. Es otra razón válida para elegir DynamoDB con Lambda.
