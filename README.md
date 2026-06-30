# MVP — Machine Learning & Analytics
## Previsão da Produção Nacional de Petróleo por Campo (ANP)

---

## 1. Visão Geral

Este MVP constrói um pipeline *end-to-end* para **prever a produção mensal de óleo por campo produtor para os próximos 5 anos (60 meses)**, a partir de duas décadas de dados públicos da ANP.

A modelagem trata o problema na granularidade **Campo-Mês** (e não no nível ruidoso de poço individual), o que estabiliza a série temporal e neutraliza a volatilidade de paradas operacionais e poços inativos. Como atributos preditivos auxiliares, o modelo consome o histórico **defasado** (lags) de Gás Natural, Gás Associado e Gás Não Associado, capturando a curva de declínio natural não-linear dos reservatórios sem incorrer em vazamento da coprodução óleo↔gás.

---

## 2. Dataset

- **Fonte:** [Produção de Petróleo e Gás Natural — ANP](https://www.gov.br/anp/pt-br/centrais-de-conteudo/dados-estatisticos) · [Consulta de Produção por Poço](https://cdp.anp.gov.br/ords/r/cdp_apex/consulta-dados-publicos-cdp/consulta-produ%C3%A7%C3%A3o-por-po%C3%A7o)
- **Período coberto:** ~2005–2025 (20 anos de histórico)
- **Repositório de dados (raw):** [rgomesa2025/MVP_MACHINE_LEARNING_E_ANALYTICS/data](https://github.com/rgomesa2025/MVP_MACHINE_LEARNING_E_ANALYTICS/tree/main/data)

> A base histórica apresenta um desafio de padronização (*data wrangling*): alguns anos vêm consolidados em um único `.csv`, outros fragmentados em arquivos mensais, com colunas que variam de formato entre os anos. O pipeline mapeia, unifica e lê os arquivos diretamente da URL pública do GitHub, dispensando uploads manuais e garantindo total reprodutibilidade.

---

## 3. Como Executar

O projeto foi desenhado para rodar **nativamente no Google Colab**, sem instalação de dependências externas — todos os pacotes utilizados já vêm pré-instalados.

1. Abra o notebook `Producao_Nacional_Petroleo.ipynb` no Google Colab.
2. Execute as células em ordem (`Ambiente de execução → Executar tudo`).
3. Os dados são baixados automaticamente do repositório público; não há upload manual.

**Ambiente local (opcional):** caso execute fora do Colab, descomente a célula de dependências:

```bash
pip install -q pandas numpy matplotlib scikit-learn requests
```

---

## 4. Bibliotecas Utilizadas

- **Ingestão e requisição:** `requests`
- **Manipulação e padronização:** `pandas`, `numpy`
- **Visualização:** `matplotlib`, `seaborn`
- **Modelagem e ML:** `scikit-learn` (`TimeSeriesSplit`, `Pipeline`, `ColumnTransformer`, `RobustScaler`, `OneHotEncoder`, `Ridge`, `RandomForestRegressor`, `DummyRegressor`, `RandomizedSearchCV`)
- **Persistência de artefatos:** `joblib`

Semente global fixada em `SEED = 42` para reprodutibilidade.

---

## 5. Estrutura do Notebook

| Seção | Conteúdo |
| :--- | :--- |
| 1. Definição do Problema | Descrição, objetivo, tipo de problema, premissas, hipóteses e critérios de sucesso |
| 2. Ambiente e Reprodutibilidade | Setup, sementes, bibliotecas e funções auxiliares |
| 3. Seleção e Carga dos Dados | Ingestão consolidada da ANP, volumetria, tipagem, ausentes, duplicatas e dicionário de dados |
| 4. Análise Exploratória (EDA) | Distribuição do target, gases coproduzidos, comportamento de declínio temporal e outliers |
| 5. Preparação dos Dados | Separação features/target, alinhamento de granularidade e divisão temporal treino/teste |
| 6. Pré-processamento e Pipeline | Pipeline blindado contra vazamento de dados |
| 7. Baseline e Modelos Candidatos | Definição de baseline temporal e modelos candidatos |
| 8. Treinamento e Avaliação Inicial | Treino e comparação inicial das métricas |
| 9. Validação e Otimização | Tuning de hiperparâmetros via `RandomizedSearchCV` |
| 10. Avaliação Final | Desempenho no conjunto de teste e diagnóstico de resíduos |
| 11. Comparação Final dos Modelos | Escolha do modelo campeão |
| 12. Boas Práticas e Rastreabilidade | Metadados, log de decisões arquiteturais e limitações |
| 13. Conclusão | Síntese, aprendizados e próximos passos |
| 14. Salvamento de Artefatos e Projeção | Persistência do pipeline e projeção de produção 2026–2030 |

---

## 6. Metodologia

- **Granularidade Campo-Mês:** agregação que elimina ruídos estocásticos do nível de poço.
- **Atributos autorregressivos:** lags `[1, 3, 6, 12]` e médias móveis `[3, 6, 12]`, calculados **dentro de cada campo** e respeitando a ordem cronológica.
- **Validação temporal:** `TimeSeriesSplit` para impedir vazamento de dados (*data leakage*) do futuro para o passado — sem embaralhamento (*shuffling*).
- **Pipeline blindado:** `RobustScaler` (numéricas) + `OneHotEncoder` (categóricas) encapsulados, evitando vazamento entre treino e teste.
- **Projeção recursiva:** forecasting autoregressivo de 60 meses (2026–2030), com restrição de não-negatividade nos volumes previstos.

---

## 7. Métricas de Avaliação

| Métrica | Papel |
| :--- | :--- |
| **MAE** (Barris/Dia) | Métrica principal, na validação 2024–2025 |
| **RMSE** (Barris/Dia) | Métrica de suporte |
| **WAPE** (Erro %) | Erro percentual ponderado pelo volume |
| **R²** | Qualidade do ajuste |

**Critério de sucesso:** o modelo campeão deve apresentar MAE pelo menos **30% menor** que o baseline temporal (mediana móvel), com custo computacional < 5 min no Colab e curva futura logicamente decrescente, sem volumes negativos.

---

## 8. Resultados

O modelo campeão é a **`RandomForestRegressor`** (versão sintonizada via `TimeSeriesSplit` adotada quando reduz o WAPE de teste), encapsulada em pipeline com `RobustScaler` e `OneHotEncoder`. Abordagens lineares (`Ridge`) captam a tendência geral, mas ficam atrás do comitê de árvores na modelagem do declínio não-linear. O modelo supera o baseline da mediana com folga e, por usar exclusivamente atributos defasados, entrega um ajuste estatisticamente legítimo.

**Principais aprendizados:** a governança de dados e a granularidade ditaram o sucesso do sistema — a agregação Campo-Mês neutralizou o excesso de zeros de poços inativos, e o uso do gás **histórico** (não contemporâneo) foi decisivo para separar capacidade preditiva real de correlação espúria.

---

## 9. Limitações e Próximos Passos

**Limitações:** o modelo é tabular — carrega memória via lags, mas não modela a dinâmica contínua de curtíssimo prazo nem eventos exógenos (gargalos logísticos, falhas de elevação artificial, paradas programadas). Na projeção recursiva, o erro tende a se acumular ao longo dos 60 meses.

**Próximos passos:**
- Comparar com modelos físicos de declínio (curvas de Arps).
- Avaliar `HistGradientBoostingRegressor` / `XGBoost`.
- Substituir a banda baseada em RMSE por previsão quantílica (*quantile regression forests*).
- Integrar logs operacionais de interdições e paradas reguladas pela ANP.

---

## 10. Artefatos Gerados

Persistidos em disco ao final da execução (pasta `artefatos_mvp/`):
- Pipeline final integrado + modelo treinado (`joblib`)
- `RobustScaler` e `OneHotEncoder` isolados
- Tabela de resultados comparativos
- Gráficos de diagnóstico de resíduos
- Projeção numérica agregada de produção 2026–2030
