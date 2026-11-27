# 04. Validación

[Inicio](../README.md) · [Anterior: Entrenamiento](03-entrenamiento.md) · [Siguiente: Registro y aprobación](05-registro-y-aprobacion.md)

## 1. Objetivo

Decidir, con criterios fijados **antes** de mirar los resultados, si un modelo candidato es lo bastante bueno, justo y comprensible para ponerlo frente a un médico. Es la **compuerta de calidad** del pipeline: el rombo "¿Cumple los umbrales?" del diagrama general.

La etapa existe porque en salud un error del modelo tiene consecuencias clínicas. Una métrica alta en promedio puede esconder que el modelo falla en mujeres, en una sede rural o en una huérfana concreta; por eso se mide por subgrupo y lo revisan médicos, no solo un número.

## 2. Entradas y salidas

| Entrada | Formato | Origen |
|---|---|---|
| Modelos candidatos con su *run* de MLflow | Artefactos de MLflow | [03. Entrenamiento](03-entrenamiento.md) |
| Particiones de evaluación | Parquet versionado con DVC (identificadores de consultas de cada partición) | Definidas en esta etapa sobre la salida de [02. Características](02-caracteristicas.md) |
| Umbrales de aprobación | Archivo `umbrales.yaml` versionado en Git | Acordados con el comité de aprobación |
| Modelo en producción (*champion*) | Versión del registro de modelos | [05. Registro y aprobación](05-registro-y-aprobacion.md), para comparar |
| Casos de referencia | Conjunto fijo de consultas con resultado esperado, revisado por médicos | Médico de referencia |

| Salida | Formato | Quién la consume |
|---|---|---|
| Reporte de validación | HTML y JSON registrados en el *run* de MLflow | Comité de aprobación |
| Decisión de la compuerta | "Aprobado para revisión" o "rechazado", con el umbral que falló | Si aprueba, [05. Registro y aprobación](05-registro-y-aprobacion.md); si no, vuelve a entrenamiento |
| Umbrales de decisión del modelo | Umbral de probabilidad por clase y por huérfana | Servicio de [07. Despliegue](07-despliegue.md), junto con el modelo |

## 3. Proceso

1. **Particiones.** Se reservan tres conjuntos que el entrenamiento nunca ve:
   - **Prueba temporal:** las consultas de los últimos 6 meses. Mide si el modelo funciona con datos más recientes que los que aprendió, como pasará en producción.
   - **Validación externa por sede:** se entrena sin una sede y se evalúa en ella, rotando por las 5 sedes. Simula llevar el modelo a una institución nueva y detecta si depende de cómo registra cada sede.
   - **Huérfanas:** como hay pocos casos, se usa validación dejando uno fuera (*leave-one-out*): cada caso confirmado se evalúa con un modelo que no lo vio.
2. **Métricas de enfermedades comunes:** sensibilidad (*recall*) por clase y promedio, precisión, AUC-PR, exactitud de los 5 primeros candidatos (la enfermedad correcta está entre los 5 que ve el médico) y calibración (error de calibración esperado y puntaje de Brier).
3. **Métricas de huérfanas:** proporción de casos confirmados en los que la huérfana correcta aparece entre los 5 candidatos o activa la alerta, y tasa de alertas falsas por cada 1.000 consultas (para que la alerta no se vuelva ruido).
4. **Equidad.** Con **Fairlearn** se calcula la sensibilidad por sexo, grupo de edad y sede, y la diferencia entre el mejor y el peor subgrupo.
5. **Explicabilidad.** Con **SHAP** se obtienen los síntomas que más pesan en cada predicción (lo que alimenta el campo `explicacion` de la salida) y la importancia global. El médico de referencia revisa que las variables importantes tengan sentido clínico.
6. **Casos de referencia (prueba de humo clínica).** El modelo debe dar el resultado esperado en el conjunto fijo de casos revisados por médicos, que incluye al menos un caso por cada uno de los 5 estados de gravedad y un caso por cada huérfana priorizada.
7. **Comparación con el modelo en producción.** El candidato no puede empeorar ninguna métrica principal en más de 0,02 frente al *champion* actual, y debe superar la línea base.
8. **Rendimiento.** Latencia de una predicción en una máquina equivalente a un PC de consultorio (4 núcleos, sin GPU, G5).
9. **Compuerta automática.** Un script compara todas las métricas contra `umbrales.yaml`. Si alguna falla, el modelo se rechaza y se registra el motivo; si todas pasan, el modelo queda "aprobado para revisión" y pasa a la revisión médica del comité en la etapa siguiente.

### Umbrales iniciales

Los valores son el punto de partida acordado con el comité y viven en `umbrales.yaml`, versionado en Git; cambiarlos exige un PR aprobado por el médico de referencia.

| Métrica | Umbral |
|---|---|
| Sensibilidad promedio en comunes (prueba temporal) | ≥ 0,80, con precisión promedio ≥ 0,50 |
| Exactitud de los 5 primeros candidatos | ≥ 0,90 |
| Error de calibración esperado | ≤ 0,05 |
| Huérfanas: caso correcto entre los 5 candidatos o con alerta | ≥ 0,70 de los casos confirmados |
| Alertas falsas de huérfanas | ≤ 20 por cada 1.000 consultas |
| Diferencia de sensibilidad entre subgrupos (sexo, edad, sede) | ≤ 0,10 |
| Validación externa: peor sede frente al promedio | No más de 0,05 por debajo |
| Casos de referencia | 100 % con el resultado esperado |
| Latencia en CPU (percentil 95) | ≤ 500 ms |
| Frente al modelo en producción | Ninguna métrica principal empeora más de 0,02 |

## 4. Tecnologías

| Necesidad | Tecnología elegida | Alternativas consideradas | Por qué sí y por qué no | Costo y operación |
|---|---|---|---|---|
| Métricas de clasificación y calibración | **scikit-learn** (`metrics`, `calibration`) | Implementación propia; `mlflow.evaluate` | Es la biblioteca de referencia, ya está en el pipeline y sus métricas son estándar y auditables. `mlflow.evaluate` es útil para registrar, pero no cubre las métricas por huérfana ni los umbrales propios | Sin licencias |
| Equidad por subgrupo | **Fairlearn** (`MetricFrame`) | AIF360; cálculo manual | Fairlearn calcula cualquier métrica por subgrupo con poco código y se integra con scikit-learn. AIF360 es más amplio pero más pesado de mantener | Sin licencias |
| Explicabilidad | **SHAP** (`TreeExplainer`) | LIME; importancia nativa de LightGBM | `TreeExplainer` es exacto y rápido para árboles y da explicaciones por predicción y globales. LIME es aproximado e inestable entre ejecuciones; la importancia nativa no explica predicciones individuales | Sin licencias |
| Umbrales como código | Archivo **YAML** en Git y un script de compuerta | Umbrales en una hoja o en la cabeza del equipo | Versionar los umbrales deja trazado quién los cambió y por qué, y permite que la compuerta sea automática | Sin costo |
| Registro del reporte | **MLflow** (artefactos del *run*) | Carpeta compartida | El reporte queda unido al modelo exacto que evaluó | Sin costo adicional |

## 5. Suposiciones

| # | Suposición | Implicación si deja de cumplirse |
|---|---|---|
| S04.1 | Los umbrales de la tabla son clínicamente aceptables para el comité | Se ajustan por PR antes del primer despliegue; la estructura de la compuerta no cambia |
| S04.2 | Los últimos 6 meses representan lo que verá el modelo en producción | Si hubo un cambio fuerte (por ejemplo, una epidemia), la prueba temporal se complementa con otra ventana y se documenta |
| S04.3 | Cada sede tiene suficientes consultas para evaluarla por separado | Si una sede rural tiene muy pocas, se agrupa con otra de condiciones parecidas para la validación externa |
| S04.4 | El médico de referencia mantiene el conjunto de casos de referencia | Sin ese conjunto, la prueba de humo clínica pierde valor; es un compromiso del rol, no opcional |
| S04.5 | Para huérfanas, *leave-one-out* da una estimación útil aunque con mucha variabilidad | Los resultados de huérfanas se reportan con su intervalo de confianza y siempre pasan por revisión de especialistas |

## 6. Enfermedades comunes y huérfanas

| Aspecto | Comunes (muchos datos) | Huérfanas (pocos datos) |
|---|---|---|
| Partición | Prueba temporal y validación externa por sede | Dejando uno fuera, por la escasez de casos |
| Métrica principal | Sensibilidad con precisión mínima, AUC-PR y calibración | Caso correcto entre los 5 candidatos o con alerta, y tasa de alertas falsas |
| Confianza estadística | Alta: miles de casos | Baja: se reporta con intervalo de confianza |
| Revisión humana | Muestra de casos | **Todos** los falsos negativos de huérfanas los revisa un especialista |

## 7. Criterios de salida y controles de calidad

- Todos los umbrales de `umbrales.yaml` cumplidos, verificados por el script de compuerta.
- Reporte de validación completo y registrado en el *run* de MLflow.
- Umbrales de decisión por clase y por huérfana calculados y guardados con el modelo.
- Si se rechaza, el motivo queda registrado y el equipo vuelve a [03. Entrenamiento](03-entrenamiento.md); un modelo rechazado nunca llega al registro.

## 8. Riesgos

| Riesgo | Mitigación |
|---|---|
| Ajustar los umbrales para que un modelo pase | Los umbrales se fijan antes y solo cambian por PR aprobado por el médico de referencia |
| Buen promedio que esconde fallas en un subgrupo | Métricas por sexo, edad y sede, con diferencia máxima |
| Sobreestimar el desempeño en huérfanas | *Leave-one-out*, intervalos de confianza y revisión de todos los falsos negativos |
| Explicaciones sin sentido clínico que generen desconfianza | Revisión de SHAP por el médico de referencia antes de aprobar |

## 9. Cambios respecto a la propuesta original

| Antes (propuesta original) | Ahora | Motivo |
|---|---|---|
| Métricas nombradas (recall, AUC-PR, calibración) sin umbrales | Tabla de umbrales concretos, versionados en YAML y aplicados por una compuerta automática | "Verificable" en lugar de "deseable" |
| "Partición estratificada y validación cruzada" | Prueba temporal, validación externa por sede y *leave-one-out* para huérfanas | Cada partición responde a un riesgo concreto del escenario |
| "Validación externa con datos de otra institución" | Validación rotando las 5 sedes de la red | El escenario ya tiene varias instituciones, no hace falta conseguir otra |
| "Pruebas de equidad por subgrupos" sin herramienta | Fairlearn con diferencia máxima entre subgrupos | Tecnología concreta y criterio medible |
| Explicabilidad mencionada solo como restricción | SHAP por predicción, que alimenta el campo `explicacion` de la salida | La explicación llega al médico |
| No había comparación con el modelo anterior | El candidato no puede empeorar más de 0,02 frente al modelo en producción | Evitar regresiones al reentrenar |
