# Guia da Solução

Acompanhamento do que já foi feito neste repositório, na ordem em que foi desenvolvido. A numeração segue exatamente os "Entregáveis Esperados" do [`README.md`](README.md) (itens 1 a 5); quando um item foi dividido em mais de um arquivo, uso sub-numeração decimal (2.1, 2.2, ...) em vez de inventar uma numeração paralela.

## Estrutura do repositório (até o momento)

```
data/                                   dados originais fornecidos
  kc_house_data.csv
  zipcode_demographics.csv
  future_unseen_examples.csv

analysis/                               notebooks Quarto (R) — cada um roda de forma independente
  01_eda.qmd / .html                      item 1 — análise exploratória completa
  02-1_modelagem.qmd / .html              item 2.1 — feature engineering, escolha e comparação de modelos
  02-2_generalizacao.qmd / .html          item 2.2 — validação aprofundada (bootstrap, split temporal, CV por CEP)

models/
  modelo_rf.rds                          modelo final treinado (Random Forest, salvo pelo item 2.1)
```

Cada `.qmd` reconstrói os dados do zero a partir de `data/` (nenhum depende do estado de execução de outro).

## Como reproduzir

Requer [R](https://www.r-project.org/) (4.x) e [Quarto](https://quarto.org/) instalados, com os pacotes: `tidyverse`, `janitor`, `skimr`, `corrplot`, `GGally`, `car`, `moments`, `scales`, `patchwork`, `lubridate`, `randomForest`, `caret`, `glmnet`.

```bash
quarto render analysis/01_eda.qmd
quarto render analysis/02-1_modelagem.qmd   # treina a Random Forest e salva em models/ (mais demorado)
quarto render analysis/02-2_generalizacao.qmd
```

## 1. Análise exploratória (`analysis/01_eda.qmd`)

Estatística descritiva completa (média, mediana, moda, desvio padrão, variância, amplitude, IQR, coeficiente de variação, assimetria, curtose) para as variáveis físicas e demográficas, distribuição do preço e decisão de usar `log(price)`, detecção de outliers, checagem de multicolinearidade (identidade `sqft_living = sqft_above + sqft_basement` quebrando o VIF), correlações com o preço, geografia e integração com os dados demográficos por CEP (com a ressalva sobre falácia ecológica).

## 2. Desenvolvimento do modelo de Machine Learning

Este item do desafio pede três coisas (variáveis importantes, escolha do modelo, generalização) — dividi entre dois notebooks porque a parte de generalização merecia um aprofundamento maior do que cabia num só arquivo.

### 2.1 Modelagem (`analysis/02-1_modelagem.qmd`)

Feature engineering (`tem_porao`, `idade_casa`, `reformado`, junção com demografia), split treino/teste estratificado, Regressão Linear e Ridge como baseline, Random Forest escolhido após validação cruzada (`mtry = 7`), comparação honesta treino (*out-of-bag*) vs. teste, e importância de variáveis (com a discordância entre `%IncMSE` e `IncNodePurity` explicada).

**Resultado no teste**: RMSE (log) = 0,176, R² = 0,892, MAE ≈ US$ 66.998 (14,9% do preço mediano).

### 2.2 Generalização (`analysis/02-2_generalizacao.qmd`)

Intervalo de confiança via bootstrap para o RMSE de teste (RMSE ∈ [0,169; 0,182]), comparação entre split aleatório e split temporal, e validação cruzada agrupada por `zipcode` para medir o desempenho em bairros nunca vistos no treino.

**Achado principal**: o modelo erra ~16% mais em CEPs ausentes do treino do que em vendas futuras (~7% a mais) — o maior risco de generalização é bairro novo, não deriva temporal.

### 2.3 Previsão em `future_unseen_examples.csv` — ainda não publicado neste repositório
