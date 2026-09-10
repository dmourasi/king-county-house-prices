# Guia do Pipeline

Este documento não é um changelog — é o mapa de como os dados viram um modelo em produção e um relatório de negócio, e onde encontrar cada etapa no repositório. A numeração segue os "Entregáveis Esperados" do [`README.md`](README.md) (itens 1 a 5); quando um item foi dividido em mais de um arquivo, uso sub-numeração decimal (2.1, 2.2, 2.3).

**Duas implementações, uma referência.** O projeto tem uma versão em R (`analysis-r/`) e uma em Python (`analysis-python/`). Comecei em R; depois de comparar 6 modelos e confirmar, na versão Python, que XGBoost generaliza melhor que Random Forest (inclusive em bairros nunca vistos no treino), adotei **Python como implementação de referência** — é dali que os itens 3, 4 e a versão apresentável do item 5 seguem. A versão R fica completa e documentada como implementação alternativa igualmente validada, com Random Forest. Cada pasta tem seu próprio `README.md` explicando seu papel: [`analysis-r/README.md`](analysis-r/README.md) (R, alternativa) e [`analysis-python/README.md`](analysis-python/README.md) (Python, referência).

## Estrutura de pastas

```
data/         dados brutos fornecidos (kc_house_data.csv, zipcode_demographics.csv, future_unseen_examples.csv)
analysis-r/     implementação R — item 1 (EDA) + item 2 alternativo (Random Forest) + item 5 alternativo
analysis-python/       implementação de referência — item 1 (EDA) + item 2 (XGBoost) + item 5 (cálculos)
docs/         entregáveis finais dos itens 3, 4 e 5 — seguem o modelo de referência, não são específicos de linguagem
models/       modelos R salvos: modelo_rf.rds (usado pela versão alternativa) e modelo_xgb.rds (comparação, evidência)
output/       previsões da versão R (item 2.3) sobre future_unseen_examples.csv — equivalente a analysis-python/output/
```

## O pipeline, etapa por etapa

```mermaid
flowchart TD
    DATA["data/*.csv"] --> EDA["Item 1 · EDA\nanalysis-r/01_eda.qmd (R)\nanalysis-python/01_eda.ipynb"]
    EDA --> MODEL["Item 2.1 · Modelagem\nanalysis-python/02-1_modelagem.ipynb → XGBoost (referência)\nanalysis-r/02-1_modelagem.qmd → Random Forest (alternativa)"]
    MODEL -->|modelo salvo| GEN["Item 2.2 · Generalização\nanalysis-python/02-2_generalizacao.ipynb\nanalysis-r/02-2_generalizacao.qmd"]
    MODEL -->|modelo salvo| PREV["Item 2.3 · Previsão\nanalysis-python/02-3_previsao_futuro.ipynb\nanalysis-r/02-3_previsao_futuro.qmd"]
    PREV --> OUT["analysis-python/output/previsoes_future_unseen.csv\noutput/previsoes_future_unseen.csv"]
    GEN --> DEPLOY["Item 3 · Deploy\ndocs/03_estrategia_deploy.html"]
    GEN --> LEARN["Item 4 · Aprendizado contínuo\ndocs/04_aprendizado_continuo.html"]
    MODEL -->|importância + coeficientes| CALC["analysis-python/05_comunicacao_stakeholders_calculos.ipynb\n(analysis-r/..._calculos.qmd = alternativa R)"]
    GEN -->|números de risco| CALC
    CALC --> STAKE["Item 5 · Comunicação com stakeholders\ndocs/05_comunicacao_stakeholders.html"]
```

| Etapa | Item | Roda sobre | Arquivo (referência, Python) | Arquivo (alternativa, R) | Produz |
|---|---|---|---|---|---|
| Análise exploratória | 1 | dados brutos | `analysis-python/01_eda.ipynb` | `analysis-r/01_eda.qmd` | decisões de feature engineering |
| Modelagem | 2.1 | dados + features | `analysis-python/02-1_modelagem.ipynb` | `analysis-r/02-1_modelagem.qmd` | `analysis-python/models/{modelo_xgb,modelo_rf}.joblib` e `models/{modelo_rf,modelo_xgb}.rds` — os 2 modelos mais fortes de cada lado, não só o escolhido |
| Generalização | 2.2 | modelo salvo | `analysis-python/02-2_generalizacao.ipynb` | `analysis-r/02-2_generalizacao.qmd` | números de risco (temporal, CEP novo) |
| Previsão em dados novos | 2.3 | modelo salvo | `analysis-python/02-3_previsao_futuro.ipynb` | `analysis-r/02-3_previsao_futuro.qmd` | `analysis-python/output/previsoes_future_unseen.csv` e `output/previsoes_future_unseen.csv` |
| Estratégia de deploy | 3 | modelo + generalização | `docs/03_estrategia_deploy.html` | — (diferenças em `analysis-python/03_estrategia_deploy.md`) | diagrama de arquitetura |
| Aprendizado contínuo | 4 | generalização | `docs/04_aprendizado_continuo.html` | — | ciclo de reentreino |
| Comunicação com stakeholders | 5 | modelo + generalização | `docs/05_comunicacao_stakeholders.html`, calculado em `analysis-python/05_comunicacao_stakeholders_calculos.ipynb` | `analysis-r/05_comunicacao_stakeholders_calculos.qmd` (números diferentes, roda sobre RF) | relatório de negócio |

Os itens 3, 4 e 5 (pasta `docs/`) são artefatos de apresentação — descrevem o modelo de referência (XGBoost) e não têm uma "versão R" própria, exceto o item 5, que tem uma bancada de cálculo alternativa em R para quem quiser comparar os números sob a Random Forest.

## Como reproduzir do zero

```bash
# 1. R — EDA + implementação alternativa (Random Forest)
quarto render analysis-r/01_eda.qmd
quarto render analysis-r/02-1_modelagem.qmd
quarto render analysis-r/02-2_generalizacao.qmd
quarto render analysis-r/02-3_previsao_futuro.qmd
quarto render analysis-r/05_comunicacao_stakeholders_calculos.qmd

# 2. Python — implementação de referência (XGBoost); rode nesta ordem, cada notebook depende do anterior
cd analysis-python
python3 -m pip install -r requirements.txt
python3 -m nbconvert --to notebook --execute --inplace 01_eda.ipynb
python3 -m nbconvert --to notebook --execute --inplace 02-1_modelagem.ipynb          # salva models/modelo_xgb.joblib
python3 -m nbconvert --to notebook --execute --inplace 02-2_generalizacao.ipynb      # usa o modelo salvo acima
python3 -m nbconvert --to notebook --execute --inplace 02-3_previsao_futuro.ipynb    # usa o modelo salvo acima
python3 -m nbconvert --to notebook --execute --inplace 05_comunicacao_stakeholders_calculos.ipynb  # usa o modelo salvo acima
```

Requisitos: **R** (4.x) + [Quarto](https://quarto.org/) com `tidyverse`, `janitor`, `skimr`, `corrplot`, `GGally`, `car`, `moments`, `scales`, `patchwork`, `lubridate`, `randomForest`, `caret`, `glmnet`, `xgboost`; **Python** (3.12), ver `analysis-python/requirements.txt`.

Os itens 3, 4 e 5 (`docs/`) são páginas HTML estáticas escritas à mão sobre os números já calculados nas etapas acima — não têm um passo de "render" próprio.

## Decisões-chave (resumo — números completos em cada notebook)

- **Target**: `log(price)`, por causa da assimetria forte da distribuição bruta (ver item 1).
- **Modelo de referência**: XGBoost, escolhido em `analysis-python/02-1_modelagem.ipynb` e confirmado em `analysis-python/02-2_generalizacao.ipynb` (a vantagem sobre Random Forest sobrevive à validação mais rigorosa: CV agrupada por CEP).
- **Maior risco de generalização**: bairro (CEP) nunca visto no treino, não deriva temporal — é o que guia a flag de confiança no item 3 e o gatilho de reentreino mais sensível no item 4.
- **Duas implementações permanecem no repositório por design**: a R não foi descartada após a troca de modelo — fica documentada como alternativa validada, útil para comparação.

## Onde encontrar cada entregável do desafio original

| # | Entregável (`README.md`) | Onde está |
|---|---|---|
| 1 | Análise e entendimento dos dados | `analysis-r/01_eda.qmd` / `analysis-python/01_eda.ipynb` |
| 2a–c | Variáveis importantes, escolha do modelo, generalização | `analysis-python/02-1_modelagem.ipynb`, `analysis-python/02-2_generalizacao.ipynb` |
| 3 | Estratégia de deploy | `docs/03_estrategia_deploy.html` |
| 4 | Aprendizado contínuo | `docs/04_aprendizado_continuo.html` |
| 5 | Comunicação com stakeholders | `docs/05_comunicacao_stakeholders.html` |
