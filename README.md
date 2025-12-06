# MVP - Engenharia de Dados | PUC-Rio

**Pipeline de Dados na Nuvem: Analise Demografica e Geografica Mundial**

**Aluno:** Rafael Moreno Soledade Franca  
**Matricula:** 4052025000202

---

## Objetivo do Projeto

Compreender e analisar os padroes de distribuicao populacional, economica e cultural dos paises ao redor do mundo, identificando tendencias que possam apoiar decisoes estrategicas de expansao internacional, estudos demograficos e analises de mercado.

### Perguntas de Negocio Respondidas (13 perguntas)

| Categoria | Perguntas |
|-----------|-----------|
| **Distribuicao Geografica** | Densidade por regiao, Correlacao area/populacao, Paises landlocked |
| **Analise Economica** | Moedas mais utilizadas, Paises que compartilham moedas |
| **Diversidade Cultural** | Idiomas com maior alcance, Distribuicao linguistica por regiao |
| **Rankings Globais** | Top 10 populosos, Top 10 por area, Top 10 por densidade |
| **Metricas Globais** | Populacao mundial, Densidade media, Total de paises |

---

## Arquitetura

### Pipeline Medallion (Bronze -> Silver -> Gold)

```
+------------------+     +------------------+     +------------------+
|     BRONZE       |     |     SILVER       |     |      GOLD        |
|  Dados Brutos    | --> | Star Schema      | --> | Agregacoes       |
|  (JSON das APIs) |     | (Fato+Dimensoes) |     | (KPIs e Insights)|
+------------------+     +------------------+     +------------------+
```

### Fontes de Dados

| API | Dados | URL |
|-----|-------|-----|
| REST Countries | 195 paises independentes | https://restcountries.com/v3.1/independent |
| Exchange Rate | Taxas de cambio USD | https://api.exchangerate-api.com/v4/latest/USD |

### Modelo Dimensional (Star Schema)

- **Tabela Fato:** `fact_country_metrics` (populacao, area, densidade)
- **Dimensoes:** `dim_countries`, `dim_currencies`, `dim_languages`

---

## Estrutura do Repositorio

```
MVP---Engenharia-de-Dados/
|
|-- README.md                          # Este arquivo
|-- .gitignore                         # Arquivos ignorados pelo Git
|
|-- 00_config.ipynb                    # Configuracoes e funcoes utilitarias
|-- 01_pipeline_bronze.ipynb           # Ingestao de dados brutos (APIs)
|-- 02_pipeline_silver.ipynb           # Transformacao e Star Schema
|-- 03_pipeline_gold.ipynb             # Agregacoes e analises de negocio
|-- 04_analise_qualidade_dados.ipynb   # Validacao de qualidade
|
|-- docs/
|   |-- MVP_ENTREGA.md                 # Documentacao completa da entrega
|   |-- CATALOGO_DE_DADOS.md           # Catalogo com linhagem dos dados
|   |-- DOCUMENTACAO_UNITY_CATALOG.md  # Documentacao do Unity Catalog
|   |-- star_schema_diagram.svg        # Diagrama do modelo dimensional
```

---

## Como Executar no Databricks

### Pre-requisitos
- Conta no Databricks (Community Edition ou superior)
- Cluster configurado com PySpark

### Passo a Passo

1. **Importar Notebooks**
   - No Databricks, va em `Workspace` > `Import`
   - Selecione os arquivos `.ipynb` deste repositorio

2. **Executar na Ordem**
   ```
   1. 00_config.ipynb       # Configuracoes iniciais
   2. 01_pipeline_bronze.ipynb  # Coleta de dados
   3. 02_pipeline_silver.ipynb  # Transformacoes
   4. 03_pipeline_gold.ipynb    # Analises
   5. 04_analise_qualidade_dados.ipynb  # Validacao
   ```

3. **Verificar Resultados**
   - Tabelas criadas em `workspace.bronze`, `workspace.silver`, `workspace.gold`
   - 13 perguntas de negocio respondidas no notebook Gold

---

## Tecnologias Utilizadas

| Tecnologia | Funcao |
|------------|--------|
| **Databricks** | Plataforma de processamento na nuvem |
| **PySpark** | Processamento distribuido de dados |
| **Delta Lake** | Armazenamento transacional com ACID |
| **Python** | Linguagem de programacao principal |
| **Unity Catalog** | Governanca e catalogacao de dados |
| **REST APIs** | Fontes de dados publicas |

---

## Resultados Obtidos

| Metrica | Valor |
|---------|-------|
| Paises processados | 195 |
| Moedas catalogadas | 146 |
| Idiomas identificados | 140+ |
| Perguntas respondidas | 13/13 (100%) |
| Qualidade dos dados | 100% |

---

## Principais Insights

1. **59% da populacao mundial** esta concentrada na Asia
2. **Euro (EUR)** e a moeda mais compartilhada (26 paises)
3. **Ingles** e idioma oficial em mais de 50 paises
4. **45 paises (23%)** sao landlocked
5. **Nao ha correlacao forte** entre area e populacao

---

## Documentacao Completa

Para detalhes completos sobre o projeto, consulte:

- [MVP_ENTREGA.md](docs/MVP_ENTREGA.md) - Documentacao completa da entrega
- [CATALOGO_DE_DADOS.md](docs/CATALOGO_DE_DADOS.md) - Catalogo de dados com linhagem

---

## Licenca

Projeto academico desenvolvido para o curso de Pos-Graduacao em Engenharia de Dados da PUC-Rio.

Os dados utilizados sao de fontes publicas e de uso livre.

