# Versão Python (item 2 + item 3)

Complemento à solução principal em R (`analysis/`, `docs/`) — mesmas decisões e o mesmo raciocínio, implementados com `pandas` + `scikit-learn`. Não substitui a versão R; serve para mostrar a mesma modelagem em outra linguagem/ferramental.

A EDA (item 1) não foi refeita aqui — as decisões de feature engineering (`log(price)`, `tem_porao`, `idade_casa`, `reformado`, remoção do imóvel de 33 quartos) vêm da análise em R, [`analysis/01_eda.qmd`](../analysis/01_eda.qmd).

## Arquivos

```
01_modelagem.ipynb / .html        item 2.1 — feature engineering, LM/Ridge/Random Forest, importância de variáveis
02_generalizacao.ipynb / .html    item 2.2 — bootstrap CI, split temporal, CV agrupada por CEP
03_previsao_futuro.ipynb / .html  item 2.3 — previsão em future_unseen_examples.csv
04_estrategia_deploy.md           item 3 — mesma arquitetura de docs/03_estrategia_deploy.md, com o ferramental Python (FastAPI, joblib)
models/modelo_rf.joblib           modelo final treinado (RandomForestRegressor)
output/previsoes_future_unseen.csv
```

## Como reproduzir

```bash
python3 -m pip install -r requirements.txt
python3 -m nbconvert --to notebook --execute --inplace 01_modelagem.ipynb   # treina e salva o modelo
python3 -m nbconvert --to notebook --execute --inplace 02_generalizacao.ipynb
python3 -m nbconvert --to notebook --execute --inplace 03_previsao_futuro.ipynb
python3 -m nbconvert --to html 01_modelagem.ipynb 02_generalizacao.ipynb 03_previsao_futuro.ipynb
```

## Resultado (conjunto de teste)

| Modelo | RMSE (log) | R² | MAE (US$) |
|---|---|---|---|
| Regressão Linear | 0,208 | 0,845 | 90.352 |
| Ridge | 0,207 | 0,848 | — |
| **Random Forest** | **0,169** | **0,898** | **68.480** |

Achado de generalização replicado de forma independente em relação à versão R: o modelo erra mais em CEPs ausentes do treino (+22,8%) do que testando estritamente no futuro (+12,0%) — a mesma ordem de risco encontrada em R (+16% vs. +7%), com números diferentes (esperado, dado que são duas implementações distintas de Random Forest), mas a mesma conclusão prática.

## Uma diferença técnica que vale registrar

O `RandomForestRegressor` do scikit-learn, sem restrição, salvou um modelo de **467MB** — grande demais para versionar em Git. O motivo: por padrão, o scikit-learn deixa cada folha crescer até ter uma única observação (`min_samples_leaf=1`), enquanto o `randomForest` do R já limita isso por padrão (`nodesize=5`). Apliquei essa mesma restrição em Python (`min_samples_leaf=5`), reduzindo o modelo para ~25MB, com perda de acurácia desprezível — decisão de paridade com o comportamento do R, não um ajuste arbitrário.
