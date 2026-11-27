# 00. Contexto y supuestos

[Inicio](../README.md) · [Siguiente: Datos](01-datos.md)

Este documento fija una sola vez el problema, el escenario en el que se usa el sistema, lo que predice el modelo y los supuestos globales. Todas las etapas (`01` a `11`) parten de lo que se define aquí y no lo repiten.

## 1. Problema

El enunciado pide un modelo que, a partir de los síntomas de un paciente, estime **si es posible que sufra alguna enfermedad**, tanto para enfermedades **comunes** (muchos datos) como **huérfanas** (pocos datos), y el proceso end-to-end para desarrollarlo y operarlo.

Para esta propuesta, el problema se precisa así:

- **Es un apoyo a la decisión clínica, no un diagnóstico.** El sistema sugiere enfermedades candidatas y un nivel de gravedad; la decisión siempre es del médico.
- **Hay dos regímenes de datos que no se pueden tratar igual:**

| | Enfermedades comunes | Enfermedades huérfanas |
|---|---|---|
| Datos propios | Miles de casos por enfermedad | Muy pocos, a veces menos de 10 por enfermedad |
| Riesgo principal | Desbalance moderado y sesgos de población | Sobreajuste y validación poco confiable |
| Enfoque | Aprendizaje supervisado | Similitud de fenotipos, transferencia de conocimiento y reglas expertas |

- **Definición de enfermedad huérfana.** Se usa la definición colombiana: enfermedad crónicamente debilitante, grave, que amenaza la vida y con prevalencia menor de 1 por cada 5.000 personas ([Ley 1392 de 2010](https://www.funcionpublica.gov.co/eva/gestornormativo/norma.php?i=39965), modificada por la Ley 1438 de 2011). El [Ministerio de Salud](https://www.minsalud.gov.co/salud/publica/PENT/paginas/enfermedades-huerfanas.aspx) reporta alrededor de 1.920 enfermedades huérfanas identificadas en el listado de la Resolución 430 de 2013. La propuesta no las cubre todas desde el inicio: arranca con un subconjunto priorizado (ver G3) y lo amplía por fases.
- **El costo del error es asimétrico.** No detectar una enfermedad (falso negativo) es más grave que una alerta de más (falso positivo). Por eso las métricas y los umbrales priorizan la sensibilidad (*recall*), con una precisión mínima acordada con los médicos ([04. Validación](04-validacion.md)).

## 2. Escenario y usuarios

El escenario es ficticio, pero sus cifras son del orden de magnitud de una red real y sirven para dimensionar cada decisión técnica.

| Aspecto | Escenario |
|---|---|
| Organización | Red privada de **5 IPS de mediana complejidad** en el Valle del Cauca |
| Sedes | **2 urbanas** en Cali con conexión estable; **3 rurales** con conexión intermitente |
| Médicos que usan el sistema | Unos **120**, entre generales y especialistas |
| Volumen | Unas **2.000 consultas al día** con el sistema; picos de pocas solicitudes por segundo |
| Historia clínica | Electrónica, en formato **HL7 FHIR**, con unos **5 años** de consultas y diagnósticos confirmados |
| Equipo de ML | Un ingeniero de ML, un ingeniero de datos y un médico de referencia, con apoyo del área de TI de la red |

Consecuencias directas del escenario:

- **Un solo nodo basta para servir el modelo.** 2.000 consultas al día son, en promedio, una cada 40 segundos en horario laboral. No se necesita procesamiento distribuido (Spark) ni un clúster de Kubernetes ([11. Orquestación e infraestructura](11-orquestacion-e-infraestructura.md) explica cuándo sí).
- **Los dos modos de despliegue tienen una razón concreta.** Las sedes urbanas consultan un servicio remoto; las rurales corren la misma solución en el computador del consultorio y sincronizan cuando hay conexión ([07. Despliegue](07-despliegue.md)).

### Usuarios y roles

| Rol | Qué hace con el sistema |
|---|---|
| Médico de consulta | Registra síntomas, recibe las sugerencias y confirma o corrige el resultado |
| Especialista de referencia | Recibe las remisiones por sospecha de enfermedad huérfana y valida los casos raros |
| Comité de aprobación | Un médico y un responsable técnico aprueban cada versión del modelo antes de producción ([05. Registro y aprobación](05-registro-y-aprobacion.md)) |
| Equipo de ML | Opera el pipeline: datos, entrenamiento, validación, despliegue y monitoreo |
| TI y seguridad de la red | Custodia la historia clínica, administra el servidor propio y los accesos |

## 3. Salida del modelo

### Entrada

Amplía el contrato de la API que ya existe en el [servicio de la Unidad 2](https://github.com/troyano/diagnostico_enfermedades-mlops-U2) (temperatura y síntomas con intensidad y duración).

| Campo | Tipo | Cambio frente a la Unidad 2 |
|---|---|---|
| `id_consulta` | Seudónimo generado en la sede | Reemplaza `nombre`: la inferencia no recibe datos que identifiquen al paciente (G7) |
| `edad`, `sexo` | Entero y categoría | Nuevo |
| `signos_vitales` | Temperatura, frecuencia cardiaca, presión arterial, saturación | Hoy solo hay temperatura |
| `sintomas` | Lista de hasta 20, cada uno con código HPO, intensidad (0 a 10) y duración en días | Hoy el síntoma es texto libre; ahora se codifica en HPO |

### Salida

| Campo | Qué contiene |
|---|---|
| `posible_enfermedad` | Sí o no: sí cuando al menos un candidato supera el umbral de decisión definido en [04. Validación](04-validacion.md). Es la respuesta directa a la pregunta del enunciado |
| `candidatos` | Hasta 5 enfermedades, cada una con su código (CIE-10 para comunes, ORPHAcode para huérfanas), tipo y **probabilidad calibrada** |
| `alerta_huerfana` | Verdadero si una enfermedad huérfana supera su umbral. La interfaz muestra "sospecha de enfermedad huérfana, remitir a especialista" |
| `estado` | Nivel de gravedad, con los **5 estados** de la Unidad 2: NO ENFERMO, ENFERMEDAD LEVE, ENFERMEDAD AGUDA, ENFERMEDAD CRÓNICA y ENFERMEDAD TERMINAL (ver G9) |
| `explicacion` | Los síntomas que más pesaron en el resultado |
| `version_modelo` | Versión del modelo que respondió, para trazabilidad |

Ejemplo de respuesta:

```json
{
  "id_consulta": "C-7F3A",
  "posible_enfermedad": true,
  "candidatos": [
    {"codigo": "J18.9", "nombre": "Neumonía", "tipo": "comun", "probabilidad": 0.62},
    {"codigo": "ORPHA:586", "nombre": "Fibrosis quística", "tipo": "huerfana", "probabilidad": 0.21}
  ],
  "alerta_huerfana": true,
  "estado": "ENFERMEDAD AGUDA",
  "explicacion": ["Fiebre (HP:0001945)", "Tos (HP:0012735)", "Disnea (HP:0002094)"],
  "version_modelo": "2.0.0"
}
```

**Ruta de migración desde lo que ya existe.** En la Unidad 2, la función `predecir()` es un conjunto de reglas que devuelve los 5 estados. En esta propuesta, el modelo entrenado reemplaza esa función **detrás del mismo servicio**: el campo `estado` se mantiene, los campos nuevos se agregan y `nombre` se cambia por `id_consulta`. Las reglas actuales quedan como línea base de comparación y como respaldo si el modelo no está disponible.

## 4. Supuestos globales

Los supuestos de cada etapa se numeran `S01.1`, `S02.1`, y así sucesivamente, dentro de su documento. Los globales se numeran `G1` a `G12` y aplican a todas las etapas.

| # | Supuesto | Qué implica para el diseño | Si deja de cumplirse |
|---|---|---|---|
| G1 | La red tiene historia clínica electrónica en **FHIR**, con unos 5 años de consultas y diagnósticos confirmados (CIE-10 y ORPHAcode). Es coherente con la [Ley 2015 de 2020](https://www.funcionpublica.gov.co/eva/gestornormativo/norma.php?i=105472), que regula la interoperabilidad de la historia clínica electrónica | La ingesta usa un estándar y las etiquetas salen de diagnósticos ya confirmados | Hace falta una etapa previa de extracción y mapeo desde exportes propios, y la fase de datos se alarga |
| G2 | Las entradas son edad, sexo, signos vitales y síntomas **codificados en HPO** con intensidad y duración | Las características son estructuradas y comparables entre sedes | Si los síntomas llegan como texto libre, se agrega extracción de síntomas con procesamiento de lenguaje en [02. Características](02-caracteristicas.md), con más error |
| G3 | En huérfanas hay pocos casos propios (a veces menos de 10 por enfermedad), que se complementan con **Orphanet** y las anotaciones de fenotipos de **HPO**. Se empieza con unas **20 huérfanas priorizadas** por los especialistas | Se necesitan técnicas de pocos datos y validación caso a caso por especialistas | Si una huérfana llega a tener suficientes casos, pasa a la rama supervisada |
| G4 | El diagnóstico confirmado llega **días o semanas después** de la consulta | El desempeño real se mide con retraso; la deriva de los datos de entrada es la alerta temprana ([08. Monitoreo](08-monitoreo.md)) | Si la confirmación fuera inmediata, el desempeño se monitorea directamente |
| G5 | El computador del consultorio tiene **CPU de 4 núcleos, 8 GB de RAM, sin GPU** y puede ejecutar Docker | El modelo debe responder en menos de 1 segundo en CPU, con una imagen menor a 1 GB: favorece modelos de árboles exportados a ONNX y descarta modelos de lenguaje grandes en modo local | Con equipos más limitados, esas sedes usan solo el modo remoto o un instalador sin Docker |
| G6 | El equipo de ML es **pequeño** (3 personas) | Herramientas de código abierto o administradas, con poca operación; sin Kubernetes ni Spark al inicio | Si el equipo y el número de modelos crecen, se evalúa un orquestador dedicado y más infraestructura ([11. Orquestación e infraestructura](11-orquestacion-e-infraestructura.md)) |
| G7 | Los datos que identifican al paciente **no salen de la red**. Se anonimizan dentro de ella antes de entrenar, y la inferencia recibe solo datos clínicos con un seudónimo | Cumple la Ley 1581 de 2012 de protección de datos personales. La anonimización corre en el servidor propio ([01. Datos](01-datos.md)) | Si la red no tuviera servidor propio, la anonimización tendría que hacerse en un equipo dedicado antes de cualquier envío |
| G8 | El sistema es **apoyo a la decisión clínica**; la decisión final es del médico | Se muestran probabilidades y explicación, nunca un diagnóstico cerrado | Un uso autónomo exigiría otra validación regulatoria, fuera del alcance de esta propuesta |
| G9 | El **estado de gravedad** se deriva de los síntomas y de la duración del cuadro, **no** de la enfermedad candidata | Las dos salidas son independientes y el campo `estado` de la Unidad 2 sigue siendo válido | Ligar la gravedad a cada enfermedad exigiría etiquetas de gravedad por diagnóstico |
| G10 | El volumen es de unas **2.000 consultas al día** con picos de pocas solicitudes por segundo | Un contenedor por sede o un servicio remoto pequeño basta | Con 10 veces más volumen, el servicio remoto pasa a varias réplicas detrás de un balanceador ([07. Despliegue](07-despliegue.md)) |
| G11 | La red **acepta usar nube pública** para entrenar y servir, siempre que solo reciba datos anonimizados | Esquema híbrido: historia clínica y anonimización en el servidor propio; entrenamiento y servicio remoto en la nube | Si la red prohíbe la nube, todo corre en el servidor propio con las mismas herramientas en contenedores |
| G12 | Las sedes urbanas tienen conexión estable y las rurales, **intermitente** | Se necesitan los dos modos de despliegue y la sincronización diferida de registros y correcciones | Si todas las sedes tuvieran buena conexión, el modo local queda solo como contingencia |

## 5. Nivel de madurez MLOps objetivo

Se usa la escala de [Google Cloud](https://docs.cloud.google.com/architecture/mlops-continuous-delivery-and-automation-pipelines-in-machine-learning?hl=es), que define tres niveles según qué tanto se automatiza:

| Nivel | Qué significa | Dónde está este proyecto |
|---|---|---|
| 0, proceso manual | Entrenamiento y despliegue a mano | Punto de partida: en las Unidades 1 y 2 el "modelo" es una función de reglas sin entrenamiento, aunque el servicio ya tiene CI/CD |
| 1, automatización del pipeline de ML | El entrenamiento se ejecuta como pipeline automático (entrenamiento continuo) y el modelo validado se entrega automáticamente | **Objetivo de esta propuesta** |
| 2, CI/CD del pipeline | Además, los cambios al propio pipeline se prueban y despliegan automáticamente | Evolución, cuando el número de modelos y de personas lo justifique |

En salud, incluso en el nivel 2, **la promoción a producción mantiene la aprobación humana** del comité ([05. Registro y aprobación](05-registro-y-aprobacion.md)).

## 6. Etapas del pipeline

El diagrama general está en el [README](../README.md#diagrama-general). Las etapas son:

| # | Etapa | Qué resuelve | Dónde corre | Tecnología principal |
|---|---|---|---|---|
| 01 | [Datos](01-datos.md) | Ingesta, anonimización, validación y versionado | Servidor de la red | FHIR `$export`, Presidio, Pandera, DVC |
| 02 | [Características](02-caracteristicas.md) | Normalizar síntomas a HPO y codificar, igual en entrenamiento y en servicio | Nube | scikit-learn Pipeline |
| 03 | [Entrenamiento](03-entrenamiento.md) | Una rama para comunes y otra para huérfanas | Nube | LightGBM, PyHPO, Optuna, MLflow Tracking |
| 04 | [Validación](04-validacion.md) | Métricas, equidad, explicabilidad y revisión médica, con umbrales automáticos | Nube | scikit-learn, Fairlearn, SHAP |
| 05 | [Registro y aprobación](05-registro-y-aprobacion.md) | Versionar el modelo y aprobarlo antes de producción | Nube y GitHub | MLflow Model Registry, *environments* de GitHub |
| 06 | [CI/CD](06-ci-cd.md) | Probar y empaquetar el servicio con el modelo aprobado | GitHub | GitHub Actions, GHCR |
| 07 | [Despliegue](07-despliegue.md) | Modo remoto y modo local con la misma imagen | Nube y sedes | FastAPI, Streamlit, Docker, ONNX Runtime, Caddy |
| 08 | [Monitoreo](08-monitoreo.md) | Salud del servicio, deriva y desempeño | Nube | Prometheus, Grafana, Evidently |
| 09 | [Retroalimentación y reentrenamiento](09-retroalimentacion-y-reentrenamiento.md) | Convertir correcciones en etiquetas y reentrenar con reglas claras | Sedes y nube | GitHub Actions programado, MLflow |
| 10 | [Seguridad, gobierno y ética](10-seguridad-gobierno-y-etica.md) | Protección de datos, accesos, auditoría y documentación del modelo | Transversal | OIDC con doble factor, Vault, Trivy, *model cards* |
| 11 | [Orquestación e infraestructura](11-orquestacion-e-infraestructura.md) | Encadenar las etapas y definir dónde corre cada una | Transversal | GitHub Actions, Terraform, Docker Compose, Oracle Cloud (Bogotá) |

Las tecnologías de esta tabla son el punto de partida. Cada documento de etapa las justifica frente a sus alternativas y puede ajustarlas; si eso pasa, esta tabla y el diagrama se actualizan en el mismo cambio.

## 7. Cómo leer esta propuesta

1. **[README](../README.md):** visión general y diagrama general.
2. **Este documento:** escenario, salida del modelo y supuestos globales.
3. **Etapas `01` a `11`:** todas siguen la misma plantilla de 9 secciones: objetivo, entradas y salidas, proceso, tecnologías con alternativas, suposiciones con su implicación, diferencias entre comunes y huérfanas, criterios de salida, riesgos y cambios respecto a la propuesta original.
4. **[CHANGELOG](../CHANGELOG.md):** qué cambió frente a la propuesta original de la semana 1.

## 8. Cambios respecto a la propuesta original

| Antes (propuesta original) | Ahora | Motivo |
|---|---|---|
| No había escenario concreto: no se sabía quién usa el sistema, con qué volumen ni con qué infraestructura | Red de 5 IPS, 120 médicos, 2.000 consultas al día, sedes urbanas y rurales | Sin cifras no se puede justificar ninguna tecnología |
| La salida decía "si es posible que sufra alguna enfermedad", mientras el servicio devolvía 5 niveles de gravedad | Salida definida: candidatos con probabilidad, alerta de huérfana, nivel de gravedad y explicación | Una sola definición para el modelo, el servicio y la validación |
| No había supuestos explícitos | 12 supuestos globales, cada uno con su implicación y lo que pasa si no se cumple | Lo exige el enunciado; el problema es abierto |
| No se definía el nivel de madurez | Nivel 1 de MLOps como objetivo, con camino al nivel 2 | Marca qué se automatiza y qué no |
| El servicio recibía el nombre del paciente | Recibe un seudónimo | Protección de datos (G7) |
