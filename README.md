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
## Diagrama de Contexto AS IS

<img src="contexto_as_is.png" width="900">

## Diagrama de Container AS IS

<img src="Conatiner_as_is.png" width="900">

## Diagrama de Contexto TO BE

<img src="Contexto_to_be.png" width="900">

## Diagrama de Container TO BE

<img src="Container_to_be.png" width="900">


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

| | Batch | Streaming |
|---|---|---|
| **Quando** | Metas anuais · INEP · IBGE · histórico | Novos indicadores · alertas urgentes · medições de desempenho |
| **Custo AS-IS** | Grátis — load jobs no free tier | Grátis — micro-lote via load job (sem insertAll pago) |
| **Custo TO-BE** | Baixo — Dataflow jobs sob demanda | Maior — Pub/Sub + Dataflow always-on |
| **Latência AS-IS** | Semanal (segunda 6h UTC) | Minutos — micro-lote Pub/Sub |
| **Latência TO-BE** | Horas / dias | Segundos |
| **Implementação AS-IS** | `ingest_bronze.py` · GitHub Actions cron | `producer.py` + `consumer.py` · Pub/Sub simulado |
| **Implementação TO-BE** | GitHub Actions cron + Dataflow Apache Beam | Pub/Sub + Dataflow Streaming always-on |
| **Idempotência** | `WRITE_TRUNCATE` garante reexecução segura | Load job em micro-lote evita duplicação |
| **Escalabilidade** | Adequada para volume atual (~3,9M linhas) | Ilimitada — Dataflow escala automaticamente |

**Decisão AS-IS:** pipeline híbrida com custo zero. Batch via script Python para o volume histórico; streaming simulado via Pub/Sub + consumer.py para demonstrar capacidade de ingestão de eventos em tempo real sem ultrapassar o free tier.

**Decisão TO-BE:** mesma lógica híbrida, com infraestrutura gerenciada. Batch para 90% do volume (dados históricos e metas periódicas); streaming real via Pub/Sub + Dataflow always-on apenas para eventos onde latência importa — evitando pagar por infraestrutura contínua para dados que chegam com baixa frequência.



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

### Custo vs Performance

| Decisão | AS-IS (Implementado) | TO-BE (Visão Profissional) | Impacto |
|---|---|---|---|
| **Formato de arquivo** | BigQuery colunar nativo | Parquet sobre GCS | AS-IS: sem custo de conversão · TO-BE: reduz I/O e armazenamento em até 80% vs CSV |
| **Particionamento** | Por ano na Bronze | Por UF e ano no GCS + Gold | Queries leem só o período/região necessária em ambos |
| **Clusterização** | Por `id_municipio` e `serie` | Por município na Gold | Reduz bytes escaneados em queries analíticas em ambos |
| **Storage histórico** | BigQuery free tier (10 GB grátis) | GCS Nearline ($0,01/GB) → Coldline ($0,004/GB) | AS-IS: grátis até 10 GB · TO-BE: até 80% mais barato que Standard para dados frios |
| **Processamento** | `ingest_bronze.py` + `transform_silver.py` Python | Dataflow serverless (Apache Beam) | AS-IS: sem custo · TO-BE: paga só pelo tempo de execução, sem cluster ocioso |
| **Orquestração** | GitHub Actions gratuito | GitHub Actions gratuito | Zero custo fixo em ambos vs Cloud Composer (~$300/mês) |
| **Consultas analíticas** | BigQuery on-demand (1 TB/mês grátis) | BigQuery on-demand | AS-IS: grátis no free tier · TO-BE: paga por dado escaneado, não por cluster ligado |
| **Ingestão streaming** | Load job micro-lote (gratuito) | Pub/Sub + Dataflow always-on | AS-IS: $0 · TO-BE: custo contínuo justificado por latência real de segundos |
| **API externa** | Não implementada | Cloud Run + API Gateway | TO-BE: Cloud Run escala a zero — custo zero em períodos ociosos |
| **Qualidade de dados** | `validate.py` Python (gratuito) | SQL assertions nativas BigQuery (gratuito) | Ambos sem custo adicional · TO-BE mais integrado ao ecossistema |
| **Custo total estimado** | **$0/mês** | **~$3–5/mês** | AS-IS ideal para projeto acadêmico · TO-BE justificado em produção corporativa |

**Decisão AS-IS: free tier primeiro, performance suficiente.** Para ~3,9M linhas de dados educacionais públicos, o BigQuery free tier entrega performance adequada com custo zero. `WRITE_TRUNCATE` garante idempotência sem acúmulo; `SELECT` explícito e clustering evitam escaneamento desnecessário.

**Decisão TO-BE: custo primeiro, performance onde importa.** GCS Nearline/Coldline para histórico frio; Dataflow serverless só executa quando necessário; particionamento e clusterização apenas na Gold, onde as queries analíticas exigem velocidade. GitHub Actions mantém orquestração gratuita em ambas as visões.

---

## 💰 FinOps — AS-IS vs TO-BE

### Estimativa de Custo Mensal

| Serviço | AS-IS (Implementado) | TO-BE (Visão Profissional) |
|---|---|---|
| **Google Cloud Storage (Standard)** | Não utilizado — BigQuery puro | ~$0,20 (10 GB Bronze ativo) |
| **Google Cloud Storage (Nearline)** | Não utilizado | ~$0,50 (50 GB histórico recente) |
| **Google Cloud Storage (Coldline)** | Não utilizado | ~$0,80 (200 GB histórico antigo) |
| **BigQuery (armazenamento)** | Grátis — ~5 GB no free tier (10 GB/mês) | ~$0,10 (5 GB Gold) |
| **BigQuery (queries)** | Grátis — ~50 GB escaneados no free tier (1 TB/mês) | ~$0,25 (1 TB grátis — mesmo consumo) |
| **BigQuery (load jobs)** | Grátis — ingestão batch e micro-lote streaming | Não utilizado — substituído pelo Dataflow |
| **Dataflow** | Não utilizado — scripts Python locais | ~$1,50 (10h de processamento batch/mês) |
| **Pub/Sub** | Grátis — ~5 GB/mês no free tier (10 GB/mês) | Grátis — mesmo consumo no free tier |
| **Cloud Run** | Não implementado | Grátis (1M requisições/mês no free tier) |
| **API Gateway (Apigee)** | Não implementado | Grátis até 2M chamadas/mês |
| **GitHub Actions** | Grátis — repositório público | Grátis — repositório público |
| **Cloud Monitoring** | Não utilizado — logging Python | Grátis (métricas básicas) |
| **Cloud Billing + Budget Alerts** | Não configurado formalmente | Grátis |
| **Total estimado** | **$0/mês** | **~$3,35/mês** |

> **AS-IS:** custo zero sustentado pelo free tier do GCP. Viável para o volume atual de dados educacionais públicos (~3,9M linhas de alunos). O BigQuery absorve armazenamento e queries dentro dos limites gratuitos.
>
> **TO-BE:** ~$3,35/mês considerando o free tier do GCP e volume típico de dados educacionais. Em produção com maior volume, o particionamento, o Coldline e o Dataflow sob demanda garantem escala sem crescimento linear de custo.

---

### Práticas adotadas

| Prática | AS-IS | TO-BE |
|---|---|---|
| **Formato de armazenamento** | BigQuery colunar nativo — evita custo de conversão | Parquet em todas as camadas — reduz I/O e armazenamento em até 80% vs CSV |
| **Ciclo de vida de dados** | BigQuery gerencia internamente (time travel 7 dias) | Lifecycle Policy no GCS move dados para Nearline (30d) e Coldline (90d) automaticamente |
| **Particionamento** | Por ano na Bronze · por id_municipio/serie na Silver | Por UF e ano no GCS · por data e município na Gold |
| **Clusterização** | Por id_municipio e serie nas tabelas grandes | Por município na Gold — queries leem só o subconjunto necessário |
| **Processamento** | Scripts Python locais — custo zero | Dataflow sob demanda — jobs existem só durante execução, sem cluster ocioso |
| **Queries** | SELECT explícito em todas as queries — nunca `SELECT *` | SELECT explícito + partição/clustering — custo mínimo por query |
| **Streaming** | Load job em micro-lote — evita insertAll pago | Pub/Sub + Dataflow — custo dentro do free tier (10 GB/mês) |
| **API externa** | Não implementada | Cloud Run escala a zero — custo zero em períodos sem requisições |
| **Orquestração** | GitHub Actions gratuito | GitHub Actions gratuito — elimina ~$300/mês do Cloud Composer |
| **Alertas de custo** | Não configurado | Budget Alerts em 50%, 80% e 100% do orçamento mensal |
| **Idempotência** | WRITE_TRUNCATE — evita reprocessamentos e duplicações custosas | WRITE_TRUNCATE + Iceberg ACID — garante consistência sem reprocessamento total |

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

## Como Executar Localmente a Arquitetura AS IS

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



### Práticas adotadas

- **Parquet** em todas as camadas — reduz I/O e armazenamento em até 80% vs CSV
- **Lifecycle Policy no GCS** — move dados automaticamente para Nearline (30 dias) e Coldline (90 dias) sem intervenção manual
- **Particionamento e clusterização no BigQuery** — queries leem apenas o subconjunto necessário, reduzindo custo por query
- **Dataflow sob demanda** — jobs existem apenas durante a execução, sem cluster ocioso
- **Cloud Run** — escala a zero fora de uso, custo zero em períodos sem requisições
- **GitHub Actions** — orquestração gratuita, eliminando ~$300/mês do Cloud Composer
- **Budget Alerts** — alertas em 50%, 80% e 100% do orçamento mensal definido

---

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
