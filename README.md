# diagnostico_enfermedades-mlops-U3

Propuesta reestructurada de un pipeline de MLOps end-to-end para predecir, a partir de los síntomas de un paciente, si es posible que sufra una enfermedad, tanto **común** (muchos datos) como **huérfana** (pocos datos).

> **Estado: en construcción.** Este README se completa al cierre del trabajo con el resumen de la propuesta, el diagrama general, la tabla de etapas y tecnologías, y la guía de lectura.

## Diagrama general

```mermaid
flowchart TB
    subgraph RED["Red de IPS · servidor propio (datos identificables)"]
        direction LR
        HCE[("Historia clínica<br/>electrónica FHIR")]
        D01["<b>01. Datos</b><br/>ingesta, anonimización,<br/>validación y versionado<br/><i>Presidio · Pandera · DVC</i>"]
        HCE -- "consultas con<br/>diagnóstico confirmado" --> D01
    end

    subgraph ML["Entorno de ML en la nube (Oracle Cloud, Bogotá) · solo datos anonimizados"]
        direction LR
        D02["<b>02. Características</b><br/>síntomas a HPO,<br/>codificación<br/><i>scikit-learn Pipeline</i>"]
        D03["<b>03. Entrenamiento</b><br/>comunes: LightGBM calibrado<br/>huérfanas: similitud HPO,<br/>pocos ejemplos, reglas expertas<br/><i>PyHPO · Optuna · MLflow Tracking</i>"]
        D04["<b>04. Validación</b><br/>recall, AUC-PR, calibración,<br/>equidad, explicabilidad,<br/>revisión médica<br/><i>Fairlearn · SHAP</i>"]
        Q{"¿Cumple<br/>los umbrales?"}
        D05["<b>05. Registro y aprobación</b><br/>aprobación clínica y técnica<br/><i>MLflow Model Registry ·<br/>environments de GitHub</i>"]
        D02 -- "matriz de<br/>características" --> D03
        D03 -- "modelos<br/>candidatos" --> D04
        D04 --> Q
        Q -- "no: se ajusta" --> D03
        Q -- "sí" --> D05
    end

    subgraph ENT["Entrega y despliegue"]
        direction LR
        D06["<b>06. CI/CD</b><br/>pruebas e imagen con<br/>el modelo en ONNX<br/><i>GitHub Actions · GHCR</i>"]
        D07R["<b>07. Despliegue · modo remoto</b><br/>sedes urbanas, por internet<br/><i>FastAPI · Streamlit · ONNX Runtime</i><br/>HTTPS con Caddy, oauth2-proxy"]
        D07L["<b>07. Despliegue · modo local</b><br/>PC del consultorio, sin conexión<br/><i>misma imagen · Docker Compose</i>"]
        D06 -- "imagen versionada" --> D07R
        D06 -- "imagen versionada" --> D07L
    end

    subgraph OP["Operación"]
        direction LR
        MED(["Médico"])
        D08["<b>08. Monitoreo</b><br/>servicio, deriva y desempeño<br/><i>Prometheus · Grafana · Evidently</i>"]
        D09["<b>09. Retroalimentación<br/>y reentrenamiento</b><br/>confirmación del médico,<br/>disparadores R1 a R6, modo sombra<br/><i>GitHub Actions programado</i>"]
        MED -- "confirma o corrige<br/>el resultado" --> D09
        D08 -- "alerta de deriva<br/>o bajo desempeño" --> D09
    end

    subgraph TR["Transversales a todas las etapas"]
        direction LR
        D10["<b>10. Seguridad, gobierno y ética</b><br/><i>Ley 1581 · OIDC · Vault · Trivy · model cards</i>"]
        D11["<b>11. Orquestación e infraestructura</b><br/><i>GitHub Actions · Terraform · Oracle Cloud Bogotá</i>"]
    end

    D01 -- "dataset anonimizado<br/>y versionado" --> D02
    D05 -- "modelo aprobado" --> D06
    D07R -- "sugerencias y gravedad" --> MED
    D07L -- "sugerencias y gravedad" --> MED
    D07R -- "registro de predicciones<br/>sin datos personales" --> D08
    D07L -. "sincroniza al<br/>tener conexión" .-> D08
    D09 -. "nuevas etiquetas" .-> D01
    D09 -. "disparo de<br/>reentrenamiento" .-> D03
    OP ~~~ TR

    classDef etapa fill:#eef4ff,stroke:#3b5b9a,color:#111
    classDef decision fill:#fff4d6,stroke:#b07d00,color:#111
    classDef actor fill:#e8f6ec,stroke:#2e7d4f,color:#111
    classDef dato fill:#f4f4f4,stroke:#777,color:#111
    class D01,D02,D03,D04,D05,D06,D07R,D07L,D08,D09,D10,D11 etapa
    class Q decision
    class MED actor
    class HCE dato
```

**Cómo leer el diagrama.** Cada caja azul es una etapa del pipeline, con su número, lo que hace y, en cursiva, sus tecnologías principales; el número coincide con el de su documento en `docs/`. Los recuadros amarillos agrupan las etapas por dónde corren: el servidor propio de la red de IPS (único lugar con datos que identifican al paciente), el entorno de ML en la nube (solo datos anonimizados), la entrega y despliegue, y la operación. Las flechas continuas son el flujo principal y llevan el nombre de lo que viaja; las punteadas son los ciclos de retroalimentación y la sincronización diferida del modo local. El rombo es la compuerta de calidad: un modelo que no cumple los umbrales vuelve a entrenamiento. Las etapas 10 y 11 son transversales y aplican a todas las demás.

El escenario, la salida del modelo y los supuestos que sostienen este diagrama están en [docs/00-contexto-y-supuestos.md](docs/00-contexto-y-supuestos.md).

## Contenido del repositorio

| Ruta | Contenido |
|---|---|
| `README.md` | Visión general de la propuesta y diagrama general |
| `docs/` | Un documento por etapa del pipeline, todos con la misma plantilla |
| `docs/img/` | Diagramas en formato fuente (Mermaid) y exportados |
| `CHANGELOG.md` | Cambios frente a la propuesta original de la semana 1 |

## Antecedentes

- Propuesta original del pipeline (Unidad 1): https://github.com/troyano/mlops-enfermedades/blob/v1.1.0/docs/brief.md
- Servicio en Docker con CI/CD en GitHub Actions (Unidad 2): https://github.com/troyano/diagnostico_enfermedades-mlops-U2

## Contexto académico

Maestría en Inteligencia Artificial Aplicada, Universidad Icesi. Materia: Machine Learning Operations (MLOps), periodo 2026-II.

## Autores

- Eduar Troyano Velásquez
- Alexander Amador
