# CHANGELOG

Cambios entre la **propuesta original** del pipeline, presentada en la semana 1 del curso ([`brief.md`, versión `v1.1.0`](https://github.com/troyano/mlops-enfermedades/blob/v1.1.0/docs/brief.md)), y esta **propuesta reestructurada** (versión `2.0.0`).

El formato sigue la idea de [Keep a Changelog](https://keepachangelog.com/es-ES/1.1.0/): los cambios se agrupan en **Añadido**, **Cambiado** y **Eliminado**. Después hay una tabla que muestra a dónde fue a parar cada sección de la propuesta original y qué le pasó. El detalle de cada cambio está en la sección 9 ("Cambios respecto a la propuesta original") de cada documento de [`docs/`](docs/00-contexto-y-supuestos.md).

## [2.0.0] - Propuesta reestructurada

### En resumen

1. **De una descripción general a una propuesta ejecutable.** La original decía *qué* hacer en cada fase; esta dice además *con qué*, *por qué esa herramienta y no otra*, *qué se supone* y *cuándo una etapa terminó bien*.
2. **Despliegue en dos modos.** La original solo contemplaba que el médico ejecutara el contenedor en su equipo; ahora hay un modo remoto y un modo local con la misma imagen y una regla para elegir.
3. **Supuestos explícitos.** La original no tenía ninguno; ahora hay 12 globales y entre 4 y 6 por etapa, cada uno con lo que pasa si deja de cumplirse.
4. **El ciclo de vida completo es verificable.** Umbrales numéricos de aprobación, aprobación humana registrada, monitoreo con una acción por alerta, disparadores de reentrenamiento y liberación gradual con vuelta atrás.
5. **Parte de lo que antes era teoría ya existe.** El servicio en Docker y el CI/CD con GitHub Actions están implementados en el [repositorio de la Unidad 2](https://github.com/troyano/diagnostico_enfermedades-mlops-U2), y la propuesta los toma como base.

### Añadido

- **Escenario concreto**: red de 5 IPS, 2 sedes urbanas y 3 rurales, unos 120 médicos y 2.000 consultas al día ([00](docs/00-contexto-y-supuestos.md)).
- **Contrato de entrada y salida del modelo**, con un ejemplo en JSON y la ruta de migración desde el servicio de la Unidad 2 ([00](docs/00-contexto-y-supuestos.md)).
- **12 supuestos globales** y supuestos por etapa, siempre con su implicación ([00](docs/00-contexto-y-supuestos.md) y sección 5 de cada etapa).
- **Nivel de madurez MLOps objetivo**: nivel 1, con camino al nivel 2 ([00](docs/00-contexto-y-supuestos.md)).
- **Una plantilla común de 9 secciones** para todas las etapas, con tabla de tecnologías y alternativas descartadas.
- **Modo de despliegue remoto** (Docker Compose en la nube con Caddy, oauth2-proxy y PostgreSQL) y **modo local** sin conexión con cola SQLite, sincronización y actualización automática ([07](docs/07-despliegue.md)).
- **Anonimización medible**: seudónimo HMAC con clave de la red, fechas relativas, Presidio para texto libre y control de *k*-anonimato ≥ 5 ([01](docs/01-datos.md)).
- **Contrato de datos** con Pandera y **versionado de datos** con DVC ([01](docs/01-datos.md)).
- **Paquete de características compartido** por entrenamiento y servicio, con prueba de paridad ([02](docs/02-caracteristicas.md)).
- **Rama de huérfanas concreta**: similitud de fenotipos con PyHPO, clasificador de pocos ejemplos y reglas expertas, con su regla de combinación ([03](docs/03-entrenamiento.md)).
- **Componente de gravedad** que conecta el modelo con los 5 estados del servicio actual ([03](docs/03-entrenamiento.md)).
- **Registro de experimentos** con MLflow Tracking y búsqueda de hiperparámetros con Optuna ([03](docs/03-entrenamiento.md)).
- **Compuerta de calidad** con umbrales numéricos versionados en `umbrales.yaml`, prueba temporal, validación por sede, *leave-one-out* en huérfanas, equidad con Fairlearn y explicabilidad con SHAP ([04](docs/04-validacion.md)).
- **Aprobación doble** (médico y técnico) con *environments* de GitHub y ***model cards*** ([05](docs/05-registro-y-aprobacion.md)).
- **Pruebas de ML en la integración continua**: paridad, contratos de datos, contrato de la API, casos de referencia y análisis de vulnerabilidades con Trivy ([06](docs/06-ci-cd.md)).
- **Monitoreo en tres capas** (servicio, deriva y desempeño) con Prometheus, Grafana y Evidently, y una tabla de señal, umbral, destinatario y acción ([08](docs/08-monitoreo.md)).
- **Mecanismo de retroalimentación** en la interfaz, **seis disparadores de reentrenamiento** (R1 a R6), **modo sombra** y **despliegue escalonado** por sedes ([09](docs/09-retroalimentacion-y-reentrenamiento.md)).
- **Etapa de seguridad, gobierno y ética**: marco normativo, zonas de datos, matriz de accesos, secretos, cifrado, auditoría, gobierno y ética ([10](docs/10-seguridad-gobierno-y-etica.md)).
- **Etapa de orquestación e infraestructura**: ocho flujos de GitHub Actions, Terraform, Oracle Cloud en la región Bogotá, copias de seguridad y criterio de cuándo usar Spark, Kubernetes, Airflow o un *feature store* ([11](docs/11-orquestacion-e-infraestructura.md)).
- **Validación automática de la documentación** en cada PR: enlaces y diagramas Mermaid (`.github/workflows/docs-check.yml`).

### Cambiado

- **Estructura**: de un documento único a un README con la visión general y 12 documentos por etapa.
- **Diagrama general**: ahora muestra las 11 etapas con sus tecnologías, dónde corre cada una (red, nube, entrega, operación), los dos modos de despliegue, la compuerta de calidad y los ciclos de retroalimentación, con una guía de lectura. Además está exportado como imagen ([README](README.md#diagrama-general)).
- **Salida del modelo**: de "si es posible que sufra alguna enfermedad" a una salida definida campo por campo, que incluye los 5 estados de la Unidad 2 ([00](docs/00-contexto-y-supuestos.md)).
- **Registro de modelos**: de las etapas *Ninguna → Staging → Producción* de MLflow, que están obsoletas, a alias `challenger`, `champion` y `anterior` ([05](docs/05-registro-y-aprobacion.md)).
- **Ambientes**: de ramas de Git (`dev`, `stage`, `main`) a lugares de ejecución (desarrollo, staging, producción); el código usa ramas de trabajo con PR a una rama protegida ([06](docs/06-ci-cd.md)).
- **Entrega continua**: de un solo disparador (promover un modelo) a dos (cambio de código y modelo aprobado) que usan el mismo flujo de construcción ([06](docs/06-ci-cd.md)).
- **ONNX**: de formato "opcional" a la forma de ejecutar el modelo en los dos modos ([03](docs/03-entrenamiento.md), [07](docs/07-despliegue.md)).
- **Validación externa**: de "datos de otra institución" a validación rotando las 5 sedes de la red ([04](docs/04-validacion.md)).
- **Reentrenamiento**: de "por calendario o alertas" a seis disparadores con condiciones verificables ([09](docs/09-retroalimentacion-y-reentrenamiento.md)).
- **Retroalimentación**: de "las correcciones se convierten en datos etiquetados" a la regla de que la etiqueta es el diagnóstico confirmado y la corrección es una señal temprana ([09](docs/09-retroalimentacion-y-reentrenamiento.md)).
- **Plan de ejecución**: de 4 fases a 6, con criterios medibles y una fase propia para el modo local ([README](README.md#plan-de-puesta-en-marcha)).
- **Marco normativo**: de la Ley 1581 de 2012 sola a la Ley 1581 (incluida la restricción de transferencia internacional), la Resolución 1995 de 1999, la Ley 2015 de 2020 y la Ley 1392 de 2010 ([10](docs/10-seguridad-gobierno-y-etica.md)).
- **Riesgos y ética**: de una tabla general a una etapa propia y una sección de riesgos en cada etapa ([10](docs/10-seguridad-gobierno-y-etica.md)).
- **Datos del servicio**: el nombre del paciente se reemplaza por un identificador aleatorio de consulta, y el archivo JSON de la Unidad 2 por PostgreSQL (modo remoto) y SQLite (modo local) ([07](docs/07-despliegue.md)).

### Eliminado

- **Datos sintéticos para huérfanas**: con pocos casos reales repiten sus sesgos y son difíciles de validar; la similitud con la descripción publicada de la enfermedad aporta conocimiento nuevo ([03](docs/03-entrenamiento.md)).
- **Etapas fijas del registro de MLflow**: reemplazadas por alias ([05](docs/05-registro-y-aprobacion.md)).
- **Ramas `dev` y `stage` como ambientes** ([06](docs/06-ci-cd.md)).
- ***Feature store* como componente de la primera versión**: se difiere, con el criterio de cuándo adoptarlo ([02](docs/02-caracteristicas.md)).
- **Imágenes, señales y texto libre como entradas del modelo**: la primera versión usa entradas estructuradas (G2); el texto libre solo se procesa para quitarle datos personales ([00](docs/00-contexto-y-supuestos.md), [01](docs/01-datos.md)).
- **Redes neuronales como candidato inicial**: quedan como evolución, porque no aportan con datos tabulares de este volumen y el modo local no tiene GPU ([03](docs/03-entrenamiento.md)).

### Se conserva de la propuesta original

- Un flujo común con **dos ramas de modelado**, una para comunes y otra para huérfanas.
- El sistema como **apoyo a la decisión clínica**: el médico decide.
- La **prioridad de la sensibilidad** (*recall*) por el costo asimétrico del error.
- Las métricas de validación: *recall*, AUC-PR, calibración, equidad y revisión médica.
- El **registro de modelos con aprobación humana** y la vuelta atrás.
- La **retroalimentación del médico** como fuente de datos.
- **Empezar por las enfermedades comunes** para poner en marcha la infraestructura con el caso de más datos.

### Antes y después por sección de la propuesta original

| Sección de la `v1.1.0` | Dónde está ahora | Qué le pasó |
|---|---|---|
| 1. Resumen ejecutivo | [README](README.md#la-propuesta-en-una-página) y [00](docs/00-contexto-y-supuestos.md) | Se mantiene la tabla de comunes y huérfanas; se agregan escenario, salida del modelo y supuestos |
| 2.1 Restricciones y limitaciones | [00](docs/00-contexto-y-supuestos.md) y [10](docs/10-seguridad-gobierno-y-etica.md) | Cada restricción tiene ahora un control concreto (por ejemplo, la privacidad se resuelve con anonimización medible y zonas de datos) |
| 2.2 Tipos de datos | [00](docs/00-contexto-y-supuestos.md) y [01](docs/01-datos.md) | De una lista de tipos posibles a un contrato de entrada concreto; imágenes y señales quedan fuera de la primera versión |
| 3.1 Fuentes y manejo de datos | [01](docs/01-datos.md) y [02](docs/02-caracteristicas.md) | Exportación FHIR, anonimización, Pandera, DVC y normalización a HPO |
| 3.2 Modelos candidatos | [03](docs/03-entrenamiento.md) | De familias posibles a modelos elegidos con su justificación; rama de huérfanas con mecanismo concreto |
| 3.3 Validación y pruebas | [04](docs/04-validacion.md) y [06](docs/06-ci-cd.md) | Umbrales numéricos y compuerta automática; las pruebas de software pasan a CI/CD |
| 4. Producción | [05](docs/05-registro-y-aprobacion.md), [07](docs/07-despliegue.md), [08](docs/08-monitoreo.md) y [09](docs/09-retroalimentacion-y-reentrenamiento.md) | Se divide en cuatro etapas: registro y aprobación, despliegue en dos modos, monitoreo y reentrenamiento |
| 5. Ambientes y CI/CD | [06](docs/06-ci-cd.md) y [11](docs/11-orquestacion-e-infraestructura.md) | Ambientes separados de las ramas; CI/CD implementado en la Unidad 2 y extendido con pruebas de ML |
| 6. Diagrama general | [README](README.md#diagrama-general) | Rehecho con etapas, tecnologías, zonas, modos de despliegue y ciclos |
| 7. Plan de ejecución | [README](README.md#plan-de-puesta-en-marcha) | De 4 a 6 fases, con el modo local como fase propia |
| 8. Riesgos y ética | [10](docs/10-seguridad-gobierno-y-etica.md) y sección 8 de cada etapa | De una tabla general a controles por etapa |
| 9. Referencias | [README](README.md#referencias) | Ampliadas con normas verificadas y documentación de las herramientas |
| No existía | [11](docs/11-orquestacion-e-infraestructura.md) | Orquestación, infraestructura, proveedor de nube y copias de seguridad |

### De idea a implementación

| En la propuesta original | Estado actual |
|---|---|
| "El modelo se empaqueta en un contenedor Docker expuesto como servicio" | Implementado en las Unidades 1 y 2 (FastAPI, Streamlit y Docker), con una función de reglas en lugar del modelo |
| "CI: cada cambio ejecuta pruebas unitarias y de integración" | Implementado en la Unidad 2: lint, tipos y 75 pruebas en cada PR con GitHub Actions |
| "CD: se construye una nueva imagen y se publica" | Implementado en la Unidad 2: publicación en GitHub Packages en cada merge a `main` |
| "Tablas de métricas" para el monitoreo | Primer paso implementado en la Unidad 2: reporte de estadísticas por categoría, que la [etapa 08](docs/08-monitoreo.md) generaliza |

## [1.1.0] - Propuesta original entregada (semana 1)

[`docs/brief.md` en la etiqueta `v1.1.0`](https://github.com/troyano/mlops-enfermedades/blob/v1.1.0/docs/brief.md). Nueve secciones: resumen ejecutivo, diseño, desarrollo, producción, ambientes y CI/CD, diagrama general, plan de ejecución, riesgos y ética, y referencias. Es la versión base de este CHANGELOG.

## [1.0.0] - Primera versión de la propuesta original

[`docs/brief.md` en la etiqueta `v1.0.0`](https://github.com/troyano/mlops-enfermedades/blob/v1.0.0/docs/brief.md). Siete secciones. La versión `1.1.0` le agregó la sección de ambientes y CI/CD, el plan de ejecución por fases, MLflow, ONNX, el *feature store* y las tablas de métricas, y rehízo el diagrama.
