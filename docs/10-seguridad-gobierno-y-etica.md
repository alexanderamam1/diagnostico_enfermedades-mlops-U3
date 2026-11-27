# 10. Seguridad, gobierno y ética

[Inicio](../README.md) · [Anterior: Retroalimentación y reentrenamiento](09-retroalimentacion-y-reentrenamiento.md) · [Siguiente: Orquestación e infraestructura](11-orquestacion-e-infraestructura.md)

## 1. Objetivo

Reunir en un solo lugar los controles que protegen a los pacientes y a la red en **todas** las etapas: qué normas aplican, quién puede acceder a qué, cómo se guardan los secretos, qué queda registrado para auditar, quién decide y cómo se usa el sistema de forma ética.

La etapa es transversal porque ningún control vive en una sola etapa: la anonimización está en [01. Datos](01-datos.md), la aprobación en [05. Registro y aprobación](05-registro-y-aprobacion.md) y la autenticación en [07. Despliegue](07-despliegue.md), pero todos responden a las mismas reglas, que se definen aquí.

## 2. Entradas y salidas

| Entrada | Formato | Origen |
|---|---|---|
| Normas aplicables | Leyes y resoluciones (sección 3.1) | Marco legal colombiano |
| Política de tratamiento de datos de la red | Documento de la red | Responsable de protección de datos de la red |
| Reportes de validación y *model cards* | HTML y Markdown | [04. Validación](04-validacion.md) y [05. Registro y aprobación](05-registro-y-aprobacion.md) |

| Salida | Formato | Quién la consume |
|---|---|---|
| Matriz de accesos por rol | Tabla versionada en el repositorio (sección 3.3) | TI de la red y equipo de ML |
| Registros de auditoría | Registros de acceso, de aprobaciones y de linaje de modelos | Auditoría interna y externa |
| Hoja de datos del conjunto (*datasheet*) | Markdown generado en cada versión de datos | Comité y auditoría |
| Aviso al paciente y texto de la interfaz | Textos aprobados por el comité | Pacientes y médicos |

## 3. Proceso

### 3.1 Marco normativo

| Norma | Qué exige | Cómo lo cumple el pipeline |
|---|---|---|
| **Ley 1581 de 2012** (protección de datos personales) | Los datos de salud son sensibles: requieren autorización, finalidad definida, seguridad y confidencialidad. Su artículo 26 prohíbe transferir datos personales a países que no ofrezcan un nivel adecuado de protección ([Circular 005 de 2017 de la SIC](https://cancilleria.gov.co/normograma/compilacion/docs/circular_superindustria_0005_2017.htm)) | Los datos identificables no salen de la red (G7). A la nube solo llegan datos anonimizados y consultas con un identificador aleatorio, y además la nube elegida está en Colombia ([11. Orquestación e infraestructura](11-orquestacion-e-infraestructura.md)) |
| **Resolución 1995 de 1999** (historia clínica), modificada por la Resolución 839 de 2017 | La historia clínica es privada y reservada; su custodia y confidencialidad son responsabilidad del prestador | La historia clínica no se copia fuera de la red; solo se exporta lo necesario y anonimizado. El Ministerio de Salud publicó en 2026 un proyecto de resolución que la reemplazaría: hay que verificar la norma vigente al implementar |
| **Ley 2015 de 2020** (interoperabilidad de la historia clínica electrónica) | Intercambio de datos clínicos con estándares | La extracción usa FHIR ([01. Datos](01-datos.md)) |
| **Ley 1392 de 2010** (enfermedades huérfanas) | Define qué es una enfermedad huérfana | Define el alcance de la rama de huérfanas ([00. Contexto y supuestos](00-contexto-y-supuestos.md)) |

El sistema es de **apoyo a la decisión clínica** (G8): no diagnostica ni prescribe. Si en el futuro se quisiera usar para decidir sin intervención del médico, habría que evaluar con el INVIMA si el software pasa a considerarse un dispositivo médico, lo que está fuera del alcance de esta propuesta.

### 3.2 Clasificación de los datos por zona

| Zona | Qué datos hay | Quién accede | Protección |
|---|---|---|---|
| Servidor de la red | Historia clínica y datos identificables; clave HMAC | TI de la red y el *runner* de la etapa de datos | Red interna, bóveda de secretos, sin puertos abiertos hacia internet |
| Entorno de ML en la nube | Conjuntos anonimizados, modelos, experimentos | Equipo de ML | Cifrado en reposo, acceso con identidad individual y doble factor |
| Servicio remoto | Consultas con `id_consulta`, predicciones y correcciones | Médicos (su propia sesión) y equipo de ML (lectura para monitoreo) | HTTPS, inicio de sesión institucional, registros de acceso |
| Computadores de sedes rurales | Cola local de predicciones | Médico de la sede | Cifrado del disco (BitLocker o equivalente), usuario de Windows por médico |
| GitHub | Código, flujos, imágenes, aprobaciones | Equipo de ML, comité | Ramas protegidas, revisores obligatorios, secretos con alcance mínimo |

### 3.3 Accesos por rol

| Rol | Historia clínica | Datos anonimizados | Servicio (interfaz) | Registro de modelos | Aprobar modelos | Desplegar |
|---|---|---|---|---|---|---|
| Médico de consulta | Sí (fuera del sistema) | No | Sí | No | No | No |
| Médico de referencia | Sí (fuera del sistema) | Lectura de reportes | Sí | Lectura | **Sí** | No |
| Responsable técnico | No | Sí | Sí | Sí | **Sí** | **Sí** |
| Ingeniero de ML y de datos | No | Sí | Pruebas | Sí | No | No (lo hace el flujo) |
| TI de la red | Sí | No | No | No | No | Mantiene el servidor y los equipos rurales |

Nadie del equipo de ML accede a datos identificables. Los cambios de rol los aprueba el responsable de protección de datos de la red.

### 3.4 Controles técnicos

1. **Identidad y acceso:** inicio de sesión con la cuenta institucional por OpenID Connect y **doble factor** para todos ([07. Despliegue](07-despliegue.md)); cuentas individuales, nunca compartidas.
2. **Secretos:**
   - En la red: la clave HMAC y las credenciales de exportación viven en la **bóveda de secretos de la red** (HashiCorp Vault, o la que la red ya tenga).
   - En la nube: credenciales de bases y almacenamiento en el servicio de bóveda del proveedor.
   - En GitHub: secretos de repositorio y de *environment*, con los permisos mínimos por flujo (como en la Unidad 2, donde cada trabajo declara solo los permisos que usa).
3. **Cifrado:** HTTPS en todo el tráfico (Caddy, sincronización de sedes, API de MLflow); cifrado en reposo en discos, bases de datos y almacenamiento de objetos; disco cifrado en los computadores rurales.
4. **Vulnerabilidades:** Dependabot revisa las dependencias del repositorio y **Trivy** analiza cada imagen en el flujo de CI ([06. CI/CD](06-ci-cd.md)); una vulnerabilidad crítica bloquea el despliegue.
5. **Auditoría:** se conservan durante al menos un año los registros de acceso del servicio (Caddy y oauth2-proxy), el historial de aprobaciones de los *environments*, el linaje de cada modelo en MLflow y el historial de Git. Los registros de predicciones nunca incluyen datos personales.
6. **Incidentes:** ante una sospecha de fuga se aísla el componente, se rota la credencial afectada, se evalúa con el responsable de protección de datos de la red y se reporta a la autoridad según la Ley 1581 y los procedimientos de la Superintendencia de Industria y Comercio.

### 3.5 Gobierno

| Decisión | Quién decide | Dónde queda registrada |
|---|---|---|
| Qué enfermedades cubre el sistema (clases comunes, huérfanas priorizadas) | Médico de referencia con los especialistas | PR al catálogo en Git |
| Umbrales de validación y de monitoreo | Comité (médico de referencia y responsable técnico) | PR a `umbrales.yaml` |
| Promover o retirar un modelo | Comité | *Environment* `produccion` |
| Accesos a datos | Responsable de protección de datos de la red | Matriz de accesos |
| Revisión general del sistema | Comité con el responsable de protección de datos | Acta semestral |

### 3.6 Ética y transparencia

- **El médico decide.** La interfaz presenta probabilidades y explicaciones, nunca un diagnóstico cerrado, e indica la versión del modelo.
- **Transparencia sobre los límites.** La *model card* y la interfaz dicen qué enfermedades cubre el sistema; una huérfana fuera de la lista no se detecta, y eso se muestra.
- **Equidad.** Las diferencias de desempeño por sexo, edad y sede se miden antes de aprobar ([04. Validación](04-validacion.md)) y se vigilan en producción ([08. Monitoreo](08-monitoreo.md)).
- **Explicabilidad.** Cada resultado muestra los síntomas que más pesaron (SHAP), para que el médico pueda contrastarlos con su criterio.
- **Información al paciente.** La red verifica que su autorización de tratamiento de datos cubra el uso de datos anonimizados para mejorar el sistema y, si no, actualiza el formato. El paciente puede ser informado de que el médico usa una herramienta de apoyo.
- **Documentación.** Una hoja de datos por versión del conjunto (origen, periodo, población, exclusiones) y una *model card* por versión del modelo.

## 4. Tecnologías

| Necesidad | Tecnología elegida | Alternativas consideradas | Por qué sí y por qué no | Costo y operación |
|---|---|---|---|---|
| Autenticación de médicos | **OpenID Connect** con el proveedor de identidad de la red y **oauth2-proxy** | Usuarios y contraseñas propios | Altas, bajas y doble factor se gestionan donde la red ya los gestiona | Sin licencias adicionales |
| Bóveda de secretos en la red | **HashiCorp Vault** (o la bóveda existente de la red) | Archivos de configuración en el servidor | Guarda la clave HMAC con control de acceso y registro de uso; un archivo de texto en disco no tiene ninguno de los dos | Edición de código abierto sin licencia |
| Secretos de los flujos | **Secretos de GitHub** por repositorio y por *environment* | Variables en el código | Nunca aparecen en el código ni en los registros; los secretos del *environment* solo se entregan a flujos aprobados | Incluido en GitHub |
| Análisis de vulnerabilidades | **Dependabot** y **Trivy** | Revisión manual | Automático en cada cambio y en cada imagen | Sin licencias |
| Cifrado del disco en sedes rurales | **BitLocker** (Windows) o equivalente | Sin cifrado | Si se roba el equipo, la cola local no se puede leer | Incluido en ediciones profesionales de Windows |
| Documentación del modelo y de los datos | ***Model cards*** y ***datasheets*** en Markdown | Documentos sueltos | Se generan con cada versión y quedan junto al modelo y los datos | Sin costo |

## 5. Suposiciones

| # | Suposición | Implicación si deja de cumplirse |
|---|---|---|
| S10.1 | La red tiene un responsable de protección de datos que participa en el gobierno del sistema | Sin ese rol, las decisiones de acceso y de incidentes quedan sin dueño; se nombra uno antes de empezar |
| S10.2 | La autorización de tratamiento que firman los pacientes permite, o se puede actualizar para permitir, el uso de datos anonimizados para mejorar el sistema | Si no, solo se usan datos de pacientes con autorización actualizada, y el volumen inicial baja |
| S10.3 | Los datos que llegan a la nube, al estar anonimizados, no se consideran datos personales; aun así, la nube elegida está en Colombia | Si un análisis legal concluyera que siguen siendo datos personales, la elección de una región en Colombia ya cubre la restricción de transferencia internacional |
| S10.4 | Las normas citadas siguen vigentes al implementar el sistema | Si cambian (por ejemplo, la norma de historia clínica), se revisan los controles de la sección 3.1 antes del despliegue |

## 6. Enfermedades comunes y huérfanas

| Aspecto | Comunes (muchos datos) | Huérfanas (pocos datos) |
|---|---|---|
| Riesgo de reidentificación | Bajo | **Alto**: el control de *k*-anonimato de [01. Datos](01-datos.md) es obligatorio antes de publicar |
| Transparencia | Métricas por clase en la *model card* | Lista explícita de huérfanas cubiertas y su confianza estadística, visible también en la interfaz |
| Riesgo ético principal | Sesgo contra grupos poco representados | Falsa tranquilidad: que el médico asuma que una huérfana fue descartada porque el sistema no la alertó |

## 7. Criterios de salida y controles de calidad

- Matriz de accesos aprobada por el responsable de protección de datos antes del primer despliegue.
- Ningún secreto en el código (verificado por el escaneo de secretos de GitHub).
- Cero vulnerabilidades críticas en la imagen desplegada.
- *Model card* y *datasheet* presentes para cada versión en producción.
- Prueba de revocación de acceso: dar de baja a un médico en el proveedor de identidad le quita el acceso al servicio de inmediato.

## 8. Riesgos

| Riesgo | Mitigación |
|---|---|
| Fuga o reidentificación de datos de pacientes | Datos identificables solo en la red, anonimización con doble control, *k*-anonimato, cifrado y accesos mínimos |
| Robo de un equipo rural | Disco cifrado; la cola local solo tiene `id_consulta` y datos clínicos codificados |
| Credencial filtrada | Secretos en bóvedas, permisos mínimos, rotación ante incidente |
| Falsos negativos con consecuencias clínicas | Umbrales orientados a sensibilidad, alerta de huérfanas, el médico decide siempre |
| Sesgo contra grupos poco representados | Medición por subgrupo antes de aprobar y en producción |
| Confianza excesiva en el sistema | Explicaciones, cobertura visible, uso declarado como apoyo |

## 9. Cambios respecto a la propuesta original

| Antes (propuesta original) | Ahora | Motivo |
|---|---|---|
| Privacidad, sesgo y explicabilidad como restricciones y una tabla de riesgos con mitigaciones generales | Etapa propia con marco normativo, zonas de datos, matriz de accesos, controles técnicos, gobierno y ética | Los controles deben ser verificables y tener responsable |
| Solo se citaba la Ley 1581 de 2012 | Ley 1581 (incluida la restricción de transferencia internacional), Resolución 1995 de 1999, Ley 2015 de 2020 y Ley 1392 de 2010 | El sistema maneja historia clínica e interoperabilidad, no solo datos personales |
| Sin manejo de secretos, cifrado ni auditoría | Bóvedas, secretos de GitHub, HTTPS, cifrado en reposo, Trivy y registros de auditoría | El modo remoto y los equipos rurales amplían la superficie de ataque |
| Sin documentación del modelo ni de los datos | *Model cards* y *datasheets* por versión | Transparencia para médicos y auditoría |
