# Pipeline Híbrido para Análise da Alfabetização no Brasil

**Tech Challenge — Fase 2 | PosTech FIAP | IA Science**

---

## Contexto do Problema

A alfabetização na infância é um dos pilares fundamentais para o desenvolvimento educacional, social e econômico do Brasil. O **Compromisso Nacional Criança Alfabetizada** mobiliza União, estados, Distrito Federal e municípios com a meta de garantir que **todas as crianças brasileiras estejam alfabetizadas até o final do 2º ano do Ensino Fundamental até 2030**.

Para medir esse avanço, o INEP criou o **Indicador Criança Alfabetizada**, que expressa o percentual de estudantes que atingem **743 pontos na escala SAEB** — ponto de corte definido pela Pesquisa Alfabetiza Brasil (2023). Compreender os fatores que influenciam esse indicador exige integrar múltiplas fontes: metas nacionais, estaduais e municipais, microdados educacionais e dados territoriais.

---

## Arquitetura da Solução

# Pipeline de Análise da Alfabetização no Brasil
## Tech Challenge — Fase 2 · PosTech FIAP

> **Nota arquitetural:** Este repositório contém duas visões da solução.
> - **AS-IS (esta seção):** arquitetura implementada e entregue, rodando 100% no free tier do GCP.
> - **TO-BE:** visão profissional de evolução, documentada na seção seguinte, representando
>   como a pipeline seria construída em um ambiente corporativo real.

---
---

## 📐 Arquitetura AS-IS (Implementada)

A pipeline implementada usa **BigQuery como plataforma única** para todas as
camadas (Bronze, Silver e Gold), com scripts Python orquestrados por GitHub
Actions. A escolha priorizou simplicidade de implementação e custo zero
dentro do free tier do GCP.


```
┌─────────────────────────────────────────────────────────────┐
│           FONTE: Base dos Dados (BigQuery público)          │
│   basedosdados.br_inep_avaliacao_alfabetizacao              │
│   ├── uf                  ├── meta_alfabetizacao_brasil     │
│   ├── municipio           ├── meta_alfabetizacao_uf         │
│   ├── alunos              └── meta_alfabetizacao_municipio  │
└───────────────────┬─────────────────────────────────────────┘
                    │
        ┌───────────┴───────────┐
        │ INGESTÃO HÍBRIDA      │
        ├───────────────────────┤
        │  BATCH (periódico)    │  STREAMING (Pub/Sub)
        │  ingest_bronze.py     │  producer.py → consumer.py
        │  6 tabelas completas  │  eventos simulados em RT
        └───────────┬───────────┘
                    │
    ╔═══════════════▼═════════════════════╗
    ║  BRONZE — Dados Brutos (BigQuery)   ║
    ║  bronze.alfabetizacao_uf            ║
    ║  bronze.meta_brasil                 ║
    ║  bronze.meta_uf                     ║
    ║  bronze.meta_municipio              ║
    ║  bronze.alfabetizacao_municipio     ║
    ║  bronze.alunos                      ║
    ║  bronze.streaming_eventos           ║
    ╚═══════════════╦═════════════════════╝
                    ║ transform_silver.py
                    ║ dedup · filtro · normalização · integração
    ╔═══════════════▼═════════════════════╗
    ║  SILVER — Dados Tratados            ║
    ║  silver.alfabetizacao_uf_clean      ║
    ║  silver.metas_consolidadas          ║
    ║  silver.alfabetizacao_municipio_clean║
    ║  silver.alunos_clean                ║
    ╚═══════════════╦═════════════════════╝
                    ║ build_gold.py
                    ║ agregação · ranking · JOIN metas
    ╔═══════════════▼═════════════════════╗
    ║  GOLD — Camada Analítica            ║
    ║  gold.indicador_por_uf_ano          ║
    ║  gold.evolucao_temporal_brasil      ║
    ║  gold.ranking_estados               ║
    ║  gold.perfil_desempenho_uf          ║
    ║  gold.painel_municipios             ║
    ╚═════════════════════════════════════╝
                    │
            quality/validate.py
            checks bronze · silver · gold
```

---

## Fontes de Dados

| Tabela Bronze | Fonte Base dos Dados | Descrição |
|---|---|---|
| `alfabetizacao_uf` | `uf` + `dicionario` + diretório UF | Taxa por estado/série/rede |
| `meta_brasil` | `meta_alfabetizacao_brasil` | Metas nacionais 2024–2030 |
| `meta_uf` | `meta_alfabetizacao_uf` | Metas por estado 2024–2030 |
| `meta_municipio` | `meta_alfabetizacao_municipio` | Metas municipais 2024–2030 |
| `alfabetizacao_municipio` | `municipio` + `dicionario` | Taxa por município/série/rede |
| `alunos` | `alunos` + `dicionario` | Microdados individuais de alunos |

---

---

## 🛠️ Tecnologias Utilizadas — AS-IS

| Componente | Tecnologia | Justificativa |
|---|---|---|
| Cloud | **GCP** | Free tier generoso; BigQuery nativo e integrado; $0/mês viável |
| Orquestração | **GitHub Actions** | Gratuito para repositórios públicos; cron semanal; autenticação via `GCP_SA_KEY` como secret; sequência ingest → transform → build → validate em um único workflow |
| Autenticação | **Service account única** | `GCP_SA_KEY` armazenada como secret do repositório; roles mínimas: BigQuery Data Editor · BigQuery Job User · Pub/Sub Editor — princípio do menor privilégio |
| Ingestão batch | **ingest_bronze.py (Python 3.11 + BigQuery client)** | 6 consultas federadas contra a Base dos Dados pública; WRITE_TRUNCATE garante idempotência; sem dependência de Dataflow ou Spark |
| Ingestão streaming | **producer.py + consumer.py (Pub/Sub)** | Simula chegada de eventos em tempo real; auto-provisiona tópico e subscription na primeira execução; grava via load job em micro-lote (gratuito — sem insertAll pago) |
| Armazenamento Bronze | **BigQuery dataset bronze** | 7 tabelas; partição por ano; cluster por id_municipio e serie; ingestao_ts preserva histórico; 3,9M linhas de alunos |
| Transformação Silver | **transform_silver.py** | Dedup via ROW_NUMBER por chave; filtro de enriquecimento incompleto; normalização de tipos e chaves; metas_consolidadas unifica brasil + UF + município |
| Camada analítica Gold | **build_gold.py + BigQuery dataset gold** | 5 tabelas analíticas prontas para BI e ML; INNER JOIN contra metas; cálculo de gap vs meta 2030 |
| Qualidade de dados | **quality/validate.py** | 35+ checks nativos em Python + BigQuery; verifica duplicidade, NULLs críticos, domínio de 27 UFs, taxa entre 0 e 100, integridade referencial; exit 1 reprova o workflow automaticamente |
| Controle de acesso | **Cloud IAM (service account única)** | Roles mínimas por princípio do menor privilégio; sem chaves expostas no código |
| Monitoramento | **logging Python + GitHub Actions** | Logging estruturado com linhas, tempos e erros por etapa; falha no validate.py reprova o workflow e envia e-mail automático do CI |
| FinOps | **BigQuery free tier** | Storage colunar nativo; partição e clustering reduzem bytes escaneados; SELECT explícito evita `SELECT *`; load jobs gratuitos; tudo dentro do free tier — $0/mês |
| Linguagem | **Python 3.11** | Bibliotecas maduras para GCP (google-cloud-bigquery, google-cloud-pubsub); padrão na engenharia de dados |

---

## ⚖️ Decisões Arquiteturais e Trade-offs — AS-IS

### Batch vs Streaming

| | Batch (AS-IS) | Streaming (AS-IS) |
|---|---|---|
| **Implementação** | `ingest_bronze.py` semanal via cron | `producer.py` + `consumer.py` via Pub/Sub |
| **Custo** | Gratuito — load jobs no free tier | Gratuito — micro-lote via load job (sem insertAll pago) |
| **Latência** | Semanal (segunda 6h UTC) | Minutos — micro-lote |
| **Dados** | Base dos Dados pública · INEP · metas | Eventos simulados: medição · meta · indicador |

**Decisão:** batch para o volume histórico (90% dos dados); streaming simulado via Pub/Sub para demonstrar a capacidade de ingestão de eventos em tempo real sem custo adicional.

---

### BigQuery puro vs Lakehouse (GCS + BigQuery)

| | AS-IS (BigQuery puro) | TO-BE (GCS + BigQuery) |
|---|---|---|
| **Complexidade** | Baixa — um serviço | Alta — dois serviços + Iceberg |
| **Custo** | $0/mês (free tier) | ~$3/mês |
| **Schema** | Rígido desde a Bronze | Flexível na Bronze/Silver |
| **Time travel** | 7 dias (BQ padrão) | Ilimitado (Iceberg) |
| **Reprocessamento** | Paga por query na Bronze | Lê GCS sem custo de query |
| **Lock-in** | Total no BigQuery | Formato aberto (Parquet) |
| **Ideal para** | Projeto acadêmico · equipe pequena | Produção · múltiplos engines |

**Decisão AS-IS:** BigQuery puro elimina camadas intermediárias e mantém custo zero. A mesma lógica de Bronze → Silver → Gold é preservada, apenas dentro do BigQuery em vez de GCS + BigQuery.

---

### Custo vs Performance — AS-IS

| Decisão | Escolha | Impacto |
|---|---|---|
| Armazenamento | BigQuery colunar nativo | Sem custo de GCS separado |
| Particionamento | Por ano na Bronze | Queries leem só o período necessário |
| Clusterização | Por id_municipio e serie | Reduz bytes escaneados em filtros comuns |
| Queries | SELECT explícito (nunca `SELECT *`) | Evita escaneamento desnecessário |
| Ingestão | WRITE_TRUNCATE idempotente | Sem duplicação em reexecuções |
| Streaming | Load job em micro-lote | insertAll pago evitado — permanece no free tier |
| Orquestração | GitHub Actions gratuito | Zero custo fixo de orquestração |

**Decisão: free tier primeiro, performance suficiente.** Para um volume de dados educacionais públicos (~3,9M linhas de alunos), o BigQuery free tier entrega performance adequada sem nenhum custo.

---

## 💰 FinOps — AS-IS

| Serviço | Uso | Custo |
|---|---|---|
| BigQuery (storage) | ~5 GB entre Bronze/Silver/Gold | Grátis (10 GB/mês free tier) |
| BigQuery (queries) | ~50 GB escaneados/mês | Grátis (1 TB/mês free tier) |
| BigQuery (load jobs) | Ingestão batch e micro-lote streaming | Grátis (sem cobrança por load job) |
| Pub/Sub | ~5 GB de mensagens/mês | Grátis (10 GB/mês free tier) |
| GitHub Actions | Workflows públicos | Grátis |
| **Total** | | **$0/mês** |

**Práticas adotadas:**
- `WRITE_TRUNCATE` garante idempotência sem acúmulo de dados duplicados
- `SELECT` explícito em todas as queries evita escaneamento de colunas desnecessárias
- Partição por ano e clustering por `id_municipio/serie` reduzem bytes escaneados nas queries analíticas
- Streaming via load job em micro-lote em vez de `insertAll` (que seria cobrado fora do free tier)
- `exit 1` no `validate.py` evita que dados inválidos contaminem camadas downstream, prevenindo reprocessamentos custosos

---

## 📊 Qualidade de Dados — AS-IS

O `quality/validate.py` executa 35+ checks automaticamente ao final de cada run:

| Categoria | O que valida |
|---|---|
| **Duplicidade** | ROW_NUMBER por chave em Bronze e Silver |
| **NULLs críticos** | co_municipio · co_uf · indicador · ano em todas as camadas |
| **Domínio** | 27 UFs válidas · taxa entre 0 e 100 · anos dentro do range esperado |
| **Integridade referencial** | Municípios da Gold existem na Silver · metas têm correspondência |
| **Consistência** | streaming_eventos_clean sem duplicata por evento+métrica |
| **CI/CD** | exit 1 reprova o workflow · GitHub envia e-mail automático de falha |

---

## 🤖 Aplicação em IA — AS-IS e TO-BE

A camada Gold está pronta para análises avançadas (evolução futura — TO-BE):

| Caso de uso | Dado de entrada (Gold) | Saída esperada |
|---|---|---|
| **Predição de alfabetização** | `evolucao_temporal_brasil` + `perfil_desempenho_uf` | Municípios com risco de não atingir meta 2030 |
| **Clustering de vulnerabilidade** | `painel_municipios` + dados socioeconômicos | Segmentos de municípios por perfil educacional |
| **Projeção meta 2030** | `indicador_por_uf_ano` (série temporal) | Projeção por UF com intervalo de confiança |
| **Predição municipal** | `painel_municipios` + features IBGE/FUNDEB | Score de risco por município |

---

## 📋 Monitoramento — AS-IS

| O que monitorar | Como | Alerta |
|---|---|---|
| Falha em qualquer etapa | GitHub Actions · exit 1 | E-mail automático do CI |
| Qualidade de dados | `validate.py` · 35+ checks | Reprova o workflow se exit 1 |
| Tempo de execução | logging Python por etapa | Log estruturado no console do Actions |
| Volume ingerido | Contagem de linhas por tabela | Log estruturado comparado ao run anterior |

---

# TO-BE — Visão Profissional (Evolução Futura)

> Esta seção descreve como a pipeline evoluiria em um ambiente corporativo real,
> com maior volume de dados, múltiplas equipes e requisitos de SLA. Não foi
> implementada neste projeto — representa a arquitetura-alvo para uma
> organização pública de análise educacional em escala nacional.

## O que muda do AS-IS para o TO-BE

| Dimensão | AS-IS (Implementado) | TO-BE (Visão Profissional) |
|---|---|---|
| **Armazenamento** | BigQuery puro | GCS (Bronze/Silver) + BigQuery (Gold) |
| **Formato** | BigQuery nativo | Apache Iceberg sobre GCS |
| **Autenticação** | `GCP_SA_KEY` como secret | Workload Identity Federation (sem chave) |
| **Qualidade** | `validate.py` em Python | SQL assertions nativas no BigQuery |
| **Governança** | Logging Python | Dataplex (opcional) ou labels/descriptions BQ |
| **Streaming** | `producer.py` simulado | Pub/Sub + Dataflow always-on |
| **Transformação** | `transform_silver.py` Python | Dataflow (Apache Beam) serverless |
| **API externa** | Não implementada | API Gateway (Apigee) + JWT + Cloud Run |
| **BI** | Não implementado | Looker Studio |
| **ML** | Não implementado | Vertex AI + Vertex AI Gemini |
| **Custo** | $0/mês (free tier) | ~$3–5/mês (estimado) |
| **Escalabilidade** | Adequada para dados educacionais públicos | Petabytes · múltiplos engines |

## Por que o TO-BE não foi implementado

- **Workload Identity Federation** exige configuração de pool no IAM que ultrapassa o escopo do free tier
- **Dataflow** tem custo por hora de worker — inviável para projeto acadêmico
- **GCS + Iceberg** adiciona complexidade de configuração sem benefício prático no volume atual (~3,9M linhas)
- **API Gateway (Apigee)** é um serviço pago sem free tier significativo
- O free tier do BigQuery é suficiente para o volume e a frequência de acesso deste projeto

---

## Como Executar Localmente

### Pré-requisitos

1. Conta GCP com projeto `project-516b6700-5d68-403c-860`
2. APIs habilitadas: BigQuery API, Pub/Sub API
3. Service Account com roles:
   - `BigQuery Data Editor`
   - `BigQuery Job User`
   - `Pub/Sub Editor`
4. Arquivo JSON da service account salvo em `credentials/service-account.json`

### Setup

```bash
pip install -r requirements.txt
export GOOGLE_APPLICATION_CREDENTIALS="credentials/service-account.json"
```

### Executar pipeline completo

```bash
python run_pipeline.py
```

### Executar etapas individualmente

```bash
# Bronze
python ingestion/batch/ingest_bronze.py

# Silver
python silver/transform_silver.py

# Gold
python gold/build_gold.py

# Qualidade
python quality/validate.py --camada all
python quality/validate.py --camada bronze
python quality/validate.py --camada silver
python quality/validate.py --camada gold
```

### Streaming (2 terminais separados)

```bash
# Terminal 1 — consumidor (deve estar rodando antes do produtor)
python streaming/consumer.py --max-mensagens 20 --timeout 60

# Terminal 2 — produtor
python streaming/producer.py --eventos 20 --intervalo 1.0
```

---

## GitHub Actions — Configuração

O workflow `.github/workflows/pipeline.yml` executa o pipeline completo automaticamente toda segunda-feira às 6h UTC e pode ser disparado manualmente.

### Criando o secret `GCP_SA_KEY`

1. No GCP Console, baixe o JSON da service account
2. No repositório GitHub: **Settings → Secrets and variables → Actions → New repository secret**
3. Nome: `GCP_SA_KEY`
4. Valor: conteúdo completo do JSON

### Permissões da service account

```
roles/bigquery.dataEditor
roles/bigquery.jobUser
roles/pubsub.editor
```

---

---

## ⚖️ Decisões Arquiteturais e Trade-offs

### Batch vs Streaming

| | Batch | Streaming |
|---|---|---|
| **Quando** | Metas anuais · INEP · IBGE · histórico | Novos indicadores · alertas urgentes |
| **Custo** | Baixo — jobs sob demanda | Maior — infra contínua (Pub/Sub always-on) |
| **Latência** | Horas / dias | Segundos |
| **Implementação** | GitHub Actions cron agendado | Pub/Sub + Dataflow always-on |

**Decisão: pipeline híbrida.** Batch para 90% do volume (dados históricos e metas periódicas); streaming apenas para eventos de atualização de indicadores, onde latência importa.

---

### Data Lake vs Data Warehouse

| | Data Lake (GCS) | Data Warehouse (BigQuery) |
|---|---|---|
| **Schema** | Flexível · formato aberto (Parquet/Iceberg) | Rígido · tipado · otimizado para query |
| **Custo/GB** | $0,004 (Coldline) a $0,02 (Standard) | $0,02 (ativo) |
| **Uso** | Bronze e Silver | Gold |
| **Engine** | Qualquer (Spark, Beam, Athena) | BigQuery |

**Decisão: lakehouse (GCS + BigQuery).** GCS para armazenar barato nas camadas Bronze e Silver; BigQuery apenas para a Gold analítica. Melhor dos dois mundos: custo de data lake com performance de data warehouse onde importa.

---

### Custo vs Performance

| Decisão | Escolha | Impacto |
|---|---|---|
| Formato de arquivo | Parquet | Reduz I/O e armazenamento em até 80% vs CSV |
| Particionamento | Por UF e ano | Queries leem só a partição necessária |
| Clusterização | Por município | Reduz bytes escaneados em queries analíticas |
| Storage histórico | Nearline / Coldline | Até 80% mais barato que Standard para dados frios |
| Processamento | Dataflow serverless | Paga só pelo tempo de execução, sem cluster ocioso |
| Orquestração | GitHub Actions | Zero custo fixo vs Cloud Composer (~$300/mês) |
| Consultas analíticas | BigQuery on-demand | Paga por dado escaneado, não por cluster ligado |
| API externa | Cloud Run | Escala a zero — custo zero em períodos ociosos |

**Decisão: custo primeiro, performance onde importa.** Storage frio e serverless para dados históricos; particionamento e clusterização apenas na Gold, onde as queries analíticas exigem velocidade.

---

## 💰 FinOps

### Estimativa de Custo Mensal

| Serviço | Uso estimado | Custo estimado |
|---|---|---|
| Google Cloud Storage (Standard) | 10 GB Bronze ativo | ~$0,20 |
| Google Cloud Storage (Nearline) | 50 GB histórico recente | ~$0,50 |
| Google Cloud Storage (Coldline) | 200 GB histórico antigo | ~$0,80 |
| BigQuery (armazenamento) | 5 GB Gold | ~$0,10 |
| BigQuery (queries) | 50 GB escaneados/mês | ~$0,25 (1 TB grátis) |
| Dataflow | 10h de processamento batch/mês | ~$1,50 |
| Pub/Sub | 5 GB eventos/mês | Grátis (free tier 10 GB) |
| Cloud Run | 1M requisições/mês | Grátis (free tier) |
| GitHub Actions | Workflows públicos | Grátis |
| Cloud Monitoring | Métricas básicas | Grátis |
| **Total estimado** | | **~$3,35/mês** |

> Os custos acima consideram o free tier do GCP e um volume típico de dados educacionais públicos. Em produção com maior volume, o particionamento e o Coldline garantem escala sem crescimento linear de custo.

### Práticas adotadas

- **Parquet** em todas as camadas — reduz I/O e armazenamento em até 80% vs CSV
- **Lifecycle Policy no GCS** — move dados automaticamente para Nearline (30 dias) e Coldline (90 dias) sem intervenção manual
- **Particionamento e clusterização no BigQuery** — queries leem apenas o subconjunto necessário, reduzindo custo por query
- **Dataflow sob demanda** — jobs existem apenas durante a execução, sem cluster ocioso
- **Cloud Run** — escala a zero fora de uso, custo zero em períodos sem requisições
- **GitHub Actions** — orquestração gratuita, eliminando ~$300/mês do Cloud Composer
- **Budget Alerts** — alertas em 50%, 80% e 100% do orçamento mensal definido

---

## 🤖 Aplicação em IA

A camada Gold é o ponto de partida para análises avançadas e inteligência artificial:

| Caso de uso | Tecnologia | Descrição |
|---|---|---|
| **Predição de alfabetização** | Vertex AI | Modelo treinado sobre série histórica para prever municípios com risco de não atingir a meta de 2030 |
| **Clusters de vulnerabilidade** | Vertex AI | Segmentação de municípios por perfil educacional e socioeconômico combinado |
| **Detecção de anomalias** | Vertex AI | Identifica queda abrupta no indicador para ação preventiva de gestores |
| **Análise de desigualdade** | BigQuery + Looker Studio | Comparação temporal do indicador entre UFs e municípios |
| **Relatórios automáticos** | Vertex AI Gemini | Gera resumos executivos em linguagem natural a partir dos dados da Gold |
| **Políticas baseadas em dados** | BigQuery + API Gateway | Subsidia decisões de alocação de recursos do FUNDEB com evidências quantitativas |

---

## 📊 Monitoramento

| O que monitorar | Ferramenta | Alerta configurado |
|---|---|---|
| Falha em workflow do GitHub Actions | Cloud Monitoring + GitHub | Notificação imediata por e-mail |
| Latência de job Dataflow | Cloud Monitoring | Alerta se > 30 min |
| Volume de dados ingeridos | Cloud Logging | Alerta se < 80% do esperado |
| Falha de entrega no Pub/Sub | Cloud Monitoring | Alerta em dead-letter queue |
| Custo mensal | Cloud Billing | Alerta em 50%, 80% e 100% do budget |
| Qualidade de dados (Silver) | Dataplex | Alerta se regras de qualidade reprovam > 1% dos registros |

## Estrutura do Repositório

```
1IAST-Fase-2-Tech-Challenge/
├── .github/
│   └── workflows/
│       └── pipeline.yml          # CI/CD — execução semanal automática
├── ingestion/
│   └── batch/
│       └── ingest_bronze.py      # Ingestão de 6 tabelas → bronze
├── silver/
│   └── transform_silver.py       # 4 tabelas silver (limpeza + integração)
├── gold/
│   └── build_gold.py             # 5 tabelas analíticas gold
├── quality/
│   └── validate.py               # Validação das 3 camadas (exit 1 em falha)
├── streaming/
│   ├── producer.py               # Publica eventos no Pub/Sub
│   └── consumer.py               # Consome Pub/Sub → bronze.streaming_eventos
├── config.py                     # IDs do projeto GCP e datasets
├── run_pipeline.py               # Orquestrador local (batch sequential)
├── requirements.txt
└── README.md
```


[def]: 2_c4_contexto.jp