# Documentação do Pipeline de Dados - Unity Catalog

## 📋 Visão Geral do Projeto

### Objetivo
Pipeline de engenharia de dados para ingestão, transformação e análise de dados geográficos e econômicos de países, implementando arquitetura Medallion (Bronze → Silver → Gold).

### Fonte de Dados
- **REST Countries API**: Dados geográficos, populacionais e culturais de 195 países
- **Exchange Rate API**: Taxas de câmbio em tempo real

### Arquitetura
- **Camada Bronze**: Dados brutos (raw data)
- **Camada Silver**: Dados limpos e normalizados (modelagem dimensional)
- **Camada Gold**: Agregações e métricas analíticas

---

## 🗄️ Schemas e Estrutura

### workspace.bronze
Camada de dados brutos com informações originais das APIs

### workspace.silver
Camada de dados transformados com modelagem dimensional

### workspace.gold
Camada de dados agregados e métricas de negócio

---

## 📊 Catálogo de Tabelas

---

## BRONZE LAYER

### 📦 workspace.bronze.countries_raw

**Descrição**: Dados brutos da REST Countries API contendo informações completas sobre países independentes.

**Tipo**: Tabela Delta (Append-only)

**Particionamento**: `execution_date`

**Schema**:
| Coluna | Tipo | Nullable | Descrição |
|--------|------|----------|-----------|
| data | STRING | NOT NULL | JSON bruto contendo todos os atributos do país |
| ingestion_timestamp | TIMESTAMP | NOT NULL | Data e hora da ingestão dos dados |
| data_source | STRING | NOT NULL | Identificador da fonte (rest_countries_api) |
| execution_date | DATE | NOT NULL | Data de execução do pipeline (partição) |

**Dados Armazenados no JSON**:
- Códigos ISO (cca2, cca3, ccn3)
- Nome comum e oficial
- Capital, região, sub-região
- População, área, densidade
- Moedas, idiomas
- Timezones, coordenadas geográficas
- Flags, fronteiras

**Frequência de Atualização**: Diária

**Retenção**: 90 dias de histórico

**Volume Estimado**: ~195 registros/dia, ~5 KB/registro

**Exemplo de Uso**:
```sql
SELECT
    data_source,
    execution_date,
    COUNT(*) as total_records
FROM workspace.bronze.countries_raw
GROUP BY data_source, execution_date
ORDER BY execution_date DESC
LIMIT 7;
```

---

### 📦 workspace.bronze.exchange_rates_raw

**Descrição**: Dados brutos da Exchange Rate API com taxas de câmbio diárias.

**Tipo**: Tabela Delta (Append-only)

**Particionamento**: `execution_date`

**Schema**:
| Coluna | Tipo | Nullable | Descrição |
|--------|------|----------|-----------|
| data | STRING | NOT NULL | JSON bruto com taxas de câmbio (base: USD) |
| ingestion_timestamp | TIMESTAMP | NOT NULL | Data e hora da ingestão |
| data_source | STRING | NOT NULL | Identificador da fonte (exchange_rate_api) |
| execution_date | DATE | NOT NULL | Data de execução do pipeline (partição) |

**Frequência de Atualização**: Diária

**Retenção**: 365 dias de histórico

**Volume Estimado**: 1 registro/dia, ~20 KB/registro

**Exemplo de Uso**:
```sql
SELECT
    execution_date,
    ingestion_timestamp,
    SUBSTRING(data, 1, 200) as sample_data
FROM workspace.bronze.exchange_rates_raw
ORDER BY execution_date DESC
LIMIT 1;
```

---

## SILVER LAYER

### 🌍 workspace.silver.dim_countries

**Descrição**: Dimensão de países com atributos geográficos e administrativos normalizados.

**Tipo**: Tabela Delta (SCD Type 1 - Overwrite)

**Grain**: Um registro por país (identificado por country_id)

**Primary Key**: `country_id` (código ISO Alpha-3)

**Schema**:
| Coluna | Tipo | Nullable | Descrição |
|--------|------|----------|-----------|
| country_id | STRING | NOT NULL | Código ISO Alpha-3 (PK) |
| country_code_2 | STRING | YES | Código ISO Alpha-2 |
| country_name_common | STRING | NOT NULL | Nome comum do país |
| country_name_official | STRING | YES | Nome oficial do país |
| capital | STRING | YES | Capital do país |
| region | STRING | YES | Região (Americas, Europe, Asia, Africa, Oceania) |
| subregion | STRING | YES | Sub-região específica |
| landlocked | BOOLEAN | YES | País sem costa marítima (true/false) |
| latitude | DOUBLE | YES | Coordenada geográfica - Latitude |
| longitude | DOUBLE | YES | Coordenada geográfica - Longitude |
| timezones | ARRAY<STRING> | YES | Lista de fusos horários |
| last_updated | TIMESTAMP | NOT NULL | Data da última atualização |

**Qualidade de Dados**:
- Validação: country_id e country_name_common são obrigatórios
- Deduplicação: Por country_id
- Registros esperados: ~195

**Relacionamentos**:
- 1:N com `fact_country_metrics` (via country_id)
- N:M com `dim_currencies` (via tabelas de países)
- N:M com `dim_languages` (via tabelas de países)

**Exemplo de Uso**:
```sql
SELECT
    country_name_common,
    capital,
    region,
    subregion,
    landlocked
FROM workspace.silver.dim_countries
WHERE region = 'Americas'
ORDER BY country_name_common;
```

---

### 💰 workspace.silver.dim_currencies

**Descrição**: Dimensão de moedas extraída dos dados de países.

**Tipo**: Tabela Delta (Overwrite)

**Grain**: Uma moeda única por código

**Primary Key**: `currency_code`

**Schema**:
| Coluna | Tipo | Nullable | Descrição |
|--------|------|----------|-----------|
| currency_code | STRING | NOT NULL | Código da moeda (ex: USD, EUR, BRL) (PK) |
| currency_name | STRING | YES | Nome completo da moeda |
| currency_symbol | STRING | YES | Símbolo da moeda (ex: $, €, R$) |

**Qualidade de Dados**:
- Validação: currency_code não pode ser nulo
- Deduplicação: Por currency_code
- Registros esperados: 146

**Exemplo de Uso**:
```sql
SELECT
    currency_code,
    currency_name,
    currency_symbol
FROM workspace.silver.dim_currencies
WHERE currency_code IN ('USD', 'EUR', 'BRL', 'GBP')
ORDER BY currency_code;
```

---

### 🗣️ workspace.silver.dim_languages

**Descrição**: Dimensão de idiomas extraída dos dados de países.

**Tipo**: Tabela Delta (Overwrite)

**Grain**: Um idioma único por código

**Primary Key**: `language_code`

**Schema**:
| Coluna | Tipo | Nullable | Descrição |
|--------|------|----------|-----------|
| language_code | STRING | NOT NULL | Código do idioma (ex: eng, spa, por) (PK) |
| language_name | STRING | YES | Nome do idioma (ex: English, Spanish, Portuguese) |

**Qualidade de Dados**:
- Validação: language_code não pode ser nulo
- Deduplicação: Por language_code
- Registros esperados: 141

**Exemplo de Uso**:
```sql
SELECT
    language_code,
    language_name
FROM workspace.silver.dim_languages
WHERE language_name LIKE '%English%'
ORDER BY language_name;
```

---

### 📈 workspace.silver.fact_country_metrics

**Descrição**: Tabela fato com métricas populacionais e geográficas dos países.

**Tipo**: Tabela Delta (Overwrite)

**Grain**: Uma linha por país por snapshot

**Foreign Keys**: `country_id` → `dim_countries.country_id`

**Schema**:
| Coluna | Tipo | Nullable | Descrição |
|--------|------|----------|-----------|
| country_id | STRING | NOT NULL | Código do país (FK) |
| population | LONG | YES | População total do país |
| area_km2 | DOUBLE | YES | Área territorial em km² |
| population_density | DOUBLE | YES | Densidade populacional (hab/km²) |
| snapshot_date | DATE | NOT NULL | Data do snapshot dos dados |

**Métricas Calculadas**:
- `population_density = population / area_km2` (quando area_km2 > 0)

**Qualidade de Dados**:
- Validação: population >= 0, area_km2 >= 0
- Integridade referencial: Todos country_id existem em dim_countries

**Exemplo de Uso**:
```sql
SELECT
    c.country_name_common,
    f.population,
    f.area_km2,
    ROUND(f.population_density, 2) as density
FROM workspace.silver.fact_country_metrics f
JOIN workspace.silver.dim_countries c ON f.country_id = c.country_id
ORDER BY f.population DESC
LIMIT 10;
```

---

## GOLD LAYER

### 🌎 workspace.gold.countries_by_region

**Descrição**: Agregações de países por região e sub-região com métricas populacionais e geográficas.

**Tipo**: Tabela Delta (Overwrite)

**Grain**: Uma linha por combinação região/sub-região

**Schema**:
| Coluna | Tipo | Nullable | Descrição |
|--------|------|----------|-----------|
| region | STRING | YES | Região geográfica principal |
| subregion | STRING | YES | Sub-região específica |
| total_countries | LONG | NOT NULL | Total de países na região |
| total_population | LONG | YES | População total da região |
| total_area_km2 | DOUBLE | YES | Área total da região em km² |
| avg_population | DOUBLE | YES | População média por país |
| avg_area_km2 | DOUBLE | YES | Área média por país |
| avg_population_density | DOUBLE | YES | Densidade populacional média |
| landlocked_count | LONG | NOT NULL | Países sem costa na região |
| max_population | LONG | YES | Maior população da região |
| min_population | LONG | YES | Menor população da região |
| calculation_date | DATE | NOT NULL | Data do cálculo |

**Casos de Uso**:
- Análises comparativas entre regiões
- Dashboards executivos
- Estudos demográficos regionais

**Exemplo de Uso**:
```sql
SELECT
    region,
    total_countries,
    FORMAT_NUMBER(total_population, 0) as population,
    ROUND(avg_population_density, 2) as avg_density
FROM workspace.gold.countries_by_region
ORDER BY total_population DESC;
```

---

### 💵 workspace.gold.currency_usage

**Descrição**: Análise de distribuição e uso de moedas pelos países.

**Tipo**: Tabela Delta (Overwrite)

**Grain**: Uma linha por moeda

**Schema**:
| Coluna | Tipo | Nullable | Descrição |
|--------|------|----------|-----------|
| currency_code | STRING | NOT NULL | Código da moeda |
| currency_name | STRING | YES | Nome da moeda |
| countries_using | ARRAY<STRING> | YES | Lista de países que usam a moeda |
| total_countries | LONG | NOT NULL | Quantidade de países |

**Casos de Uso**:
- Análise de moedas mais utilizadas
- Estudos de zonas monetárias
- Análises econômicas regionais

**Exemplo de Uso**:
```sql
SELECT
    currency_code,
    currency_name,
    total_countries,
    SIZE(countries_using) as country_count
FROM workspace.gold.currency_usage
ORDER BY total_countries DESC
LIMIT 10;
```

---

### 🗨️ workspace.gold.language_distribution

**Descrição**: Distribuição de idiomas falados pelos países.

**Tipo**: Tabela Delta (Overwrite)

**Grain**: Uma linha por idioma

**Schema**:
| Coluna | Tipo | Nullable | Descrição |
|--------|------|----------|-----------|
| language_code | STRING | NOT NULL | Código do idioma |
| language_name | STRING | YES | Nome do idioma |
| countries_speaking | ARRAY<STRING> | YES | Lista de países que falam o idioma |
| total_countries | LONG | NOT NULL | Quantidade de países |

**Casos de Uso**:
- Análise de idiomas mais falados
- Planejamento de internacionalização
- Estudos linguísticos globais

**Exemplo de Uso**:
```sql
SELECT
    language_name,
    total_countries,
    countries_speaking
FROM workspace.gold.language_distribution
ORDER BY total_countries DESC
LIMIT 15;
```

---

### 📊 workspace.gold.geographic_metrics

**Descrição**: KPIs e métricas globais agregadas do dataset.

**Tipo**: Tabela Delta (Overwrite)

**Grain**: Uma linha por métrica

**Schema**:
| Coluna | Tipo | Nullable | Descrição |
|--------|------|----------|-----------|
| metric_name | STRING | NOT NULL | Nome da métrica |
| metric_value | DOUBLE | NOT NULL | Valor da métrica |
| calculation_date | DATE | NOT NULL | Data do cálculo |

**Métricas Disponíveis** (9 métricas):
- `total_countries`: Total de países no dataset (195)
- `world_population`: População mundial total (~7,97B)
- `world_land_area_km2`: Área terrestre total (~133M km²)
- `avg_country_density`: Densidade média por país (311.54 hab/km²)
- `world_population_density`: Densidade global real (59.70 hab/km²)
- `landlocked_countries`: Total de países sem costa (45)
- `total_regions`: Número de regiões geográficas (5)
- `largest_country_area_km2`: Maior país por área (Rússia - 17.098.246 km²)
- `largest_country_population`: Maior país por população (Índia - 1.417.492.000)

**Casos de Uso**:
- Dashboards executivos
- KPIs globais
- Métricas de qualidade de dados

**Exemplo de Uso**:
```sql
SELECT
    metric_name,
    FORMAT_NUMBER(metric_value, 0) as value,
    calculation_date
FROM workspace.gold.geographic_metrics
ORDER BY metric_name;
```

---

## 🔄 Fluxo de Execução

### Pipeline Completo

1. **01_pipeline_bronze.ipynb**
   - Ingestão das APIs
   - Validação de conectividade
   - Salvamento de dados brutos
   - ⏱️ Tempo estimado: 2-3 minutos

2. **02_pipeline_silver.ipynb**
   - Parse e normalização de JSON
   - Criação de dimensões e fatos
   - Validações de qualidade
   - ⏱️ Tempo estimado: 3-5 minutos

3. **03_pipeline_gold.ipynb**
   - Agregações e KPIs
   - Análises Top N
   - Métricas globais
   - ⏱️ Tempo estimado: 2-4 minutos

**Tempo Total**: ~10 minutos

---

## 📅 Agendamento Recomendado

### Databricks Jobs

```yaml
Job Name: daily_countries_pipeline
Schedule: 0 2 * * * (Diariamente às 2h AM)
Cluster: Shared cluster (Runtime 4.0.0+)
Notebooks:
  - Task 1: 01_pipeline_bronze.ipynb
  - Task 2: 02_pipeline_silver.ipynb (depends_on: Task 1)
  - Task 3: 03_pipeline_gold.ipynb (depends_on: Task 2)
Timeout: 30 minutos
Retries: 2
Email Alerts: On failure
```

---

## 🎯 Casos de Uso e Análises

### Análises de Negócio

1. **Análise Demográfica Global**
   ```sql
   SELECT
       region,
       SUM(total_population) as population,
       SUM(total_area_km2) as area,
       AVG(avg_population_density) as density
   FROM workspace.gold.countries_by_region
   GROUP BY region;
   ```

2. **Top 10 Países Mais Populosos**
   ```sql
   SELECT
       c.country_name_common,
       f.population,
       c.capital,
       c.region
   FROM workspace.silver.fact_country_metrics f
   JOIN workspace.silver.dim_countries c ON f.country_id = c.country_id
   ORDER BY f.population DESC
   LIMIT 10;
   ```

3. **Moedas Mais Utilizadas**
   ```sql
   SELECT
       currency_name,
       total_countries,
       countries_using
   FROM workspace.gold.currency_usage
   ORDER BY total_countries DESC
   LIMIT 5;
   ```

4. **Densidade Populacional por Região**
   ```sql
   SELECT
       region,
       ROUND(AVG(avg_population_density), 2) as avg_density,
       total_countries
   FROM workspace.gold.countries_by_region
   GROUP BY region, total_countries
   ORDER BY avg_density DESC;
   ```

---

## 🔍 Qualidade de Dados

### Validações Implementadas

**Bronze Layer**:
- ✅ Verificação de status HTTP 200
- ✅ Validação de formato JSON
- ✅ Timestamp de ingestão

**Silver Layer**:
- ✅ Campos obrigatórios (country_id, country_name)
- ✅ Validação de ranges (population >= 0, area >= 0)
- ✅ Deduplicação por primary key
- ✅ Integridade referencial (FK constraints)
- ✅ Cálculos derivados (population_density)

**Gold Layer**:
- ✅ Consistência de agregações
- ✅ Validação de totais
- ✅ Data de cálculo sempre atualizada

### Monitoramento

**Métricas a Acompanhar**:
- Contagem de registros por camada
- Tempo de execução de cada pipeline
- Taxa de falha nas APIs
- Variação de população entre execuções
- Registros órfãos (falha de integridade referencial)

---

## 📝 Metadados e Tags

### Tags Recomendadas no Unity Catalog

**Bronze Tables**:
- `data_classification`: raw
- `pii`: false
- `retention_days`: 90
- `source`: rest_countries_api, exchange_rate_api

**Silver Tables**:
- `data_classification`: curated
- `model`: star_schema
- `pii`: false
- `business_critical`: true

**Gold Tables**:
- `data_classification`: aggregated
- `usage`: analytics, bi_reporting
- `update_frequency`: daily
- `business_critical`: true

---

## 👥 Contatos e Suporte

**Engenheiro de Dados**: Rafael França
**Email**: rafaelmoreno007@gmail.com
**Última Atualização**: 2025-12-05

---

## 📚 Referências

- [REST Countries API Documentation](https://restcountries.com)
- [Exchange Rate API](https://exchangerate-api.com)
- [Databricks Delta Lake Best Practices](https://docs.databricks.com/delta/)
- [Medallion Architecture](https://www.databricks.com/glossary/medallion-architecture)
