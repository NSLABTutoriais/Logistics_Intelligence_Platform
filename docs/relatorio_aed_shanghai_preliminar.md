# NSLAB-LOGISTICS-06 — Relatório de consolidação da AED

**Fonte:** notebook executado `02_aed_shanghai(5).ipynb`. **Status:** achados exploratórios, sem validação de SLA oficial.

## Síntese

- 1.424.406 linhas, 19 colunas, 1.425 row groups; nenhuma duplicata nas chaves testadas.
- GPS da coleta ausente em 441.333 registros (30,984%); GPS do aceite ausente em 654.425 (45,944%).
- `ds` coincide com mês/dia de `pickup_time` em 100% das linhas; meses de maio a outubro; ano não está explícito no texto dos timestamps.
- Atraso **provisório** de 2,565% (36.530 registros), definido por `pickup_time > time_window_end`; não é ainda SLA oficial.
- Distribuições com caudas longas e pontos GPS muito distantes, exigindo investigação, não exclusão automática.
- Segmentos geográficos e operacionais apresentam taxas distintas, sem evidência de causalidade.

## Matriz de recomendações preliminares — 19 colunas

| Coluna | Recomendação preliminar | Justificativa e disponibilidade no aceite |
|---|---|---|
| `order_id` | Manter como chave | Único na amostra; não usar diretamente como preditor. |
| `region_id` | Investigar / codificar | 30 regiões; confirmar disponibilidade no aceite. |
| `city` | Manter / avaliar descarte no ETL | Constante em Shanghai; útil como origem em expansões. |
| `courier_id` | Investigar / codificar | 4.494 entregadores; confirmar atribuição e evitar memorização. |
| `accept_time` | Transformar | Disponível no aceite; extrair atributos operacionais, validar ano. |
| `time_window_start` | Transformar | Janela presumivelmente programada; confirmar disponibilidade. |
| `time_window_end` | Transformar | Necessária ao alvo provisório; confirmar regra de SLA. |
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

## Pendências antes do ETL e da modelagem

1. Confrontar todos os significados e códigos com a documentação oficial do LaDe-P.
2. Confirmar o ano, a cobertura e as regras para datas sem ano explícito.
3. Validar a definição operacional de atraso e o instante exato da previsão.
4. Investigar extremos temporais, coordenadas afastadas e ausências de GPS.
5. Definir a estratégia de avaliação que evite vazamento entre entregadores, áreas e períodos, conforme o cenário.
6. Documentar regras de transformação antes de executá-las no ETL; preservar integralmente a fonte bruta.

**Nota metodológica:** as recomendações são preliminares e não significam descarte ou transformação já realizados.
