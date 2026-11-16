# NSLAB Logistics Intelligence Platform
## Relatório de execução e validação — ETL Silver Shanghai

**Data de consolidação:** 16/11/2026  
**Fonte de evidências:** `02_etl_shanghai.ipynb` (saídas de execução salvas), confrontado com `01_aed_shanghai.ipynb` e `relatorio_aed_shanghai.md`.  
**Dataset:** Cainiao-AI LaDe-P, subconjunto Pickup Shanghai.  
**Escopo:** transformação RAW → Silver, preservação de rastreabilidade, validação e persistência.  
**Status:** execução, gravação e releitura aprovadas **nas saídas registradas do notebook**. A presença atual do arquivo no computador de origem não foi verificada independentemente.

### 1. Objetivo e premissas

Materializar a camada Silver para consumo analítico, preservando o RAW original e as 19 colunas de origem. A unidade analítica aprovada é uma tarefa de coleta por `order_id`. O instante da futura previsão é `accept_time`. A janela requerida é uma referência operacional; o atraso em relação a `time_window_end` **não é um SLA contratual**.

### 2. Ambiente, entradas e saídas

| Item | Evidência registrada |
|---|---|
| Ambiente | Windows, `.venv`, Python 3.13.16, pandas 2.3.3, PyArrow 26.0.0 |
| RAW | `data/raw/pickup/pickup_sh-00000-of-00001-79fabe8088e723a2.parquet` |
| Tamanho RAW | 98.191.560 bytes |
| Silver | `data/silver/pickup_shanghai_silver.parquet` |
| Tamanho Silver gravado | 56.461.223 bytes |
| Registros | 1.424.406 antes e depois |
| Colunas | 19 de origem + 22 derivadas = 41 |
| Colunas descartadas | 0 |

Os caminhos são relativos à raiz do repositório. A execução registrada ocorreu no computador local; não foi executada novamente durante a elaboração deste relatório.

### 3. Transformações executadas

**Conversão temporal.** Seis campos foram convertidos para `datetime64[ns]`, prefixando o ano técnico **2022** aos valores de mês-dia: `accept_time`, `pickup_time`, `time_window_start`, `time_window_end`, `accept_gps_time`, `pickup_gps_time`. Foram registradas **zero falhas de parsing** e preservadas as quantidades de nulos: 654.425 em `accept_gps_time` e 441.333 em `pickup_gps_time`; os quatro campos temporais essenciais não tinham nulos. A AED histórica utilizou 2000 como ano técnico; a Silver usa 2022, conforme decisão documentada.

**Sete métricas operacionais (minutos).** `janela_min`, `aceite_ate_coleta_min`, `aceite_ate_inicio_janela_min`, `aceite_ate_fim_janela_min`, `desvio_fim_janela_min` (assinado), `minutos_apos_limite` (mínimo zero) e `minutos_antes_inicio` (mínimo zero). As diferenças são calculadas vetorialmente, sem imputação automática.

**Classificação da janela.** Para registros com início, fim e coleta disponíveis e janela não invertida:

- `ANTES_DA_JANELA`: coleta anterior ao início;
- `DENTRO_DA_JANELA`: coleta entre início e fim, inclusive;
- `APOS_A_JANELA`: coleta posterior ao fim.

Em caso de entrada incompleta ou janela invertida, a classificação fica ausente. A flag booleana `coleta_apos_limite` usa a comparação entre coleta e fim da janela, conforme contrato analítico.

**Treze indicadores de qualidade.** Flags de ausência temporal essencial, janela invertida, coleta antes do aceite, aceite após fim da janela, coleta antes do início, ausência e preenchimento parcial de GPS no aceite e na coleta, coordenadas fora de limites físicos para referência/aceite/coleta, e divergência entre `ds` e mês-dia de `pickup_time`. As flags sinalizam condições; não eliminam nem corrigem registros automaticamente.

### 4. Resultados da classificação

| Categoria | Registros | Percentual aproximado |
|---|---:|---:|
| `DENTRO_DA_JANELA` | 1.027.561 | 72,14% |
| `ANTES_DA_JANELA` | 360.315 | 25,30% |
| `APOS_A_JANELA` | 36.530 | 2,56% |
| **Total** | **1.424.406** | **100%** |

Os valores correspondem às saídas registradas no notebook; `APOS_A_JANELA` é atraso relativo ao limite da janela requerida, não comprovação de descumprimento contratual.

### 5. Reconciliação com a AED

| Indicador | AED | Silver | Resultado |
|---|---:|---:|---|
| Registros | 1.424.406 | 1.424.406 | OK |
| Coletas após o limite | 36.530 | 36.530 | OK |
| Mediana da janela (min) | 120 | 120 | OK |
| Mediana aceite → coleta (min) | 124 | 124 | OK |
| Mediana do desvio ao fim (min) | -88 | -88 | OK |
| GPS de coleta ausente | 441.333 | 441.333 | OK |
| GPS de aceite ausente | 654.425 | 654.425 | OK |
| `ds` divergente | 0 | 0 | OK |

**Resultado registrado:** `Reconciliação AED: OK` (oito verificações concordantes).

### 6. Integridade, persistência e releitura

O notebook confirmou a unicidade e não nulidade de `order_id`, a preservação das 19 colunas originais e de suas contagens de nulos, e a coerência entre a flag de atraso e o desvio calculado. A persistência foi explicitamente autorizada com `AUTORIZAR_GRAVACAO = True`; a rotina escreveu um arquivo temporário, verificou metadados e publicou o Parquet sem sobrescrita automática. A releitura confirmou 1.424.406 linhas, 41 colunas, ordenação do esquema, chave única, tipos temporais e contagem de 36.530 coletas após o limite. **Saída final: `Validação física e lógica após releitura: OK`.**

### 7. Restrições para Gold, BI e Machine Learning

A Silver contém tanto dados potencialmente disponíveis no aceite quanto resultados posteriores à coleta. **Não usar automaticamente todas as 41 colunas como preditores.** São posteriores ao aceite ou derivadas do desfecho, entre outras: `pickup_time`, campos GPS de coleta, `ds`, `classificacao_janela`, `coleta_apos_limite`, `aceite_ate_coleta_min`, `desvio_fim_janela_min`, `minutos_apos_limite` e `minutos_antes_inicio`. Outras flags ou métricas calculadas com informações futuras também exigem exclusão do conjunto de preditores. A disponibilidade efetiva de `accept_gps_time`, `accept_gps_lng` e `accept_gps_lat` no instante do aceite permanece não confirmada.

A futura camada Gold deve ser modelada para necessidades de negócio e consumo, e o conjunto de treino deve ter seleção explícita de alvo, features e estratégia de validação para prevenir *data leakage*. A Silver não é, por si só, uma tabela de treinamento pronta.

### 8. Limitações e pendências documentais

1. As evidências são as saídas salvas do notebook; não houve verificação independente do arquivo no ambiente local nesta revisão.
2. Os textos de resultados/conclusões de diversas etapas do notebook ETL ainda indicam “Pendente de execução local”, apesar de as células apresentarem saídas. Devem ser atualizados para refletir os resultados executados.
3. A conclusão da seção 13 da AED também precisa ser atualizada com a distribuição das três classes.
4. As flags de qualidade foram criadas, mas o notebook não apresenta uma tabela consolidada com a contagem individual de todas as 13 flags; não atribuir contagens não medidas.
5. A semântica dos códigos de AOI, o referencial geográfico e a disponibilidade operacional de determinados atributos continuam com ressalvas documentadas na AED.
6. Ainda faltam revisão de Git, commit e publicação das alterações, sujeitos à conferência do estado do repositório.

### 9. Conclusão e decisão de passagem

**ETL Silver tecnicamente concluído nas evidências executadas:** transformação, reconciliação, integridade, gravação e releitura aprovadas, com preservação integral do RAW e dos registros. O relatório encerra a consolidação documental do ETL, mas não substitui a atualização dos Markdown internos dos notebooks nem a conferência de Git.

**Próxima etapa recomendada:** atualizar os resultados documentais pendentes, conferir `git status` e então definir os contratos de dados Gold e de treinamento. Machine Learning + MLflow pode iniciar após a seleção e validação das features e alvos, sem necessidade de finalizar previamente todas as páginas do Power BI.
