# 08. Monitoreo

[Inicio](../README.md) · [Anterior: Despliegue](07-despliegue.md) · [Siguiente: Retroalimentación y reentrenamiento](09-retroalimentacion-y-reentrenamiento.md)

## 1. Objetivo

Saber, todos los días, si el sistema **está funcionando**, si **los datos que recibe se parecen** a los que vio al entrenar y si **sigue acertando**, y convertir cada problema en una **acción definida de antemano**.

La etapa existe porque un modelo se degrada sin hacer ruido: el servicio puede responder rápido y sin errores mientras sus sugerencias empeoran porque cambió la población, una sede registra distinto o apareció una epidemia. Como el diagnóstico confirmado llega con días o semanas de retraso (G4), no se puede esperar a medir los aciertos: hay que vigilar también las señales tempranas.

## 2. Entradas y salidas

| Entrada | Formato | Origen |
|---|---|---|
| Métricas del servicio | Punto final `/metrics` en formato Prometheus | Servicio remoto ([07. Despliegue](07-despliegue.md)) |
| Registro de predicciones | Tabla en PostgreSQL (modo remoto) y lotes sincronizados desde la cola local (modo local) | Despliegue |
| Diagnósticos confirmados y correcciones | Etiquetas nuevas por `id_consulta` | [01. Datos](01-datos.md) (exportación semanal) y [09. Retroalimentación y reentrenamiento](09-retroalimentacion-y-reentrenamiento.md) |
| Datos de referencia | Muestra del conjunto de entrenamiento del modelo `champion` | Versión de DVC registrada con el modelo |
| Umbrales | `umbrales.yaml` y umbrales de monitoreo en el mismo repositorio | [04. Validación](04-validacion.md) |

| Salida | Formato | Quién la consume |
|---|---|---|
| Tableros | Grafana: servicio, datos y desempeño, por sede | Equipo de ML, responsable técnico y médico de referencia |
| Reportes de deriva | HTML de Evidently, uno diario y uno semanal | Equipo de ML |
| Alertas | Mensaje al canal del equipo y correo, con el umbral que se cruzó | Equipo de ML o comité, según el tipo |
| Señales para reentrenar o volver atrás | Evento al flujo de [09. Retroalimentación y reentrenamiento](09-retroalimentacion-y-reentrenamiento.md) o solicitud de vuelta atrás en [05. Registro y aprobación](05-registro-y-aprobacion.md) | Reentrenamiento y registro |

## 3. Proceso

El monitoreo tiene **tres capas**, de la más rápida a la más lenta:

### 3.1 Capa 1: salud del servicio (cada 15 segundos)

**Prometheus** lee `/metrics` del servicio remoto: solicitudes por minuto, latencia (percentiles 50 y 95), proporción de errores, disponibilidad y si el modelo cargó o el servicio está respondiendo solo con reglas. **Grafana** muestra el tablero y evalúa las reglas de alerta.

### 3.2 Capa 2: deriva de datos y de predicciones (diaria)

Un trabajo programado cada noche compara las predicciones del día y de los últimos 7 días con los datos de referencia usando **Evidently**:

- **Deriva de entradas:** distribución de edad, sexo, signos vitales y frecuencia de los síntomas HPO más comunes.
- **Deriva de predicciones:** distribución de los estados de gravedad, de las clases más sugeridas y de la tasa de alertas de huérfanas.
- **Calidad de entrada:** proporción de síntomas sin correspondencia en HPO y de signos vitales faltantes.

Cada métrica se calcula en total y por sede, porque un cambio en una sola sede (por ejemplo, un médico nuevo que registra distinto) se diluye en el promedio. Es la generalización del reporte `/estadisticas` de la Unidad 2, que ya cuenta predicciones por categoría.

### 3.3 Capa 3: desempeño con etiquetas (semanal)

Cada semana, cuando [01. Datos](01-datos.md) publica los diagnósticos confirmados, se unen con las predicciones por `id_consulta` y se recalculan las métricas de [04. Validación](04-validacion.md) sobre los casos reales: sensibilidad, exactitud de los 5 primeros candidatos, calibración, detección de huérfanas y diferencia entre subgrupos. También se mide la proporción de resultados que los médicos corrigieron.

### 3.4 Qué alerta dispara qué acción

| Señal | Umbral | Quién recibe la alerta | Acción |
|---|---|---|---|
| Servicio caído | `/health` sin respuesta por 2 minutos | Responsable técnico | Reinicio automático; si persiste, sedes urbanas a la instalación local de respaldo |
| Latencia alta | Percentil 95 > 500 ms durante 15 minutos | Equipo de ML | Revisar carga y recursos de la máquina |
| Errores | > 1 % de solicitudes con error durante 15 minutos | Equipo de ML | Si empezó tras un despliegue, vuelta a la imagen anterior |
| Modelo no cargado | El servicio responde con reglas | Responsable técnico | Atender de inmediato: el médico no recibe candidatos |
| Deriva de entradas | Evidently marca deriva en más del 30 % de las variables monitoreadas, durante 3 días seguidos | Equipo de ML | Investigar la causa; si es real y sostenida, evento de reentrenamiento a [09](09-retroalimentacion-y-reentrenamiento.md) |
| Calidad de entrada | Síntomas sin correspondencia > 5 % en una sede | Equipo y médico de referencia | Ampliar la tabla de correspondencias ([02. Características](02-caracteristicas.md)) |
| Alertas de huérfanas | Tasa del doble de la observada en validación durante una semana | Médico de referencia | Revisar si son falsas alarmas; ajustar umbral con el ciclo normal de validación |
| Desempeño | Sensibilidad o detección de huérfanas por debajo del umbral de [04](04-validacion.md) en la medición semanal | Comité de aprobación | Reentrenamiento prioritario; si la caída es mayor a 0,05, vuelta atrás al modelo `anterior` |
| Correcciones | Más del 30 % de los resultados corregidos por los médicos en una semana | Comité | Igual que la fila anterior |

## 4. Tecnologías

| Necesidad | Tecnología elegida | Alternativas consideradas | Por qué sí y por qué no | Costo y operación |
|---|---|---|---|---|
| Métricas del servicio | **Prometheus** (con la instrumentación de FastAPI) | Registros de texto revisados a mano; servicio de monitoreo del proveedor de nube | Prometheus es el estándar abierto para métricas de servicios, funciona con cualquier proveedor y el servicio solo tiene que exponer `/metrics` | Sin licencias; corre en la misma máquina |
| Tableros y alertas | **Grafana** | Kibana; tableros del proveedor de nube | Grafana lee Prometheus y PostgreSQL en el mismo tablero y gestiona las alertas. Kibana exige Elasticsearch, otro servicio pesado | Sin licencias |
| Deriva de datos y predicciones | **Evidently** | NannyML; cálculo propio de pruebas estadísticas | Evidently trae pruebas de deriva por tipo de variable y genera reportes HTML legibles para el médico de referencia. NannyML destaca en estimar el desempeño sin etiquetas, útil como evolución; el cálculo propio no da reportes uniformes | Sin licencias |
| Almacenar el registro de predicciones | **PostgreSQL** (el mismo de [07. Despliegue](07-despliegue.md)) | Archivos CSV como en la Unidad 2 | Permite unir predicciones con etiquetas por `id_consulta` con SQL y consultarlas desde Grafana | Incluido en la base del despliegue |
| Ejecutar los trabajos diarios y semanales | Flujos programados de **GitHub Actions** ([11. Orquestación e infraestructura](11-orquestacion-e-infraestructura.md)) | Cron en la máquina | Mismo orquestador que el resto del pipeline, con historial de ejecuciones | Sin costo adicional |

## 5. Suposiciones

| # | Suposición | Implicación si deja de cumplirse |
|---|---|---|
| S08.1 | Los diagnósticos confirmados se pueden unir con las predicciones por `id_consulta` (el identificador queda guardado en la historia clínica, dentro de la red) | Sin esa unión no hay capa 3; solo quedarían la deriva y las correcciones de los médicos como señal de desempeño |
| S08.2 | El retraso de las etiquetas es de días a semanas (G4) | Con retrasos de meses, la capa 3 llega tarde y la vuelta atrás depende más de la deriva y de las correcciones |
| S08.3 | Los datos del modo local llegan con retraso, pero llegan (S07.2) | Las sedes rurales se monitorean con días de atraso; sus tableros muestran la fecha de la última sincronización |
| S08.4 | Los umbrales de la tabla 3.4 son un punto de partida razonable | Se ajustan con la experiencia del primer trimestre, por PR, como los umbrales de validación |

## 6. Enfermedades comunes y huérfanas

| Aspecto | Comunes (muchos datos) | Huérfanas (pocos datos) |
|---|---|---|
| Señal principal | Deriva diaria y desempeño semanal con suficientes casos | La **tasa de alertas** (diaria) y la **revisión caso a caso** de cada huérfana confirmada |
| Desempeño | Métricas estadísticas por clase | Con tan pocos casos, cada caso confirmado se revisa individualmente: ¿el sistema lo alertó o no? Un caso no detectado se analiza con el especialista |
| Riesgo de fatiga | Bajo | Alto: si la alerta suena demasiado, los médicos la ignoran. Por eso se vigila su frecuencia |

## 7. Criterios de salida y controles de calidad

- Los tres tableros (servicio, datos y desempeño) existen y muestran datos de todas las sedes.
- Cada fila de la tabla 3.4 tiene una regla de alerta configurada y probada (se dispara a propósito una vez en staging).
- Los trabajos diario y semanal registran su ejecución y fallan de forma visible si no pueden correr.
- Toda alerta de desempeño queda registrada con la decisión que se tomó.

## 8. Riesgos

| Riesgo | Mitigación |
|---|---|
| Demasiadas alertas que nadie atiende | Umbrales con duración mínima (por ejemplo, 3 días de deriva) y destinatario definido para cada alerta |
| Deriva real que no se detecta porque se promedia entre sedes | Métricas por sede |
| Datos personales en los registros | El registro de predicciones solo tiene `id_consulta` y entradas codificadas (G7) |
| Monitoreo ciego en sedes rurales sin conexión | Fecha de última sincronización visible y alerta si una sede pasa más de 14 días sin sincronizar |

## 9. Cambios respecto a la propuesta original

| Antes (propuesta original) | Ahora | Motivo |
|---|---|---|
| Monitoreo de deriva, desempeño y salud del servicio en "tablas de métricas con umbrales" | Tres capas con frecuencia, herramienta y umbral definidos: Prometheus y Grafana, Evidently, y desempeño semanal con etiquetas | La rúbrica pide tecnologías y nada a interpretación |
| No decía qué hacer ante cada alerta | Tabla de señal, umbral, destinatario y acción | Una alerta sin acción no protege al paciente |
| No consideraba el retraso de las etiquetas | Deriva y correcciones como señales tempranas; desempeño cuando llegan las etiquetas | El diagnóstico confirmado tarda (G4) |
| Sin seguimiento por sede ni del modo local | Métricas por sede y control de la sincronización de las sedes rurales | Los dos modos de despliegue y las diferencias entre sedes |
