# FSx: Windows File Server, Lustre, NetApp ONTAP y OpenZFS

**Dominio:** RES | PERF | COST
**Semana:** 03

## Qué es

- **FSx:** una familia de sistemas de archivos **"de marca"** administrados por AWS (analogía: **bibliotecas especializadas**, mientras que EFS es la biblioteca general de Linux). Hay cuatro:
  - **FSx for Windows File Server:** la biblioteca de la oficina Windows (SMB, NTFS, Active Directory).
  - **FSx for Lustre:** la sala de lectura ultrarrápida para investigadores (HPC, ML), enlazada a S3.
  - **FSx for NetApp ONTAP:** la biblioteca "políglota" (NFS + SMB + iSCSI) para quien ya usa NetApp.
  - **FSx for OpenZFS:** para migrar cargas que usan ZFS (NFS, latencia muy baja).

## Cuándo usarlo

- **Windows:** migrar file servers de Windows, carpetas compartidas con permisos de **Active Directory**, apps .NET/SQL Server que usan SMB.
- **Lustre:** **HPC**, entrenamiento de **ML**, renderizado de video, simulaciones, modelos financieros, sobre todo cuando los datos están en **S3**.
- **ONTAP:** migrar almacenamiento **NetApp** on-premises, clientes **Linux y Windows** sobre los mismos datos, volúmenes **iSCSI**.
- **OpenZFS:** migrar servidores **ZFS** sin cambiar la app, con clonado instantáneo para entornos de prueba.

## Cuándo NO usarlo (y qué usar en su lugar)

- **EFS** para Windows/SMB/AD → es solo NFS → **FSx for Windows**.
- **EFS Max I/O** para HPC con datos en S3 → **FSx for Lustre**.
- **FSx for Windows** o **EFS** cuando piden NFS + SMB + iSCSI a la vez → **FSx for NetApp ONTAP**.
- **Lustre Persistent** para un procesamiento de pocos días con datos ya en S3 → **Lustre Scratch** (más barato).
- **Lustre Scratch** para datos importantes o de largo plazo → **Lustre Persistent** (Scratch no replica).
- **Single-AZ + backups** cuando piden failover automático ante la caída de una AZ → **Multi-AZ** (un backup es una restauración manual, no un failover).
- **OpenZFS** para clientes Windows por SMB → no tiene SMB → **FSx for Windows** u **ONTAP**.

## Datos clave

**FSx for Windows File Server**
- **SMB** + **NTFS**, integración con **Active Directory** (AWS Managed Microsoft AD u on-premises), permisos **ACL** de Windows.
- **DFS Namespaces:** agrupar varias carpetas compartidas bajo una sola ruta.
- **Multi-AZ** (standby en otra AZ, **failover automático**) o **Single-AZ**.
- Almacenamiento **SSD** (latencia baja) o **HDD** (más barato).
- Deduplicación de datos, shadow copies (versiones anteriores para el usuario), backups diarios a S3.
- Se accede desde on-premises por **VPN** o **Direct Connect**. Linux también lo puede montar por SMB.

**FSx for Lustre**
- Sistema de archivos **paralelo**: cientos de GB/s, millones de IOPS, sub-ms. **Solo Linux**.
- **Integración con S3 (data repository):** ves los objetos del bucket como archivos, los carga al leerlos (*lazy load*) y puede **escribir los resultados de vuelta a S3**.

| | Scratch | Persistent |
|---|---|---|
| Replicación | **No** (si falla un servidor, se pierden los datos) | Sí, **en la misma AZ** |
| Uso | Procesamiento **corto**, datos ya en S3 | Cargas **largas**, datos importantes |
| Costo | Más barato | Más caro |

**FSx for NetApp ONTAP**
- **NFS + SMB + iSCSI** → Linux, Windows y macOS sobre **los mismos datos**, y además volúmenes de **bloques**.
- **Snapshots**, **clonado instantáneo** (copias sin duplicar datos), **deduplicación y compresión**, replicación con **SnapMirror**.
- La capacidad crece automáticamente. Multi-AZ o Single-AZ.

**FSx for OpenZFS**
- **Solo NFS** (v3 y v4).
- Hasta **~1 millón de IOPS**, latencia **< 0,5 ms**.
- Snapshots, **clonado instantáneo** y compresión.

## Se confunde con…

### Árbol de decisión
```
¿Qué pide el enunciado?
├─ Windows / SMB / Active Directory .......... FSx for Windows
├─ HPC / ML / procesamiento paralelo / S3 .... FSx for Lustre
│     ├─ temporal, datos ya en S3 ............ Scratch
│     └─ largo plazo, datos importantes ...... Persistent
├─ NetApp / multi-protocolo / iSCSI .......... FSx for NetApp ONTAP
├─ ZFS ....................................... FSx for OpenZFS
└─ Linux compartido "normal", NFS ........... EFS
```

### Protocolos
| | NFS | SMB | iSCSI (bloques) |
|---|---|---|---|
| EFS | ✅ | ❌ | ❌ |
| FSx for Windows | ❌ | ✅ | ❌ |
| FSx for Lustre | Cliente Lustre (Linux) | ❌ | ❌ |
| FSx for NetApp ONTAP | ✅ | ✅ | ✅ |
| FSx for OpenZFS | ✅ | ❌ | ❌ |
> Cuenta los protocolos que pide el enunciado: si son **varios**, casi siempre es **ONTAP**.

### Lustre Scratch vs Persistent
> Pregúntate: **¿es temporal?** y **¿los datos originales ya están a salvo en S3?** Si las dos respuestas son sí → Scratch.

## Trampas de examen

**Frases → respuesta**
- "Windows", "SMB", "Active Directory", "carpetas compartidas de Windows" → **FSx for Windows**
- "...y debe seguir funcionando si cae una AZ" → **FSx for Windows Multi-AZ**
- "HPC", "machine learning", "renderizado", "procesamiento paralelo" → **FSx for Lustre**
- "dataset en S3", "procesar a muy alta velocidad", "unos días" → **Lustre Scratch enlazado a S3**
- "NetApp", "multi-protocolo", "Linux y Windows sobre los mismos datos", "iSCSI" → **FSx for NetApp ONTAP**
- "ZFS" → **FSx for OpenZFS**

**Opciones trampa**
- EFS para Windows → solo NFS.
- FSx for Windows para HPC → no es un sistema de archivos paralelo.
- Lustre Scratch para datos que no se pueden perder → no replica.
- Single-AZ + backups diarios como "alta disponibilidad" → restaurar no es un failover automático.
- OpenZFS para SMB → solo NFS.

**Método (Día 8)**
1. Busca la **marca o el protocolo** en el enunciado (Windows, NetApp, ZFS, HPC/ML) → eso te da la familia.
2. Después lee la **disponibilidad o el costo** → Multi-AZ vs Single-AZ, Scratch vs Persistent.

## Lo vi en mis proyectos

- Ninguno lo usó todavía. Las dos apps (TMS serverless y la plataforma three-tier) no tenían servidores Windows ni cargas HPC.
