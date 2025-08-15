
# Diagramas de Arquitectura (Estilo Arquitecto de Soluciones)

> Nota: Los siguientes diagramas están en **Mermaid**. Puedes pegarlos en cualquier visor/editor compatible (como Draw.io/Excalidraw mermaid plugin, GitHub, Obsidian o VS Code con extensión Mermaid) para visualizarlos.

---

## 1) Serverless (AWS) — Alto nivel (Componentes)

```mermaid
flowchart LR
  subgraph Client["Cliente / Navegador"]
    UI["Frontend Next.js (S3+CloudFront)"]
  end

  subgraph AWS["AWS"]
    APIGW["API Gateway (REST)"]
    L1["Lambda API (FastAPI/NestJS)"]
    SF["Step Functions (Orquestación)"]
    DL["EventBridge (Cron/Eventos)"]
    Q["SQS (Colas/DLQ)"]
    DB[(Aurora Serverless PostgreSQL)]
    S3[(S3 - XML/PDF/Exports)]
    ATH["Athena (SQL sobre S3)"]
    QS["QuickSight (Dashboards)"]
    SEC["Secrets Manager + KMS"]
    WAF["AWS WAF"]
    XRY["X-Ray + CloudWatch"]
  end

  UI -->|HTTPS| WAF --> APIGW --> L1
  L1 -->|invoke| SF
  DL --> SF
  SF -->|descarga| L1
  L1 -->|SRI| SRI[(Portal SRI via Playwright)]
  L1 --> S3
  L1 --> DB
  SF --> Q
  L1 --> Q
  ATH --> QS
  SEC --> L1
  XRY --- L1
```

### Flujo principal (Descarga → Parseo → Persistencia → Reporte)
```mermaid
sequenceDiagram
  autonumber
  participant UI as UI (Next.js)
  participant APIG as API Gateway
  participant L as Lambda API
  participant SF as Step Functions
  participant F as Lambda Fetch/Parse
  participant S3 as S3
  participant DB as Aurora PG
  participant ATH as Athena/QS

  UI->>APIG: POST /descargas {rango, tenant}
  APIG->>L: Invoke
  L->>SF: StartExecution
  loop Orquestación
    SF->>F: Tarea: Descargar desde SRI
    F->>S3: Guardar XML/PDF
    F->>DB: Upsert metadatos/valores
  end
  UI->>APIG: GET /reportes
  APIG->>L: Invoke
  alt Analítica
    L->>ATH: Consultas en Parquet/Views
  else OLTP
    L->>DB: SELECTs por filtros
  end
  L-->>UI: JSON/Excel/PDF/ZIP (pre-signed)
```

---

## 2) Microservicios en Contenedores (ECS Fargate) — Alto nivel

```mermaid
flowchart TB
  subgraph Client["Cliente / Navegador"]
    UI["Next.js (S3+CloudFront)"]
  end

  subgraph Edge["Perímetro"]
    APIG["API Gateway"]
  end

  subgraph ECS["ECS Fargate Cluster"]
    BFF["bff-api (NestJS/FastAPI)"]
    FETCH["fetcher-sri (Playwright)"]
    PARSE["parser-xml"]
    RPT["reports-svc"]
    VOID["void-checker"]
    ZIP["zipper-svc"]
  end

  SQS["SQS (work queues)"]
  SNS["SNS (notificaciones)"]
  DB[(RDS PostgreSQL)]
  S3[(S3 Archivos)]
  REDIS[ElastiCache (Redis)]
  OBS["CloudWatch / X-Ray"]
  SEC["Secrets Manager / KMS"]

  %% flujo cliente -> API -> BFF
  UI --> APIG --> BFF

  %% BFF acceso a datos y publicación de jobs
  BFF --> DB
  BFF --> S3
  BFF --> SQS

  %% fetcher descarga y publica trabajos
  FETCH --> S3
  FETCH --> DB
  FETCH --> SQS

  %% consumidores de colas
  SQS --> PARSE
  SQS --> RPT
  SQS --> VOID
  SQS --> ZIP

  %% servicios a datos/archivos
  PARSE --> DB
  PARSE --> S3
  RPT --> DB
  RPT --> S3
  VOID --> DB
  ZIP --> S3

  %% notificaciones
  ZIP --> SNS

  %% cache
  REDIS --- BFF

  %% observabilidad (líneas separadas por servicio)
  OBS --- BFF
  OBS --- FETCH
  OBS --- PARSE
  OBS --- RPT
  OBS --- VOID
  OBS --- ZIP

  %% secretos/cifrado (líneas separadas por servicio)
  SEC --> BFF
  SEC --> FETCH
  SEC --> PARSE
  SEC --> RPT
  SEC --> VOID
  SEC --> ZIP
```

---

## 3) Kubernetes Gestionado (EKS) + GitOps — Alto nivel

```mermaid
flowchart LR
  subgraph Client["Cliente / Navegador"]
    UI["Next.js (CloudFront)"]
  end

  subgraph Edge["Perímetro"]
    Ingress["ALB/NGINX Ingress"]
  end

  subgraph EKS["Amazon EKS"]
    BFF["BFF (FastAPI/NestJS)"]
    SVC1["fetcher-sri (Deployment)"]
    SVC2["parser-xml (Deployment)"]
    SVC3["reports-svc (Deployment)"]
    SVC4["void-checker (Cron/Deployment)"]
    SVC5["zipper-svc (Deployment)"]
    MQ["Kafka/RabbitMQ (MSK/AmazonMQ)"]
    RED["Redis (ElastiCache)"]
    MESH["Istio/Linkerd (mTLS, CB, Retry)"]
  end

  DB[(RDS PostgreSQL)]
  S3[(S3/MinIO Archivos)]
  OBS["Prometheus/Grafana/Loki/Tempo + OpenTelemetry"]
  SEC["IRSA, NetworkPolicies, OPA/Kyverno, cert-manager"]
  CD["Argo CD (GitOps) + Argo Workflows/Cron"]

  UI --> Ingress --> BFF
  BFF -->|REST| SVC3
  BFF --> DB
  BFF --> S3
  SVC1 --> S3
  SVC1 --> DB
  SVC1 --> MQ
  SVC2 --> DB
  SVC2 --> S3
  SVC2 --> MQ
  SVC3 --> DB
  SVC3 --> S3
  SVC4 --> DB
  SVC5 --> S3
  MQ --- SVC1 & SVC2 & SVC3 & SVC4 & SVC5
  RED --- BFF
  MESH --- BFF & SVC1 & SVC2 & SVC3 & SVC4 & SVC5
  OBS --- EKS
  SEC --- EKS
  CD --- EKS
```

---

## 4) Lakehouse (S3 + Glue + Athena/Trino) + APIs Ligeras — Alto nivel

```mermaid
flowchart TB
  subgraph Client["Cliente / Navegador"]
    UI["Next.js (CloudFront)"]
  end

  subgraph API["APIs Ligeras"]
    API1["FastAPI/NestJS (Lambda/Fargate)"]
  end

  subgraph Lake["Data Lake en S3"]
    RAW["Raw"]
    BRZ["Bronze"]
    SLV["Silver"]
    GLD["Gold (Iceberg/Delta)"]
  end

  GLUE["AWS Glue (Jobs/Catalog)"]
  ATH["Athena/Trino (SQL)"]
  QS["QuickSight (BI)"]
  DB[(Aurora PostgreSQL - Metadatos/OLTP)]
  ORCH["Step Functions / Airflow (MWAA)"]
  SEC["Lake Formation + KMS"]
  OBS["CloudWatch + Job Metrics + Data Quality"]

  UI --> API1
  API1 --> DB
  API1 --> ATH
  ORCH --> GLUE
  GLUE --> RAW --> BRZ --> SLV --> GLD
  GLUE -->|Catálogo| ATH
  ATH --> QS
  SEC --- Lake
  OBS --- ORCH & GLUE
```

---

## 5) Solo Web / On-Prem / VPS — Alto nivel

```mermaid
flowchart LR
  subgraph Users["Usuarios"]
    UI["Next.js (Nginx)"]
  end

  subgraph Server["Servidor/VPS On-Prem"]
    NGINX["Nginx (Reverse Proxy)"]
    API["FastAPI/NestJS (Gunicorn/Uvicorn/PM2)"]
    JOBS["Cron/Systemd Timers (Jobs)"]
    MQ["RabbitMQ/Redis (Colas)"]
    DB[(PostgreSQL)]
    FS[(MinIO/NAS - Archivos)]
    EXP["wkhtmltopdf / OpenPyXL / zip"]
    OBS["Prometheus + Grafana + Loki"]
    AUTH["Keycloak/JWT"]
    BKP["Backups: pg_dump + restic/borg"]
  end

  UI --> NGINX --> API
  API --> DB
  API --> FS
  API --> MQ
  JOBS --> API
  JOBS --> MQ
  EXP --> FS
  AUTH --> API
  OBS --- Server
  BKP --- DB & FS
```

---

## Anexos — Capas/Responsabilidades (Capas o Hexagonal)

```mermaid
flowchart LR
  UI["Capa Presentación (Next.js)"]
  APP["Capa Aplicación (AppServices/UseCases)"]
  DOM["Capa Dominio (Entidades/VOs/Reglas)"]
  INFRA["Capa Infraestructura (DB/Storage/Colas/SRI/Export)"]

  UI --> APP --> DOM
  APP --> INFRA
```

```mermaid
flowchart LR
  INB["Puertos Inbound (REST/Cron/Cola)"]
  APP2["Casos de Uso"]
  OUT["Puertos Outbound (Repo/Storage/SRI/Export/Bus)"]

  INB --> APP2 --> OUT
```
