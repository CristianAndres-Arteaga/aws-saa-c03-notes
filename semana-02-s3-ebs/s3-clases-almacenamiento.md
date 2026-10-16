# S3: clases de almacenamiento

**Dominio:** COST | PERF
**Semana:** 02

## Qué es

Las clases de almacenamiento son opciones dentro de S3 para guardar objetos (no son servicios aparte: el servicio es S3). Son 8 y todas tienen durabilidad de 11 nueves (99.999999999 %), o sea que no pierden datos. Lo que cambia entre ellas es el precio de guardar y de leer, la disponibilidad (si puedo leer el dato de inmediato o tarda) y la resiliencia ante la caída de una AZ. Elegir una es apostar por cuánto voy a leer ese dato.

## Cuándo usarlo

- **Standard** → acceso frecuente.
- **Intelligent-Tiering** → patrón de acceso desconocido o cambiante.
- **Standard-IA** → acceso poco frecuente (~1 vez al mes), pero inmediato cuando se pide.
- **One Zone-IA** → igual que Standard-IA, pero solo para datos recreables (1 AZ).
- **Glacier Instant Retrieval** → archivo que se consulta ~1 vez por trimestre y necesita milisegundos.
- **Glacier Flexible Retrieval** → archivo que puede esperar minutos (Expedited) u horas.
- **Glacier Deep Archive** → retención legal de años, casi nunca se lee y se aceptan 12–48 h de espera.
- **Express One Zone** → latencia de milisegundos de un dígito (analítica intensiva, ML) con datos recreables.

## Cuándo NO usarlo (y qué usar en su lugar)

- **Standard-IA** para archivos que se borran antes de 30 días → usar **Standard**, porque IA cobra 30 días mínimo aunque borres antes.
- **One Zone-IA / Express One Zone** para datos que NO se pueden recrear → usar **Standard-IA** (o Standard), porque guardan en 1 sola AZ y si la AZ se destruye se pierden los datos.
- **Glacier Flexible / Deep Archive** cuando piden acceso inmediato o en milisegundos → usar **Glacier Instant Retrieval** o **Standard-IA**, porque tardan minutos u horas en recuperar.
- **Glacier Flexible** para retención de años cuando se aceptan 12–48 h de espera y piden costo mínimo → usar **Deep Archive**, porque es ~4 veces más barato. (Si piden recuperar en minutos, Flexible sigue siendo la correcta.)
- **Intelligent-Tiering** con millones de objetos muy pequeños (<128 KB) → usar **Standard**, porque los objetos chicos no se mueven de tier y no ahorras nada.
- **Intelligent-Tiering** cuando el patrón de acceso es predecible → usar **Standard + regla de lifecycle**, porque hace lo mismo sin cobrar monitoreo.
- **Standard-IA / One Zone-IA** con archivos de menos de 128 KB → usar **Standard**, porque IA cobra cada objeto como si pesara 128 KB.
- **Standard** cuando los datos pasan meses sin leerse → usar **Standard-IA**, **Glacier Instant Retrieval** o una **regla de lifecycle**, porque Standard es la clase más cara para guardar.

## Datos clave

**Iguales en todas las clases**
- Durabilidad: 11 nueves (99.999999999 %) → ninguna "pierde" datos por sí sola.
- Lo que cambia: precio de guardar, precio de leer, velocidad de recuperación y número de AZs.

**Número de AZs**
- 1 sola AZ: One Zone-IA y Express One Zone → si la AZ se destruye, se pierden los datos.
- El resto: 3 AZs o más.

**Duración mínima (te cobran aunque borres antes)**
- Standard e Intelligent-Tiering: sin mínimo
- Standard-IA y One Zone-IA: 30 días
- Glacier Instant y Glacier Flexible: 90 días
- Deep Archive: 180 días

**Tamaño mínimo facturable**
- Standard-IA y One Zone-IA: 128 KB por objeto
- Intelligent-Tiering: los objetos de menos de 128 KB no se monitorean (se quedan en el tier frecuente)

**Tiempo de recuperación**
| Clase | Tiempo |
|---|---|
| Standard, IA, One Zone-IA, Glacier Instant | milisegundos |
| Express One Zone | milisegundos de un dígito (lo más rápido) |
| Glacier Flexible: Expedited / Standard / Bulk | 1–5 min / 3–5 h / 5–12 h |
| Deep Archive: Standard / Bulk | hasta 12 h / hasta 48 h |

**Precio aproximado de guardar (USD por GB al mes, us-east-1)**
- Standard 0.023 → IA 0.0125 → One Zone-IA 0.01 → Glacier Instant 0.004
  → Glacier Flexible 0.0036 → Deep Archive ~0.001
- Regla: cuanto más barato guardar, más caro o más lento sacar.

**Cobros extra**
- IA, One Zone-IA y todas las Glacier cobran por GB recuperado.
- Intelligent-Tiering no cobra recuperación, pero cobra monitoreo por objeto.
- Glacier Expedited "garantizado" → requiere Provisioned Capacity.

## Se confunde con…

### Standard-IA vs Glacier Instant Retrieval
| | Standard-IA | Glacier Instant Retrieval |
|---|---|---|
| Para qué sirve | Datos que se leen poco (~1 vez al mes) | Archivo que se lee muy poco (~1 vez por trimestre) |
| Diferencia clave | Guardar más caro, leer más barato. Mínimo 30 días | Guardar más barato, leer más caro. Mínimo 90 días |
| Velocidad | milisegundos | milisegundos (igual) |
> Las dos son instantáneas: decide la FRECUENCIA de lectura.

### Glacier Flexible Retrieval vs Deep Archive
| | Glacier Flexible | Deep Archive |
|---|---|---|
| Para qué sirve | Archivo que puede esperar minutos u horas | Retención de años, casi nunca se lee |
| Diferencia clave | Recuperación Expedited en 1–5 min. Mínimo 90 días | La más barata, pero 12–48 h. Mínimo 180 días |
> Decide el PLAZO: si piden minutos, Flexible. Si aceptan 12 h o más y el costo es mínimo, Deep Archive.

### Standard-IA vs One Zone-IA
| | Standard-IA | One Zone-IA |
|---|---|---|
| Para qué sirve | Datos poco usados que NO puedes perder | Datos poco usados que se pueden RECREAR |
| Diferencia clave | 3 AZs o más | 1 AZ, ~20 % más barata |
> Decide si los datos se pueden recrear.

### Intelligent-Tiering vs Standard + regla de lifecycle
| | Intelligent-Tiering | Lifecycle |
|---|---|---|
| Para qué sirve | Patrón de acceso IMPREDECIBLE | Patrón PREDECIBLE (por edad) |
| Diferencia clave | Mueve según el uso real y cobra monitoreo por objeto | Mueve según los días, sin costo de monitoreo |
> Decide si el patrón de acceso es predecible.


## Trampas de examen

**Frases → respuesta**
- "patrón de acceso impredecible / desconocido / cambiante" → Intelligent-Tiering
- "se puede recrear / regenerar / es una copia" → One Zone-IA (o Express One Zone si piden latencia)
- "latencia de milisegundos de un dígito / ML / analítica de alto rendimiento" → Express One Zone
- "poco frecuente pero acceso inmediato" → Standard-IA
- "~1 vez por trimestre + milisegundos" → Glacier Instant Retrieval
- "en minutos" → Glacier Flexible con Expedited
- "garantizar la recuperación Expedited" → Provisioned Capacity
- "retención legal de años + 12 o 48 horas + costo mínimo" → Deep Archive
- "patrón predecible (por edad) + minimizar costo" → Standard + regla de lifecycle

**Opciones trampa (parecen buenas pero no lo son)**
- Datos que viven menos de 30 días en IA → más caro que Standard (mínimo de 30 días).
- Millones de archivos chiquitos en IA → cobra 128 KB por objeto.
- One Zone para datos que NO se pueden recrear → si cae la AZ, se pierden.
- Glacier Flexible o Deep Archive cuando piden "inmediato" → tardan minutos u horas.
- Elegir la opción "más rápida" o "más segura" cuando el enunciado pide "minimizar costo".

**Método para cada pregunta**
1. Subrayar TODAS las restricciones: plazo, frecuencia, ¿recreable?, ¿predecible?, costo.
2. Mirar los números: días, horas, KB, cantidad de objetos.
3. Descartar las clases cuya letra chica choca con el enunciado.
4. Entre las que quedan, elegir la más barata que cumple.

## Lo vi en mis proyectos

**TMS: frontend estático en S3**
- Clase usada: Standard (la clase por defecto).
- ¿Fue correcta? Sí:
  - Los archivos del frontend son pequeños (muchos pesan menos de 128 KB). En IA
    se cobrarían como si pesaran 128 KB, y en Intelligent-Tiering no se moverían de tier.
  - Se leen cada vez que alguien abre la app (acceso frecuente). En IA pagaría
    cobro por GB leído en cada visita.
  - El volumen total es mínimo (pocos MB), así que el costo de guardar es casi cero
    en cualquier clase. Optimizar aquí no ahorra nada.
- Aprendizaje: para sitios estáticos, Standard es la correcta. El ahorro real
  está en el CDN (CloudFront), que reduce las lecturas a S3, no en cambiar de clase.
- Otros buckets que no se ven: SAM también crea un bucket para los artefactos de
  despliegue. Ese sí podría llevar una regla de lifecycle que borre las versiones viejas.
