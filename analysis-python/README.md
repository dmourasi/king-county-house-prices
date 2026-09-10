# Versão Python — modelo de referência do projeto (itens 1, 2 e 3)

Esta pasta é a **implementação de referência** do projeto: o modelo final (XGBoost) e tudo que depende dele — generalização, previsão, estratégia de deploy, comunicação com stakeholders — segue a partir daqui. A versão R (`analysis-r/`, `docs/`) fica documentada com Random Forest como implementação alternativa, testada e validada da mesma forma, mas não é mais a que os demais itens do projeto seguem.

A EDA (item 1) foi refeita de forma independente em [`01_eda.ipynb`](01_eda.ipynb) — mesma análise da versão R (`analysis-r/01_eda.qmd`), com `scipy`/`statsmodels`/`seaborn` no lugar de `moments`/`car`/`ggplot2`. Os números batem entre as duas implementações (mesmo dataset), o que serve como confirmação cruzada das decisões de feature engineering (`log(price)`, `tem_porao`, `idade_casa`, `reformado`, remoção do imóvel de 33 quartos) usadas a partir de `02-1_modelagem.ipynb`.

## Por que XGBoost, e não Random Forest

Comparei 4 modelos em `02-1_modelagem.ipynb` (Regressão Linear, Ridge, Random Forest, XGBoost). A versão R testa mais dois (Lasso e KNN, ver `analysis-r/02-1_modelagem.qmd`) — não refiz esses dois aqui porque, em R, nenhum superava a Random Forest/XGBoost (Lasso ficou nivelado com a regressão linear simples; KNN ficou atrás dos modelos baseados em árvore, embora não muito), e o objetivo desta versão era validar a troca de modelo, não repetir toda a bateria de baselines. XGBoost venceu num único split de teste, mas a decisão de trocar por Random Forest (que já estava validada e em uso) só veio depois de checar, em `02-2_generalizacao.ipynb`, se essa vantagem sobrevivia ao teste mais rigoroso que tenho: validação cruzada agrupada por CEP, simulando bairro nunca visto no treino. Sobreviveu — e a degradação proporcional do XGBoost nesse teste foi até um pouco menor que a da Random Forest, descartando a hipótese de que um modelo de *boosting* pudesse generalizar pior por memorizar mais o treino.

## Arquivos

```
01_eda.ipynb / .html                                item 1 — análise exploratória completa
02-1_modelagem.ipynb / .html                        item 2.1 — 4 modelos comparados, XGBoost escolhido, importância de variáveis
02-2_generalizacao.ipynb / .html                    item 2.2 — bootstrap CI, split temporal, CV agrupada por CEP
02-3_previsao_futuro.ipynb / .html                  item 2.3 — checagem de covariate shift (incl. idade_casa) + previsão em future_unseen_examples.csv
03_estrategia_deploy.md                             item 3 — notas específicas do stack Python (ver docs/03_estrategia_deploy.html para o diagrama principal)
05_comunicacao_stakeholders_calculos.ipynb / .html  item 5 — cálculos por trás de docs/05_comunicacao_stakeholders.html
models/modelo_xgb.joblib                            modelo final (XGBRegressor) — usado pelos itens 2.2, 2.3, 5 e deploy
models/modelo_rf.joblib                             Random Forest, referência de comparação (não usada por nenhum notebook, mas mantida versionada como evidência de que os 4 modelos foram de fato treinados)
output/previsoes_future_unseen.csv
```

## Como reproduzir

```bash
python3 -m pip install -r requirements.txt
python3 -m nbconvert --to notebook --execute --inplace 01_eda.ipynb
python3 -m nbconvert --to notebook --execute --inplace 02-1_modelagem.ipynb   # treina e salva os modelos
python3 -m nbconvert --to notebook --execute --inplace 02-2_generalizacao.ipynb
python3 -m nbconvert --to notebook --execute --inplace 02-3_previsao_futuro.ipynb
python3 -m nbconvert --to notebook --execute --inplace 05_comunicacao_stakeholders_calculos.ipynb
python3 -m nbconvert --to html 01_eda.ipynb 02-1_modelagem.ipynb 02-2_generalizacao.ipynb 02-3_previsao_futuro.ipynb 05_comunicacao_stakeholders_calculos.ipynb
```

## Resultado (conjunto de teste)

| Modelo | RMSE (log) | R² | MAE (US$) |
|---|---|---|---|
| Regressão Linear | 0,208 | 0,845 | 90.352 |
| Ridge | 0,207 | 0,848 | — |
| Random Forest | 0,169 | 0,898 | 68.480 |
| **XGBoost** | **0,162** | **0,907** | **63.707** |

**Generalização** (item 2.2, modelo XGBoost): intervalo de confiança via bootstrap RMSE ∈ [0,156; 0,168] (estável); split temporal piora o RMSE em **+13,0%**; CV agrupada por CEP piora em **+20,5%** — mesma ordem de risco (bairro novo > tempo) encontrada quando o modelo era Random Forest, e replicada de forma independente na versão R.

**Diagnóstico de resíduos** (item 2.1, seção 8): o modelo subestima sistematicamente os imóveis mais caros — viés médio de **+US\$ 78.004** no top 10% mais caro do teste (real acima do previsto), contra viés próximo de zero (-US\$ 3.222) no restante. É a base empírica, calculada aqui em cima do XGBoost, da ressalva usada na comunicação com stakeholders.

## Uma diferença técnica que vale registrar

O modelo XGBoost final salva em **~0,4MB** — cerca de 60× menor que a Random Forest (~25MB), e treina em segundos em vez de minutos. Não foi o motivo da escolha (a decisão foi por generalização), mas é um efeito colateral relevante para o item 3 (deploy): modelo menor, cold start mais rápido, custo de armazenamento desprezível no Model Registry.
