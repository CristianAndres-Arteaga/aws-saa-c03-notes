# EBS: tipos de volumen, snapshots e instance store

**Dominio:** PERF | COST | RES | SEC
**Semana:** 02

## Qué es

- **EBS (Elastic Block Store):** un disco de red que se conecta a una instancia EC2 (analogía: un cuaderno que vive en **un solo salón**, la AZ). Persiste aunque la instancia se detenga.
- **Snapshot:** copia de un volumen EBS guardada en S3 (analogía: la fotocopia del cuaderno en el archivo central). Es **incremental**: solo guarda los bloques que cambiaron desde el snapshot anterior.
- **Instance store:** disco físico pegado al servidor que aloja la instancia (analogía: una **pizarra** del salón). Es rapidísimo pero **efímero**.

## Cuándo usarlo

- **gp3** → uso general por defecto: boot, apps, bases de datos pequeñas o medianas, dev/test.
- **io2 Block Express** → bases de datos críticas (Oracle, SAP HANA, SQL Server) con IOPS muy altas, latencia sub-ms y máxima durabilidad.
- **io1/io2** → IOPS provisionadas sostenidas, o **Multi-Attach**.
- **st1** → datos **secuenciales** y grandes con mucho throughput: big data, data warehouses, logs, ETL.
- **sc1** → datos fríos de **acceso poco frecuente**, al menor costo posible.
- **Snapshot** → backups, mover un volumen a otra AZ o región, cifrar un volumen sin cifrar, crear AMIs.
- **Instance store** → caché, buffers, scratch o datos temporales, o datos **ya replicados** en otros nodos (Hadoop, Cassandra), cuando se quiere el máximo de IOPS.

## Cuándo NO usarlo (y qué usar en su lugar)

- **gp2** sobredimensionado solo para tener más IOPS → usar **gp3**: sus IOPS y su throughput no dependen del tamaño, así que puedes bajar los GB y fijar las IOPS.
- **io2** para datos desechables → usar **instance store**: pagar 99.999% de durabilidad por datos que se pueden perder es tirar dinero.
- **st1/sc1** como volumen de arranque → **no se puede**: los HDD no son bootables. Usar gp3.
- **st1/sc1** para muchas IOPS pequeñas y aleatorias (una base de datos OLTP) → usar SSD (**gp3/io2**).
- **Instance store** para datos que no puedes perder → usar **EBS** + snapshots.
- **EBS** compartido entre AZs → **no se puede**: el volumen vive en una AZ. Para un disco compartido entre AZs, usar **EFS**.
- **Multi-Attach** con gp3 o entre AZs → solo **io1/io2**, en la **misma AZ**.
- "Activar cifrado" en un volumen existente o "encryption by default" para cifrar lo que ya existe → usar **snapshot → copia cifrada → volumen nuevo**.
- **Lambda o script** que crea snapshots con cron → usar **DLM** (o **AWS Backup** si hay varios servicios o varias cuentas).

## Datos clave

**Tipos de volumen** (cifras "de examen")

| Tipo | Disco | IOPS máx. | Throughput máx. | Clave |
|---|---|---|---|---|
| **gp2** | SSD | 16.000 | 250 MB/s | 3 IOPS por GB, burst a 3.000. Las IOPS **dependen del tamaño** |
| **gp3** | SSD | 16.000 | 1.000 MB/s | Base de **3.000 IOPS / 125 MB/s** con cualquier tamaño. Se suben **por separado**. ~20% más barato que gp2 |
| **io1** | SSD | 64.000 | 1.000 MB/s | IOPS provisionadas. Multi-Attach |
| **io2 Block Express** | SSD | **256.000** | 4.000 MB/s | Hasta 64 TiB, latencia **sub-ms**, durabilidad **99.999%**. Multi-Attach |
| **st1** | HDD | — | 500 MB/s | Throughput optimizado. **No bootable** |
| **sc1** | HDD | — | 250 MB/s | El más barato. Acceso poco frecuente. **No bootable** |

> Nota: en 2025 AWS subió los límites de gp3 (hasta 80.000 IOPS / 2.000 MB/s). Las preguntas del examen suelen usar las cifras de la tabla.

**Volúmenes**
- Atados a **una AZ**. Para llevarlos a otra AZ: snapshot → volumen nuevo en la otra AZ.
- **Delete on Termination:** activado por defecto en el volumen **raíz** y desactivado en los volúmenes **adicionales**.
- **Multi-Attach:** io1/io2, misma AZ, hasta **16** instancias Nitro, y necesita un **sistema de archivos de clúster** (no ext4/xfs normal).
- **Elastic Volumes:** permite cambiar el tipo, **crecer** el tamaño y cambiar las IOPS **en línea**. **No se puede encoger**.

**Snapshots**
- **Incrementales**, guardados en S3 (no los ves en tu bucket). **No dependen de la AZ**.
- Se pueden **copiar a otra región** (DR, migración).
- **Cifrado:** de un snapshot sin cifrar se puede hacer una **copia cifrada**. Un volumen creado desde un snapshot cifrado está cifrado.
- **Fast Snapshot Restore (FSR):** el volumen da **rendimiento completo desde el primer segundo** (sin la latencia de la primera lectura). Cuesta extra y se activa por snapshot y por AZ.
- **Snapshot Archive:** hasta **75% más barato**, restaurar tarda **24-72 h**, mínimo **90 días**.
- **Recycle Bin:** recupera snapshots (y AMIs) **borrados por accidente** durante el periodo de retención de la regla.
- **DLM (Data Lifecycle Manager):** automatiza crear, retener y borrar snapshots de EBS y AMIs.
- **AWS Backup:** backups centralizados de **muchos servicios** (EBS, RDS, DynamoDB, EFS…), con soporte entre cuentas y regiones.

**Instance store**
- Millones de IOPS, latencia mínima (disco local NVMe).
- **Sobrevive a un reboot**.
- **Se pierde** con **stop**, **hibernate**, **terminate** o si falla el hardware.
- El tamaño viene fijado por el tipo de instancia. No se puede añadir después.

## Se confunde con…

### gp2 vs gp3
| | gp2 | gp3 |
|---|---|---|
| IOPS | Ligadas al tamaño (3/GB) | 3.000 base + se suben aparte |
| Costo | Más caro | ~20% más barato |
> Si la pregunta dice "volumen sobredimensionado solo por las IOPS" → **gp3** con menos GB.

### st1 vs sc1
| | st1 | sc1 |
|---|---|---|
| Para qué sirve | Throughput frecuente (logs, big data) | Datos fríos, poco acceso |
| Diferencia clave | 500 MB/s | 250 MB/s, el más barato |
> Decide según **cada cuánto** se lee.

### EBS vs instance store
| | EBS | Instance store |
|---|---|---|
| Persistencia | Sobrevive al stop | Se pierde con stop/terminate |
| Rendimiento | Alto (por la red) | Máximo (disco local) |
| Snapshots | Sí | No |
> Pregúntate: **¿pasa algo si se pierde el disco?**

### DLM vs AWS Backup
| | DLM | AWS Backup |
|---|---|---|
| Alcance | Solo EBS y AMIs | Muchos servicios, varias cuentas |
> Si el enunciado nombra **un solo servicio** (EBS) → DLM. Si nombra **varios** o habla de **políticas centralizadas** → AWS Backup.

## Trampas de examen

**Frases → respuesta**
- "100k+ IOPS", "sub-millisecond", "máxima durabilidad", "Oracle/SAP" → **io2 Block Express**
- "bajar costo manteniendo IOPS", "gp2 sobredimensionado" → **gp3**
- "logs secuenciales de varios TB", "big data", "throughput, el más barato" → **st1**
- "acceso poco frecuente", "lo más barato posible" → **sc1**
- "varias instancias escriben en el mismo volumen", "misma AZ" → **io2 Multi-Attach**
- "temporal", "máximas IOPS", "replicado en otros nodos", "se puede perder" → **instance store**
- "cifrar un volumen existente sin cifrar" → **snapshot → copia cifrada → volumen nuevo**
- "mover el volumen a otra AZ/región" → **snapshot** (+ copia a la otra región)
- "primeras lecturas lentas tras restaurar" → **Fast Snapshot Restore**
- "snapshots que se guardan años y casi nunca se restauran" → **Snapshot Archive**
- "borraron un snapshot por error" → **Recycle Bin**
- "automatizar snapshots diarios con retención" → **DLM**

**Opciones trampa**
- Arrancar desde st1/sc1 → los HDD no son bootables.
- Activar cifrado directamente en el volumen → no existe.
- "EBS encryption by default" para cifrar lo existente → solo afecta a volúmenes nuevos.
- Asignar una clave KMS a la instancia → las claves van en el volumen o el snapshot.
- Instance store "persistente" o "con snapshots" → no tiene ninguna de las dos cosas.
- Multi-Attach en gp3 o entre AZs → solo io1/io2, misma AZ.
- Encoger un volumen con Elastic Volumes → solo crece.
- Snapshot Archive para restaurar rápido → tarda 24-72 h; lo rápido es FSR.

**Método (Días 6-8)**
1. Primero decide **SSD vs HDD**: ¿IOPS aleatorias o throughput secuencial? ¿Hay que arrancar desde él?
2. Después decide **persistente vs efímero**: ¿se puede perder el dato?
3. Al final, el **costo**: entre las opciones que cumplen, la más barata (gp3 > io2, sc1 > st1).

## Lo vi en mis proyectos

- **Plataforma three-tier (SAA):** las EC2 del ASG arrancaban con un volumen raíz EBS, con Delete on Termination por defecto. Por eso las instancias se podían reemplazar sin dejar discos huérfanos.
- **Módulo 11 HA/DR:** usé la misma idea de los snapshots con RDS: `DeletionPolicy: Snapshot` (el snapshot final antes de borrar) y la copia de snapshots entre regiones como estrategia de DR backup & restore.
