# Catálogo de Dados - MVP Engenharia de Dados PUC-Rio

## 📋 Informações Gerais

**Projeto**: Pipeline de Dados - Análise Demográfica e Geográfica Mundial\
**Plataforma**: Databricks + Delta Lake\
**Arquitetura**: Medalhão (Bronze → Silver → Gold)\

---

## 🗺️ Visão Geral da Arquitetura

```
APIs REST
    ↓
┌─────────────────────────────────────────────┐
│ BRONZE (workspace.bronze)                   │
│ • countries_raw (585 registros)             │
│ • exchange_rates_raw (3 registros)          │
└─────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────┐
│ SILVER (workspace.silver)                   │
│ • dim_countries (195 registros)             │
│ • dim_currencies (146 registros)            │
│ • dim_languages (140 registros)             │
│ • fact_country_metrics (585 registros)      │
└─────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────┐
│ GOLD (workspace.gold)                       │
│ • countries_by_region (24 registros)        │
│ • currency_usage (146 registros)            │
│ • language_distribution (141 registros)     │
│ • geographic_metrics (8 registros)          │
└─────────────────────────────────────────────┘
```

---

## 🥉 Camada BRONZE - Dados Brutos

### workspace.bronze.countries_raw

**Descrição**: Dados brutos em formato JSON da REST Countries API. Armazena snapshot completo de 195 países independentes por execução.

**Características**:
- **Tipo**: Delta Lake (Append-only)
- **Particionamento**: execution_date
- **Frequência**: Diária
- **Retenção**: 90 dias
- **Grain**: 1 registro por país por execução

#### Schema

| Campo | Tipo | Nullable | Descrição | Domínio |
|-------|------|----------|-----------|---------|
| data | STRING | NOT NULL | JSON bruto completo da API | JSON válido contendo objeto de país |
| ingestion_timestamp | TIMESTAMP | NOT NULL | Timestamp exato da ingestão | >= 2025-12-05 |
| data_source | STRING | NOT NULL | Identificador da fonte | Valor fixo: 'rest_countries_api' |
| execution_date | DATE | NOT NULL | Data de execução do pipeline (PARTITION KEY) | YYYY-MM-DD |

#### Exemplo de Registro

```json
{
  "data": "{\"name\":{\"common\":\"Brazil\",\"official\":\"Federative Republic of Brazil\"},\"cca3\":\"BRA\",\"population\":213993437,\"area\":8515767,...}",
  "ingestion_timestamp": "2025-12-05T15:51:26.501Z",
  "data_source": "rest_countries_api",
  "execution_date": "2025-12-05"
}
```

#### Linhagem

```
[REST Countries API]
  https://restcountries.com/v3.1/independent?status=true
    ↓ HTTP GET (requests.get)
    ↓ JSON serialization (json.dumps)
    ↓ Metadata addition (timestamp, source, date)
    ↓
[workspace.bronze.countries_raw]
  Modo: APPEND
  195 países por execução
```

---

### workspace.bronze.exchange_rates_raw

**Descrição**: Dados brutos de taxas de câmbio em relação ao USD.

**Características**:
- **Tipo**: Delta Lake (Append-only)
- **Particionamento**: execution_date
- **Frequência**: Diária
- **Grain**: 1 registro por execução

#### Schema

| Campo | Tipo | Nullable | Descrição | Domínio |
|-------|------|----------|-----------|---------|
| data | STRING | NOT NULL | JSON contendo taxas de câmbio | JSON: {base, date, rates} |
| ingestion_timestamp | TIMESTAMP | NOT NULL | Timestamp da ingestão | >= 2025-12-05 |
| data_source | STRING | NOT NULL | Identificador da fonte | Valor: 'exchange_rate_api' |
| execution_date | DATE | NOT NULL | Data de execução | YYYY-MM-DD |

---

## 🥈 Camada SILVER - Dados Curados

### workspace.silver.dim_countries

**Descrição**: Dimensão de países com atributos geográficos e administrativos normalizados. Implementa SCD Type 1 (snapshot atual).

**Características**:
- **Tipo**: Delta Lake (Overwrite)
- **Grain**: 1 país único (country_id)
- **Chave Primária**: country_id
- **Frequência**: Diária (substituição completa)
- **Registros**: 195

#### Schema

| Campo | Tipo | Nullable | PK/FK | Descrição | Domínio/Exemplos |
|-------|------|----------|-------|-----------|------------------|
| country_id | STRING | NOT NULL | PK | Código ISO Alpha-3 | BRA, USA, CHN, IND, FRA (195 valores únicos) |
| country_code_2 | STRING | YES | - | Código ISO Alpha-2 | BR, US, CN, IN, FR |
| country_name_common | STRING | NOT NULL | - | Nome comum em inglês | Brazil, United States, China, India |
| country_name_official | STRING | YES | - | Nome oficial completo | Federative Republic of Brazil |
| capital | STRING | YES | - | Capital (primeira se múltiplas) | Brasília, Washington, Beijing, New Delhi |
| region | STRING | YES | - | Região geográfica | {Africa, Americas, Asia, Europe, Oceania} (5 valores) |
| subregion | STRING | YES | - | Sub-região específica | South America, Western Europe, Eastern Asia (24 valores) |
| landlocked | BOOLEAN | YES | - | País sem acesso ao mar | 45 True (23.1%), 150 False (76.9%) |
| latitude | DOUBLE | YES | - | Latitude do centro | [-90.0, 90.0] |
| longitude | DOUBLE | YES | - | Longitude do centro | [-180.0, 180.0] |
| timezones | ARRAY<STRING> | YES | - | Fusos horários | ["UTC-03:00", "UTC-04:00"] |
| last_updated | TIMESTAMP | NOT NULL | - | Última atualização | Current timestamp |

#### Linhagem

```
[workspace.bronze.countries_raw]
  Campo: data (JSON string)
    ↓
[PySpark Transformation]
  1. from_json(data, country_schema)
  2. Flatten nested structures:
     - parsed_data.cca3 → country_id
     - parsed_data.name.common → country_name_common
     - parsed_data.capital[0] → capital
     - parsed_data.latlng[0,1] → latitude, longitude
  3. Validações:
     - filter(country_id IS NOT NULL)
     - dropDuplicates(['country_id'])
    ↓
[workspace.silver.dim_countries]
  Modo: OVERWRITE (SCD Type 1)
  195 registros
```

#### Regras de Qualidade

- ✅ country_id obrigatório e único
- ✅ country_name_common obrigatório
- ✅ Latitude ∈ [-90, 90]
- ✅ Longitude ∈ [-180, 180]
- ✅ 0 duplicatas
- ✅ 0 nulos em chaves

---

### workspace.silver.dim_currencies

**Descrição**: Dimensão de moedas extraída dos dados de países.

**Características**:
- **Tipo**: Delta Lake (Overwrite)
- **Grain**: 1 moeda única (currency_code)
- **Chave Primária**: currency_code
- **Registros**: 146

#### Schema

| Campo | Tipo | Nullable | PK/FK | Descrição | Domínio/Exemplos |
|-------|------|----------|-------|-----------|------------------|
| currency_code | STRING | NOT NULL | PK | Código ISO 4217 | USD, EUR, BRL, GBP, JPY (146 valores) |
| currency_name | STRING | YES | - | Nome completo em inglês | United States dollar, Euro, Brazilian real |
| currency_symbol | STRING | YES | - | Símbolo da moeda | $, €, R$, £, ¥ |

#### Linhagem

```
[workspace.bronze.countries_raw]
  Campo: data.currencies (map<string,object>)
    ↓
[PySpark Transformation]
  1. from_json(data, country_schema)
  2. explode(parsed_data.currencies)
     Map{USD: {name, symbol}} → Rows
  3. Select:
     - currency_code → currency_code
     - currency_data.name → currency_name
     - currency_data.symbol → currency_symbol
  4. dropDuplicates(['currency_code'])
    ↓
[workspace.silver.dim_currencies]
  Modo: OVERWRITE
  146 registros únicos
```

#### Relacionamentos

- **N:M com dim_countries**: 1 moeda em N países (EUR em 20 países da Zona do Euro)
- **1:N com gold.currency_usage**: Agregação por moeda

---

### workspace.silver.dim_languages

**Descrição**: Dimensão de idiomas extraída dos dados de países.

**Características**:
- **Tipo**: Delta Lake (Overwrite)
- **Grain**: 1 idioma único (language_code)
- **Chave Primária**: language_code
- **Registros**: 140

#### Schema

| Campo | Tipo | Nullable | PK/FK | Descrição | Domínio/Exemplos |
|-------|------|----------|-------|-----------|------------------|
| language_code | STRING | NOT NULL | PK | Código ISO 639 | eng, spa, por, fra, ara (140 valores) |
| language_name | STRING | YES | - | Nome em inglês | English, Spanish, Portuguese, French, Arabic |

#### Linhagem

```
[workspace.bronze.countries_raw]
  Campo: data.languages (map<string,string>)
    ↓
[PySpark Transformation]
  1. from_json(data, country_schema)
  2. explode(parsed_data.languages)
     Map{por: Portuguese} → Rows
  3. dropDuplicates(['language_code'])
    ↓
[workspace.silver.dim_languages]
  Modo: OVERWRITE
  140 registros únicos
```

#### Insight

- **English (eng)**: ~67 países (maior alcance global)
- **French (fra)**: ~51 países
- **Arabic (ara)**: ~25 países

---

### workspace.silver.fact_country_metrics

**Descrição**: Tabela fato contendo métricas populacionais e geográficas quantitativas.

**Características**:
- **Tipo**: Delta Lake (Overwrite)
- **Grain**: 1 país por snapshot
- **Chave Estrangeira**: country_id → dim_countries.country_id
- **Registros**: 195 (1 execução única após deduplicação)

#### Schema

| Campo | Tipo | Nullable | PK/FK | Descrição | Domínio |
|-------|------|----------|-------|-----------|---------|
| country_id | STRING | NOT NULL | FK | Código do país | 100% existem em dim_countries |
| population | LONG | YES | - | População total | [882, 1.417.492.000] (Vaticano → Índia) |
| area_km2 | DOUBLE | YES | - | Área territorial em km² | [0.49, 17.098.246] (Monaco → Rússia) |
| population_density | DOUBLE | YES | - | Densidade (hab/km²) **CALCULADO** | [2.27, 19.021.29] (Mongólia → Monaco) |
| snapshot_date | DATE | NOT NULL | - | Data do snapshot | 2025-12-05 |

#### Métricas Calculadas

```sql
population_density = CASE
  WHEN area_km2 > 0 THEN population / area_km2
  ELSE 0
END
```

#### Linhagem

```
[workspace.bronze.countries_raw]
  Campos: data.population, data.area
    ↓
[PySpark Transformation]
  1. from_json(data, country_schema)
  2. Select:
     - parsed_data.cca3 → country_id
     - parsed_data.population → population
     - parsed_data.area → area_km2
     - current_date() → snapshot_date
  3. Cálculo derivado:
     - population_density = population / area_km2 (when area > 0)
  4. Validações:
     - filter(population >= 0)
     - filter(area_km2 >= 0)
    ↓
[workspace.silver.fact_country_metrics]
  Modo: OVERWRITE
  195 registros por execução
```

#### Regras de Qualidade

- ✅ Integridade referencial: 100% (0 country_id órfãos)
- ✅ population >= 0 (0 inválidos)
- ✅ area_km2 >= 0 (0 inválidos)
- ✅ population_density corretamente calculado
- ✅ Completude: 100%

---

## 🥇 Camada GOLD - Métricas Analíticas

### workspace.gold.countries_by_region

**Descrição**: Agregações por região e sub-região com estatísticas populacionais e geográficas.

**Características**:
- **Tipo**: Delta Lake (Overwrite)
- **Grain**: 1 registro por (region, subregion)
- **Registros**: 24

#### Schema

| Campo | Tipo | Nullable | Descrição | Domínio |
|-------|------|----------|-----------|---------|
| region | STRING | YES | Região geográfica | {Africa, Americas, Asia, Europe, Oceania} |
| subregion | STRING | YES | Sub-região específica | 24 valores |
| total_countries | LONG | NOT NULL | Total de países | [1, 48] |
| total_population | LONG | YES | População total somada | Máx: Southern Asia (6.0B triplicado) |
| total_area_km2 | DOUBLE | YES | Área total somada | Em km² |
| avg_population | DOUBLE | YES | População média por país | Média calculada |
| avg_area_km2 | DOUBLE | YES | Área média por país | Em km² |
| avg_population_density | DOUBLE | YES | Densidade média | Southern Asia: 475.25, SE Asia: 909.47 hab/km² |
| landlocked_count | LONG | NOT NULL | Países sem costa | [0, 18] |
| max_population | LONG | YES | Maior população da sub-região | Ex: Índia (1.417B) |
| min_population | LONG | YES | Menor população | Valores mínimos |
| calculation_date | DATE | NOT NULL | Data do cálculo | 2025-12-05 |

#### Top 3 Sub-regiões por Densidade

1. **South-Eastern Asia**: 909.47 hab/km² (Singapura, Filipinas, Indonésia)
2. **Southern Asia**: 475.25 hab/km² (Índia, Bangladesh, Paquistão)
3. **Western Asia**: 288.35 hab/km² (Israel, Líbano, Bahrein)

---

### workspace.gold.currency_usage

**Descrição**: Ranking de moedas por alcance global.

**Características**:
- **Tipo**: Delta Lake (Overwrite)
- **Grain**: 1 moeda única
- **Registros**: 146

#### Schema

| Campo | Tipo | Nullable | Descrição | Domínio |
|-------|------|----------|-----------|---------|
| currency_code | STRING | NOT NULL | Código ISO 4217 | USD, EUR, BRL, GBP, JPY |
| currency_name | STRING | YES | Nome completo | United States dollar, Euro |
| countries_using | ARRAY<STRING> | YES | Lista de países que usam | ["USA", "Ecuador", "El Salvador"] |
| total_countries | LONG | NOT NULL | Quantidade de países | [1, 20] |

---

### workspace.gold.language_distribution

**Descrição**: Ranking de idiomas por alcance global.

**Características**:
- **Tipo**: Delta Lake (Overwrite)
- **Grain**: 1 idioma único
- **Registros**: 141

#### Schema

| Campo | Tipo | Nullable | Descrição | Domínio |
|-------|------|----------|-----------|---------|
| language_code | STRING | NOT NULL | Código ISO 639 | eng, fra, spa, ara |
| language_name | STRING | YES | Nome em inglês | English, French, Spanish, Arabic |
| countries_speaking | ARRAY<STRING> | YES | Lista de países | ["USA", "UK", "Canada"] |
| total_countries | LONG | NOT NULL | Quantidade de países | [1, 67] |

---

### workspace.gold.geographic_metrics

**Descrição**: KPIs globais agregados.

**Características**:
- **Tipo**: Delta Lake (Particionado por calculation_date)
- **Grain**: 1 registro por métrica por data
- **Registros**: 9 (inclui avg_country_density e world_population_density)

#### Schema

| Campo | Tipo | Nullable | Descrição | Domínio |
|-------|------|----------|-----------|---------|
| metric_name | STRING | NOT NULL | Nome da métrica | total_countries, world_population, etc. |
| metric_value | DOUBLE | NOT NULL | Valor numérico | Varia por métrica |
| calculation_date | DATE | NOT NULL | Data do cálculo | 2025-12-05 |

#### Métricas Disponíveis

| metric_name | Valor (195 registros) | Descrição |
|-------------|----------------------|-----------|
| total_countries | 195.0 | Total de países independentes |
| world_population | ~7.974.764.646 | População mundial real |
| world_land_area_km2 | ~133.568.019 | Área terrestre total (km²) |
| avg_country_density | 311.54 | Densidade média por país (média aritmética) |
| world_population_density | 59.70 | Densidade global real (população/área total) |
| landlocked_countries | 45.0 | Países sem acesso ao mar |
| total_regions | 5.0 | Número de regiões (Africa, Americas, Asia, Europe, Oceania) |
| largest_country_area_km2 | 17.098.246 | Rússia (maior país por área territorial) |
| largest_country_population | 1.417.492.000 | Índia (país mais populoso) |

---

## 📊 Resumo Estatístico

### Registros por Camada

- **Bronze**: 196 registros (195 countries + 1 exchange rate)
- **Silver**: 676 registros (195 + 146 + 140 + 195)
- **Gold**: 320 registros (24 + 146 + 141 + 9)
- **Total**: 1.192 registros

### Qualidade de Dados

- **Completude**: 100% em campos críticos
- **Integridade Referencial**: 100% (0 órfãos)
- **Unicidade**: 100% em chaves primárias
- **Validação de Ranges**: 100% (0 valores fora de faixa)

---

## 🔄 Resumo de Linhagem Completa

```
[REST Countries API]
  195 países (JSON)
    ↓
[workspace.bronze.countries_raw]
  195 registros (append mode, 1 execução única)
    ↓
  ├─→ [workspace.silver.dim_countries] (195)
  ├─→ [workspace.silver.dim_currencies] (146)
  ├─→ [workspace.silver.dim_languages] (140)
  └─→ [workspace.silver.fact_country_metrics] (195)
        ↓
        ├─→ [workspace.gold.countries_by_region] (24)
        ├─→ [workspace.gold.currency_usage] (146)
        ├─→ [workspace.gold.language_distribution] (141)
        └─→ [workspace.gold.geographic_metrics] (9)

[Exchange Rate API]
  Taxas de câmbio (JSON)
    ↓
[workspace.bronze.exchange_rates_raw]
  3 registros (append mode)
    └─→ [Uso futuro para enriquecimento]
```

---

## 📝 Convenções e Padrões

### Nomenclatura

- **Schemas**: workspace.{layer} (bronze | silver | gold)
- **Tabelas**: snake_case, prefixos dim_ / fact_
- **Campos**: snake_case descritivo

### Tipos de Dados

- **Identificadores**: STRING (preserva zeros à esquerda)
- **Contadores**: LONG (suporta valores grandes)
- **Medidas**: DOUBLE (precisão em cálculos)
- **Datas**: DATE (sem timezone)
- **Timestamps**: TIMESTAMP (milissegundos)
- **Flags**: BOOLEAN
- **Listas**: ARRAY<type>
- **Maps**: MapType<key, value>
