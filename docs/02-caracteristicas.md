# 02. Características

[Inicio](../README.md) · [Anterior: Datos](01-datos.md) · [Siguiente: Entrenamiento](03-entrenamiento.md)

## 1. Objetivo

Transformar cada consulta anonimizada en el **vector numérico** que usan los modelos, con una regla fundamental: **la misma transformación, con el mismo código y la misma versión, se usa al entrenar y al responder al médico**.

La etapa existe para evitar el error más común al llevar un modelo a producción, el desajuste entre entrenamiento y servicio (*training-serving skew*): si en entrenamiento la fiebre se codifica de una forma y en el servicio de otra, el modelo responde mal sin que nada falle de forma visible.

## 2. Entradas y salidas

| Entrada | Formato | Origen |
|---|---|---|
| Conjunto de datos anonimizado y versionado | Parquet | [01. Datos](01-datos.md) |
| Ontología HPO y tabla de correspondencias hacia HPO | Archivo de la ontología (versión fijada) y tabla CSV versionada en Git | Proyecto HPO y médico de referencia |
| Consulta individual (en producción) | JSON con el contrato de entrada de [00. Contexto y supuestos](00-contexto-y-supuestos.md) | Servicio de [07. Despliegue](07-despliegue.md) |

| Salida | Formato | Quién la consume |
|---|---|---|
| Matriz de características para entrenar | Parquet versionado con DVC | [03. Entrenamiento](03-entrenamiento.md) |
| Transformador ajustado | Objeto `Pipeline` de scikit-learn serializado, registrado en MLflow junto al modelo | Entrenamiento, validación y servicio |
| Paquete `diagnostico_features` | Paquete Python versionado (por ejemplo `features 1.3.0`) | Pipeline de entrenamiento y servicio, que lo instalan en la misma versión |

## 3. Proceso

1. **Normalización de síntomas a HPO.** Cada síntoma se lleva a un término de la ontología HPO:
   - Si llega en HPO, se valida que exista en la versión fijada de la ontología.
   - Si llega en SNOMED CT o CIE-10, se traduce con la **tabla de correspondencias**, mantenida por el médico de referencia y versionada en Git (cada cambio entra por PR y queda registrado).
   - Si no tiene correspondencia, el síntoma se marca como "no mapeado", se cuenta en el reporte y no se usa. Si más del 5 % de los síntomas de una versión quedan sin mapear, la etapa falla (ver criterios de salida).
2. **Expansión por la jerarquía.** A cada término se le agregan sus términos ancestros en HPO (por ejemplo, "fiebre alta" también activa "fiebre" y "alteración de la temperatura"). Esto permite que dos pacientes descritos con distinto nivel de detalle se parezcan, algo clave para las huérfanas.
3. **Construcción del vector**, con columnas de cuatro tipos:
   - **Síntomas:** una columna por término HPO frecuente (los que aparecen en al menos 20 consultas, más todos los términos de las huérfanas priorizadas), con la intensidad (0 a 10) como valor. Es una matriz dispersa.
   - **Duración:** duración máxima del cuadro en días, transformada con logaritmo, y un indicador de duración de 30 días o más (el mismo corte que usa hoy la regla de enfermedad crónica de la Unidad 2).
   - **Signos vitales:** temperatura, frecuencia cardiaca, presión arterial y saturación, recortados a rangos clínicos válidos e imputados con la mediana por grupo de edad cuando faltan, con una columna indicadora de "faltante".
   - **Demografía:** edad (numérica) y sexo (codificación *one-hot*).
4. **Ajuste y empaquetado.** Los pasos con parámetros (medianas, vocabulario de términos) se ajustan **solo con los datos de entrenamiento** de cada partición, dentro de un `Pipeline` de scikit-learn, para que la validación no vea información del conjunto de prueba.
5. **Publicación.** La matriz se versiona con DVC; el transformador ajustado se registra en MLflow con el modelo; el código vive en el paquete `diagnostico_features`, con pruebas unitarias y versión semántica.

## 4. Tecnologías

| Necesidad | Tecnología elegida | Alternativas consideradas | Por qué sí y por qué no | Costo y operación |
|---|---|---|---|---|
| Encadenar transformaciones de forma reproducible | **scikit-learn `Pipeline`** y `ColumnTransformer` dentro del paquete `diagnostico_features` | Código suelto en notebooks; Spark ML | Un `Pipeline` se ajusta y aplica igual en entrenamiento y servicio y evita la fuga de datos entre particiones. Los notebooks no garantizan que producción haga lo mismo. Spark sobra para el volumen del escenario (G10) | Sin licencias |
| Trabajar con la ontología HPO | **PyHPO** (biblioteca de Python para HPO) | Cargar el archivo de la ontología con código propio | PyHPO ya resuelve ancestros, información contenida y similitud entre conjuntos de términos, que también usa [03. Entrenamiento](03-entrenamiento.md) para huérfanas | Sin licencias |
| Guardar y servir características | **Sin *feature store*** al inicio: las características se calculan a partir de la propia consulta | Feast; *feature store* administrado de un proveedor de nube | Todas las características salen de la consulta que llega; no hay que buscar historia del paciente en línea ni compartir características entre muchos modelos. Un *feature store* agregaría un servicio que operar sin beneficio para un equipo de 3 (G6) | Sin costo |
| Mantener la tabla de correspondencias | CSV en Git, con cambios por PR | Hoja de cálculo compartida; base de datos de terminologías | Git da historial, revisión y versión exacta para cada modelo. Una hoja compartida no tiene trazabilidad | Sin costo |

**Cuándo sí tendría sentido un *feature store*.** Si el modelo empieza a usar la historia del paciente (consultas anteriores, resultados de laboratorio previos), habría que consultar esos datos en el momento de la predicción de la misma forma en entrenamiento y en servicio. Ese es el caso de uso de Feast, y se evaluaría entonces.

## 5. Suposiciones

| # | Suposición | Implicación si deja de cumplirse |
|---|---|---|
| S02.1 | La tabla de correspondencias hacia HPO cubre al menos el 95 % de los síntomas registrados | Más síntomas "no mapeados" significa información perdida; el médico de referencia debe ampliar la tabla antes de entrenar |
| S02.2 | La ontología HPO se fija en una versión por cada versión del paquete de características | Si se actualiza sin control, un mismo síntoma podría codificarse distinto entre entrenamiento y servicio. Por eso la versión de la ontología es parte de la versión del paquete |
| S02.3 | La intensidad de 0 a 10 la registra el médico con un criterio parecido en todas las sedes | Si cada sede usa la escala distinto, la intensidad introduce ruido; se puede pasar a una escala de tres niveles más robusta |
| S02.4 | El servicio puede instalar el mismo paquete de Python que el entrenamiento | Si el servicio cambiara de lenguaje, habría que exportar también la transformación (no solo el modelo) a un formato portable |

## 6. Enfermedades comunes y huérfanas

| Aspecto | Comunes (muchos datos) | Huérfanas (pocos datos) |
|---|---|---|
| Términos HPO incluidos | Los frecuentes (al menos 20 consultas) | **Todos** los términos de las huérfanas priorizadas, aunque casi no aparezcan en los datos, porque son justamente los que las distinguen |
| Uso de la jerarquía | Ayuda a agrupar descripciones parecidas | Es **esencial**: la similitud de fenotipos compara al paciente con la descripción de la enfermedad en HPO, que suele estar a otro nivel de detalle |
| Representación adicional | Ninguna | Además del vector, se conserva el **conjunto de términos HPO** de la consulta, que usa la rama de huérfanas para calcular similitud semántica |

## 7. Criterios de salida y controles de calidad

- Pruebas unitarias del paquete `diagnostico_features` en verde, incluidas pruebas de **paridad**: un mismo lote de consultas transformado por la ruta de entrenamiento y por la ruta del servicio produce exactamente la misma matriz.
- Síntomas "no mapeados" por debajo del 5 %.
- Ninguna columna constante ni completamente vacía.
- El transformador ajustado y la versión del paquete quedan registrados en el mismo *run* de MLflow que el modelo.

## 8. Riesgos

| Riesgo | Mitigación |
|---|---|
| Desajuste entre entrenamiento y servicio | Un solo paquete versionado para las dos rutas y prueba de paridad en CI ([06. CI/CD](06-ci-cd.md)) |
| Fuga de información del conjunto de prueba | Todos los pasos con parámetros se ajustan dentro del `Pipeline`, solo con datos de entrenamiento |
| Correspondencias erróneas hacia HPO | Cambios por PR revisados por el médico de referencia; el reporte muestra los síntomas más frecuentes sin mapear |
| Demasiadas columnas dispersas | Umbral mínimo de frecuencia para comunes; los modelos de árboles y la similitud manejan bien matrices dispersas |

## 9. Cambios respecto a la propuesta original

| Antes (propuesta original) | Ahora | Motivo |
|---|---|---|
| "Normalización de síntomas a un vocabulario común" sin decir cuál | Normalización a HPO con una tabla de correspondencias versionada y expansión por la jerarquía | HPO es el vocabulario que también usan Orphanet y la rama de huérfanas |
| Las características se guardaban en un *feature store* "para usar las mismas en entrenamiento e inferencia" | Sin *feature store*: un paquete versionado y una prueba de paridad cumplen ese objetivo con menos operación; se deja escrito cuándo sí haría falta | Proporcional al escenario y al tamaño del equipo (G6) |
| Sin herramienta ni control contra el desajuste entre entrenamiento y servicio | scikit-learn `Pipeline` y prueba de paridad automática | El riesgo existía pero no tenía control verificable |
