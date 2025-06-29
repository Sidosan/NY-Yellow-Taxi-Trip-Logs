# Arquitetura de Dados Orientada a Eventos – NYC Taxi

Este documento apresenta uma solução completa de Data Reliability usando exclusivamente ferramentas open-source em Docker Compose para três pipelines JSON em tempo real das bases Yellow, Green e FHV de táxis de Nova Iorque. A arquitetura prioriza Governança e Qualidade de Dados, cobrindo também Confiabilidade, Observabilidade e Rastreabilidade end-to-end.

## 1. Contextualização

Cada pipeline referencia a documentação oficial da NYC Taxi and Limousine Commission (TLC) para extrair campos, tipos e regras de negócio rigorosas. Os dicionários de dados oficiais de março de 2025 incluem Yellow Taxi (23 campos), Green Taxi (com `trip_type` específico) e For-Hire Vehicles (estrutura simplificada com `dispatching_base_num`).

O enriquecimento utiliza o catálogo oficial que mapeia LocationIDs (1-265) para borough e nome da zona. Os arquivos Parquet seguem a nomenclatura oficial: `yellow_tripdata_2025-01.parquet`, `green_tripdata_2025-01.parquet` e `fhv_tripdata_2025-01.parquet`.

### JSON Schema Completo

O JSON Schema Draft-07 define validação para cada tipo de veículo, com VendorID restrito aos valores atuais válidos:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "YellowTaxiRecord",
  "type": "object",
  "required": [
    "trace_id",
    "VendorID", 
    "tpep_pickup_datetime",
    "passenger_count",
    "PULocationID",
    "fare_amount"
  ],
  "properties": {
    "trace_id": {"type": "string", "format": "uuid"},
    "VendorID": {"type": "integer", "enum": [1, 2]},
    "tpep_pickup_datetime": {"type": "string", "format": "date-time"},
    "passenger_count": {"type": "integer", "minimum": 1},
    "PULocationID": {"type": "integer", "minimum": 1},
    "fare_amount": {"type": "number", "minimum": 0.0}
  },
  "additionalProperties": true
}
```

## 2. Arquitetura

A arquitetura implementa seis camadas horizontais, garantindo separação de responsabilidades e escalabilidade.\
![diagrama](https://github.com/Sidosan/NY-Yellow-Taxi-Trip-Logs/raw/main/img.png)

**Camada 1 - Ingestão**: Simulador Python converte dados Parquet linha-a-linha em JSON streams, gravando no MinIO S3-compatible storage.\
**Camada 2 - Processamento**: Spark Structured Streaming lê diretamente arquivos JSON do bucket MinIO via protocolo s3a://.\
**Camada 3 - Armazenamento**: PostgreSQL recebe dados processados com exactly-once semantics.\
**Camada 4 - Governança**: DataHub cataloga metadados e lineage automaticamente.\
**Camada 5 - Qualidade**: Great Expectations executa validações inline no Spark.\
**Camada 6 - Observabilidade**: Stack Prometheus/Grafana/Alertmanager monitora SLIs/SLOs.


O Kafka atua apenas como componente opcional de buffering e replay, sem ser requisito para leitura pelo Spark.

## 3. Simulação Parquet → JSON

O simulador Python usa PyArrow e pandas para converter cada linha de Parquet em JSON individualmente, respeitando rigorosamente os schemas oficiais TLC. O processamento utiliza os arquivos oficiais `yellow_tripdata_2025-01.parquet`, `green_tripdata_2025-01.parquet` e `fhv_tripdata_2025-01.parquet`.

### Formato Único de JSON

O sistema utiliza **arquivo JSON único** por registro em `/json-stream/<fonte>/YYYY/MM/DD/<uuid>.json`, facilitando reprocessamento e lineage detalhado. Cada registro inclui metadados de pipeline: `trace_id` UUID único, `source` identifier e `processing_timestamp` para rastreabilidade completa.

**Exemplo do formato escolhido**:

```
/json-stream/yellow_taxi/2025/01/01/550e8400-e29b-41d4-a716-446655440000.json
/json-stream/yellow_taxi/2025/01/01/a1b2c3d4-e5f6-7g8h-9i0j-k1l2m3n4o5p6.json
/json-stream/green_taxi/2025/01/01/f47ac10b-58cc-4372-a567-0e02b2c3d479.json
```

### Snippet Python de Simulação Completo

```python
import pyarrow.parquet as pq
import pandas as pd
import json
import uuid
import time
from datetime import datetime
import boto3

def simulate_parquet_to_json_stream(fonte, parquet_file):
    s3_client = boto3.client('s3',
        endpoint_url='http://minio:9000',
        aws_access_key_id='minioadmin',
        aws_secret_access_key='minioadmin123')

    pq_file = pq.ParquetFile(parquet_file)
    for batch in pq_file.iter_batches(batch_size=1):
        df = batch.to_pandas()
        for _, row in df.iterrows():
            rec = {
                "trace_id": str(uuid.uuid4()),
                "source": fonte,
                "processing_timestamp": datetime.utcnow().isoformat(),
                **row.to_dict()
            }
            dt = datetime.utcnow()
            key = f"json-stream/{fonte}/{dt.strftime('%Y/%m/%d')}/{uuid.uuid4()}.json"
            s3_client.put_object(
                Bucket='minio',
                Key=key,
                Body=json.dumps(rec)
            )
            time.sleep(0.1)
```

## 4. Pipeline Spark Structured Streaming

O Spark lê JSON direto de MinIO, aplica validação Great Expectations inline e enriquece com dados de zona, persistindo em PostgreSQL com exactly-once semantics.

```python
spark.readStream \
    .format("json") \
    .schema(schema_yellow_taxi) \
    .option("maxFilesPerTrigger", 1) \
    .load("s3a://minio/json-stream/yellow_taxi/")
```

### Configuração de Quarentena

Registros que falharem nas validações Great Expectations são roteados para quarentena usando **tabelas PostgreSQL** com DDLs específicos:

**Yellow Taxi Quarantine**:

```sql
CREATE TABLE IF NOT EXISTS yellow_taxi_quarantine (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    trace_id UUID NOT NULL,
    record JSONB NOT NULL,
    processing_timestamp TIMESTAMPTZ NOT NULL DEFAULT now(),
    error_details TEXT NOT NULL,
    validation_failures JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Green Taxi Quarantine**:

```sql
CREATE TABLE green_taxi_quarantine (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    trace_id UUID NOT NULL,
    record JSONB NOT NULL,
    processing_timestamp TIMESTAMPTZ NOT NULL DEFAULT now(),
    error_details TEXT NOT NULL,
    validation_failures JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**FHV Quarantine**:

```sql
CREATE TABLE fhv_quarantine (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    trace_id UUID NOT NULL,
    record JSONB NOT NULL,
    processing_timestamp TIMESTAMPTZ NOT NULL DEFAULT now(),
    error_details TEXT NOT NULL,
    validation_failures JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

A quarentena permite auditoria de dados inválidos sem interromper o fluxo principal, incluindo análise de padrões de falha e reprocessamento controlado.

```python
writeStream \
  .format("jdbc") \
  .option("url","jdbc:postgresql://postgres:5432/analytics") \
  .option("dbtable","yellow_taxi_processed") \
  .option("checkpointLocation","s3a://minio/checkpoints/yellow_taxi") \
  .outputMode("append") \
  .start()
```

## 5. Governança de Metadados (DataHub)

DataHub cataloga datasets MinIO e tabelas PostgreSQL, gerenciando entidades Dataset, SchemaField e Lineage automaticamente. A ingestão unificada é definida em YAML cobrindo os três pipelines simultaneamente. Os JSON Schemas são versionados em Git com tags semânticas, permitindo schema evolution controlada via pull requests documentados.

### YAML de Ingestão no DataHub

```yaml
source:
  type: postgresql
  config:
    host_port: postgres:5432
    database: analytics
    username: datahub_user
    password: datahub_pass
    table_pattern:
      allow:
        - yellow_taxi_processed
        - green_taxi_processed  
        - fhv_processed
        - yellow_taxi_quarantine
        - green_taxi_quarantine
        - fhv_quarantine

transformers:
  - type: simple_add_dataset_tags
    config:
      tag_urns:
        - "urn:li:tag:nyc_taxi_data"
        - "urn:li:tag:streaming_pipeline"

sink:
  type: datahub-rest
  config:
    server: http://datahub-gms:8080

pipeline_name: nyc_taxi_unified_ingestion
run_id: taxi_ingestion_2025
```

## 6. Qualidade de Dados (Great Expectations)

Suites por fonte definem expectativas rigorosas baseadas nos dicionários oficiais TLC. Yellow Taxi: 7 expectativas incluindo validação de `VendorID` enum, ranges de `passenger_count` ≥1 e unicidade de `trace_id`. Green Taxi: 6 expectativas com `VendorID` restrito e validação específica de `trip_type`. FHV: 8 expectativas focando em padrão regex para base numbers e ranges de LocationID [1-265].

Falhas críticas bloqueiam gravação principal, enviam para quarentena e disparam alertas via Alertmanager.

## 7. Observabilidade & SLIs/SLOs

Prometheus coleta métricas de latência P99 (<500ms), throughput (>1000/s) e erros de validação (<1%/min). Grafana exibe dashboards específicos por pipeline com visualização de SLOs em tempo real. Alertmanager gerencia notificações baseadas em regras YAML versionadas em Git, integrando webhooks para violações de SLO.

### Regras de Alertmanager

```yaml
# alert_rules.yml
groups:
  - name: taxi_pipeline_alerts
    rules:
      - alert: HighLatencyP99
        expr: histogram_quantile(0.99, rate(pipeline_processing_duration_seconds_bucket[5m])) > 0.5
        for: 2m
        labels:
          severity: critical
          pipeline: taxi_data
          team: data_engineering
        annotations:
          summary: "Pipeline latency P99 exceeded 500ms"
          description: "Pipeline {{ $labels.pipeline }} has P99 latency of {{ $value }}s for more than 2 minutes"
          runbook_url: "URL do runbook interno"
          
      - alert: ValidationErrorRateHigh  
        expr: rate(great_expectations_failed_total[1m]) > 0.01
        for: 1m
        labels:
          severity: warning
          pipeline: taxi_data
          team: data_quality
        annotations:
          summary: "Data quality validation failure rate > 1%/min"
          description: "Validation error rate is {{ $value | humanizePercentage }} per minute for pipeline {{ $labels.pipeline }}"
          runbook_url: "URL do runbook interno"

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093

rule_files:
  - "alert_rules.yml"

# alertmanager.yml
global:
  slack_api_url: 'https://hooks.slack.com/services/SEU/WEBHOOK/AQUI'

route:
  group_by: ['alertname', 'pipeline']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'taxi-pipeline-alerts'

receivers:
  - name: 'taxi-pipeline-alerts'
    slack_configs:
      - channel: '#data-alerts'
        title: 'NYC Taxi Pipeline Alert - {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
        send_resolved: true
    webhook_configs:
      - url: 'http://slack-webhook:3000/alerts'
        send_resolved: true
        http_config:
          basic_auth:
            username: 'alert_user'
            password: 'alert_password'
```

## 8. Rastreabilidade & Debug

Logs JSON estruturados incluem `trace_id`, `source`, `step` e `error_detail`, permitindo investigação end-to-end completa. O runbook SQL permite seguir qualquer `trace_id` através das tabelas `<fonte>_processed`, quarentena e métricas de qualidade.

```sql
-- Investigação completa de trace_id
WITH trace_investigation AS (
    SELECT 'yellow_taxi' as pipeline, trace_id, processing_timestamp, created_at
    FROM yellow_taxi_processed WHERE trace_id = :trace_id_param
    UNION ALL
    SELECT 'green_taxi' as pipeline, trace_id, processing_timestamp, created_at  
    FROM green_taxi_processed WHERE trace_id = :trace_id_param
    UNION ALL
    SELECT 'fhv' as pipeline, trace_id, processing_timestamp, created_at
    FROM fhv_processed WHERE trace_id = :trace_id_param
)
SELECT * FROM trace_investigation ORDER BY processing_timestamp;
```

## 9. Estrutura de Pastas do Repositório

A organização modular facilita manutenção, deploy e colaboração entre equipes de dados.

```text
nyc-taxi-data-pipeline/
├── docker-compose.yml              # Orquestração completa
├── .env                           # Variáveis de ambiente
├── schemas/                       # JSON Schemas para validação
│   ├── yellow_taxi_schema.json
│   ├── green_taxi_schema.json
│   └── fhv_schema.json
├── simulator/                     # Container Python de geração
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── simulator.py              # Script principal
│   └── config/
│       └── simulation_config.yaml
├── spark/                         # Spark Structured Streaming  
│   ├── Dockerfile
│   ├── streaming_job.py          # Job principal
│   ├── schemas/
│   │   └── spark_schemas.py      # StructType schemas
│   └── checkpoints/              # Checkpoint locations
├── expectations/                  # Great Expectations
│   ├── great_expectations.yml
│   └── expectations/
│       ├── yellow_taxi_suite.json
│       ├── green_taxi_suite.json
│       └── fhv_suite.json
├── datahub/                      # Configurações DataHub
│   ├── ingestion.yml             # Ingestão unificada
│   └── glossary.yml              # Termos de negócio
├── monitoring/                   # Stack observabilidade
│   ├── prometheus/
│   │   ├── prometheus.yml
│   │   └── rules/
│   │       └── alert_rules.yml
│   ├── grafana/
│   │   └── dashboards/
│   └── alertmanager/
│       └── alertmanager.yml
├── init-scripts/                 # Inicialização PostgreSQL
│   ├── 01_create_databases.sql
│   ├── 02_create_tables.sql
│   └── 03_create_quarantine_tables.sql
└── docs/                        # Documentação
    ├── architecture.md
    └── deployment_guide.md
```

### Deploy e Execução

O sistema inclui script automatizado de inicialização que processa consistentemente os arquivos oficiais `yellow_tripdata_2025-01.parquet`, `green_tripdata_2025-01.parquet` e `fhv_tripdata_2025-01.parquet`. O ambiente é completamente containerizado com Docker Compose, facilitando deploy local e portabilidade para Kubernetes.

Interfaces Web Disponíveis: Grafana Dashboard (localhost:3000), MinIO Console (localhost:9001), Spark UI (localhost:8080), DataHub (localhost:9002) e Alertmanager (localhost:9093).

---

## 10. Próximo Passo: Agentes de IA em Data Reliability

A incorporação de agentes de Data Reliability no pipeline eleva a governança de dados de reativa para proativa, reduzindo falhas e acelerando correções com automação inteligente.

### Definição e Arquitetura

**Agentes de IA** são microserviços autônomos que consomem eventos, métricas e logs e atuam em tempo real para monitorar, validar, remediar e documentar o fluxo de dados. Cada agente pode ser empacotado como container Docker e expor APIs REST ou mensageria (Kafka, gRPC) para integração com o pipeline existente.

1. **Event Listener**
Escuta tópicos Kafka ou bucket MinIO e dispara pipelines de verificação.
2. **Validation Engine**
Aplica regras de schema e qualidade (Great Expectations ou UDFs Spark) e registra resultados em Prometheus.
3. **Remediation Orchestrator**
Executa jobs de limpeza ou replay de offsets via Kafka Connect/Spark, escalonando incidentes persistentes.
4. **Metadata Reporter**
Atualiza DataHub via API, documentando correções, falhas e versões de schema.

### Casos de Uso

- **Detecção e Quarentena Automática**
Ao receber JSONs inválidos, o agente move registros para `quarantine/` no MinIO e dispara alerta no Alertmanager.
- **Autocorreção por Padrões**
Identifica outliers em `fare_amount` e executa script Python de normalização antes do `writeStream` em PostgreSQL.
- **Reprocessamento Inteligente**
Fracassos de conexão com o banco geram replays de Kafka, usando backoff exponencial e notificação em Slack via webhook.
- **Enriquecimento Dinâmico**
Sempre que `PULocationID` estiver faltando, o agente consulta API de zonas e injeta dados faltantes antes do sink final.
- **Auditoria Contínua**
Gera relatórios diários de conformidade por `trace_id` no DataHub e persiste logs estruturados para análise histórica.

### Implementação

- **Em Python/Flask ou FastAPI**
Containers leves ou Lambdas que consomem eventos de Kafka, aplicam validações e chamam endpoints de remediação.
- **Como Jobs Spark**
Utilização de **Structured Streaming with UDF triggers** para acionar funções de remediação baseadas em estatísticas de DataFrame.
- **Frameworks Recomendados**
    - **LangChain Agents**: coordena sub-agentes para descoberta de schemas e execução de correções (LangChain)​.
    - **Monte Carlo AI Observability**: “Data Quality AI” para detecção preditiva de falhas e análise de causa raiz automática.


### Benefícios

- **Mitigação Proativa** de incidentes, reduzindo tempo médio de resolução em até 80%.
- **Governança Automatizada**: atualizações de catálogo e lineage em segundos.
- **Escalabilidade**: novos agentes podem ser adicionados sem impacto em produção.
- **Visibilidade Centralizada**: todos os eventos de confiabilidade consolidados em dashboards Grafana.

---

## 11. Fontes e Documentação de Referência

### Dados e Documentação Oficial (NYC TLC)

Fontes centrais que fornecem o acesso aos datasets brutos e os guias oficiais para sua utilização.

- **Portal TLC Trip Record Data**
    - **Descrição**: Página oficial da NYC Taxi and Limousine Commission que centraliza o acesso a todos os datasets (Yellow, Green, FHV), notícias e atualizações.
    - **URL**: `https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page`
- **Guia do Usuário dos Registros de Viagem**
    - **Descrição**: Documento fundamental que detalha os formatos disponíveis, particularidades sobre a coleta e o significado dos campos.
    - **URL**: `https://www.nyc.gov/assets/tlc/downloads/pdf/trip_record_user_guide.pdf`


### Dicionários de Dados e Enriquecimento

Documentos essenciais para a validação de schemas, definição de tipos de dados, regras de negócio e a implementação das suítes de qualidade de dados.

- **Dicionário de Dados - Yellow Taxi**
    - **URL**: `https://www.nyc.gov/assets/tlc/downloads/pdf/data_dictionary_trip_records_yellow.pdf`
- **Dicionário de Dados - Green Taxi**
    - **URL**: `https://www.nyc.gov/assets/tlc/downloads/pdf/data_dictionary_trip_records_green.pdf`
- **Dicionário de Dados - For-Hire Vehicle (FHV)**
    - **URL**: `https://www.nyc.gov/assets/tlc/downloads/pdf/data_dictionary_trip_records_fhv.pdf`
- **Catálogo de Zonas para Enriquecimento**
    - **Descrição**: Arquivo CSV utilizado para enriquecer os dados, mapeando `LocationID` para `Borough`, `Zone` e `service_zone`.
    - **URL**: `https://d37ci6vzurychx.cloudfront.net/misc/taxi_zone_lookup.csv`


---

