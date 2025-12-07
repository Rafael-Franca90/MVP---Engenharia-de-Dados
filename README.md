# MVP - Engenharia de Dados | PUC-Rio

**Pipeline de Dados na Nuvem: Analise Demografica e Geografica Mundial**

**Aluno:** Rafael Moreno Soledade Franca  
**Matricula:** 4052025000202

---

## 1. Objetivo do Projeto

Compreender e analisar os padroes de distribuicao populacional, economica e cultural dos paises ao redor do mundo, identificando tendencias que possam apoiar decisoes estrategicas de expansao internacional, estudos demograficos e analises de mercado.

### Perguntas de Negocio (13 perguntas)

| # | Categoria | Pergunta |
|---|-----------|----------|
| 1 | Distribuicao Geografica | Quais regioes concentram a maior densidade populacional? |
| 2 | Distribuicao Geografica | Existe correlacao entre tamanho territorial e populacao? |
| 3 | Distribuicao Geografica | Paises landlocked tem caracteristicas distintas? |
| 4 | Analise Economica | Quais sao as moedas mais utilizadas globalmente? |
| 5 | Analise Economica | Quantos paises compartilham a mesma moeda? |
| 6 | Diversidade Cultural | Quais idiomas tem o maior alcance global? |
| 7 | Diversidade Cultural | Qual a distribuicao de idiomas por regiao? |
| 8 | Rankings Globais | Quais sao os Top 10 paises mais populosos? |
| 9 | Rankings Globais | Quais sao os Top 10 paises com maior area? |
| 10 | Rankings Globais | Quais sao os Top 10 paises com maior densidade? |
| 11 | Metricas Globais | Qual a populacao total mundial? |
| 12 | Metricas Globais | Qual a densidade populacional media global? |
| 13 | Metricas Globais | Quantos paises independentes existem? |

---

## 2. Justificativa das Escolhas

### Plataforma: Databricks

O **Databricks** foi escolhido como plataforma por oferecer:

- Ambiente integrado para engenharia de dados na nuvem
- Suporte nativo a Delta Lake e PySpark
- Unity Catalog para governanca de dados
- Versao gratuita disponivel (Community Edition)
- Facilidade de reproducao academica

### Fontes de Dados: APIs Publicas

Uma das principais dificuldades foi **encontrar APIs gratuitas e confiaveis**. Muitas APIs de dados demograficos sao pagas ou exigem cadastro complexo.

As APIs selecionadas sao **100% publicas e gratuitas**:

| API | Dados | Vantagem |
|-----|-------|----------|
| REST Countries | 195 paises independentes | Sem cadastro, sem API key |
| Exchange Rate API | Taxas de cambio USD | Acesso livre, dados atualizados |

**Isso garante total reprodutibilidade** - qualquer pessoa pode executar o pipeline sem barreiras.

---

## 3. Arquitetura e Modelagem

### Pipeline Medallion (Bronze -> Silver -> Gold)

```
+------------------+     +------------------+     +------------------+
|     BRONZE       |     |     SILVER       |     |      GOLD        |
|  Dados Brutos    | --> | Star Schema      | --> | Agregacoes       |
|  (JSON das APIs) |     | (Fato+Dimensoes) |     | (KPIs e Insights)|
+------------------+     +------------------+     +------------------+
     2 tabelas              4 tabelas              4 tabelas
```

### Modelo Dimensional: Star Schema

```
                    +-------------------+
                    |  dim_countries    |
                    |  (195 registros)  |
                    +--------+----------+
                             |
+------------------+         |         +------------------+
|  dim_currencies  +---------+---------+  dim_languages   |
|  (146 registros) |         |         |  (141 registros) |
+------------------+         |         +------------------+
                             |
                    +--------+----------+
                    | fact_country_     |
                    | metrics           |
                    | (195 registros)   |
                    +-------------------+
```

**Tabela Fato:** `fact_country_metrics` (populacao, area, densidade)

**Dimensoes:** `dim_countries`, `dim_currencies`, `dim_languages`

---

## 4. Pipeline ETL

### Bronze: Ingestao de Dados Brutos

- Requisicoes HTTP via biblioteca `requests`
- Retry logic com 3 tentativas automaticas
- Timeout de 30 segundos por requisicao
- Metadados de controle: `source`, `ingestion_timestamp`, `execution_date`
- Persistencia em Delta Lake (modo append)

### Silver: Transformacao e Normalizacao

- Parse de JSON com schema estruturado
- Flatten de estruturas aninhadas
- Explode de arrays (moedas, idiomas)
- Criacao de chaves surrogate
- Deduplicacao preventiva com `dropDuplicates()`
- Persistencia em Delta Lake (SCD Type 1)

### Gold: Agregacoes e Insights

- Agregacoes por regiao/sub-regiao
- Rankings (Top 10)
- KPIs globais calculados
- Particionamento por `calculation_date`
- Otimizacao com OPTIMIZE e VACUUM

---

## 5. Resultados e Insights

### Metricas Obtidas

| Metrica | Valor |
|---------|-------|
| Paises processados | 195 |
| Moedas catalogadas | 146 |
| Idiomas identificados | 141 |
| Sub-regioes mapeadas | 24 |
| Tabelas criadas | 10 |
| Perguntas respondidas | 13/13 (100%) |
| Qualidade dos dados | 100% |

### Top 5 Insights Principais

1. **59% da populacao mundial** esta concentrada na Asia
2. **Euro (EUR)** e a moeda mais utilizada (26 paises)
3. **Ingles** e idioma oficial em 59 paises
4. **45 paises (23%)** sao landlocked (sem costa maritima)
5. **Nao ha correlacao forte** entre area territorial e populacao

### Resumo das Respostas

| Pergunta | Resposta |
|----------|----------|
| Maior densidade | Western Europe (2.617 hab/km²) |
| Correlacao area/pop | Fraca - tamanho nao determina populacao |
| Landlocked | 45 paises, densidade menor (151 vs 360 hab/km²) |
| Moeda mais usada | Euro (EUR) - 26 paises |
| Idioma mais falado | Ingles - 59 paises |
| Pais mais populoso | India - 1.417 bilhoes |
| Maior area | Russia - 17.098.246 km² |
| Maior densidade | Monaco - 19.021 hab/km² |
| Pop. mundial | ~7,97 bilhoes |
| Densidade media | 311 hab/km² (por pais) / 59.7 hab/km² (global) |
| Total paises | 195 independentes |

---

## 6. Qualidade de Dados

A analise completa esta disponivel em [04_analise_qualidade_dados.ipynb](Evidencias/04_analise_qualidade_dados.ipynb).

### Validacoes Implementadas

| Categoria | Validacao | Resultado |
|-----------|-----------|-----------|
| **Completude** | Analise de Nulos | 0 nulos em campos criticos (country_id, population, area) |
| **Conformidade** | Codigos ISO Alpha-2/3 | 100% validos |
| **Consistencia** | Coordenadas geograficas | Latitude [-90, 90], Longitude [-180, 180] |
| **Integridade** | Chaves estrangeiras | 100% FKs validas entre dimensoes e fato |
| **Regras de Negocio** | Valores positivos | Populacao >= 0, Area >= 0, Densidade consistente |

### Estatisticas Descritivas

| Metrica | Populacao | Area (km²) | Densidade |
|---------|-----------|------------|-----------|
| Minimo | 882 (Vaticano) | 0.49 (Monaco) | 2.27 (Mongolia) |
| Maximo | 1.417.492.000 (India) | 17.098.246 (Russia) | 19.021 (Monaco) |
| Media | 40.896.741 | 685.477 | 311.54 |
| Mediana | 8.703.000 | 99.700 | 94.5 |

### Deteccao de Outliers

**Top 5 por Populacao:**
1. India - 1.417 bilhoes
2. China - 1.408 bilhoes
3. Estados Unidos - 340 milhoes
4. Indonesia - 278 milhoes
5. Paquistao - 231 milhoes

**Top 5 por Densidade:**
1. Monaco - 19.021 hab/km²
2. Singapura - 8.606 hab/km²
3. Bahrein - 2.085 hab/km²
4. Maldivas - 1.802 hab/km²
5. Malta - 1.666 hab/km²

### Scorecard de Qualidade

| Dimensao | Peso | Score |
|----------|------|-------|
| Completude | 25% | 100% |
| Conformidade | 25% | 100% |
| Consistencia | 25% | 100% |
| Integridade | 25% | 100% |
| **Total** | **100%** | **100%** |

**Conclusao:** Dados sem problemas criticos, aptos para analise e tomada de decisao.

---

## 7. Catalogo de Dados

### Visao Geral das Tabelas

```
APIs REST
    ↓
┌─────────────────────────────────────────────┐
│ BRONZE (workspace.bronze)                   │
│ • countries_raw (195 registros/execucao)    │
│ • exchange_rates_raw (1 registro/execucao)  │
└─────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────┐
│ SILVER (workspace.silver)                   │
│ • dim_countries (195 registros)             │
│ • dim_currencies (146 registros)            │
│ • dim_languages (141 registros)             │
│ • fact_country_metrics (195 registros)      │
└─────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────┐
│ GOLD (workspace.gold)                       │
│ • countries_by_region (24 registros)        │
│ • currency_usage (146 registros)            │
│ • language_distribution (141 registros)     │
│ • geographic_metrics (9 registros)          │
└─────────────────────────────────────────────┘
```

### Camada Bronze - Dados Brutos

| Tabela | Descricao | Tipo | Particionamento |
|--------|-----------|------|-----------------|
| `countries_raw` | JSON bruto da REST Countries API | Append-only | execution_date |
| `exchange_rates_raw` | Taxas de cambio USD | Append-only | execution_date |

### Camada Silver - Star Schema

| Tabela | Tipo | PK | Registros | Descricao |
|--------|------|-----|-----------|-----------|
| `dim_countries` | Dimensao | country_id | 195 | Paises com atributos geograficos |
| `dim_currencies` | Dimensao | currency_code | 146 | Moedas com nome e simbolo |
| `dim_languages` | Dimensao | language_code | 141 | Idiomas com codigo ISO |
| `fact_country_metrics` | Fato | country_id | 195 | Populacao, area, densidade |

### Camada Gold - Agregacoes

| Tabela | Registros | Descricao |
|--------|-----------|-----------|
| `countries_by_region` | 24 | Agregacoes por regiao/sub-regiao |
| `currency_usage` | 146 | Ranking de moedas por alcance |
| `language_distribution` | 141 | Ranking de idiomas por alcance |
| `geographic_metrics` | 9 | KPIs globais (populacao, area, densidade) |

### Linhagem de Dados

```
[REST Countries API] → [Bronze] → [Silver: Star Schema] → [Gold: KPIs]
         ↓                ↓              ↓                    ↓
    195 paises       JSON bruto    Fato + Dimensoes      Agregacoes
```

### Dominios de Valores Principais

| Campo | Min | Max | Exemplos |
|-------|-----|-----|----------|
| population | 882 (Vaticano) | 1.417.492.000 (India) | Inteiro positivo |
| area_km2 | 0.49 (Monaco) | 17.098.246 (Russia) | Decimal positivo |
| density | 2.27 (Mongolia) | 19.021 (Monaco) | hab/km² |
| region | - | - | Africa, Americas, Asia, Europe, Oceania |

---

## 8. Integracao Unity Catalog

O projeto utiliza o **Unity Catalog** do Databricks para documentar todas as tabelas e colunas, garantindo governanca de dados e facilitando a descoberta de informacoes por outros usuarios.

### Metadados Registrados

Cada tabela possui as seguintes propriedades documentadas:

| Propriedade | Descricao | Exemplo |
|-------------|-----------|---------|
| `comment` | Descricao detalhada da tabela | "Dimensao de paises com atributos geograficos..." |
| `layer` | Camada do pipeline | bronze, silver, gold |
| `model` | Tipo de modelagem | Star Schema - Dimension, Fact Table |
| `grain` | Granularidade dos dados | 1 registro por pais |
| `source` | Fonte dos dados | REST Countries API v3.1 |
| `source_tables` | Tabelas de origem | silver.dim_countries, silver.fact_country_metrics |

### Colunas Documentadas

Todas as colunas possuem comentarios explicativos:

```
country_id: "Chave primaria - Codigo ISO Alpha-3 do pais (ex: BRA, USA, CHN)"
population: "Populacao total do pais (numero de habitantes)"
area_km2: "Area territorial do pais em quilometros quadrados"
```

### Como Consultar a Documentacao

**Via SQL:**
```sql
-- Ver propriedades da tabela
DESCRIBE TABLE EXTENDED workspace.silver.dim_countries;

-- Ver comentarios das colunas
DESCRIBE workspace.silver.dim_countries;
```

**Via Interface Unity Catalog:**
1. Navegue ate **Catalog** → **workspace** → **silver**
2. Selecione a tabela desejada (ex: `dim_countries`)
3. Veja a aba **Details** para propriedades e **Schema** para colunas

### Beneficios da Documentacao

- **Descoberta de Dados**: Usuarios encontram facilmente tabelas relevantes
- **Governanca**: Metadados claros sobre origem e responsabilidade
- **Colaboracao**: Novos membros entendem rapidamente o modelo
- **Auditoria**: Rastreabilidade completa de fonte e transformacoes

---

## 9. Databricks Genie - IA Generativa

O **Databricks Genie** e a inteligencia artificial generativa integrada ao Databricks que permite consultar dados usando **linguagem natural**. Neste projeto, o Genie e utilizado para responder as 13 perguntas de negocio de forma interativa.

### O que e o Genie?

O Genie e um assistente de IA que:
- Entende perguntas em portugues ou ingles
- Converte automaticamente para SQL
- Executa consultas nas tabelas Gold
- Retorna respostas formatadas e insights

### Como Usar

1. **Acessar o Genie** no Databricks (menu lateral)
2. **Criar um Genie Space** com as tabelas:
   - `workspace.gold.countries_by_region`
   - `workspace.gold.currency_usage`
   - `workspace.gold.language_distribution`
   - `workspace.gold.geographic_metrics`
3. **Fazer perguntas** em linguagem natural

### Exemplos de Perguntas

| Pergunta Natural | O Genie Responde |
|------------------|------------------|
| "Qual a moeda mais usada no mundo?" | Euro (EUR) - 26 paises |
| "Quais paises falam ingles?" | Lista de 59 paises |
| "Qual a populacao total mundial?" | ~7,97 bilhoes |
| "Quais sao os 10 paises mais populosos?" | India, China, EUA... |

### Video de Demonstracao

Um video demonstrando o Genie respondendo as perguntas de negocio esta disponivel em:

🎥 **[Assistir Demo do Genie no OneDrive](https://1drv.ms/v/c/877ff80325bb67be/IQB_Asu2QJuFTIFnpl-_-WW3AfaBDggtmfm7l66sMu5fFbE?e=VsnlWS)**

---

## 10. Licoes Aprendidas

### Dificuldades Encontradas

| Dificuldade | Solucao Aplicada |
|-------------|------------------|
| Encontrar API gratuita e confiavel | REST Countries API - sem autenticacao |
| Parse de JSON aninhado complexo | `MapType` e `explode()` no PySpark |
| Dados inconsistentes (nulos) | `coalesce()` e valores default |
| Duplicatas entre execucoes | `dropDuplicates()` preventivo |
| Metricas ambiguas | Separar `avg_country_density` vs `world_population_density` |
| Small files no Delta Lake | OPTIMIZE e VACUUM frequentes |

### Tecnicas de Engenharia Aplicadas

- **Delta Lake**: Transacoes ACID, time travel, schema evolution
- **Star Schema**: Modelagem dimensional para analytics
- **Arquitetura Medallion**: Separacao clara Bronze/Silver/Gold
- **Particionamento**: Por `execution_date` e `calculation_date`
- **Deduplicacao**: Preventiva em cada camada
- **Retry Logic**: Resiliencia na coleta de APIs
- **Metadados de Controle**: Rastreabilidade completa

### Conceitos Consolidados

- Governanca de Dados com Unity Catalog
- Diferenca entre metricas agregadas vs ponderadas
- Importancia da documentacao junto ao codigo
- Validacao critica de agregacoes previne erros em producao

---

## 11. Tecnologias Utilizadas

| Tecnologia | Funcao |
|------------|--------|
| **Databricks** | Plataforma de processamento na nuvem |
| **PySpark** | Processamento distribuido de dados |
| **Delta Lake** | Armazenamento transacional com ACID |
| **Python** | Linguagem de programacao principal |
| **SQL** | Queries analiticas |
| **Unity Catalog** | Governanca e catalogacao de dados |
| **Databricks Genie** | IA generativa para consultas em linguagem natural |
| **REST APIs** | Fontes de dados publicas |

---

## 12. Como Executar

### Pre-requisitos

- Conta no Databricks (Community Edition ou superior)
- Cluster configurado com PySpark

### Passo a Passo

1. **Clonar ou Importar Repositorio**
   - No Databricks: `Repos` > `Add Repo` > Cole a URL do GitHub
   - Ou: `Workspace` > `Import` > Selecione os arquivos `.ipynb`

2. **Executar Notebooks na Ordem**
   ```
   1. 01_pipeline_bronze.ipynb              # Coleta dados das APIs
   2. 02_pipeline_silver.ipynb              # Transforma e cria Star Schema
   3. 03_pipeline_gold.ipynb                # Gera agregacoes e insights
   ```

3. **Executar Evidencias (Opcional)**
   ```
   4. Evidencias/04_analise_qualidade_dados.ipynb  # Valida qualidade
   ```

4. **Verificar Resultados**
   - Tabelas em `workspace.bronze`, `workspace.silver`, `workspace.gold`
   - 13 perguntas respondidas no notebook Gold

5. **Consultar via Genie (IA Generativa)**
   - Acesse o Databricks Genie para fazer perguntas em linguagem natural
   - O Genie utiliza as tabelas Gold para responder as perguntas de negocio

---

## 13. Evidencias

### Notebooks de Pipeline

| Notebook | Descricao |
|----------|-----------|
| `01_pipeline_bronze.ipynb` | Ingestao de dados das APIs |
| `02_pipeline_silver.ipynb` | Transformacao e Star Schema |
| `03_pipeline_gold.ipynb` | Agregacoes e respostas das 13 perguntas |

### Documentacao e Evidencias Complementares

| Arquivo | Conteudo |
|---------|----------|
| [04_analise_qualidade_dados.ipynb](Evidencias/04_analise_qualidade_dados.ipynb) | Analise de qualidade (100%) |
| [Demo_Genie.mp4](https://1drv.ms/v/c/877ff80325bb67be/IQB_Asu2QJuFTIFnpl-_-WW3AfaBDggtmfm7l66sMu5fFbE?e=VsnlWS) | Video demonstrando o Genie (OneDrive) |
| [DOCUMENTACAO_UNITY_CATALOG.md](Evidencias/DOCUMENTACAO_UNITY_CATALOG.md) | Documentacao completa do Unity Catalog |
| [Unitcatalgo_evidencia.png](Evidencias/Unitcatalgo_evidencia.png) | Screenshot do Unity Catalog - Tabelas documentadas |
| [Unitcatalgo_evidencia_02.png](Evidencias/Unitcatalgo_evidencia_02.png) | Screenshot do Unity Catalog - Colunas documentadas |
| [star_schema_diagram.svg](Evidencias/star_schema_diagram.svg) | Diagrama do modelo dimensional |
| [workflows/README.md](workflows/README.md) | Configuracao do Job Databricks |

---

## 14. Estrutura do Repositorio

```
MVP---Engenharia-de-Dados/
|
|-- README.md                          # Este arquivo (documentacao completa)
|-- .gitignore                         # Arquivos ignorados pelo Git
|
|-- 01_pipeline_bronze.ipynb           # Ingestao de dados brutos
|-- 02_pipeline_silver.ipynb           # Transformacao e Star Schema
|-- 03_pipeline_gold.ipynb             # Agregacoes e analises
|
|-- Evidencias/
|   |-- 04_analise_qualidade_dados.ipynb  # Analise de qualidade
|   |-- DOCUMENTACAO_UNITY_CATALOG.md     # Documentacao Unity Catalog (catalogo completo)
|   |-- Unitcatalgo_evidencia.png         # Screenshot Unity Catalog - Tabelas
|   |-- Unitcatalgo_evidencia_02.png      # Screenshot Unity Catalog - Colunas
|   |-- star_schema_diagram.svg           # Diagrama Star Schema
|   |-- Demo_Genie.mp4                    # Video (hospedado no OneDrive)
|
|-- workflows/
|   |-- README.md                      # Documentacao do Job Databricks
```

---

## Licenca

Projeto academico desenvolvido para o curso de Pos-Graduacao em Engenharia de Dados da PUC-Rio.

Os dados utilizados sao de fontes publicas e de uso livre.

---

*Documentacao gerada como parte do MVP de Engenharia de Dados.*
