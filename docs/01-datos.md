# 01. Datos

[Inicio](../README.md) · [Anterior: Contexto y supuestos](00-contexto-y-supuestos.md) · [Siguiente: Características](02-caracteristicas.md)

## 1. Objetivo

Convertir la historia clínica de la red en un **conjunto de datos anonimizado, validado y versionado**, que pueda salir del servidor de la red hacia el entorno de ML sin exponer a ningún paciente.

La etapa existe porque todo lo que viene después depende de ella. Si los datos llegan con identificadores, el proyecto incumple la ley; si llegan con errores, el modelo los aprende; si no están versionados, ningún entrenamiento se puede reproducir ni auditar. Es también la **única etapa que corre dentro de la red** con datos identificables (G7).

## 2. Entradas y salidas

| Entrada | Formato | Origen |
|---|---|---|
| Consultas con síntomas, signos vitales y diagnóstico confirmado | Recursos FHIR `Patient`, `Encounter`, `Observation` y `Condition`, exportados en NDJSON | Historia clínica electrónica de la red (G1) |
| Correcciones de los médicos (etiquetas nuevas) | Registros JSON con `id_consulta`, diagnóstico confirmado y fecha | [09. Retroalimentación y reentrenamiento](09-retroalimentacion-y-reentrenamiento.md) |
| Catálogos de referencia | Listado de enfermedades huérfanas priorizadas con su ORPHAcode, códigos CIE-10 de las comunes incluidas | Médico de referencia y especialistas |
| Clave de seudonimización | Secreto de 256 bits | Bóveda de secretos del servidor de la red; nunca sale de ella |

| Salida | Formato | Quién la consume |
|---|---|---|
| Conjunto de datos anonimizado | Archivos Parquet, una fila por consulta | [02. Características](02-caracteristicas.md) |
| Versión del conjunto | Etiqueta `datos-vAAAA.MM.DD` en Git con su archivo `.dvc` (hash SHA-256 de cada archivo) | Entrenamiento, validación y registro, para trazar qué datos produjeron cada modelo |
| Reporte de calidad | HTML y JSON con el resultado de cada regla | Equipo de ML; bloquea la publicación si falla |
| Reporte de riesgo de reidentificación | JSON con el *k* mínimo por combinación de cuasi identificadores | Responsable de seguridad de la red ([10. Seguridad, gobierno y ética](10-seguridad-gobierno-y-etica.md)) |

## 3. Proceso

1. **Exportación incremental.** Cada semana, un trabajo programado pide a la historia clínica las consultas nuevas o modificadas desde la última exportación, usando la exportación masiva de FHIR (`$export`, formato NDJSON). Solo se exportan consultas con diagnóstico confirmado; las que no lo tienen esperan a la siguiente semana (G4).
2. **Aplanado.** Los recursos FHIR se convierten en una tabla con una fila por consulta: edad en años, sexo, sede, signos vitales, lista de síntomas (código, intensidad y duración) y diagnóstico confirmado (CIE-10 u ORPHAcode).
3. **Anonimización**, en este orden:
   - Se eliminan los identificadores directos: nombre, documento, dirección, teléfono y correo.
   - El identificador del paciente y el de la consulta se reemplazan por un **seudónimo HMAC-SHA256** calculado con la clave de la red. El mismo paciente produce siempre el mismo seudónimo, lo que permite unir sus consultas sin saber quién es; sin la clave, el seudónimo no se puede revertir.
   - Las fechas se convierten en **días relativos** a la primera consulta del paciente y en mes y año de la consulta. La fecha de nacimiento desaparece; queda la edad, agrupada en "90 o más" desde los 90 años.
   - Los campos de texto libre que todavía existan (por ejemplo, una descripción de síntoma escrita a mano) pasan por **Presidio** para detectar y quitar nombres, documentos y teléfonos antes de cualquier otro uso.
4. **Control de riesgo de reidentificación.** Se calcula el *k*-anonimato sobre los cuasi identificadores (grupo de edad de 5 años, sexo, sede y mes). Si alguna combinación tiene menos de 5 personas, se generaliza (por ejemplo, la sede pasa a "urbana" o "rural") hasta cumplir *k* ≥ 5.
5. **Validación de calidad** con **Pandera**: tipos, valores obligatorios, rangos clínicos (temperatura entre 35 y 42 °C, los mismos límites que ya valida la API de la Unidad 2), códigos existentes en los catálogos, intensidad de 0 a 10, máximo 20 síntomas por consulta, ausencia de duplicados y proporción de faltantes por columna por debajo del umbral.
6. **Publicación versionada.** Si la validación y el control de riesgo pasan, los archivos Parquet se registran con **DVC**: el contenido se sube al almacenamiento de objetos del entorno de ML y en Git queda solo el archivo `.dvc` con los hashes. El commit se etiqueta con la versión del conjunto.
7. **Aviso al pipeline.** La publicación de una versión nueva dispara la evaluación de los disparadores de reentrenamiento ([09. Retroalimentación y reentrenamiento](09-retroalimentacion-y-reentrenamiento.md)).

Los pasos 1 a 6 corren en el servidor de la red, en un contenedor ejecutado por un *runner* propio de GitHub Actions instalado allí ([11. Orquestación e infraestructura](11-orquestacion-e-infraestructura.md)). Así la orquestación es la misma que en el resto del pipeline, pero los datos identificables nunca salen de la red.

## 4. Tecnologías

| Necesidad | Tecnología elegida | Alternativas consideradas | Por qué sí y por qué no | Costo y operación |
|---|---|---|---|---|
| Extraer datos de la historia clínica | Exportación masiva **FHIR `$export`** (NDJSON) leída con Python | Consultas SQL directas a la base de la historia clínica; integración por mensajes HL7 v2 | FHIR es el estándar de la historia clínica de la red (G1) y no depende del esquema interno del proveedor. SQL directo se rompe con cada actualización del sistema; HL7 v2 obliga a interpretar mensajes uno por uno | Sin licencias. Requiere que TI habilite el permiso de exportación |
| Quitar datos personales del texto libre | **Microsoft Presidio** con modelo de idioma en español | Expresiones regulares propias; servicios de anonimización en la nube | Presidio es de código abierto, corre dentro de la red y combina reglas con reconocimiento de entidades. Las regex solas no detectan nombres; un servicio en la nube obligaría a sacar el texto sin anonimizar, contra G7 | Sin licencias. Se ejecuta en CPU |
| Seudonimizar identificadores | **HMAC-SHA256** con la biblioteca estándar de Python | Hash simple (SHA-256 sin clave); tablas de correspondencia | Un hash sin clave se revierte probando todos los números de documento posibles; con HMAC hace falta la clave, que no sale de la red. Una tabla de correspondencia es un archivo más que proteger | Sin costo. La clave vive en la bóveda de secretos de la red |
| Validar la calidad de los datos | **Pandera** | Great Expectations; validaciones escritas a mano | Pandera define el contrato de datos como código Python junto al pipeline y se prueba con pytest, como el resto del proyecto. Great Expectations es más completo pero exige su propia configuración, almacén de resultados y más operación para un equipo de 3 personas (G6). Las validaciones a mano no generan un reporte uniforme | Sin licencias |
| Versionar los datos | **DVC** sobre Git, con almacenamiento de objetos compatible con S3 | LakeFS; carpetas con fecha; versionado del propio almacenamiento | DVC usa el mismo flujo de Git y PR que el equipo ya domina en las Unidades 1 y 2, y deja en el repositorio el hash exacto de cada versión. LakeFS exige un servidor más; las carpetas con fecha no dan trazabilidad | Sin licencias. Costo del almacenamiento: unos pocos GB por versión |
| Formato de almacenamiento | **Parquet** | CSV; base de datos relacional | Parquet es columnar, comprimido y conserva tipos. CSV pierde tipos y pesa más; una base de datos agrega un servicio que operar sin necesidad | Sin costo |

## 5. Suposiciones

| # | Suposición | Implicación si deja de cumplirse |
|---|---|---|
| S01.1 | La historia clínica permite la exportación masiva FHIR (`$export`) con un usuario técnico de solo lectura | Se usa una vista de solo lectura acordada con TI y un mapeo propio a la tabla aplanada; el resto del proceso no cambia |
| S01.2 | Los síntomas están registrados como códigos (SNOMED CT, CIE-10 o HPO) en la mayoría de las consultas, y el texto libre es la excepción | Si la mayoría es texto libre, la extracción de síntomas pasa a ser un paso de procesamiento de lenguaje en [02. Características](02-caracteristicas.md), con más error y revisión médica adicional |
| S01.3 | Un diagnóstico confirmado es fiable como etiqueta: lo registró un médico después de exámenes o seguimiento | Si hay muchos diagnósticos provisionales, se filtra por tipo de diagnóstico y el volumen útil baja |
| S01.4 | La red tiene un servidor propio capaz de ejecutar contenedores y una bóveda de secretos | Sin servidor, se instala un equipo dedicado dentro de la red solo para esta etapa |
| S01.5 | Una exportación semanal es suficiente: las etiquetas llegan con días o semanas de retraso (G4) | Si se necesitara reaccionar más rápido, la exportación pasa a diaria sin cambiar las herramientas |
| S01.6 | Con *k* ≥ 5 y la generalización descrita, el riesgo de reidentificación es aceptable para el responsable de datos de la red | Si el comité de la red exige más, se sube *k* o se agregan técnicas como privacidad diferencial, a costa de perder detalle en huérfanas |

## 6. Enfermedades comunes y huérfanas

| Aspecto | Comunes (muchos datos) | Huérfanas (pocos datos) |
|---|---|---|
| Volumen esperado | Miles de consultas por diagnóstico | Pocas consultas por enfermedad, a veces menos de 10 (G3) |
| Riesgo de reidentificación | Bajo: muchas personas comparten edad, sexo y sede | **Alto**: un diagnóstico raro con edad y sede puede señalar a una persona. Por eso el control de *k*-anonimato se aplica antes de publicar y, si hace falta, se generaliza la sede |
| Fuentes adicionales | No se necesitan | Anotaciones de fenotipos de HPO y Orphanet (enfermedad con sus síntomas típicos y su frecuencia). Son públicas y no tienen datos de pacientes, así que no pasan por anonimización |
| Reglas de calidad | Proporción de faltantes y rangos | Además, **revisión manual** de cada caso nuevo de huérfana por el médico de referencia antes de incluirlo, porque un error en pocos casos pesa mucho |

## 7. Criterios de salida y controles de calidad

Una versión del conjunto de datos solo se publica si cumple **todo** lo siguiente; si algo falla, la publicación se detiene y se notifica al equipo con el reporte:

- Cero identificadores directos detectados en una muestra aleatoria de 1.000 filas revisada con Presidio después de la anonimización.
- *k*-anonimato ≥ 5 sobre los cuasi identificadores.
- Todas las reglas de Pandera en verde.
- Cada consulta tiene al menos un síntoma y un diagnóstico de los catálogos.
- La versión nueva no tiene menos filas que la anterior (las exportaciones son acumulativas); si las tiene, se investiga antes de publicar.
- El archivo `.dvc` y la etiqueta de versión están en Git.

## 8. Riesgos

| Riesgo | Mitigación |
|---|---|
| Fuga de datos identificables fuera de la red | Toda la etapa corre en el servidor de la red; la salida pasa por el doble control (Presidio y *k*-anonimato) antes de subir; la clave HMAC nunca sale de la bóveda |
| Cambio en la historia clínica que rompe la exportación | FHIR desacopla del esquema interno; las reglas de Pandera detectan columnas o códigos inesperados y detienen la publicación |
| Etiquetas erróneas | Solo diagnósticos confirmados; revisión manual en huérfanas; el monitoreo de desempeño compara contra correcciones de los médicos |
| Sesgo de representación (por ejemplo, pocas consultas de sedes rurales) | El reporte de calidad incluye la distribución por sede, sexo y grupo de edad; la equidad se mide en [04. Validación](04-validacion.md) |

## 9. Cambios respecto a la propuesta original

| Antes (propuesta original) | Ahora | Motivo |
|---|---|---|
| Fuentes nombradas en general (historias clínicas con HL7/FHIR, registros de raras, Orphanet, literatura) | Exportación FHIR `$export` semanal, con recursos concretos y catálogos de referencia | Una fuente sin mecanismo de extracción no es ejecutable |
| "Eliminación de identificadores directos, seudonimización de los restantes" | Anonimización en pasos definidos: HMAC con clave, fechas relativas, Presidio para texto libre y control de *k*-anonimato | El riesgo de reidentificación es mayor en huérfanas y necesita un control medible |
| "Validación de esquema, rangos y valores faltantes" sin herramienta | Contrato de datos en Pandera con reglas y umbrales explícitos, que bloquean la publicación | La rúbrica pide tecnologías justificadas y criterios verificables |
| "Datos versionados" sin herramienta | DVC con etiqueta por versión y hash SHA-256 | Reproducir y auditar cada entrenamiento |
| No decía dónde corre el proceso | Corre en el servidor de la red con un *runner* propio | Los datos identificables no salen de la red (G7) |
