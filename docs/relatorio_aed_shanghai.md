# NSLAB-LOGISTICS-06 — Relatório de consolidação da AED

**Fonte:** `01_aed_shanghai.ipynb` (resultados históricos). **Status:** AED documentada e três decisões de Data Owner aprovadas para o laboratório; métricas relativas à janela requerida, sem SLA contratual definido.

## Síntese

- 1.424.406 linhas, 19 colunas, 1.425 row groups; nenhuma duplicata nas chaves testadas.
- GPS da coleta ausente em 441.333 registros (30,984%); GPS do aceite ausente em 654.425 (45,944%).
- `ds` coincide com mês/dia de `pickup_time` em 100% das linhas; meses de maio a outubro; ano não está explícito no texto dos timestamps.
- Coletas após o fim da janela requerida de 2,565% (36.530 registros), definido por `pickup_time > time_window_end`; não representa SLA contratual.
- Distribuições com caudas longas e pontos GPS muito distantes, exigindo investigação, não exclusão automática.
- Segmentos geográficos e operacionais apresentam taxas distintas, sem evidência de causalidade.

## Matriz de recomendações preliminares — 19 colunas

| Coluna | Recomendação preliminar | Justificativa e disponibilidade no aceite |
|---|---|---|
| `order_id` | Manter como chave | Identificador no arquivo; documentação usa `package_id`. Equivalência não confirmada; não usar como preditor. |
| `region_id` | Investigar / codificar | 30 regiões; confirmar disponibilidade no aceite. |
| `city` | Manter / avaliar descarte no ETL | Constante em Shanghai; útil como origem em expansões. |
| `courier_id` | Investigar / codificar | 4.494 entregadores; confirmar atribuição e evitar memorização. |
| `accept_time` | Transformar | Disponível no aceite; extrair atributos operacionais, validar ano. |
| `time_window_start` | Transformar | Janela presumivelmente programada; confirmar disponibilidade. |
| `time_window_end` | Transformar | Necessária ao alvo provisório; interpretar janela requerida. |
| `lng` | Investigar / transformar | Coordenada de referência completa; confirmar semântica. |
| `lat` | Investigar / transformar | Par de longitude; conferir referencial e plausibilidade. |
| `aoi_id` | Investigar / codificar | 5.867 áreas; alta cardinalidade. |
| `aoi_type` | Investigar / codificar | 15 códigos; falta definição semântica oficial. |
| `pickup_time` | Manter para construir alvo; excluir de preditores no aceite | Evento futuro, vazamento direto se usado como feature. |
| `pickup_gps_time` | Manter para auditoria; excluir de preditores no aceite | Evento posterior e 30,984% ausente. |
| `pickup_gps_lng` | Manter para auditoria; excluir de preditores no aceite | Posição futura e incompleta. |
| `pickup_gps_lat` | Manter para auditoria; excluir de preditores no aceite | Posição futura e incompleta. |
| `accept_gps_time` | Investigar | 45,944% ausente; validar captura e latência. |
| `accept_gps_lng` | Investigar / transformar | Disponibilidade operacional incerta; valores afastados. |
| `accept_gps_lat` | Investigar / transformar | Disponibilidade operacional incerta; valores afastados. |
| `ds` | Manter para auditoria / avaliar redundância | MMDD consistente com pickup_time; risco de vazamento se conhecido só após coleta. |

## Pontos de controle para o ETL e a modelagem

1. Confrontar todos os significados e códigos com a documentação oficial do LaDe-P.
2. Período oficial: maio a outubro de 2022 (CST, UTC+8); registrar ano técnico 2000 usado na AED e 2022 no ETL.
3. Instante da previsão **aprovado: aceite**; atraso relativo ao fim da janela **aprovado** como regra analítica, sem pressupor SLA contratual.
4. Investigar extremos temporais, coordenadas afastadas e ausências de GPS.
5. Definir a estratégia de avaliação que evite vazamento entre entregadores, áreas e períodos, conforme o cenário.
6. Documentar regras de transformação antes de executá-las no ETL; preservar integralmente a fonte bruta.

**Nota metodológica:** as recomendações são preliminares e não significam descarte ou transformação já realizados.


---

## Dicionário de dados — LaDe-P Shanghai (19 campos)

**Fontes consultadas:** [Hugging Face LaDe-P](https://huggingface.co/datasets/Cainiao-AI/LaDe-P), [Hugging Face LaDe](https://huggingface.co/datasets/Cainiao-AI/LaDe) e [GitHub dos autores](https://github.com/wenhaomin/LaDe).

**Critério de evidência:** a coluna «Significado» traduz a documentação oficial, exceto `order_id`, cuja nomenclatura foi observada no arquivo distribuído. Formatos refletem o arquivo e a AED. Observações não confirmadas são explicitadas como pendências.

**Período documentado:** 2022-05-01 a 2022-10-31, China Standard Time (UTC+8). O ano 2022 é sustentado pelo projeto dos autores, embora omitido nos timestamps individuais. A AED histórica usou 2000 como ano técnico para cálculos; o ETL adotou 2022. Não misturar anos técnicos sem registrar a distinção.

**SLA:** a documentação consultada descreve uma janela requerida, mas não define SLA contratual. Neste laboratório, `pickup_time > time_window_end` é denominado *coleta após o limite* ou *atraso relativo ao fim da janela*. Coleta anterior a `time_window_start` será descrita como *antes da janela*, sem classificação automática como falha.

| Campo | Significado | Formato observado | Observações / limites |
|---|---|---|---|
| `order_id` | Identificador da ordem no arquivo distribuído; a documentação textual usa package_id | ID inteiro | Divergência de nomenclatura; equivalência exata não demonstrada |
| `region_id` | Identificador da região | ID inteiro | Códigos de regiões não decodificados |
| `city` | Cidade da operação | Texto | Shanghai neste subconjunto |
| `courier_id` | Identificador do entregador responsável | ID inteiro | Momento da atribuição não especificado |
| `accept_time` | Horário em que o entregador aceita a tarefa | MM-DD HH:MM:SS | Ano omitido no campo; período documentado em 2022 |
| `time_window_start` | Início da janela requerida | MM-DD HH:MM:SS | Não há definição de SLA contratual |
| `time_window_end` | Fim da janela requerida | MM-DD HH:MM:SS | Base para desvio e atraso relativo ao limite |
| `lng` | Longitude da parada associada à tarefa | Float | Sistema geodésico não explicitado |
| `lat` | Latitude da parada associada à tarefa | Float | Sistema geodésico não explicitado |
| `aoi_id` | Identificador da área de interesse (AOI) | ID inteiro | Códigos sem legenda geográfica |
| `aoi_type` | Categoria/tipo da área de interesse | Inteiro categórico | Significado dos códigos não documentado |
| `pickup_time` | Horário em que o entregador realiza a coleta | MM-DD HH:MM:SS | Evento posterior ao aceite; não usar como preditor no aceite |
| `pickup_gps_time` | Horário do ponto GPS mais próximo ao evento de coleta | MM-DD HH:MM:SS | Pode estar ausente; não é necessariamente igual à coleta |
| `pickup_gps_lng` | Longitude GPS associada ao evento de coleta | Float | Pode estar ausente |
| `pickup_gps_lat` | Latitude GPS associada ao evento de coleta | Float | Pode estar ausente |
| `accept_gps_time` | Horário do ponto GPS mais próximo ao evento de aceite | MM-DD HH:MM:SS | Pode estar ausente; latência de disponibilidade desconhecida |
| `accept_gps_lng` | Longitude GPS associada ao aceite | Float | Pode estar ausente |
| `accept_gps_lat` | Latitude GPS associada ao aceite | Float | Pode estar ausente |
| `ds` | Data da coleta, observada como código mês/dia | Inteiro MMDD | 100% consistente com pickup_time na AED; pós-evento para previsão no aceite |

### Decisões aprovadas pelo Data Owner — simulação do laboratório (12/11/2026)

As regras abaixo foram **aprovadas para o laboratório**, não atribuídas à documentação oficial do LaDe-P.

| ID | Tema | Decisão aprovada | Implicação |
|---|---|---|---|
| DO-01 | Janela requerida | `ANTES_DA_JANELA` se `pickup_time < time_window_start`; `DENTRO_DA_JANELA` se `time_window_start <= pickup_time <= time_window_end`; `APOS_A_JANELA` se `pickup_time > time_window_end` | Três categorias descritivas; atraso somente após o limite final |
| DO-02 | Momento da previsão | Previsão no instante `accept_time` | Atributos posteriores ao aceite não poderão ser usados como preditores |
| DO-02A | Disponibilidade | Considerar janela requerida, região, localização de referência e identificação do entregador disponíveis no aceite | Premissa operacional do laboratório, não confirmada empiricamente na fonte |
| DO-03 | Granularidade | Uma tarefa de coleta por `order_id` | Preservar 1.424.406 linhas e todas as 19 colunas de origem na Silver; não presumir equivalência com pacote físico |
| DO-04 | Nomenclatura | Não empregar SLA para a janela requerida | Usar desvio/atraso relativo ao fim da janela; sem inferir penalidade ou obrigação contratual |

**Indicadores a materializar na Silver:** duração da janela, tempo entre aceite e coleta, tempo entre aceite e início/fim da janela, diferença assinada entre coleta e fim da janela, minutos após o limite (mínimo zero), minutos antes do início, classificação em três categorias e flags de consistência. As métricas serão calculadas sem descartar registros; valores ausentes ou cronologias incoerentes devem permanecer auditáveis.

**Prevenção de vazamento:** `pickup_time`, GPS da coleta e indicadores derivados do desfecho são informações posteriores ao aceite. Podem permanecer na Silver para auditoria, BI e definição do alvo, mas não entrarão como preditores no cenário aprovado. A disponibilidade real do GPS do aceite e de `ds` continua incerta; não os liberar automaticamente como features.

**Limitações ainda documentadas (não bloqueantes para a Silver):** não há campos explícitos de criação do pedido, autorização, prioridade, motivo da janela, agrupamento em viagens ou relação entre `order_id` e `package_id`; códigos de `aoi_type` e referencial das coordenadas carecem de legenda/confirmação. Não inventar regras ou semânticas.

**Próxima fase:** implementar o ETL Silver completo em `02_etl_shanghai.ipynb`, preservando o RAW e validando as métricas contra os resultados históricos da AED antes de gravar o Parquet. A modelagem Gold e a escolha final das features pertencem às etapas posteriores.


## Revisão de decisões para Silver — 12/11/2026

**Retificação metodológica:** investigações e distribuições pertencem à AED; a Silver executa transformações e validações de contrato. A seção 13 do notebook AED acrescenta a contagem das três categorias, ainda **pendente de execução**.

**Colunas:** a autorização do Data Owner permite excluir colunas *comprovadamente dispensáveis*, mas não determina exclusão automática. Neste recorte, `city` identifica a origem, `ds` apoia auditoria e os campos GPS apoiam qualidade/rastreabilidade; por isso, **nenhuma das 19 colunas está comprovadamente dispensável apenas com as evidências atuais**. A Silver inicial preserva as 19, e qualquer descarte futuro exige justificativa por coluna e revisão da regra DO-03.

**Tratamentos aprovados:** converter os seis campos de data para 2022, criar métricas temporais e classes, preservar nulos e sinalizar incoerências por flags. Não imputar coordenadas, não corrigir cronologias sem evidência e não eliminar linhas automaticamente.

**Controles do ETL:** verificação mínima de chave, esquema, reconciliação com resultados históricos e releitura do Parquet não constituem nova AED. A medição das novas três classes será documentada somente após execução.

**Status:** notebooks preparados, não executados nesta revisão; Silver ainda não persistida ou validada com o RAW local.
