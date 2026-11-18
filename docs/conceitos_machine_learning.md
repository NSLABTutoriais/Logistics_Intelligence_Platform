# Fundamentos de Machine Learning: Classificação, Regressão e Clustering

**Machine Learning (Aprendizado de Máquina)** é um ramo da inteligência artificial em que computadores aprendem padrões a partir de dados para tomar decisões ou fazer previsões sem serem programados explicitamente para cada regra.

As tarefas mais comuns se dividem em **classificação**, **regressão** (ambas de aprendizado supervisionado, onde usamos dados já rotulados) e **clustering** (aprendizado não supervisionado, onde os dados não têm rótulos prévios).

---

## 1. Classificação (Classification)

É a tarefa de prever uma **categoria ou classe discreta** para um dado de entrada. O modelo analisa as características de um item e decide a qual grupo predefinido ele pertence.

### Exemplos Práticos
* **Filtro de spam:** Analisar o texto e o remetente de um e-mail para classificá-lo como "Spam" ou "Não Spam".
* **Detecção de fraudes:** Avaliar o valor, local e horário de uma compra no cartão de crédito para decidir se a transação é "Legítima" ou "Fraudulenta".
* **Diagnóstico por imagem:** Examinar uma radiografia para identificar se um tecido é "Saudável" ou "Doente".

---

## 2. Regressão (Regression)

É a tarefa de prever um **valor numérico contínuo** com base em variáveis de entrada. Em vez de escolher uma categoria, o algoritmo calcula uma quantidade ou medida exata.

### Exemplos Práticos
* **Precificação de imóveis:** Estimar o valor de venda de uma casa em reais com base na metragem, bairro e número de quartos.
* **Previsão de demanda:** Calcular quantas unidades de um produto serão vendidas no próximo mês considerando o histórico de vendas e a sazonalidade.
* **Previsão do tempo:** Estimar a temperatura exata em graus Celsius para o dia seguinte com base na umidade e pressão atmosférica.

---

## 3. Clustering (Agrupamento)

É a tarefa de organizar dados em **grupos (clusters) baseados na similaridade** entre eles, sem saber previamente quais são esses grupos. O algoritmo descobre estruturas ocultas nos dados por conta própria.

### Exemplos Práticos
* **Segmentação de clientes:** Agrupar consumidores de um e-commerce por hábitos de compra e frequência para criar campanhas de marketing direcionadas.
* **Sistemas de recomendação:** Agrupar usuários com gostos parecidos em plataformas de streaming para sugerir filmes ou músicas que outros membros do mesmo grupo assistiram.
* **Agrupamento de notícias:** Reunir automaticamente reportagens de diferentes portais que falam sobre o mesmo assunto do dia.

---

## Resumo Comparativo

| Abordagem | Tipo de Aprendizado | O que prevê/faz | Exemplo de Saída |
| :--- | :--- | :--- | :--- |
| **Classificação** | Supervisionado | Categorias ou rótulos | "Aprovado" ou "Reprovado" |
| **Regressão** | Supervisionado | Valores numéricos contínuos | R$ 450.000,00 ou 28,5 °C |
| **Clustering** | Não Supervisionado | Grupos por semelhança | Grupo A, Grupo B, Grupo C |
