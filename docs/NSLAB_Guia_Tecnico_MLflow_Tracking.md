# NSLAB Logistics Intelligence Platform

# Guia técnico e didático — MLflow Tracking local

**Laboratório:** LOGISTICS-05 — Otimização de hiperparâmetros (Shanghai)
**Etapa documentada:** 2.4 — Instalação, configuração, persistência e validação do MLflow Tracking
**Data das validações:** 19/11/2026
**Versão do MLflow:** 3.18.0
**Repositório:** `NSLABTutoriais/Logistics_Intelligence_Platform`
**Notebook:** `notebooks/05_otimizacao_hiperparametros_shanghai.ipynb`
**Ambiente:** Windows 11, VS Code, Python 3.13.16, `.venv`
**Status:** infraestrutura validada; otimização ainda não iniciada.

> **Escopo e evidências.** Este guia consolida as operações e saídas compartilhadas durante as etapas 2.4.1–2.4.12. Comandos de treinamento e persistência analítica apresentados como exemplos são identificados como *futuros* e não foram executados. O arquivo não contém dados nem modelos treinados.

---

## 1. Contexto e objetivo de negócio

O laboratório utiliza o dataset **LaDe-P Shanghai**, de coletas (*pickup*), para investigar a pergunta de negócio: **como identificar antecipadamente coletas com risco de atraso?** O evento de atraso refere-se à **coleta após o limite da janela operacional**, não à entrega de mercadorias. A classificação será avaliada por **Average Precision** e métricas operacionais **Recall@K** e **Precision@K**.

Na preparação da otimização, foram selecionados **Random Forest D** e **XGBoost D**. O planejamento prevê `RandomizedSearchCV` com **12 iterações por modelo**, `StratifiedKFold` com **três folds** e **11 features** (três categóricas e oito numéricas), com `OneHotEncoder` e `StandardScaler`. O conjunto de teste está reservado.

Distribuição dos dados informada para esta etapa:

| Partição / indicador | Quantidade |
|---|---:|
| Coletas elegíveis | 1.424.195 |
| Coletas atrasadas | 36.319 (2,55%) |
| Treinamento | 996.936 |
| Validação | 213.629 |
| Teste reservado | 213.630 |

**Por que MLflow?** Porque uma otimização gera múltiplas configurações e resultados. É necessário saber **qual execução** produziu **quais parâmetros, métricas e arquivos**, e recuperar essas informações sem repetir treinamentos.

## 2. Fundamentos: o que é o MLflow?

O **MLflow** é uma plataforma para organizar o ciclo de vida de projetos de Machine Learning. Neste laboratório usamos inicialmente o componente **Tracking**, que registra e permite consultar experimentos. O MLflow também oferece recursos para modelos, avaliação, rastreamento de aplicações de IA e outros fluxos; nem todos fazem parte do escopo atual.

### 2.1 Vocabulário essencial

| Termo | Definição | Exemplo no laboratório |
|---|---|---|
| **Tracking** | Registro e consulta de execuções | Consultar otimizações de RF D e XGBoost D |
| **Experiment** | Agrupamento lógico de execuções | `NSLAB_Logistics_Shanghai_Hyperparameter_Optimization` |
| **Experiment ID** | Identificador do agrupamento | `1` |
| **Run** | Uma execução registrada | `validacao_tecnica_tracking` |
| **Run ID** | Identificador único da execução | `2f6a353579e34aed98c2817dcde0a323` |
| **Parameter** | Valor de configuração registrado | Futuramente, `max_depth` |
| **Metric** | Valor numérico de avaliação | Futuramente, Average Precision |
| **Tag** | Metadado descritivo | `tipo_execucao=validacao_infraestrutura` |
| **Artifact** | Arquivo associado à execução | `validacao/validacao.txt` |
| **Backend Store** | Armazenamento dos metadados das execuções | Banco SQLite `mlflow.db` |
| **Artifact Store** | Local dos arquivos vinculados às execuções | `mlruns/artifacts/` |
| **Tracking URI** | Endereço pelo qual o cliente acessa o backend | `sqlite:///C:/.../mlflow.db` |
| **Model Registry** | Gerenciamento e versionamento de modelos registrados | Planejado para depois da seleção do modelo |
| **Monitoring / Drift** | Acompanhamento de comportamento e mudanças dos dados | Fora do escopo deste laboratório |

**Experiment não é Run.** Um experimento contém diversas runs. Cada run pode registrar parâmetros, métricas e artefatos diferentes. Uma run de validação técnica não representa um modelo treinado.

**Run ID não é Experiment ID.** O `experiment_id` identifica o grupo; o `run_id` identifica uma execução individual. Usaremos o `run_id` como chave de integração com os resultados analíticos em Parquet.

### 2.2 Tracking local versus Databricks

No **Databricks**, a experiência com MLflow pode estar integrada ao workspace, com recursos gerenciados e interfaces prontas para experimentos e modelos. No **laboratório local**, precisamos definir explicitamente onde ficam os metadados, os artefatos e como a interface acessa esses dados. Os conceitos centrais — experiment, run, parâmetros, métricas e artefatos — continuam os mesmos. **Não é necessário utilizar Databricks para começar a usar MLflow.** A integração com Databricks será uma etapa posterior do projeto.

### 2.3 O que o MLflow não substitui

MLflow Tracking não substitui o código-fonte, o Git, os datasets de origem, a documentação das regras de negócio nem os arquivos analíticos destinados ao Power BI. Ele complementa esses elementos, associando evidências às execuções. O registro de uma métrica no MLflow não demonstra, por si só, que ela foi calculada corretamente: a validação metodológica continua necessária.

---

## 3. Arquitetura escolhida

```text
notebooks/05_otimizacao_hiperparametros_shanghai.ipynb
                   |
                   | cliente Python MLflow
                   v
             MLflow Tracking
              /           \
             v             v
   Backend Store       Artifact Store
   mlruns/mlflow.db    mlruns/artifacts/
      (SQLite)         (arquivos por run)

Resultados analíticos complementares, posteriormente:
  data/ml_results/experimentos.parquet
  data/ml_results/otimizacao_cv.parquet
  data/ml_results/metricas_operacionais.parquet
```

**SQLite** foi escolhido por oferecer persistência local sem instalar um servidor de banco de dados adicional. O banco contém os registros estruturados. **Artefatos** são arquivos armazenados separadamente. **Parquet** será utilizado para análise tabular e integração com Power BI; seus arquivos não são criados automaticamente pelo MLflow.

Estrutura adotada:

```text
Logistics_Intelligence_Platform/
├── .venv/
├── .gitignore
├── requirements.txt
├── mlruns/
│   ├── mlflow.db
│   └── artifacts/
├── data/
│   └── ml_results/
└── notebooks/
    └── 05_otimizacao_hiperparametros_shanghai.ipynb
```

**Persistência:** fechar o VS Code ou o processo Python não apaga os registros, desde que o banco e os arquivos permaneçam no disco. **Portabilidade:** os caminhos absolutos locais podem precisar de adaptação em outra máquina. **Backup:** ignorar o Git não constitui backup; o banco e os artefatos precisam ser preservados em conjunto por um procedimento de cópia apropriado, preferencialmente com o servidor parado.

---

## 4. Histórico detalhado das etapas executadas

> **Como estudar:** leia primeiro o objetivo de cada etapa, depois execute mentalmente os comentários numerados junto ao código. Em seguida, compare a saída registrada com a interpretação. Os blocos representam o histórico do laboratório: não é necessário executá-los novamente no ambiente já configurado.

### 2.4.1 — Inspeção do ambiente

**Objetivo:** confirmar repositório, ambiente Python, dependências e presença do MLflow antes de instalar qualquer pacote.

**Comandos executados (com comentários para reprodução):**

```powershell
# ============================================================
# LOGISTICS-05 — ETAPA 2.4.1
# Código comentado para estudo e reprodução
# ============================================================

# 1. Confirmar o repositório, alterações e último commit
git status --short
git branch --show-current
git log -1 --oneline
# 2. Confirmar a versão e o executável Python da .venv
python --version
python -c "import sys; print('Executável:', sys.executable)"
python -m pip --version
# 3. Verificar se MLflow já está instalado
python -m pip show mlflow
# 4. Inspecionar as bibliotecas do laboratório
python -m pip show scikit-learn xgboost pandas pyarrow
# 5. Procurar a dependência no requirements.txt
Select-String -Path requirements.txt -Pattern "mlflow" -ErrorAction SilentlyContinue
```

**Resultados observados:** branch `main`, commit `6968e93`, Python `3.13.16` dentro da `.venv`, pip `26.2.1`; `scikit-learn 1.9.1`, `xgboost 3.4.1`, `pandas 2.3.3`, `pyarrow 26.0.0`. `pip show mlflow` retornou `Package(s) not found: mlflow`; não havia entrada do MLflow no `requirements.txt`.

**Arquivos não rastreados já presentes:** dois PDFs de documentação em `docs/` e o notebook `05_otimizacao_hiperparametros_shanghai.ipynb`. Foram preservados. **Conclusão:** ambiente virtual correto; instalação necessária.


**Leitura didática:** `git status --short` mostra arquivos alterados e não rastreados; `git branch --show-current` informa a branch; `git log -1 --oneline` informa o commit mais recente. `sys.executable` confirma qual interpretador executa o código. `pip show` consulta pacotes instalados sem alterar o ambiente. `Select-String` procura texto no arquivo de dependências.

### 2.4.2 — Instalação e validação do MLflow

**Objetivo:** instalar o MLflow somente na `.venv` e confirmar importação e consistência das dependências.

```powershell
# ============================================================
# LOGISTICS-05 — ETAPA 2.4.2
# Código comentado para estudo e reprodução
# ============================================================

# 1. Instalar MLflow na .venv do projeto
python -m pip install mlflow
# 2. Confirmar a versão disponível na linha de comando
python -m mlflow --version
# 3. Validar que o Python consegue importar o pacote
python -c "import mlflow; print('MLflow:', mlflow.__version__)"
# 4. Conferir conflitos declarados entre dependências
python -m pip check
```

**Resultados observados:** MLflow `3.18.0` instalado e importado; `pip check` retornou `No broken requirements found.` O aviso de atualização do pip não exigiu ação. **Conceito:** `python -m pip` usa o gerenciador de pacotes associado ao interpretador selecionado, reduzindo o risco de instalar fora da `.venv`. O `pip check` verifica conflitos declarados de dependências, mas não substitui testes funcionais.


**Leitura didática:** instalar um pacote não significa que seu import funcionará; por isso validamos a instalação em três níveis: gerenciador de pacotes, módulo Python e integridade das dependências. O aviso de nova versão do pip não significa que a instalação falhou.

### 2.4.3 — Registro em requirements.txt

**Objetivo:** tornar explícita a dependência principal utilizada pelo projeto.

**Alteração realizada:** adicionada a linha:

```text
mlflow==3.18.0
```

**Validação:**

```powershell
# ============================================================
# LOGISTICS-05 — ETAPA 2.4.3
# Código comentado para estudo e reprodução
# ============================================================

# 1. Confirmar que a versão foi registrada
Select-String -Path requirements.txt -Pattern "^mlflow=="
# 2. Inspecionar exatamente a alteração feita
git diff -- requirements.txt
# 3. Validar whitespace e formatação
git diff --check
# 4. Conferir os arquivos modificados e não rastreados
git status --short
```

**Resultado:** linha 11 do `requirements.txt`; diff mostrou apenas a nova dependência. `git diff --check` não apontou problemas. **Conceito:** fixar a versão direta melhora a reprodutibilidade; isso não é equivalente a congelar todas as dependências transitivas.


**Leitura didática:** `==` fixa a versão direta do MLflow. O `git diff` é uma revisão antes do commit; `git diff --check` procura problemas de whitespace. O comando `git status` não publica nada: apenas mostra o estado local.

### 2.4.4 — Decisão de arquitetura (planejamento)

**Objetivo:** escolher um backend e um local para artefatos antes de criar registros.

**Decisão aprovada:** SQLite em `mlruns/mlflow.db`, artefatos em `mlruns/artifacts/`, e arquivos analíticos futuros em `data/ml_results/`. Model Registry somente após seleção do modelo; monitoramento e drift em outro laboratório. **Não houve execução técnica nessa subetapa.**

### 2.4.5 — Criação dos diretórios

**Objetivo:** preparar a estrutura sem sobrescrever diretórios existentes.

```powershell
# ============================================================
# LOGISTICS-05 — ETAPA 2.4.5
# Código comentado para estudo e reprodução
# ============================================================

# 1. Listar os três diretórios planejados
$diretorios = @(
    "mlruns",
    "mlruns/artifacts",
    "data/ml_results"
)
# 2. Criar somente os diretórios ausentes
foreach ($diretorio in $diretorios) {
    if (-not (Test-Path $diretorio)) {
        New-Item -ItemType Directory -Path $diretorio | Out-Null
        Write-Host "CRIADO: $diretorio"
    }
    else {
        Write-Host "EXISTENTE: $diretorio"
    }
}
# 3. Confirmar a existência de cada diretório
foreach ($diretorio in $diretorios) {
    Write-Host "$diretorio : $(Test-Path $diretorio)"
}
# 4. Verificar o estado do Git
git status --short
```

**Resultados:** os três diretórios foram criados; todas as verificações `Test-Path` retornaram `True`. **Conceito:** o Git não versiona diretórios vazios; por isso a criação não alterou a saída do status.


**Leitura didática:** o array PowerShell `@(...)` guarda os caminhos. `foreach` percorre cada item. `Test-Path` devolve verdadeiro ou falso; `New-Item -ItemType Directory` cria pastas; `Out-Null` suprime a saída detalhada da criação.

### 2.4.6 — Proteção no .gitignore

**Objetivo:** impedir que banco, artefatos e resultados locais sejam incluídos no GitHub.

**Inspeção prévia:**

```powershell
# ============================================================
# LOGISTICS-05 — ETAPA 2.4.6
# Código comentado para estudo e reprodução
# ============================================================

# 1. Ler as regras existentes, sem sobrescrevê-las
Get-Content .gitignore
# 2. Verificar se o banco SQLite já está ignorado
git check-ignore -v --no-index mlruns/mlflow.db
# 3. Verificar se os artefatos já estão ignorados
git check-ignore -v --no-index mlruns/artifacts/modelo_teste.pkl
# 4. Verificar se os resultados Parquet já estão ignorados
git check-ignore -v --no-index data/ml_results/experimentos.parquet
# 5. Confirmar que o .gitignore é rastreado pelo Git
git ls-files .gitignore
```

**Resultado inicial:** nenhuma regra correspondia aos três caminhos. O `.gitignore` já era versionado e possuía regras para `.venv`, cache Python, VS Code, `.env`, datasets RAW/Silver e cache do Power BI.

**Comando PowerShell utilizado para acrescentar as regras (após inspeção):**

```powershell
# 1. Acrescentar ao final do arquivo, sem sobrescrever as regras anteriores
@"

# MLflow - banco de dados e artefatos locais
/mlruns/

# Resultados analiticos de Machine Learning
/data/ml_results/
"@ | Add-Content -Path .gitignore -Encoding utf8

# 2. Exibir o arquivo atualizado
Get-Content .gitignore

# 3. Verificar os três caminhos que devem ser ignorados
git check-ignore -v --no-index mlruns/mlflow.db
git check-ignore -v --no-index mlruns/artifacts/modelo_teste.pkl
git check-ignore -v --no-index data/ml_results/experimentos.parquet

# 4. Revisar o diff, a formatação e o status
git diff -- .gitignore
git diff --check
git status --short
```

**Regras acrescentadas:**

```gitignore
# MLflow - banco de dados e artefatos locais
/mlruns/

# Resultados analiticos de Machine Learning
/data/ml_results/
```

**Validação:** `git check-ignore -v --no-index` confirmou `/mlruns/` na linha 24 para banco e artefatos, e `/data/ml_results/` na linha 27 para Parquet. `git diff --check` não encontrou erro. Houve aviso de conversão LF → CRLF, relacionado a quebras de linha no Windows, sem impacto identificado nas regras. **Conceito:** `.gitignore` impede inclusão futura de arquivos não rastreados; não apaga dados nem substitui backup.


**Leitura didática:** o prefixo `/` ancora a regra na raiz do repositório. A regra `/mlruns/` cobre o banco e os artefatos abaixo dessa pasta. `git check-ignore -v` informa qual linha da configuração causou a exclusão. O aviso LF/CRLF observado não representou falha de validação.

### 2.4.7 — Inicialização do Backend Store SQLite

**Objetivo:** criar o banco persistente e suas tabelas internas, sem treinar modelos.

```powershell
# ============================================================
# LOGISTICS-05 — ETAPA 2.4.7
# Código comentado para estudo e reprodução
# ============================================================

# 1. Montar o caminho absoluto do arquivo SQLite
$dbPath = (Join-Path (Get-Location).Path "mlruns\mlflow.db")
# 2. Construir uma URI SQLite com barras normais
$trackingUri = "sqlite:///" + ($dbPath -replace "\\", "/")
# 3. Exibir a URI para conferência
Write-Host "Tracking URI: $trackingUri"
# 4. Disponibilizar a URI ao processo Python neste terminal
$env:MLFLOW_TRACKING_URI = $trackingUri
# 5. Conectar ao backend e consultar os experimentos
python -c "import mlflow; print('Tracking URI:', mlflow.get_tracking_uri()); print('Experimentos:', len(mlflow.search_experiments()))"
# 6. Confirmar a criação física do banco
Get-Item "mlruns/mlflow.db" | Select-Object Name, Length, LastWriteTime
# 7. Verificar a regra do .gitignore
git check-ignore -v --no-index mlruns/mlflow.db
```

**Resultados:**

```text
Tracking URI: sqlite:///C:/Users/nslab/Documents/labs/Logistics_Intelligence_Platform/mlruns/mlflow.db
INFO ... Creating initial MLflow database tables...
INFO ... Updating database tables
Experimentos: 1
mlflow.db  1032192 bytes  19/11/2026 11:45:47
.gitignore:24:/mlruns/  mlruns/mlflow.db
```

**Interpretação:** `mlflow.search_experiments()` acessou o backend e provocou sua inicialização. A contagem `1` corresponde ao experimento padrão `Default` criado pelo MLflow; não representa um treinamento. A variável `$env:MLFLOW_TRACKING_URI` vale para a sessão PowerShell em que foi definida, não para todas as sessões futuras.


**Leitura didática:** `Join-Path` compõe o caminho no Windows. `-replace "\", "/"` normaliza as barras para a URI SQLite. O prefixo `sqlite:///` indica o backend de banco local. A variável `$env:MLFLOW_TRACKING_URI` vale no ambiente do terminal atual. A consulta `search_experiments()` inicializou as tabelas do MLflow; o experimento padrão não é um treinamento.

### 2.4.8 — Criação do experimento e configuração do Artifact Store

**Objetivo:** cadastrar um agrupamento específico para a otimização, apontando para o armazenamento local de artefatos.

```python
# ============================================================
# LOGISTICS-05 — ETAPA 2.4.8
# Código comentado para estudo e reprodução
# ============================================================

# 1. Importar bibliotecas e identificar a raiz do projeto
from pathlib import Path
import mlflow

# 2. Definir o banco SQLite e a pasta de artefatos
raiz_projeto = Path.cwd()
caminho_banco = raiz_projeto / "mlruns" / "mlflow.db"
caminho_artefatos = raiz_projeto / "mlruns" / "artifacts"
# 3. Configurar a URI de rastreamento
tracking_uri = f"sqlite:///{caminho_banco.as_posix()}"
mlflow.set_tracking_uri(tracking_uri)

# 4. Definir o nome do experimento do laboratório
nome_experimento = "NSLAB_Logistics_Shanghai_Hyperparameter_Optimization"
# 5. Consultar o cadastro para evitar duplicidade
experimento = mlflow.get_experiment_by_name(nome_experimento)
# 6. Criar o experimento apenas se necessário
if experimento is None:
    experiment_id = mlflow.create_experiment(
        name=nome_experimento,
        artifact_location=caminho_artefatos.as_uri()
    )
    print("Experimento criado.")
else:
    experiment_id = experimento.experiment_id
    print("Experimento já existente.")

# 7. Recuperar o cadastro persistido e mostrar os campos
experimento = mlflow.get_experiment(experiment_id)
print("Tracking URI:", mlflow.get_tracking_uri())
print("Experiment ID:", experimento.experiment_id)
print("Nome:", experimento.name)
print("Artifact URI:", experimento.artifact_location)
print("Lifecycle Stage:", experimento.lifecycle_stage)
```

**Resultado observado:**

```text
Experimento criado.
Experiment ID: 1
Nome: NSLAB_Logistics_Shanghai_Hyperparameter_Optimization
Artifact URI: file:///C:/Users/nslab/Documents/labs/Logistics_Intelligence_Platform/mlruns/artifacts
Lifecycle Stage: active
```

**Conceitos:** `get_experiment_by_name` permite reutilizar o cadastro e evitar duplicações; `create_experiment` grava a definição e o caminho-base de artefatos. `Path.as_uri()` produz uma URI de arquivo adequada ao sistema. O `experiment_id=1` identifica o experimento criado; `Default` é separado. Nenhuma run foi criada nesta etapa.


**Leitura didática:** `Path.cwd()` retorna a pasta de execução; `/` em objetos `Path` une componentes de caminhos; `as_posix()` usa barras `/` e `as_uri()` produz uma URI `file:///...`. `get_experiment_by_name()` consulta o cadastro; `create_experiment()` grava o novo cadastro apenas se não existir. `lifecycle_stage=active` significa que o experimento não está marcado como excluído.

### 2.4.9 — Primeiro registro de run de validação

**Objetivo:** testar o ciclo completo de registro e leitura sem treinar modelos e sem acessar o conjunto de teste.

```python
# ============================================================
# LOGISTICS-05 — ETAPA 2.4.9
# Código comentado para estudo e reprodução
# ============================================================

# 1. Importar bibliotecas, incluindo diretório temporário
from pathlib import Path
from tempfile import TemporaryDirectory
import mlflow

# 2. Apontar para o banco SQLite já existente
raiz = Path.cwd()
banco = raiz / "mlruns" / "mlflow.db"
mlflow.set_tracking_uri(f"sqlite:///{banco.as_posix()}")
# 3. Recuperar o experimento de otimização
nome_experimento = "NSLAB_Logistics_Shanghai_Hyperparameter_Optimization"
experimento = mlflow.get_experiment_by_name(nome_experimento)
if experimento is None:
    raise RuntimeError("Experimento MLflow não encontrado.")

# 4. Iniciar uma run de validação, sem treinar modelos
with mlflow.start_run(
    experiment_id=experimento.experiment_id,
    run_name="validacao_tecnica_tracking"
) as run:
    mlflow.set_tag("tipo_execucao", "validacao_infraestrutura")
    mlflow.log_param("modelo", "nenhum")
    mlflow.log_metric("metrica_validacao_tecnica", 1.0)
    with TemporaryDirectory() as pasta_temporaria:
        arquivo = Path(pasta_temporaria) / "validacao.txt"
        arquivo.write_text(
            "Validacao tecnica do MLflow Tracking - NSLAB Logistics",
            encoding="utf-8"
        )
        mlflow.log_artifact(str(arquivo), artifact_path="validacao")
    run_id = run.info.run_id

# 8. Recuperar a execução e listar os artefatos
execucao = mlflow.get_run(run_id)
artefatos = mlflow.artifacts.list_artifacts(
    run_id=run_id, artifact_path="validacao"
)
# 9. Exibir dados persistidos e validar com assert
print("Run ID:", run_id)
print("Status:", execucao.info.status)
print("Parametro:", execucao.data.params)
print("Metrica:", execucao.data.metrics)
print("Artifact URI:", execucao.info.artifact_uri)
print("Artefatos:", [item.path for item in artefatos])
assert execucao.info.status == "FINISHED"
assert execucao.data.params["modelo"] == "nenhum"
assert execucao.data.metrics["metrica_validacao_tecnica"] == 1.0
assert any(item.path == "validacao/validacao.txt" for item in artefatos)
```

**Resultados reais:**

```text
Run ID: 2f6a353579e34aed98c2817dcde0a323
Status: FINISHED
Parametro: {'modelo': 'nenhum'}
Metrica: {'metrica_validacao_tecnica': 1.0}
Artifact URI: file:///C:/Users/nslab/Documents/labs/Logistics_Intelligence_Platform/mlruns/artifacts/2f6a353579e34aed98c2817dcde0a323/artifacts
Artefatos: ['validacao/validacao.txt']
VALIDACAO CONCLUIDA COM SUCESSO
```

**Conceitos:** `start_run` abre uma execução; `log_param` registra configuração; `log_metric` registra valor numérico; `set_tag` fornece contexto; `log_artifact` persiste um arquivo; `get_run` e `list_artifacts` permitem recuperar os registros. `TemporaryDirectory` apenas fornece um local temporário para criar o arquivo de origem; o artefato registrado foi copiado para o Artifact Store. **A métrica `1.0` é fictícia e não representa desempenho de ML.**


**Leitura didática:** `with mlflow.start_run(...)` abre e fecha o ciclo da execução. Uma *tag* descreve seu propósito; um *parameter* guarda uma configuração; uma *metric* guarda um número. O `TemporaryDirectory` remove a origem temporária, não o arquivo já copiado ao Artifact Store. `assert` valida condições sem representar uma avaliação de modelo.

### 2.4.10 — Persistência entre processos Python

**Objetivo:** verificar que a run continua disponível após o encerramento do processo que a criou.

```python
# ============================================================
# LOGISTICS-05 — ETAPA 2.4.10
# Código comentado para estudo e reprodução
# ============================================================

# 1. Importar dependências e reconstruir a conexão SQLite
from pathlib import Path
import mlflow

raiz = Path.cwd()
banco = raiz / "mlruns" / "mlflow.db"
mlflow.set_tracking_uri(f"sqlite:///{banco.as_posix()}")
# 2. Informar o run_id criado no processo anterior
run_id = "2f6a353579e34aed98c2817dcde0a323"
# 3. Recuperar a execução pelo identificador
execucao = mlflow.get_run(run_id)
# 4. Recuperar o experimento associado
experimento = mlflow.get_experiment(execucao.info.experiment_id)
# 5. Listar os artefatos persistidos
artefatos = mlflow.artifacts.list_artifacts(
    run_id=run_id, artifact_path="validacao"
)
# 6. Exibir as informações recuperadas
print("Experimento:", experimento.name)
print("Experiment ID:", experimento.experiment_id)
print("Run ID:", execucao.info.run_id)
print("Status:", execucao.info.status)
print("Parametros:", execucao.data.params)
print("Metricas:", execucao.data.metrics)
print("Artefatos:", [item.path for item in artefatos])
# 7. Validar status, parâmetros, métricas e arquivo
assert execucao.info.status == "FINISHED"
assert execucao.data.params["modelo"] == "nenhum"
assert execucao.data.metrics["metrica_validacao_tecnica"] == 1.0
assert any(item.path == "validacao/validacao.txt" for item in artefatos)
```

**Resultado:** a run foi recuperada em novo processo com status `FINISHED`, parâmetro, métrica e artefato corretos; mensagem `PERSISTENCIA VALIDADA COM SUCESSO`. **Interpretação:** os dados estavam persistidos no disco e não apenas em variáveis Python do processo anterior.


**Leitura didática:** a variável `run_id` é reconstruída manualmente em outro processo; `get_run(run_id)` consulta o SQLite, não a memória da execução anterior. O `experiment_id` da run permite recuperar o agrupamento correspondente. A listagem de artefatos confirma que os arquivos persistem independentemente da sessão anterior.

### 2.4.11 — Interface gráfica local

**Objetivo:** visualizar o experimento, a run e os artefatos no navegador.

```powershell
# ============================================================
# LOGISTICS-05 — ETAPA 2.4.11
# Código comentado para estudo e reprodução
# ============================================================

# 1. Definir o caminho do banco SQLite
$dbPath = Join-Path (Get-Location).Path "mlruns\mlflow.db"
# 2. Construir o Tracking URI
$trackingUri = "sqlite:///" + ($dbPath -replace "\\", "/")
# 3. Iniciar a interface local do MLflow na porta 5000
python -m mlflow ui --backend-store-uri $trackingUri --host 127.0.0.1 --port 5000
```

**Acesso:** `http://127.0.0.1:5000`.

**Saída observada:** Uvicorn iniciou em `127.0.0.1:5000`; a interface abriu no navegador e o experimento `1` apareceu. O MLflow 3.18.0 apresentou inicialmente a área **GenAI**, com uma tela voltada a traces. Para localizar as runs de classificadores tradicionais, foi necessário selecionar **Model training**, no canto superior esquerdo.

Na página da run `validacao_tecnica_tracking` foram vistos: status **Finished**, métrica `metrica_validacao_tecnica=1`, parâmetro `modelo=nenhum`, tag `tipo_execucao=validacao_infraestrutura`, Experiment ID `1` e o Run ID esperado. Na aba **Artifacts**, a pasta `validacao/` continha `validacao.txt`, com **54 bytes** e conteúdo:

```text
Validacao tecnica do MLflow Tracking - NSLAB Logistics
```

**Avisos do terminal:**

- `MLflow job backend does not support Windows system`: o servidor iniciou sem suporte ao backend de execução de jobs gerenciados. Isso **não impediu** o Tracking usado no laboratório.
- `StarletteDeprecationWarning`: aviso de API depreciada em dependência da interface; o servidor continuou funcional.
- Algumas requisições de interface retornaram `404` para um endpoint de usuário, mas as operações relevantes de experimentos/runs responderam e a interface funcionou. Não há evidência de falha no Tracking.

**Como operar:** mantenha o terminal que executa `mlflow ui` aberto enquanto usa a interface. Para interromper, use `Ctrl+C` nesse terminal. Em outra sessão, repita o comando a partir da raiz do projeto. A interface não precisa estar aberta para que scripts Python registrem dados diretamente no SQLite local.

**Observação importante sobre a URI:** o comando `mlflow ui` aponta para o mesmo backend SQLite, enquanto o notebook usa `mlflow.set_tracking_uri(...)`. Ambos acessam os mesmos registros. Não se deve confundir a URL HTTP da interface com a URI SQLite do backend.


**Leitura didática:** `--backend-store-uri` indica o SQLite existente; `--host 127.0.0.1` limita o acesso à máquina local; `--port 5000` escolhe a porta HTTP. O navegador abre a interface, mas o notebook pode continuar gravando diretamente no SQLite sem manter essa interface aberta. Para parar o servidor, use `Ctrl+C` no terminal que o iniciou.

### 2.4.12 — Configuração reutilizável no notebook 05

**Objetivo:** eliminar dependência da sessão PowerShell e reutilizar o banco/experimento sem criar novos registros.

**Markdown inicial:** registrar objetivo, arquitetura, procedimento e critérios de validação antes do código.

**Célula Python executada:**

```python
# ============================================================
# LOGISTICS-05 — ETAPA 2.4.12
# Código comentado para estudo e reprodução
# ============================================================

# 1. Importar as bibliotecas
from pathlib import Path
import mlflow

# 2. Identificar a raiz do repositório pelo requirements.txt
diretorio_atual = Path.cwd().resolve()
if (diretorio_atual / "requirements.txt").is_file():
    raiz_projeto = diretorio_atual
elif (diretorio_atual.parent / "requirements.txt").is_file():
    raiz_projeto = diretorio_atual.parent
else:
    raise FileNotFoundError("Raiz do projeto não identificada.")

# 3. Definir caminhos persistentes de banco e artefatos
caminho_banco = raiz_projeto / "mlruns" / "mlflow.db"
caminho_artefatos = raiz_projeto / "mlruns" / "artifacts"
# 4. Confirmar que o banco existente está acessível
if not caminho_banco.is_file():
    raise FileNotFoundError(f"Banco MLflow não encontrado: {caminho_banco}")

# 5. Configurar a URI SQLite
tracking_uri = f"sqlite:///{caminho_banco.as_posix()}"
mlflow.set_tracking_uri(tracking_uri)
# 6. Recuperar o experimento previamente criado
nome_experimento = "NSLAB_Logistics_Shanghai_Hyperparameter_Optimization"
experimento = mlflow.get_experiment_by_name(nome_experimento)
if experimento is None:
    raise RuntimeError("Experimento MLflow não encontrado.")

# 7. Validar ID, status e URI de artefatos
artifact_uri_esperada = caminho_artefatos.as_uri()
assert experimento.experiment_id == "1"
assert experimento.lifecycle_stage == "active"
assert experimento.artifact_location == artifact_uri_esperada
# 8. Guardar o experiment_id para as próximas células
experiment_id = experimento.experiment_id

# 9. Exibir o resumo da configuração
print("CONFIGURAÇÃO MLFLOW TRACKING")
print("Versão MLflow:", mlflow.__version__)
print("Raiz do projeto:", raiz_projeto)
print("Tracking URI:", mlflow.get_tracking_uri())
print("Experiment ID:", experiment_id)
print("Experimento:", experimento.name)
print("Artifact URI:", experimento.artifact_location)
print("Status:", experimento.lifecycle_stage)
print("Configuração validada com sucesso.")
```

**Resultados reais:** versão `3.18.0`, raiz do projeto correta, URI SQLite local, Experiment ID `1`, experimento esperado, URI de artefatos esperada, status `active` e mensagem `Configuração validada com sucesso.`

**Conceitos:** `Path.cwd()` identifica o diretório corrente; `requirements.txt` é usado como marcador simples para a raiz do repositório; `set_tracking_uri` define o backend; `get_experiment_by_name` apenas recupera o cadastro; `assert` verifica as condições; `experiment_id` fica disponível para as próximas células do notebook. **Limitação conhecida:** a descoberta de raiz considera apenas o diretório atual e seu pai; se o notebook for executado de outro local, a lógica precisará ser adaptada.

**Conclusão:** configuração reutilizável validada sem treinamento e sem novas runs. A quarta célula Markdown, com resultados e interpretação, foi fornecida na conversa para inclusão no notebook; a saída de execução foi confirmada, mas a gravação final dessa célula no arquivo local ainda depende de confirmação do usuário.

---

## 5. Guia de operação: iniciar, consultar e reutilizar


**Leitura didática:** a célula identifica a raiz procurando `requirements.txt` no diretório atual ou no pai. O banco deve existir antes da conexão. `get_experiment_by_name` recupera sem duplicar. Os três `assert` garantem que ID, estado e Artifact Store correspondem ao experimento já aprovado. A variável `experiment_id` será reutilizada no notebook.

### 5.1 Iniciar a interface

A partir da raiz do projeto, com `.venv` ativa:

```powershell
$dbPath = Join-Path (Get-Location).Path "mlruns\mlflow.db"
$trackingUri = "sqlite:///" + ($dbPath -replace "\\", "/")
python -m mlflow ui --backend-store-uri $trackingUri --host 127.0.0.1 --port 5000
```

Abra `http://127.0.0.1:5000`, escolha **Model training**, selecione o experimento e entre em **Runs**. Clique em uma run para consultar **Overview**, métricas, parâmetros, tags e **Artifacts**.

### 5.2 Consultar uma run sem abrir a interface

```python
from pathlib import Path
import mlflow

raiz = Path.cwd()  # executar a partir da raiz do repositório
mlflow.set_tracking_uri(f"sqlite:///{(raiz / 'mlruns' / 'mlflow.db').as_posix()}")
run_id = "2f6a353579e34aed98c2817dcde0a323"
run = mlflow.get_run(run_id)
print(run.info.status)
print(run.data.params)
print(run.data.metrics)
```

O exemplo **consulta** a run de infraestrutura já existente; não cria outra.

### 5.3 O que registrar em runs futuras (planejamento, não executado)

Para cada execução real de otimização, registrar de forma consistente:

- **Identificação:** algoritmo, variante (`Random Forest D` ou `XGBoost D`), etapa, data e `run_id`.
- **Parâmetros:** espaço de busca, parâmetros selecionados, `n_iter=12`, `cv=3`, semente aleatória e configurações relevantes do pipeline.
- **Métricas:** Average Precision de validação cruzada e métricas operacionais Recall@K e Precision@K, identificando claramente partição e capacidade K.
- **Artefatos:** tabelas de resultados de CV, relatórios e arquivos necessários à rastreabilidade.
- **Proveniência:** versão do código, características do dataset e definição do alvo, evitando armazenar dados sensíveis desnecessários.

**Atenção:** registrar as 12 configurações de busca como runs individuais, uma run por algoritmo ou uma hierarquia de runs é uma decisão de desenho que ainda precisa ser tomada. O MLflow não transforma automaticamente cada candidato do `RandomizedSearchCV` em uma run individual apenas porque o experimento foi criado.

### 5.4 Relação entre MLflow e Parquet

Arquivos previstos em `data/ml_results/`:

| Arquivo | Finalidade planejada |
|---|---|
| `experimentos.parquet` | Resumo tabular das execuções e principais métricas |
| `otimizacao_cv.parquet` | Resultados das combinações de hiperparâmetros e validação cruzada |
| `metricas_operacionais.parquet` | Métricas por capacidade operacional K |

Incluir `run_id` nas tabelas que descrevem runs para associar os resultados ao Tracking. Se uma tabela detalhar vários candidatos de CV dentro de uma mesma run, será necessário também um identificador do candidato/configuração; **`run_id` sozinho não identifica cada linha de CV**. Os schemas e a granularidade serão definidos antes de gerar os Parquet. Os arquivos não foram criados nesta etapa.

---

## 6. Reprodutibilidade, segurança e cuidados

1. **Preservar o teste:** os 213.630 registros de teste continuam reservados para a avaliação final. O ajuste de hiperparâmetros não deve utilizar o teste.
2. **Evitar data leakage:** features devem representar apenas informações disponíveis no momento da previsão. Métricas de atraso calculadas após a coleta não podem entrar como preditores antecipados.
3. **Distinguir avaliação e infraestrutura:** a run `validacao_tecnica_tracking` contém um valor fictício (`1.0`) e deve permanecer identificada como validação de infraestrutura, não como benchmark.
4. **Não versionar os dados locais:** `/mlruns/` e `/data/ml_results/` estão no `.gitignore`; notebooks, código, documentação e `requirements.txt` permanecem versionáveis.
5. **Backups:** para preservar resultados entre máquinas, planejar cópia consistente de `mlflow.db` **e** `mlruns/artifacts/`; apenas o Git não basta.
6. **Caminhos locais:** a URI `file:///C:/...` é específica desta máquina; ao migrar o projeto, reavaliar a localização do backend e dos artefatos, especialmente dos experimentos já criados.
7. **Interface local:** `127.0.0.1` restringe o acesso à própria máquina. Não expor o servidor à rede sem planejamento de segurança.
8. **SQLite:** adequado ao laboratório local; para colaboração simultânea e maior escala, avaliar posteriormente um backend gerenciado, com configuração de artefatos apropriada.
9. **Versões:** `requirements.txt` contém `mlflow==3.18.0`; a compatibilidade foi verificada por importação, execução de Tracking e `pip check`, não por uma bateria completa de testes de integração com todos os pacotes.
10. **Commit:** até a última verificação de Git compartilhada, `.gitignore` e `requirements.txt` estavam modificados, e o notebook e dois PDFs ainda não rastreados. A publicação dessas alterações não foi demonstrada nesta etapa.

---

## 7. Checklist final e evidências

| Etapa | Evidência | Status |
|---|---|---|
| 2.4.1 Ambiente | Python, `.venv`, dependências e ausência inicial do MLflow | Validada |
| 2.4.2 Instalação | MLflow 3.18.0; `pip check` sem conflitos | Validada |
| 2.4.3 Dependência | `mlflow==3.18.0` em `requirements.txt` | Validada |
| 2.4.4 Arquitetura | SQLite + artefatos locais + Parquet futuro | Aprovada |
| 2.4.5 Diretórios | Três diretórios criados e verificados | Validada |
| 2.4.6 Gitignore | Caminhos ignorados e diff verificado | Validada |
| 2.4.7 Backend | `mlflow.db` criado; experimento padrão detectado | Validada |
| 2.4.8 Experimento | ID `1`, nome e Artifact URI persistidos | Validada |
| 2.4.9 Run técnica | Run `FINISHED`, parâmetro, métrica, artefato | Validada |
| 2.4.10 Persistência | Recuperação em novo processo Python | Validada |
| 2.4.11 Interface | Model training, Overview e Artifacts conferidos | Validada |
| 2.4.12 Notebook | Conexão reutilizável e asserts aprovados | Validada |

**Conclusão:** a infraestrutura local do MLflow Tracking foi instalada, configurada, testada e integrada ao notebook 05. O experimento de otimização existe e pode receber runs futuras. **Nenhum treinamento ou RandomizedSearchCV foi executado durante a Etapa 2.4.**

## 8. Próximos passos — não executados

1. Conferir a inclusão da célula Markdown final da etapa 2.4.12 no notebook.
2. Revisar alterações do Git e decidir quais arquivos de documentação devem entrar no commit.
3. Definir o desenho de runs para `RandomizedSearchCV`, incluindo a relação entre candidatos de CV, métricas operacionais e Parquet.
4. Executar a otimização somente após aprovação da configuração e do plano de rastreamento.
5. Avaliar Model Registry depois de selecionar o modelo, e deixar Monitoring/Data Drift para laboratório futuro.

---

## 9. Referência rápida de comandos

```powershell
# Ambiente e instalação
python -m mlflow --version
python -m pip check

# Verificação do versionamento
git diff --check
git status --short

# Iniciar interface (a partir da raiz do projeto)
$dbPath = Join-Path (Get-Location).Path "mlruns\mlflow.db"
$trackingUri = "sqlite:///" + ($dbPath -replace "\\", "/")
python -m mlflow ui --backend-store-uri $trackingUri --host 127.0.0.1 --port 5000

# Encerrar a interface: Ctrl+C no terminal que a iniciou
```

```python
# Conectar no Python (a partir da raiz do projeto)
from pathlib import Path
import mlflow
raiz = Path.cwd()
mlflow.set_tracking_uri(f"sqlite:///{(raiz / 'mlruns' / 'mlflow.db').as_posix()}")
experimento = mlflow.get_experiment_by_name(
    "NSLAB_Logistics_Shanghai_Hyperparameter_Optimization"
)
print(experimento.experiment_id, experimento.artifact_location)
```

**Documentação oficial para consulta:** https://mlflow.org/docs/latest/ml/tracking/
**Observação:** a referência oficial é complementar; os resultados e caminhos descritos neste guia são os efetivamente compartilhados durante a execução local do laboratório.
