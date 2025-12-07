# Databricks Workflows - Pipeline de Dados

## Visao Geral

Este documento descreve a configuracao do Job no Databricks para orquestracao do pipeline de dados.

---

## Job: Pipeline Countries Data

### Configuracao do Job

| Propriedade | Valor |
|-------------|-------|
| **Nome do Job** | pipeline_countries_data |
| **Tipo** | Multi-task Job |
| **Schedule** | Manual ou Diario (configuravel) |
| **Timeout** | 30 minutos |
| **Retries** | 2 tentativas por task |
| **Email Alerts** | On Failure |

---

## Tasks e Ordem de Execucao

```
Task 1: bronze
    |
    v
Task 2: silver
    |
    v
Task 3: gold
    |
    v
Task 4: quality (opcional)
    |
    v
Task 5: dashboard (opcional)
```

### Detalhamento das Tasks

| Task | Notebook | Depende de | Descricao |
|------|----------|------------|-----------|
| **bronze** | 01_pipeline_bronze.ipynb | - | Ingesta dados das APIs |
| **silver** | 02_pipeline_silver.ipynb | bronze | Transforma e cria Star Schema |
| **gold** | 03_pipeline_gold.ipynb | silver | Gera agregacoes e insights |
| **quality** | Evidencias/04_analise_qualidade_dados.ipynb | gold | Valida qualidade dos dados |
| **dashboard** | Evidencias/05_dashboard.ipynb | gold | Visualizacoes interativas (opcional) |

---

## Configuracao do Cluster

### Cluster Recomendado

| Propriedade | Valor |
|-------------|-------|
| **Tipo** | Shared / Single Node |
| **Runtime** | Databricks Runtime 13.3 LTS ou superior |
| **Node Type** | Standard_DS3_v2 (ou equivalente) |
| **Workers** | 0-2 (auto-scaling) |
| **Spark Config** | Default |

### Para Databricks Community Edition

| Propriedade | Valor |
|-------------|-------|
| **Cluster Mode** | Single Node |
| **Runtime** | 13.3 LTS (Scala 2.12, Spark 3.4.1) |
| **Terminate After** | 120 minutos de inatividade |

---

## Exemplo de Configuracao JSON

```json
{
  "name": "pipeline_countries_data",
  "email_notifications": {
    "on_failure": ["seu-email@exemplo.com"]
  },
  "timeout_seconds": 1800,
  "max_concurrent_runs": 1,
  "tasks": [
    {
      "task_key": "bronze",
      "notebook_task": {
        "notebook_path": "/Workspace/MVP/01_pipeline_bronze",
        "source": "WORKSPACE"
      },
      "existing_cluster_id": "seu-cluster-id"
    },
    {
      "task_key": "silver",
      "depends_on": [{"task_key": "bronze"}],
      "notebook_task": {
        "notebook_path": "/Workspace/MVP/02_pipeline_silver",
        "source": "WORKSPACE"
      },
      "existing_cluster_id": "seu-cluster-id"
    },
    {
      "task_key": "gold",
      "depends_on": [{"task_key": "silver"}],
      "notebook_task": {
        "notebook_path": "/Workspace/MVP/03_pipeline_gold",
        "source": "WORKSPACE"
      },
      "existing_cluster_id": "seu-cluster-id"
    },
    {
      "task_key": "quality",
      "depends_on": [{"task_key": "gold"}],
      "notebook_task": {
        "notebook_path": "/Workspace/MVP/04_analise_qualidade_dados",
        "source": "WORKSPACE"
      },
      "existing_cluster_id": "seu-cluster-id"
    }
  ]
}
```

---

## Como Configurar o Job

### Passo 1: Criar o Job

1. Acesse **Workflows** no menu lateral do Databricks
2. Clique em **Create Job**
3. Nomeie como `pipeline_countries_data`

### Passo 2: Adicionar Tasks

1. Adicione a primeira task `bronze` apontando para `01_pipeline_bronze.ipynb`
2. Adicione `silver` com dependencia de `bronze`
3. Adicione `gold` com dependencia de `silver`
4. (Opcional) Adicione `quality` com dependencia de `gold`

### Passo 3: Configurar Cluster

1. Selecione um cluster existente ou crie um novo
2. Recomendado: usar cluster compartilhado para economia

### Passo 4: Configurar Alertas

1. Em **Email notifications**, adicione emails para falhas
2. Configure timeout de 30 minutos

---

## Tempo de Execucao Estimado

| Task | Tempo Medio |
|------|-------------|
| bronze | 2-3 min |
| silver | 3-5 min |
| gold | 5-8 min |
| quality | 2-3 min |
| **Total** | **~15 min** |

---

## Monitoramento

### Metricas a Acompanhar

- **Sucesso/Falha**: Taxa de execucoes bem-sucedidas
- **Duracao**: Tempo total de cada run
- **Registros**: Contagem de registros por camada
- **Erros de API**: Falhas de conexao com APIs externas

### Verificar Logs

1. Acesse **Workflows** > **Job Runs**
2. Clique na execucao desejada
3. Visualize logs de cada task individualmente

---

## Troubleshooting

| Problema | Causa Provavel | Solucao |
|----------|---------------|---------|
| Timeout na task bronze | API lenta ou indisponivel | Aumentar timeout ou verificar status da API |
| Falha de schema | Database nao existe | Verificar criacao de schemas no notebook bronze |
| Dados duplicados | Multiplas execucoes no dia | Verificar logica de dedup na Silver |
| Cluster indisponivel | Cluster terminado | Iniciar cluster antes de executar |

---

## Referencias

- [Databricks Workflows Documentation](https://docs.databricks.com/workflows/index.html)
- [Job Scheduling Best Practices](https://docs.databricks.com/workflows/jobs/jobs.html)
