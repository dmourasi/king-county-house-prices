# Aprendizado Contínuo

Como eu descreveria a estratégia para o modelo continuar aprendendo com dados novos — apoiada diretamente no que encontrei no item 2.2, generalização ([`analysis/02-2_generalizacao.qmd`](../analysis/02-2_generalizacao.qmd)), não numa receita genérica de MLOps.

## Quando reentreinar

Uso três gatilhos, não só um calendário fixo:

1. **Periódico (linha de base).** Mensal é razoável: venda de imóvel não é um evento de alta frequência, e treinar toda semana só adicionaria ruído operacional sem dado novo suficiente para justificar. Esse é o gatilho de "manutenção", não o mais importante.

2. **Por zipcode novo (o gatilho que eu considero mais importante).** No item 2.2, descobri que o modelo generaliza **pior para um CEP nunca visto** (RMSE +16%) do que para uma janela de tempo futura (RMSE +7%) — o maior risco de generalização não é o modelo "envelhecer", é aparecer um bairro fora do que ele já viu (um novo empreendimento, uma área que começou a vender casas). Por isso, o gatilho mais sensível que eu proporia não é baseado em tempo: é monitorar a % de previsões vindas de CEPs ausentes do conjunto de treino (a mesma flag da estratégia de deploy) e disparar um reentreino assim que esse volume passar de um limiar — sem esperar o ciclo mensal.

3. **Por degradação de performance observada.** Quando o preço real de venda for confirmado (com o atraso natural do mercado), comparo o erro real contra o intervalo de confiança que já calculei via bootstrap (RMSE de teste, IC 95% [0,169; 0,182] em escala log). Se o erro real ficar consistentemente **fora** desse intervalo por um período, é sinal de que a relação entre as features e o preço mudou (ex.: uma variação de mercado que o modelo não capturou) — gatilho de reentreino antecipado, independente do calendário.

## Como avaliar um modelo novo antes de substituir o de produção (champion/challenger)

Um ponto que aprendi com o próprio item 2.2: **uma média de erro única esconde fraqueza específica**. Por isso, antes de qualquer modelo novo (`challenger`) substituir o de produção (`champion`), ele passa pelas mesmas três validações que já usei:

| Validação | O que testa |
|---|---|
| Split aleatório estratificado | Desempenho geral, comparável ao histórico |
| Split temporal | Desempenho ao prever "o futuro" a partir do passado |
| CV agrupada por CEP | Desempenho em bairros fora do treino |

O `challenger` só substitui o `champion` se **não piorar em nenhuma das três** (dentro da margem do intervalo de confiança, não exigindo melhora estatisticamente perfeita em todas) — evita o cenário de um modelo novo melhorar a média geral às custas de piorar exatamente o ponto fraco que eu já sei que é o mais sensível (CEPs novos).

## Como o reentreino aconteceria, na prática

1. Puxar as vendas confirmadas desde o último treino, e a versão mais recente de `zipcode_demographics` (dado que muda pouco, mas vale checar).
2. Rodar o mesmo pipeline de feature engineering encapsulado (a mesma lógica descrita na estratégia de deploy, para não haver divergência entre o que treina e o que serve).
3. Re-treinar com a mesma busca de `mtry` por 5-fold CV usada no item 2.1.
4. Avaliar como `challenger` contra as três validações da tabela acima.
5. Se aprovado, versionar no Model Registry e trocar o alias de produção (rollback imediato disponível, como descrito na estratégia de deploy).

## Uma decisão que eu deixaria em aberto para revisão futura

Não decidi ainda se o treino deveria usar **todo o histórico acumulado** ou uma **janela deslizante** (ex.: últimos 3–5 anos). Mercado imobiliário muda (inflação, tendência de preços por região), então dado muito antigo pode ter relação diferente entre features e preço da que existe hoje — mas também não tenho, neste momento, evidência própria de que isso é um problema real neste dataset (só um ano de dados de treino, 2014–2015). Registro como algo a revisitar quando houver histórico multi-ano suficiente para testar as duas abordagens uma contra a outra, em vez de decidir isso sem dado.
