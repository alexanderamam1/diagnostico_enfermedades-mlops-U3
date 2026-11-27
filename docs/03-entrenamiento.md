# 03. Entrenamiento

[Inicio](../README.md) · [Anterior: Características](02-caracteristicas.md) · [Siguiente: Validación](04-validacion.md)

## 1. Objetivo

Producir los **modelos candidatos** que generan cada parte de la salida definida en [00. Contexto y supuestos](00-contexto-y-supuestos.md), con cada experimento registrado de forma que cualquiera pueda repetirlo y obtener el mismo resultado.

La etapa existe porque comunes y huérfanas no se pueden resolver con un mismo modelo: con miles de casos conviene aprender de los datos; con menos de 10, el modelo tiene que apoyarse en el conocimiento médico publicado. Por eso hay **dos ramas de modelado** y un componente aparte para la gravedad.

## 2. Entradas y salidas

| Entrada | Formato | Origen |
|---|---|---|
| Matriz de características y conjunto de términos HPO por consulta | Parquet versionado con DVC | [02. Características](02-caracteristicas.md) |
| Anotaciones de fenotipos por enfermedad | Archivo de anotaciones de HPO (`phenotype.hpoa`) y datos de Orphanet, en versión fijada | Proyecto HPO y Orphanet |
| Reglas expertas de huérfanas | YAML versionado en Git | Especialistas de referencia |
| Disparo de reentrenamiento | Evento del flujo de trabajo | [09. Retroalimentación y reentrenamiento](09-retroalimentacion-y-reentrenamiento.md) |

| Salida | Formato | Quién la consume |
|---|---|---|
| Modelo de comunes calibrado | Artefacto de MLflow (formato nativo y exportado a ONNX) | [04. Validación](04-validacion.md) |
| Modelo de huérfanas | Artefacto de MLflow: parámetros de similitud, calibración, clasificadores de pocos ejemplos y reglas | Validación |
| Reglas de gravedad | Módulo versionado (punto de partida: las reglas de la Unidad 2) | Validación y servicio |
| *Run* de MLflow | Parámetros, métricas internas, versión de datos, versión del paquete de características, commit de código y archivo de dependencias | Validación y [05. Registro y aprobación](05-registro-y-aprobacion.md), para la trazabilidad |

## 3. Proceso

### 3.1 Rama de comunes

1. **Definición de clases.** Las 30 agrupaciones de diagnóstico CIE-10 más frecuentes en la red forman las clases; el resto se agrupa en "otra enfermedad" y las consultas sin enfermedad confirmada en "sin enfermedad". El listado lo aprueba el médico de referencia.
2. **Línea base.** Una **regresión logística** multiclase regularizada. Es interpretable y fija el mínimo que el modelo principal tiene que superar.
3. **Modelo principal.** **LightGBM** multiclase, con pesos por clase para el desbalance.
4. **Búsqueda de hiperparámetros** con **Optuna**: 50 ensayos por entrenamiento, con validación cruzada estratificada interna. La métrica a maximizar es el AUC-PR promedio por clase, coherente con la prioridad en sensibilidad.
5. **Calibración.** Las probabilidades se calibran con regresión isotónica sobre una partición reservada, para que un 0,62 signifique de verdad alrededor de 62 % (lo que ve el médico en la salida).
6. **Exportación a ONNX** con `onnxmltools`, para servir el modelo sin depender de LightGBM y con una imagen más liviana ([07. Despliegue](07-despliegue.md)).

### 3.2 Rama de huérfanas

Para cada una de las ~20 huérfanas priorizadas (G3) se calcula un puntaje combinando tres fuentes:

1. **Similitud de fenotipos** (funciona con cero casos propios). Se compara el conjunto de términos HPO del paciente con el conjunto de términos que HPO y Orphanet asocian a la enfermedad, usando la similitud semántica de **PyHPO**. Esto es transferencia de conocimiento: el modelo aprende de la descripción médica publicada de la enfermedad, no solo de los casos de la red.
2. **Clasificador de pocos ejemplos** (solo si la enfermedad tiene al menos 10 casos confirmados). Regresión logística con regularización fuerte sobre las características HPO expandidas, una por enfermedad contra el resto.
3. **Reglas expertas.** Combinaciones de síntomas de alarma definidas por los especialistas, que activan la alerta aunque los otros puntajes sean bajos. Son la red de seguridad para enfermedades sin casos.

El puntaje final por enfermedad es el **máximo** entre la similitud calibrada y el clasificador, y la alerta se activa si supera el umbral de la enfermedad o si se cumple una regla. La similitud se convierte en probabilidad con una calibración logística ajustada sobre los casos confirmados y una muestra de consultas sin esa enfermedad.

### 3.3 Componente de gravedad

El campo `estado` (5 estados) se deriva de síntomas y duración, no de la enfermedad candidata (G9). Como la historia clínica no tiene etiquetas de gravedad confiables, la primera versión usa **las reglas de la Unidad 2** (`simulado-0.2.0`), revisadas por el médico de referencia y versionadas como código. Cuando existan etiquetas de gravedad (por ejemplo, a partir de hospitalizaciones o remisiones registradas), se entrena un modelo ordinal y se compara contra estas reglas.

### 3.4 Registro de cada experimento

Cada ejecución crea un *run* en **MLflow Tracking** con: parámetros, métricas internas, gráficos, versión del conjunto de datos (etiqueta y hash de DVC), versión del paquete de características, commit de Git, archivo `uv.lock` y semilla. El entrenamiento corre en un contenedor con dependencias fijadas, así que repetir un *run* con los mismos insumos produce el mismo modelo.

## 4. Tecnologías

| Necesidad | Tecnología elegida | Alternativas consideradas | Por qué sí y por qué no | Costo y operación |
|---|---|---|---|---|
| Modelo principal de comunes | **LightGBM** | XGBoost; bosques aleatorios; redes neuronales | Los datos son tabulares y dispersos; los árboles potenciados son el estado del arte práctico para este tipo de datos. LightGBM entrena rápido en CPU y se exporta a ONNX. XGBoost es equivalente en calidad (queda como alternativa válida); los bosques aleatorios suelen rendir menos y pesan más; una red neuronal no aporta con este volumen tabular y exige GPU | Sin licencias; entrena en CPU |
| Línea base | **Regresión logística** (scikit-learn) | Las reglas de la Unidad 2 | La regresión logística es interpretable y sí predice enfermedades. Las reglas de la Unidad 2 solo dan gravedad, por eso se usan como línea base del componente de gravedad y no de enfermedades | Sin costo |
| Similitud de fenotipos para huérfanas | **PyHPO** | Exomiser; LIRICAL; código propio | Exomiser y LIRICAL priorizan enfermedades usando también datos genéticos, que la red no tiene. PyHPO hace la similitud semántica entre conjuntos HPO en Python, dentro del mismo pipeline | Sin licencias |
| Búsqueda de hiperparámetros | **Optuna** | Búsqueda en rejilla; búsqueda aleatoria | Optuna explora de forma adaptativa y corta los ensayos malos antes, con menos tiempo de cómputo, y se integra con MLflow | Sin licencias |
| Registro de experimentos | **MLflow Tracking** (servidor propio en la nube con base PostgreSQL y almacenamiento de objetos) | Weights & Biases; experimentos de DVC | MLflow es de código abierto, se aloja en la infraestructura propia (los metadatos de experimentos no salen a un tercero) y el mismo servidor ofrece el registro de modelos de [05. Registro y aprobación](05-registro-y-aprobacion.md). Weights & Biases es un servicio externo de pago; los experimentos de DVC no tienen registro de modelos | Una máquina pequeña y una base de datos administrada |
| Formato del modelo para servir | **ONNX** (exportado con `onnxmltools`) | Pickle o joblib de Python | ONNX no depende de la biblioteca de entrenamiento, es más rápido en CPU y no ejecuta código arbitrario al cargarse, a diferencia de pickle | Sin costo |
| Cómputo | Máquina virtual con CPU (8 a 16 núcleos) bajo demanda | GPU; servicio de entrenamiento administrado | Ningún modelo necesita GPU. Un servicio administrado agrega costo y dependencia del proveedor sin necesidad | Pago solo durante el entrenamiento |

**Cuándo evolucionar.** Si los síntomas pasan a ser texto libre o si se quieren aprovechar representaciones aprendidas de la literatura médica, se puede agregar un modelo de *embeddings* biomédicos (por ejemplo, con PyTorch y modelos preentrenados de Hugging Face) en la rama de huérfanas. Hoy no se justifica: la entrada es estructurada (G2) y el modo local no tiene GPU (G5).

## 5. Suposiciones

| # | Suposición | Implicación si deja de cumplirse |
|---|---|---|
| S03.1 | Las 30 agrupaciones de diagnóstico más frecuentes cubren la mayoría de las consultas con enfermedad común | Si la cobertura es baja, se aumentan las clases o se pasa a un esquema jerárquico (primero el capítulo de la CIE-10, luego el diagnóstico) |
| S03.2 | Las anotaciones de fenotipos de HPO y Orphanet describen bien las huérfanas priorizadas | Si una enfermedad está mal anotada, su similitud será pobre; los especialistas la refuerzan con reglas expertas |
| S03.3 | Los especialistas pueden escribir y revisar reglas de alarma para las huérfanas priorizadas | Sin reglas, las enfermedades sin casos dependen solo de la similitud y su sensibilidad baja |
| S03.4 | Un entrenamiento completo tarda menos de 2 horas en una máquina de 8 a 16 núcleos | Si tarda mucho más, se reduce el número de ensayos de Optuna o se paraleliza la búsqueda |
| S03.5 | La historia clínica no tiene etiquetas confiables de gravedad | Si las tiene, el componente de gravedad pasa de reglas a modelo desde la primera versión |
| S03.6 | La conversión de LightGBM a ONNX conserva las predicciones (diferencia máxima de 1e-5) | Si no, se sirve el formato nativo de LightGBM; la imagen pesa más pero sigue cabiendo en G5 |

## 6. Enfermedades comunes y huérfanas

| Aspecto | Comunes (muchos datos) | Huérfanas (pocos datos) |
|---|---|---|
| Enfoque | Aprendizaje supervisado: LightGBM calibrado | Similitud de fenotipos, clasificador de pocos ejemplos cuando hay 10 casos o más, y reglas expertas |
| De dónde aprende | De los casos confirmados de la red | Sobre todo de la descripción publicada de la enfermedad (HPO y Orphanet), y de los pocos casos propios |
| Desbalance | Pesos por clase | Umbral por enfermedad, ajustado para priorizar sensibilidad |
| Salida | Probabilidad calibrada por clase | Puntaje calibrado por enfermedad y señal de alerta |
| Paso de una rama a otra | | Cuando una huérfana acumula suficientes casos (por ejemplo, 200), se evalúa incorporarla como clase de la rama supervisada |

## 7. Criterios de salida y controles de calidad

Un *run* de entrenamiento termina bien y pasa a [04. Validación](04-validacion.md) solo si:

- Terminó sin errores y registró en MLflow todos los metadatos de la sección 3.4.
- El modelo de comunes supera a la regresión logística en AUC-PR promedio en la validación interna.
- La exportación a ONNX reproduce las predicciones del modelo nativo con una diferencia máxima de 1e-5.
- Cada huérfana priorizada tiene al menos una fuente de puntaje (similitud o regla).

Los umbrales clínicos de aprobación no se evalúan aquí: son la compuerta de calidad de la etapa siguiente.

## 8. Riesgos

| Riesgo | Mitigación |
|---|---|
| Sobreajuste en huérfanas | Regularización fuerte; clasificador solo con 10 casos o más; la similitud no se ajusta a los casos propios |
| Probabilidades mal calibradas que confunden al médico | Calibración isotónica y medición explícita de la calibración en la validación |
| Experimentos no reproducibles | Contenedor con dependencias fijadas, semillas y registro completo en MLflow |
| Optimizar una métrica que no importa clínicamente | La métrica de búsqueda es el AUC-PR, alineada con la sensibilidad; la decisión final la toman umbrales clínicos en la validación |

## 9. Cambios respecto a la propuesta original

| Antes (propuesta original) | Ahora | Motivo |
|---|---|---|
| Lista de familias de modelos posibles (regresión logística, árboles potenciados, bosques aleatorios, redes neuronales) | Línea base y modelo principal elegidos (regresión logística y LightGBM), con las razones para descartar las demás | La rúbrica pide tecnologías concretas y argumentadas |
| Huérfanas: transferencia, pocos ejemplos, datos sintéticos y reglas, sin decir cómo se combinan | Tres fuentes concretas (similitud de fenotipos, clasificador de pocos ejemplos, reglas) y una regla de combinación definida | Nada queda a interpretación |
| Datos sintéticos como técnica para huérfanas | Se retiran de la primera versión | Con pocos casos reales, los datos sintéticos repiten los mismos sesgos y son difíciles de validar; la similitud con la descripción publicada aporta conocimiento nuevo |
| No había registro de experimentos | MLflow Tracking con versión de datos, código y dependencias | Reproducibilidad y trazabilidad |
| La gravedad no aparecía en el modelado | Componente de gravedad explícito, que parte de las reglas de la Unidad 2 | Une la salida del modelo con el servicio que ya existe |
