# Estratégia de Deploy — notas desta pasta

O diagrama e a explicação completa da arquitetura estão em [`docs/03_estrategia_deploy.html`](../docs/03_estrategia_deploy.html) — já escrito em cima do modelo desta pasta (XGBoost, FastAPI, `joblib`), que é a referência do projeto.

## Se fosse a implementação R (Random Forest) em vez desta

A decisão de arquitetura não muda com o modelo ou a linguagem — só a ferramenta concreta de cada camada:

| Camada | Aqui (Python, referência) | Na versão R (`analysis/02-1_modelagem.qmd`) |
|---|---|---|
| Modelo servido | `models/modelo_xgb.joblib` (XGBoost, ~0,4MB) | `models/modelo_rf.rds` (Random Forest, ~44MB) |
| Feature pipeline | Módulo Python compartilhado entre notebook de treino e API | Função R compartilhada entre `.qmd` e serviço |
| API | FastAPI (tipagem via `pydantic`, `/docs` gerado sozinho) | `plumber` |
| Versionamento | Alias no Model Registry apontando pro `.joblib` | Mesmo conceito, apontando pro `.rds` |

Infraestrutura e monitoramento não mudam — essas decisões são independentes de linguagem ou modelo, e já estão descritas em `docs/03_estrategia_deploy.html`.
