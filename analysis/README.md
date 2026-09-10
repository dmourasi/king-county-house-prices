# Versão R — implementação alternativa do projeto (itens 1, 2 e 5-alternativo)

Esta pasta é a **implementação original** do projeto, em R/Quarto. O modelo final aqui é **Random Forest** — testado e validado com o mesmo rigor da versão Python (`../python/`), mas não é mais o que os itens 3, 4 e a versão apresentável do item 5 seguem: depois de comparar 6 modelos e confirmar, na versão Python, que XGBoost generaliza melhor que Random Forest (inclusive em bairros nunca vistos no treino), adotei **Python como implementação de referência** do projeto. Esta versão R fica completa e documentada como alternativa igualmente validada, não como rascunho.

A EDA (item 1, `01_eda.qmd`) foi a primeira coisa feita neste projeto — as decisões de feature engineering (`log(price)`, `tem_porao`, `idade_casa`, `reformado`, remoção do imóvel de 33 quartos) nasceram aqui, e depois foram replicadas de forma independente em Python (`../python/01_eda.ipynb`), com os mesmos números.

## Por que Random Forest continua nesta versão (mesmo XGBoost tendo vencido aqui também)

`02-1_modelagem.qmd` testa **6 modelos**: Regressão Linear, Ridge, Lasso, Random Forest, XGBoost, KNN. XGBoost venceu por RMSE também na versão R (0,166 contra 0,176 da Random Forest) — mesmo padrão da versão Python. A diferença é o que aconteceu depois: **nesta implementação R, o restante do pipeline (itens 2.2 e 2.3 desta pasta) continua em cima da Random Forest**, porque foi o modelo já validado quando o XGBoost surgiu como candidato; a troca de modelo de referência do projeto como um todo foi decidida e formalizada na implementação Python, não retroativamente aqui. Os dois modelos ficam documentados lado a lado em `02-1_modelagem.qmd`.

## Arquivos

```
01_eda.qmd / .html                                  item 1 — análise exploratória completa
02-1_modelagem.qmd / .html                          item 2.1 — 6 modelos comparados, Random Forest mantido nesta versão
02-2_generalizacao.qmd / .html                      item 2.2 — bootstrap CI, split temporal, CV agrupada por CEP
02-3_previsao_futuro.qmd / .html                    item 2.3 — previsão em future_unseen_examples.csv, salva em ../output/
05_comunicacao_stakeholders_calculos.qmd / .html    item 5 — versão alternativa dos cálculos, rodando sobre a Random Forest desta pasta
```

A versão apresentável do item 5 (`docs/05_comunicacao_stakeholders.html`) e os itens 3 e 4 (`docs/03_estrategia_deploy.html`, `docs/04_aprendizado_continuo.html`) seguem o modelo de referência (Python) e vivem em `../docs/`, não aqui.

## Como reproduzir

Requer R (4.x) + [Quarto](https://quarto.org/), pacotes: `tidyverse`, `janitor`, `skimr`, `corrplot`, `GGally`, `car`, `moments`, `scales`, `patchwork`, `lubridate`, `randomForest`, `caret`, `glmnet`, `xgboost`.

```bash
quarto render 01_eda.qmd
quarto render 02-1_modelagem.qmd
quarto render 02-2_generalizacao.qmd
quarto render 02-3_previsao_futuro.qmd
quarto render 05_comunicacao_stakeholders_calculos.qmd
```

## Resultado (conjunto de teste)

| Modelo | RMSE (log) | MAE (log) | R² |
|---|---|---|---|
| **XGBoost** | **0,166** | **0,117** | **0,903** |
| Random Forest | 0,176 | 0,122 | 0,892 |
| KNN | 0,201 | 0,144 | 0,859 |
| Lasso | 0,211 | 0,157 | 0,843 |
| Regressão Linear | 0,211 | 0,157 | 0,843 |
| Ridge | 0,212 | 0,157 | 0,843 |

Em dólar (MAE): Regressão Linear US\$ 89.810 (20,0% da mediana) · Random Forest US\$ 66.998 (14,9%) · XGBoost US\$ 63.424 (14,1%) · KNN US\$ 80.123 (17,8%).

**Generalização** (item 2.2, modelo Random Forest): split temporal piora o RMSE em **+7%**; CV agrupada por CEP piora em **+16%** — mesma ordem de risco (bairro novo > tempo) encontrada na versão Python (+13,0% / +20,5%, respectivamente, com XGBoost).
