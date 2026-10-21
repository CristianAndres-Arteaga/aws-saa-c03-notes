# S3: versionado, lifecycle y replicación (CRR/SRR)

**Dominio:** RES | COST | SEC
**Semana:** 02

## Qué es

- **Versionado:** cuando subo un archivo con el mismo nombre, S3 no tira el anterior: lo apila debajo y le da a cada uno su propio Version ID, así sé qué versión es cada una (analogía: pila de hojas).
- **Lifecycle:** un conserje automático que, según los días desde que se creó el objeto, lo mueve a una clase más barata (transición) o lo borra cuando ya no se usa (expiración).
- **Replicación:** copia automática de los objetos de un bucket a otro (analogía: fotocopiadora entre oficinas). Entre regiones distintas es CRR; dentro de la misma región (otra oficina en la misma ciudad) es SRR.

## Cuándo usarlo

- **Versionado** → para recuperar datos sobrescritos o borrados por accidente.
- **MFA Delete** → para que los datos tengan mejor protección y no sean fácilmente eliminados, ya que se necesita el código MFA de root para borrar versiones para siempre (protege incluso de un admin con credenciales robadas).
- **Lifecycle (transiciones)** → para mover objetos a clases más baratas a medida que envejecen y se usan menos.
- **Lifecycle (expiración / NoncurrentVersionExpiration)** → Expiration pone un post-it de borrado (con versionado no libera espacio) y NoncurrentVersionExpiration borra de verdad las versiones noncurrent X días después de que dejaron de ser current.
- **CRR** → recuperación ante desastre regional; cumplimiento ("copia a X km"); menor latencia para usuarios en otra región.
- **SRR** → juntar logs de varios buckets en uno; copiar de producción a una cuenta de pruebas; datos que por ley no pueden salir de la región pero necesitan copia en otra cuenta.
- **S3 Batch Replication** → para copiar los objetos que ya existían antes de activar la replicación.
- **S3 RTC** → cuando piden un tiempo de replicación garantizado (SLA de 15 minutos).

## Cuándo NO usarlo (y qué usar en su lugar)

- **Suspender el versionado** para bajar la factura → usar **lifecycle con NoncurrentVersionExpiration**, porque al suspender las versiones viejas se quedan y se siguen cobrando.
- **Expiration** en un bucket con versionado para liberar espacio → usar **NoncurrentVersionExpiration**, porque Expiration solo pone un post-it (delete marker) y no tritura nada.
- **Glacier Instant** en un tramo de lifecycle de 60 días → usar **Standard-IA**, porque Glacier Instant cobra mínimo 90 días (pagas 30 de más) y Standard-IA solo 30.
- **Glacier Flexible** cuando nadie lo mira y 48 h está bien → usar **Deep Archive**, porque también cumple las 48 h y es más barato.
- **CRR** cuando los datos no pueden salir de la región → usar **SRR**, porque copia dentro de la misma región (y puede ir a otra cuenta).
- **Delete marker replication** cuando la copia debe seguir legible tras un borrado → **dejarla desactivada**, porque si el post-it se copia, el GET en destino da 404.

## Datos clave

**Estados del versionado**
| Estado | Cuándo |
|---|---|
| Unversioned | Nunca se activó |
| Enabled | Activado |
| Suspended | Se activó y luego se pausó |
- De Enabled **nunca se puede** volver a Unversioned: solo suspender.
- Al suspender, las versiones existentes **se quedan y se siguen cobrando**.
- Subidas nuevas con versionado suspendido: Version ID **`null`** (se sobrescriben entre ellas).

**Operaciones con versionado**
| Operación | Resultado |
|---|---|
| PUT con el mismo nombre | Nueva versión current; la anterior pasa a noncurrent |
| GET sin versionId | Devuelve la current |
| GET con versionId | Devuelve esa versión concreta |
| DELETE sin versionId | Crea un delete marker (post-it) → GET da 404; nada se borra |
| DELETE del delete marker (con su versionId) | Restaura: la versión anterior vuelve a ser current |
| DELETE con versionId de una versión | Borrado permanente, no se puede deshacer |
- Facturación: se cobran **todas** las versiones. foto.jpg 3 × 2 MB → se cobran **6 MB**, porque las noncurrent siguen guardadas aunque un GET normal no las muestre.
- Crear otro delete marker NO restaura: el nuevo post-it queda arriba y sigue dando 404.

**MFA Delete**
- Exige MFA para: **borrar permanentemente una versión** y **cambiar el estado del versionado**.
- Lo activa: **solo la cuenta root** — solo desde: **CLI/API** (no consola).
- Requisito: **versionado activado**.

**Lifecycle: duración mínima (pagas los días que faltan si sale antes)**
| Clase | Mínimo |
|---|---|
| Standard | sin mínimo |
| Standard-IA / One Zone-IA | 30 días |
| Glacier Instant / Glacier Flexible | 90 días |
| Deep Archive | 180 días |
- Standard → Standard-IA: no antes del día **30** desde la creación.
- Las transiciones solo van **hacia abajo** (de caro a barato).
- Filtros posibles: **prefijo** (ej. `logs/`) y **tags**.
- Ejemplo del Día 3 (logs de auditoría): 0–30 Standard → 30–90 Standard-IA → 90+ Deep Archive.

**Lifecycle: acciones de limpieza**
| Acción | Qué hace |
|---|---|
| Expiration (current) | Sin versionado: borra el objeto. Con versionado: solo crea un delete marker |
| NoncurrentVersionExpiration | Borra de verdad las versiones noncurrent X días después (puede conservar las N más recientes) |
| ExpiredObjectDeleteMarker | Quita delete markers que ya no tienen versiones debajo |
| AbortIncompleteMultipartUpload | Borra partes de subidas grandes fallidas (no se ven, pero se cobran) |

**Replicación: requisitos**
1. **Versionado activado en origen Y destino.**
2. **Rol IAM** que permita a S3 leer del origen y escribir en el destino.
3. Otra cuenta: el dueño del bucket destino debe **permitirlo en su bucket policy**.
- Es **asíncrona** (llega segundos o minutos después).

**Replicación: qué se copia y qué no**
| Situación | ¿Se replica? |
|---|---|
| Objetos nuevos | Sí |
| Objetos que ya existían | No (usar **S3 Batch Replication**) |
| Delete markers | Opcional (delete marker replication) |
| DELETE con versionId | Nunca (por qué: para que un borrado malicioso en origen no destruya la copia) |
| Cadena A→B→C | No (configurar A→C directamente) |
| Objetos SSE-KMS | No por defecto: hay que activarlo e indicar una clave KMS de la región destino |
- RTC: **99.99 %** en **15** minutos, costo extra.
- Clase en destino: puede ser distinta (ej. **Deep Archive** para backup barato).

## Se confunde con…

### CRR vs SRR
| | CRR | SRR |
|---|---|---|
| Para qué sirve | Desastre regional, cumplimiento por distancia, latencia en otra región | Juntar logs, prod → cuenta de pruebas, datos que no pueden salir de la región |
| Diferencia clave | Destino en otra región | Destino en la misma región |
> Decide si los datos **pueden o deben** salir de la región.

### Expiration vs NoncurrentVersionExpiration
| | Expiration | NoncurrentVersionExpiration |
|---|---|---|
| Qué afecta | La versión current | Las versiones noncurrent (las de abajo de la pila) |
| ¿Libera espacio con versionado? | No (solo pone el post-it) | Sí (las tritura) |
> Decide si quieres **esconder** el objeto o **liberar espacio** de versiones viejas.

### Suspender versionado vs lifecycle
| | Suspender | Lifecycle (NoncurrentVersionExpiration) |
|---|---|---|
| ¿Baja la factura? | No (las versiones viejas siguen) | Sí |
| ¿Rompe la replicación? | Sí (la replicación exige versionado) | No |

## Trampas de examen

**Frases → respuesta**
- "recuperar de sobrescrituras / borrados accidentales" → versionado
- "nadie, ni un admin, pueda borrar versiones" → MFA Delete (lo activa root por CLI/API)
- "reducir costo de versiones antiguas, conservar N días" → NoncurrentVersionExpiration de N días
- "copia en otra región ante desastre regional" → CRR
- "no puede salir de la región + copia en otra cuenta" → SRR a otra cuenta
- "replicar también los datos existentes" → S3 Batch Replication
- "replicación garantizada en 15 minutos / SLA" → S3 RTC
- "costos ocultos de subidas grandes que fallan" → AbortIncompleteMultipartUpload

**Opciones trampa**
- Suspender el versionado para ahorrar → no baja la factura y rompe la replicación.
- Expiration en bucket versionado para liberar espacio → solo crea delete markers.
- Tramo de lifecycle más corto que el mínimo de la clase → pagas los días que faltan.
- Esperar que la CRR copie lo existente → solo copia lo nuevo; falta Batch Replication.
- Replicar delete markers cuando la copia debe seguir legible → el destino da 404.
- Crear otro delete marker para "restaurar" → hay que **borrar** el delete marker existente.

**Método (lo que aprendí el Día 3)**
1. Subrayar TODAS las restricciones (región, días, plazo, cuenta, costo).
2. Para lifecycle: contar los días de cada tramo y compararlos con el mínimo de la clase.
3. Si dos opciones se parecen, buscar la restricción que las separa (ej. B vs D: delete markers).
4. Descartar cada opción con su motivo antes de marcar.

## Lo vi en mis proyectos

**loan-collections, Módulo 11: copia de snapshot RDS a otra región**
- Es la misma idea que CRR: una copia en otra región (Ohio → Oregon) por si cae toda la región.
- La copia quedó cifrada con una clave KMS de us-west-2, porque las claves KMS son regionales: la de Ohio no se puede usar en Oregon. Es lo mismo que pasa al replicar objetos SSE-KMS: hay que indicar una clave de la región destino.

**TMS: bucket de artefactos de SAM**
- Le pondría NoncurrentVersionExpiration (y/o Expiration por prefijo) para borrar artefactos de despliegues viejos, porque se acumulan en cada deploy y nadie los vuelve a usar.
