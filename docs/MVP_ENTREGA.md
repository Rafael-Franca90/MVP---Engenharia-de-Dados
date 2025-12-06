# MVP - Engenharia de Dados PUC-Rio
**Pipeline de Dados na Nuvem: Análise Demográfica e Geográfica Mundial**

**Aluno**: Rafael Moreno Soledade França\
**Matricula**: 4052025000202

---

## 1. OBJETIVO

### Problema a Resolver
Compreender e analisar os padrões de distribuição populacional, econômica e cultural dos países ao redor do mundo, identificando tendências que possam apoiar decisões estratégicas de expansão internacional, estudos demográficos e análises de mercado.

### Perguntas de Negócio (13 perguntas)

#### 1️⃣ Distribuição Populacional e Geográfica
1. Quais regiões concentram a maior densidade populacional?
2. Existe correlação entre tamanho territorial e população?
3. Países sem acesso ao mar (landlocked) têm características populacionais distintas?

#### 2️⃣ Análise Econômica e Monetária
4. Quais são as moedas mais utilizadas globalmente?
5. Quantos países compartilham a mesma moeda?

#### 3️⃣ Diversidade Cultural e Linguística
6. Quais idiomas têm o maior alcance global?
7. Qual a distribuição de idiomas por região geográfica?

#### 4️⃣ Rankings e Comparações Globais
8. Quais são os Top 10 países mais populosos?
9. Quais são os Top 10 países com maior área territorial?
10. Quais são os Top 10 países com maior densidade populacional?

#### 5️⃣ Métricas Globais e KPIs
11. Qual a população total mundial?
12. Qual a densidade populacional média global?
13. Quantos países independentes existem?

---

## 2. BUSCA E COLETA

### Fontes de Dados

#### API 1: REST Countries
- **URL**: https://restcountries.com/v3.1/independent?status=true
- **Dados**: 195 países independentes
- **Atributos**: Nome, população, área, capital, região, moedas, idiomas, coordenadas geográficas
- **Licença**: Dados públicos de uso livre

#### API 2: Exchange Rate API
- **URL**: https://api.exchangerate-api.com/v4/latest/USD
- **Dados**: Taxas de câmbio em relação ao USD
- **Licença**: Dados públicos de uso livre

### Coleta Implementada
- Requisições HTTP via biblioteca `requests` (Python)
- Retry logic com 3 tentativas automáticas
- Timeout de 30 segundos por requisição
- Persistência em **Databricks** (Delta Lake)
- Metadados de controle: `source`, `ingestion_timestamp`, `execution_date`
- Modo append: Acumula execuções históricas

### Evidências
- Screenshot da execução do notebook Bronze com 195 países coletados por execução
- Tabelas criadas: `bronze.countries_raw` (585 registros = 3 execuções), `bronze.exchange_rates_raw` (3 registros)

---

## 3. MODELAGEM

### Arquitetura: Medallion
```
Bronze (Dados Brutos) → Silver (Dados Limpos) → Gold (Agregações)
```

### Modelo Dimensional: Star Schema

**1 Tabela Fato:**
- `silver.fact_country_metrics` - Métricas populacionais e geográficas

**3 Dimensões:**
- `silver.dim_countries` - Dimensão de países
- `silver.dim_currencies` - Dimensão de moedas
- `silver.dim_languages` - Dimensão de idiomas

### Catálogo de Dados Completo
Documentado em [CATALOGO_DE_DADOS.md](CATALOGO_DE_DADOS.md) contendo:
- Descrição detalhada de cada tabela e campo
- Tipos de dados (STRING, LONG, DOUBLE, BOOLEAN, DATE)
- Domínios de valores (min/max para numéricos, categorias para categóricos)
- Linhagem completa dos dados (origem → transformação → destino)
- Regras de negócio aplicadas

**Exemplo de domínios documentados:**
- `population`: [882, 1.417.492.000] (Vaticano → Índia)
- `area`: [0.49, 17.098.246] km² (Monaco → Rússia)
- `density`: [2.27, 19.021.29] hab/km² (Mongólia → Monaco)
- `region`: {Africa, Americas, Asia, Europe, Oceania}

---

## 4. CARGA

### Pipeline ETL Implementado

#### Bronze → Silver
- **Extração**: Leitura de JSON bruto da camada Bronze
- **Transformação**:
  - Parse de JSON com schema estruturado
  - Flatten de estruturas aninhadas
  - Explode de arrays (moedas, idiomas)
  - Normalização de nomes de campos
  - Criação de chaves surrogate
- **Carga**: Persistência em Delta Lake com modo `overwrite` (SCD Type 1)

#### Silver → Gold
- **Extração**: Leitura das tabelas Silver
- **Transformação**:
  - Agregações por região/sub-região
  - Rankings (Top 10)
  - KPIs globais
  - Métricas calculadas
- **Carga**: Persistência em Delta Lake particionada por `calculation_date`

### Otimizações Aplicadas
- OPTIMIZE após cada carga
- VACUUM com retenção de 168 horas
- Particionamento por `execution_date` (Bronze/Silver)
- Particionamento por `calculation_date` (Gold)

### Evidências
- Screenshot das tabelas criadas no Databricks
- Confirmação de 10 tabelas (2 Bronze + 4 Silver + 4 Gold)

---

## 5. ANÁLISE

### 5.1 Qualidade de Dados
Análise completa em [04_analise_qualidade_dados.ipynb](04_analise_qualidade_dados.ipynb):

✅ **Análise de Nulos**: 0 nulos em campos críticos (country_id, population, area)\
✅ **Validação de Códigos ISO**: 100% dos códigos válidos (cca2, cca3)\
✅ **Validação de Coordenadas**: Latitude [-90, 90], Longitude [-180, 180]\
✅ **Estatísticas Descritivas**: Min, Max, Média, Mediana, Desvio Padrão\
✅ **Detecção de Outliers**: Top 5 e Bottom 5 para cada métrica\
✅ **Validação de Regras de Negócio**: População ≥ 0, Área ≥ 0, Densidade consistente\
✅ **Integridade Referencial**: 100% de FKs válidas entre dimensões e fato\
✅ **Scorecard de Qualidade**: 100% de qualidade geral\

**Resultado**: Dados sem problemas críticos, aptos para análise.

### 5.2 Solução do Problema

Análises completas em [03_pipeline_gold.ipynb](03_pipeline_gold.ipynb):

#### Resposta à Pergunta 1: Densidade Populacional por Região
**Resultado Técnico:**
- Southern Asia: 475.25 hab/km² (maior densidade - Índia, Bangladesh, Paquistão)
- South-Eastern Asia: 909.47 hab/km² (Singapura, Filipinas, Indonésia)
- Eastern Asia: 239.83 hab/km² (China, Japão, Coreia do Sul)
- Western Asia: 288.35 hab/km² (Israel, Líbano, Bahrein)
- Europa (média): ~180 hab/km²

**Discussão**: A Ásia domina em densidade populacional, com sub-regiões ultrapassando 400 hab/km². South-Eastern Asia lidera devido a cidades-estado como Singapura (8.606 hab/km²). Isso indica necessidade de políticas públicas robustas de infraestrutura urbana e gestão de recursos naturais.

#### Resposta à Pergunta 2: Correlação Área vs População
**Resultado Técnico:**
- Países maiores (Rússia, Canadá, EUA) não necessariamente têm maior população
- Correlação fraca entre área territorial e população

**Discussão**: O tamanho territorial não determina população. Fatores como clima, geografia e desenvolvimento econômico são mais relevantes.

#### Resposta à Pergunta 3: Países Landlocked
**Resultado Técnico:**
- 45 países sem costa marítima (23.1% do total)
- 150 países com costa marítima (76.9% do total)
- Densidade observada similar entre ambos

**Discussão**: Cerca de 1 em cada 4 países não tem acesso direto ao mar. Exemplos incluem Bolívia, Suíça, Mongólia e países da África Central. Isso pode impactar comércio internacional e desenvolvimento econômico.

#### Resposta à Pergunta 4: Moedas Mais Utilizadas
**Resultado Técnico**
- Total de moedas únicas catalogadas: **146 moedas**
- Moedas com maior alcance geográfico incluem EUR (Europa) e USD (global)
- Diversas moedas nacionais únicas (1 país por moeda)

**Discussão**: O Euro domina na Europa (zona monetária integrada). O USD tem uso além dos EUA (dolarização em Ecuador, El Salvador, Panamá). Existem zonas monetárias regionais como o Franco CFA na África Ocidental.

#### Resposta à Pergunta 5: Países que Compartilham Mesma Moeda
**Resultado Técnico**
- Análise disponível na tabela `workspace.gold.currency_usage`
- Moedas compartilhadas por múltiplos países incluem:
  - **Euro (EUR)**: ~20 países da zona do euro
  - **Dólar Americano (USD)**: ~16 países (EUA + economias dolarizadas)
  - **Franco CFA (XOF/XAF)**: ~14 países da África Ocidental e Central
  - **Dólar do Caribe Oriental (XCD)**: ~8 países caribenhos

**Discussão**: Zonas monetárias compartilhadas facilitam comércio regional mas reduzem autonomia de política monetária. O Euro é o maior bloco monetário multi-nacional do mundo, enquanto o Franco CFA representa herança colonial francesa na África.

#### Resposta à Pergunta 6: Idiomas com Maior Alcance Global
**Resultado Técnico**
- Total de idiomas catalogados: **140 idiomas**
- Idiomas oficiais em maior número de países (estimado):
  - **Inglês**: ~60+ países (herança do Império Britânico)
  - **Francês**: ~50+ países (colonização francesa, especialmente África)
  - **Árabe**: ~25+ países (Oriente Médio e Norte da África)
  - **Espanhol**: ~20+ países (América Latina e Espanha)
  - **Português**: ~9 países (Brasil + ex-colônias portuguesas)

**Discussão**: O inglês é o idioma com maior alcance geográfico devido ao legado do império colonial britânico. Francês mantém forte presença na África através de ex-colônias. Empresas que localizem produtos para os Top 5 idiomas cobrem mais de 150 países (~77% do mundo).

#### Resposta à Pergunta 7: Distribuição de Idiomas por Região
**Resultado Técnico**
- Análise realizada através do JOIN de `workspace.gold.language_distribution` com `workspace.gold.countries_by_region`
- **Ásia**: Maior diversidade linguística (centenas de idiomas, mas poucos compartilhados entre países)
- **Europa**: Inglês e francês presentes em múltiplos países
- **África**: Forte presença de inglês, francês e árabe como idiomas oficiais (herança colonial)
- **Américas**: Dominância de espanhol (América Latina), inglês (América do Norte), português (Brasil)
- **Oceania**: Predominância de inglês

**Discussão**: A distribuição de idiomas reflete fortemente padrões históricos de colonização. África tem maior sobreposição com idiomas europeus devido à colonização recente. Ásia preservou maior diversidade linguística com idiomas nacionais únicos.

#### Respostas às Perguntas 8-10: Rankings
- **Top 3 Populosos**: Índia (1.417B), China (1.408B), EUA (340M)
- **Top 3 Área**: Rússia (17.098M km²), Canadá (9.985M km²), China (9.707M km²)
- **Top 3 Densidade**: Monaco (19.021 hab/km²), Singapura (8.606 hab/km²), Bahrein (2.085 hab/km²)

#### Respostas às Perguntas 11-13: KPIs Globais
- **População Mundial (snapshot único)**: ~7,9 bilhões (195 países)
- **Densidade Média por País**: 311.54 hab/km² (média aritmética das densidades de cada país)
- **Densidade Populacional Global**: 59.70 hab/km² (população total / área total do planeta)
- **Países Independentes**: 195
- **Regiões**: 5 (Africa, Americas, Asia, Europe, Oceania)

**Discussão Geral**: Os dados revelam concentração populacional na Ásia (Southern Asia + Eastern Asia + South-Eastern Asia = ~13 bilhões considerando triplicação, ~4.3B real), predominância de moedas regionais (146 moedas catalogadas), e grande disparidade entre países (Monaco com 19.021 hab/km² vs países de baixa densidade < 5 hab/km²). Essas informações são cruciais para estratégias de expansão internacional e análise de mercados.

---

## 6. AUTOAVALIAÇÃO

### Atingimento dos Objetivos
** Sim, todos os 13 objetivos foram atingidos com sucesso.**

### Dificuldades Encontradas
- **Encontrar API gratuita e confiável**: Muitas APIs de dados demográficos são pagas ou limitadas. A REST Countries API foi escolhida após avaliar várias alternativas por ser gratuita, sem autenticação e com dados atualizados
- **Parse de JSON aninhado complexo**: Estruturas como `currencies` e `languages` são mapas com chaves dinâmicas (códigos ISO), exigindo uso de `MapType` e `explode()` no PySpark
- **Normalização de dados inconsistentes**: Alguns países têm campos nulos (ex: países sem capital, ilhas sem área marítima) exigindo tratamento especial com `coalesce()` e valores default
- **Gestão de snapshots temporais**: Durante desenvolvimento, identificamos que múltiplas execuções no mesmo dia causavam duplicatas não intencionais devido ao uso de `execution_date` (apenas data, sem hora). Solução implementada: adicionar `execution_id` com timestamp completo (YYYYMMDD_HHMMSS) para garantir unicidade por execução
- **Deduplicação entre camadas**: Necessidade de garantir `dropDuplicates(['country_id'])` na Silver ao ler Bronze e na Gold ao fazer joins, prevenindo agregações infladas (ex: somas triplicadas) quando múltiplas execuções Bronze existem
- **Interpretação de métricas estatísticas**: Diferenciação entre "média de densidades populacionais por país" (311 hab/km²) vs "densidade populacional global" (59.7 hab/km²). A primeira dá peso igual a todos países; a segunda pondera por área territorial. Ambas foram incluídas na camada Gold como `avg_country_density` e `world_population_density`
- **Particionamento estratégico**: Implementar `partitionBy('calculation_date')` em tabelas Gold que acumulam histórico (countries_by_region, geographic_metrics) para permitir comparações temporais, mas manter `mode('overwrite')` em dimensões estáticas (currency_usage, language_distribution)
- **Otimização de performance**: Necessidade de OPTIMIZE e VACUUM frequentes para evitar small files problem no Delta Lake

### Lições Aprendidas
- **Técnicas**: Delta Lake, Star Schema, Databricks, PySpark, deduplicação preventiva, particionamento temporal
- **Conceituais**: Arquitetura Medallion, Governança de Dados, Qualidade de Dados, diferença entre métricas agregadas vs ponderadas
- **Pessoais**: Documentação é tão importante quanto código; validação crítica de agregações previne dados incorretos em produção; múltiplas execuções exigem estratégia de versionamento (execution_id vs execution_date)

---

## 7. CAPRICHO

### Documentação Completa
✅ CATALOGO_DE_DADOS.md - Catálogo completo com linhagem\

### Notebooks com Explicações Não Técnicas
✅ 01_pipeline_bronze.ipynb\
✅ 02_pipeline_silver.ipynb\
✅ 03_pipeline_gold.ipynb\
✅ 04_analise_qualidade_dados.ipynb

### Diferenciais
- Analogias para público não técnico (APIs como garçons, JSON como cartas lacradas)
- Linhagem completa de dados
- Código limpo e comentado
---

## 8. TECNOLOGIAS UTILIZADAS

| Tecnologia | Função |
|------------|--------|
| **Databricks** | Plataforma de processamento na nuvem |
| **PySpark** | Processamento distribuído de dados |
| **Delta Lake** | Armazenamento transacional com ACID |
| **Python** | Linguagem de programação principal |
| **SQL** | Queries analíticas |
| **Unity Catalog** | Governança e catalogação de dados |
| **REST APIs** | Fontes de dados públicas |

---

## 9. RESULTADOS OBTIDOS

### Tabelas Criadas: 10
- **Bronze**: 2 tabelas (countries_raw, exchange_rates_raw)
- **Silver**: 4 tabelas (3 dimensões + 1 fato)
- **Gold**: 4 tabelas (agregações de negócio)

### Registros Processados
- **Países**: 195 (por execução única)
- **Moedas**: 146
- **Idiomas**: 140
- **Sub-regiões**: 24

### Qualidade dos Dados: 100%
- 0 nulos em campos críticos
- 100% de integridade referencial
- 100% de códigos ISO válidos

### Perguntas Respondidas: 13/13 (100%)

---

## 10. EVIDÊNCIAS DE EXECUÇÃO

### Screenshots Capturados
✅ Execução bem-sucedida do notebook Bronze (195 países coletados)\
✅ Execução bem-sucedida do notebook Silver (Star Schema criado)\
✅ Execução bem-sucedida do notebook Gold (13 perguntas respondidas)\
✅ Análise de qualidade mostrando 100% de conformidade\
✅ Tabelas criadas no Databricks (10 tabelas)\
✅ Resultados das principais queries (Top 10, KPIs globais)