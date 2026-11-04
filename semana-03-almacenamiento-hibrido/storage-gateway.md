# Storage Gateway: S3 File, FSx File, Volume y Tape Gateway

**Dominio:** RES | PERF | COST
**Semana:** 03

## Qué es

- **Storage Gateway:** servicio **híbrido** que conecta tus servidores on-premises con el almacenamiento de AWS (analogía: una **sucursal local** de un banco cuya bóveda está en AWS: atiendes rápido en la sucursal y el grueso del dinero vive en la bóveda).
- Se instala un **appliance** on-premises: una **VM** (VMware, Hyper-V, KVM), un **hardware appliance** de AWS o una instancia EC2.
- Guarda una **caché local** para que los datos recientes se lean con baja latencia.
- Hay cuatro tipos:
  - **S3 File Gateway:** archivos por NFS/SMB → **objetos en S3**.
  - **FSx File Gateway:** acceso local con caché, por SMB, a un **FSx for Windows**.
  - **Volume Gateway:** **discos (bloques) por iSCSI**, con respaldo como **snapshots de EBS**.
  - **Tape Gateway:** **librería de cintas virtual (VTL)** → S3 / Glacier / Deep Archive.

## Cuándo usarlo

- **S3 File Gateway:** apps on-prem que escriben archivos y quieres que lleguen a S3 como objetos (lifecycle, analítica, backup).
- **FSx File Gateway:** oficinas con usuarios Windows que acceden a un FSx for Windows en AWS y necesitan baja latencia.
- **Volume Gateway cached:** ampliar capacidad de disco on-prem sin comprar hardware (el dataset principal vive en S3).
- **Volume Gateway stored:** todo el dataset en local con baja latencia + backup asíncrono a AWS (DR).
- **Tape Gateway:** eliminar cintas físicas **sin cambiar el software de backup** (Veeam, NetBackup, Backup Exec…).

## Cuándo NO usarlo (y qué usar en su lugar)

- **Migración de una sola vez** o copias programadas → **DataSync** (transfiere datos; no es un share permanente con caché).
- Volume Gateway cuando piden "ver los archivos como objetos en S3" → **S3 File Gateway**.
- S3 File Gateway cuando piden **bloques / iSCSI** → **Volume Gateway**.
- FSx File Gateway con destino S3 → no; su destino es **FSx for Windows**.
- Volume **cached** cuando piden "**todo** el dataset con baja latencia local" → **stored**.
- S3 File Gateway + lifecycle para reemplazar cintas manteniendo el software de backup → **Tape Gateway** (el software espera una librería de cintas).

## Datos clave

**S3 File Gateway**
- Protocolos **NFS** (v3, v4.1) y **SMB**.
- Cada archivo = **un objeto en S3**, visible en la consola y usable por otros servicios.
- Clases destino: S3 Standard, Standard-IA, One Zone-IA, Intelligent-Tiering. Para Glacier → **lifecycle policy**.
- Integración con **Active Directory** para SMB.

**FSx File Gateway**
- Solo **SMB**. Destino: **FSx for Windows File Server**.
- Caché local de los datos usados con frecuencia.

**Volume Gateway**
- Discos expuestos por **iSCSI**.
- Respaldos = **snapshots de EBS** point-in-time → se pueden restaurar como volúmenes EBS en EC2 (DR / migración).

| Modo | Dónde vive el dataset completo | Qué queda en local | Cuándo |
|---|---|---|---|
| **Cached** | **S3** (almacenamiento principal) | Solo lo usado con frecuencia | Ampliar capacidad sin discos locales |
| **Stored** | **Local** (almacenamiento principal) | Todo | Baja latencia sobre todo el dataset + backup asíncrono a AWS |

> **Cached = la nube manda. Stored = lo local manda.**

**Tape Gateway**
- **VTL por iSCSI**: el software de backup cree que escribe en cintas físicas.
- Cintas virtuales en **S3**; al archivarlas pasan a **Glacier Flexible Retrieval** o **Glacier Deep Archive** (lo más barato para retención de años).

## Se confunde con…

### Árbol de decisión
```
¿Qué pide el enunciado?
├─ Archivos (NFS/SMB)
│    ├─ destino S3 como objetos ................. S3 File Gateway
│    └─ destino FSx for Windows (SMB) ........... FSx File Gateway
├─ Bloques (iSCSI)
│    ├─ dataset principal en AWS, lo caliente local ... Volume Gateway CACHED
│    └─ dataset completo local + backup asíncrono ..... Volume Gateway STORED
├─ Cintas / software de backup existente ........ Tape Gateway
└─ Mover datos una vez o por calendario ......... DataSync (no Storage Gateway)
```

### Storage Gateway vs DataSync
| | Storage Gateway | DataSync |
|---|---|---|
| Para qué | Acceso **continuo** on-prem ↔ AWS | **Transferir / migrar** datos |
| Caché local | Sí | No |
| Uso típico | Share montado, discos, cintas | Migración, copias programadas |

## Trampas de examen

**Frases → respuesta**
- "NFS/SMB on-prem", "objetos en S3", "lifecycle a Glacier", "caché local" → **S3 File Gateway**
- "usuarios de oficina", "FSx for Windows", "baja latencia local" → **FSx File Gateway**
- "iSCSI", "ampliar capacidad", "datos principales en AWS" → **Volume Gateway cached**
- "iSCSI", "todo el dataset local", "backup asíncrono", "snapshots" → **Volume Gateway stored**
- "cintas físicas", "sin cambiar el software de backup", "retención de años", "menor costo" → **Tape Gateway + Glacier Deep Archive**

**Opciones trampa**
- Volume Gateway para "objetos en S3" → guarda volúmenes/snapshots, no objetos visibles.
- DataSync como "acceso local con caché" → solo transfiere.
- FSx File Gateway hacia S3 → su destino es FSx for Windows.
- Cached cuando piden todo el dataset en local → es stored.

**Método (Día 9)**
1. ¿**Archivos, bloques o cintas**? → te da el tipo de gateway.
2. Después lee el **destino** (S3 vs FSx) o **dónde vive el dataset** (cached vs stored).

## Lo vi en mis proyectos

- Ninguno lo usó todavía. Las dos apps (TMS serverless y la plataforma three-tier) nacieron en AWS, sin infraestructura on-premises que conectar.
