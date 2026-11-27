# 05. Registro y aprobación

[Inicio](../README.md) · [Anterior: Validación](04-validacion.md) · [Siguiente: CI/CD](06-ci-cd.md)

## 1. Objetivo

Guardar cada modelo que pasó la validación con **todo lo necesario para saber de dónde salió**, y exigir la **aprobación de un médico y de un responsable técnico** antes de que llegue a producción. También permite volver a la versión anterior en minutos.

La etapa existe porque en salud no basta con que un modelo cumpla métricas: alguien con criterio clínico debe aceptar el riesgo de ponerlo en uso, y esa decisión tiene que quedar registrada con nombre y fecha.

## 2. Entradas y salidas

| Entrada | Formato | Origen |
|---|---|---|
| Modelo aprobado por la compuerta, con su reporte de validación | *Run* de MLflow | [04. Validación](04-validacion.md) |
| Decisión del comité | Aprobación o rechazo en un *environment* protegido de GitHub | Médico de referencia y responsable técnico |

| Salida | Formato | Quién la consume |
|---|---|---|
| Versión registrada del modelo | Versión numerada en **MLflow Model Registry**, con alias y etiquetas | [06. CI/CD](06-ci-cd.md), que construye la imagen con esa versión |
| *Model card* | Documento Markdown generado al registrar y guardado con la versión | Médicos, comité y auditoría ([10. Seguridad, gobierno y ética](10-seguridad-gobierno-y-etica.md)) |
| Registro de la aprobación | Historial del *environment* de GitHub: quién aprobó, cuándo y qué versión | Auditoría |

## 3. Proceso

1. **Registro.** Si la compuerta aprueba, el pipeline registra el modelo en el registro de MLflow con el nombre `diagnostico-candidatos` y una versión nueva (1, 2, 3…). La versión queda unida a:
   - el *run* que la produjo (parámetros, métricas y artefactos),
   - la versión del conjunto de datos (etiqueta y hash de DVC),
   - la versión del paquete de características y el commit de Git,
   - el reporte de validación y los umbrales de decisión,
   - el modelo exportado a ONNX.
2. **Alias `challenger`.** La versión nueva recibe el alias `challenger` y la etiqueta `validacion=aprobada`. El alias `champion` sigue apuntando al modelo en producción.
3. ***Model card*.** Se genera automáticamente con el propósito del modelo, los datos usados, las métricas por subgrupo, las limitaciones conocidas y las huérfanas cubiertas.
4. **Aprobación humana.** Un flujo de GitHub Actions llamado `promover-modelo` (que se lanza a mano con la versión a promover) usa un *environment* de GitHub llamado `produccion` con **revisores obligatorios**: el médico de referencia y el responsable técnico. El flujo se detiene hasta que ambos revisan el reporte y la *model card* y aprueban desde GitHub. Un integrante no puede aprobar su propio despliegue.
5. **Promoción.** Con la aprobación, el flujo mueve el alias `champion` a la versión nueva y deja en la anterior el alias `anterior`. Ese cambio dispara [06. CI/CD](06-ci-cd.md), que construye y publica la imagen del servicio con el modelo `champion`.
6. **Vuelta atrás.** Si el monitoreo detecta un problema ([08. Monitoreo](08-monitoreo.md)), el mismo flujo, con la misma aprobación, mueve `champion` a la versión con alias `anterior`. No hace falta reentrenar ni reconstruir nada a mano.
7. **Archivo.** Las versiones que nunca se promovieron o que quedaron dos promociones atrás se etiquetan `archivada`; no se borran, para poder auditarlas.

## 4. Tecnologías

| Necesidad | Tecnología elegida | Alternativas consideradas | Por qué sí y por qué no | Costo y operación |
|---|---|---|---|---|
| Registro de modelos | **MLflow Model Registry** con **alias** (`champion`, `challenger`, `anterior`) y etiquetas | Etapas de MLflow (*Staging*, *Production*); registro de un proveedor de nube; carpetas versionadas | MLflow ya está en el pipeline para el seguimiento de experimentos ([03. Entrenamiento](03-entrenamiento.md)), así que el registro no agrega otro servicio. Se usan alias porque las etapas fijas están marcadas como obsoletas desde MLflow 2.9 ([anuncio de MLflow](https://github.com/mlflow/mlflow/issues/10336)). Un registro de un proveedor ata el pipeline a esa nube | Mismo servidor de MLflow |
| Aprobación humana | ***Environment* de GitHub con revisores obligatorios** | Aprobación por correo; botón de aprobación dentro de MLflow; herramienta de tickets | El equipo ya trabaja en GitHub con PR y ramas protegidas (Unidad 2). El *environment* deja registro de quién aprobó y bloquea el flujo hasta la aprobación. Un correo no deja un registro verificable; MLflow de código abierto no tiene flujo de aprobación con permisos | Sin costo en repositorios públicos (ver S05.2) |
| *Model card* | Plantilla Markdown llenada por un script con los datos del *run* | Herramientas dedicadas de documentación de modelos | Un archivo Markdown se versiona, se revisa en GitHub y lo lee cualquiera sin instalar nada | Sin costo |

## 5. Suposiciones

| # | Suposición | Implicación si deja de cumplirse |
|---|---|---|
| S05.1 | El médico de referencia y el responsable técnico tienen cuenta de GitHub y responden en un plazo razonable (por ejemplo, 3 días hábiles) | Sin ellos, ningún modelo llega a producción. Se designa un suplente para cada rol |
| S05.2 | El repositorio del sistema permite revisores obligatorios en *environments*. Según la [documentación de GitHub](https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments), en los planes Free, Pro y Team esa función solo está disponible para repositorios públicos | Si el repositorio de la red es privado, se necesita GitHub Enterprise, o la aprobación se hace con un PR que cambia un archivo `modelo-en-produccion.yaml`, protegido con revisión obligatoria de propietarios de código (CODEOWNERS) |
| S05.3 | Una versión anterior del modelo sigue siendo segura como respaldo durante el tiempo que tarda en corregirse el problema | Si la versión anterior también está afectada (por ejemplo, por un cambio en los datos de entrada), se activa el respaldo por reglas descrito en [00. Contexto y supuestos](00-contexto-y-supuestos.md) |
| S05.4 | El servidor de MLflow tiene copias de seguridad diarias | Sin copias, una falla del servidor borraría la trazabilidad; las copias son parte de la infraestructura en [11. Orquestación e infraestructura](11-orquestacion-e-infraestructura.md) |

## 6. Enfermedades comunes y huérfanas

| Aspecto | Comunes (muchos datos) | Huérfanas (pocos datos) |
|---|---|---|
| Qué se registra | El modelo LightGBM calibrado y sus umbrales | Los parámetros de similitud, las calibraciones, los clasificadores de pocos ejemplos y la versión de las reglas expertas |
| Revisión del comité | Métricas por subgrupo y muestra de casos | Además, cada huérfana nueva o con cambio de umbral la revisa un especialista de esa enfermedad |
| *Model card* | Métricas por clase | Lista explícita de las huérfanas cubiertas y de su confianza estadística, para que el médico sepa que una huérfana fuera de la lista no se detecta |

Ambas ramas se registran como **una sola versión** del modelo, porque el servicio las usa juntas. Así nunca se mezcla un modelo de comunes de una versión con uno de huérfanas de otra.

## 7. Criterios de salida y controles de calidad

- La versión registrada tiene todos los vínculos de la sección 3 (*run*, datos, características, commit, reporte, ONNX y *model card*).
- La aprobación del *environment* `produccion` la dieron las dos personas requeridas.
- El alias `champion` apunta a la versión aprobada y `anterior` a la que estaba antes.
- Ningún modelo llega a [06. CI/CD](06-ci-cd.md) sin pasar por aquí: el flujo de construcción solo lee el alias `champion`.

## 8. Riesgos

| Riesgo | Mitigación |
|---|---|
| Promover un modelo sin revisión | El flujo de CD solo lee `champion`, y `champion` solo cambia con la aprobación del *environment* |
| Aprobación automática por rutina, sin leer el reporte | El reporte resume en la primera página las métricas por subgrupo y los cambios frente al modelo en producción; la *model card* exige firmar las limitaciones |
| Pérdida del historial de modelos | Las versiones no se borran, solo se archivan; copias de seguridad del servidor de MLflow |
| Demora en la aprobación que deja un modelo degradado en producción | Suplente por rol; la vuelta atrás no necesita reentrenar |

## 9. Cambios respecto a la propuesta original

| Antes (propuesta original) | Ahora | Motivo |
|---|---|---|
| Etapas *Ninguna → Staging → Producción* "por ejemplo, con MLflow" | MLflow Model Registry con alias `challenger`, `champion` y `anterior` | Las etapas fijas están obsoletas en MLflow desde la versión 2.9; los alias permiten la vuelta atrás inmediata |
| "La promoción a producción requiere aprobación humana" sin decir quién ni cómo | *Environment* `produccion` de GitHub con dos revisores obligatorios (médico y técnico), con registro de la aprobación | Hacer la aprobación verificable y auditable |
| "Siempre se puede volver a la versión anterior" sin mecanismo | Alias `anterior` y el mismo flujo de promoción para volver | La vuelta atrás tarda minutos y queda registrada |
| No había documentación del modelo para el médico | *Model card* generada en cada versión | Transparencia sobre limitaciones y huérfanas cubiertas |
