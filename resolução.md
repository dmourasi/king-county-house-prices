# Guia da Solução

Acompanhamento do que já foi feito neste repositório para o desafio descrito em [`README.md`](README.md), na ordem em que foi desenvolvido.

## Estrutura do repositório (até o momento)

```
data/                                   dados originais fornecidos
  kc_house_data.csv
  zipcode_demographics.csv
  future_unseen_examples.csv

analysis/                               notebooks Quarto (R) — cada um roda de forma independente
  01_eda.qmd / .html                      análise exploratória completa
  02_modelagem.qmd / .html                feature engineering, escolha e comparação de modelos
  03_generalizacao.qmd / .html            validação aprofundada (bootstrap, split temporal, CV por CEP)

models/
  modelo_rf.rds                          modelo final treinado (Random Forest, salvo por 02_modelagem.qmd)
```

Cada `.qmd` reconstrói os dados do zero a partir de `data/` (nenhum depende do estado de execução de outro).

## Como reproduzir

Requer [R](https://www.r-project.org/) (4.x) e [Quarto](https://quarto.org/) instalados, com os pacotes: `tidyverse`, `janitor`, `skimr`, `corrplot`, `GGally`, `car`, `moments`, `scales`, `patchwork`, `lubridate`, `randomForest`, `caret`, `glmnet`.

```bash
quarto render analysis/01_eda.qmd
quarto render analysis/02_modelagem.qmd   # treina a Random Forest e salva em models/ (mais demorado)
quarto render analysis/03_generalizacao.qmd
```

## 1. Análise exploratória (`analysis/01_eda.qmd`)

Estatística descritiva completa (média, mediana, moda, desvio padrão, variância, amplitude, IQR, coeficiente de variação, assimetria, curtose) para as variáveis físicas e demográficas, distribuição do preço e decisão de usar `log(price)`, detecção de outliers, checagem de multicolinearidade (identidade `sqft_living = sqft_above + sqft_basement` quebrando o VIF), correlações com o preço, geografia e integração com os dados demográficos por CEP (com a ressalva sobre falácia ecológica).

## 2. Modelagem (`analysis/02_modelagem.qmd`)

Feature engineering (`tem_porao`, `idade_casa`, `reformado`, junção com demografia), split treino/teste estratificado, Regressão Linear e Ridge como baseline, Random Forest escolhido após validação cruzada (`mtry = 7`), comparação honesta treino (*out-of-bag*) vs. teste, e importância de variáveis (com a discordância entre `%IncMSE` e `IncNodePurity` explicada).

**Resultado no teste**: RMSE (log) = 0,176, R² = 0,892, MAE ≈ US$ 66.998 (14,9% do preço mediano).

## 3. Generalização (`analysis/03_generalizacao.qmd`)

Intervalo de confiança via bootstrap para o RMSE de teste (RMSE ∈ [0,169; 0,182]), comparação entre split aleatório e split temporal, e validação cruzada agrupada por `zipcode` para medir o desempenho em bairros nunca vistos no treino.

**Achado principal**: o modelo erra ~16% mais em CEPs ausentes do treino do que em vendas futuras (~7% a mais) — o maior risco de generalização é bairro novo, não deriva temporal.
