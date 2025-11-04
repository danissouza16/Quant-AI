# Quant-AI: Projeto de Finanças Quantitativas

Este repositório contém um conjunto de scripts e estratégias para análise de mercado, engenharia de features e backtesting de sistemas de trading algorítmico. O projeto explora desde modelos de Machine Learning para um único ativo até uma construção avançada de portfólio para o S&P 500, utilizando Teoria dos Grafos.

---

## Principais Funcionalidades

* **Coleta de Dados Multi-Fonte:** Pipeline para agregar dados de mercado (YFinance), macroeconômicos (FRED), de sentimento (Reddit) e de "hype" (Google Trends).
* **Estratégia 1 (Single-Asset):** Um modelo preditivo (XGBoost) para a **Nvidia (NVDA)**, que utiliza volatilidade (GARCH) e dados alternativos (Google Trends) para dimensionar posições.
* **Estratégia 2 (Event-Driven):** Um backtest para o **NASDAQ-100** que analisa o *Post-Earnings Announcement Drift* (PEAD), testando a persistência de momentum após gaps de balanço.
* **Estratégia 3 (Portfolio):** Um sistema avançado de construção de portfólio para o **S&P 500** que:
    1.  Resolve o viés de sobrevivência (survivorship bias) fazendo web scraping do histórico de constituintes do índice.
    2.  Utiliza **Clustering Hierárquico** e covariância **Ledoit-Wolf** para agrupar ativos por resíduos (removendo o beta do mercado).
    3.  Aplica **Teoria dos Grafos (Maximal Independent Set - MIS)** para selecionar o conjunto mais diversificado de clusters de ativos.
    4.  Realiza uma **Otimização Bayesiana (skopt)** com validação cruzada `PurgedKFold` para encontrar os hiperparâmetros ideais da estratégia.
    5.  Executa um backtest *walk-forward* e calcula métricas avançadas, como o **Deflated Sharpe Ratio (DSR)**.

---

##Estrutura do Projeto

Aqui está uma descrição do que cada arquivo principal faz:

* `itauquant.py`
    * **Coleta de Dados (Nvidia):** Script principal para baixar e agregar todos os dados necessários para a Estratégia 1. Coleta preços (YFinance), dados macro (FRED), sentimento (Reddit PRAW) e interesse de busca (Google Trends).

* `itau1.py`
    * **Estratégia 1 (Nvidia) - Análise de Features:** Realiza a engenharia de features para o modelo da NVDA. Cria fatores como volatilidade **GARCH**, momentum e o "Hype Factor" (baseado no Google Trends). Testa a significância estatística (ANOVA) de cada feature.

* `itauquant3.py`
    * **Estratégia 1 (Nvidia) - Backtest:** Carrega os dados processados, treina um classificador **XGBoost** para prever a direção do preço ("Sobe", "Desce", "Neutro") e executa um backtest vetorizado. Inclui um modelo híbrido avançado que usa a convicção do ML para dimensionar uma posição-alvo definida pela volatilidade (GARCH).

* `itaun100.py`
    * **Estratégia 2 (NASDAQ-100) - Backtest de Evento (PEAD):** Script dedicado a testar a tese do *Post-Earnings Announcement Drift*. Ele baixa os dados de balanço (earnings dates) do N100, calcula o gap de preço overnight e simula a compra ou venda (momentum vs. reversão) com diferentes períodos de holding (N=1, 5, 20 dias).

* `itaugrafo.py`
    * **Estratégia 3 (S&P 500) - Portfólio com Teoria dos Grafos:** O script mais avançado do projeto. Implementa um backtest completo de construção de portfólio:
        1.  **Células 1-7:** Faz scraping da Wikipedia para construir um banco de dados *point-in-time* dos constituintes do S&P 500, resolvendo o viés de sobrevivência.
        2.  **Células 12-14:** Baixa todos os dados de preço históricos para o universo de ativos.
        3.  **Célula 15:** Define as funções-chave: `get_residual_returns`, `get_hierarchical_clusters` (com Ledoit-Wolf), `get_selected_clusters_mis` (o motor da Teoria dos Grafos) e `get_min_variance_allocation`.
        4.  **Células 15.A-16.B:** Implementa um framework de validação cruzada robusto (`PurgedKFold`) e usa **Otimização Bayesiana (skopt)** para calibrar os hiperparâmetros da estratégia (ex: `n_clusters`, `Z_threshold`).
        5.  **Células 17-21:** Executa o backtest *walk-forward* final, gerando métricas de performance (Sharpe, Drawdown, HHI, DSR) e comparando com o benchmark (SPY).

---

##Detalhes das Estratégias

### 1. Estratégia Single-Asset (XGBoost + GARCH)

* **Tese:** A combinação de fatores quantitativos clássicos (momentum, volatilidade) com dados alternativos (hype) pode prever a direção futura do preço de um ativo volátil como a NVDA.
* **Implementação:**
    1.  Os dados são coletados e processados (`itauquant.py`, `itau1.py`).
    2.  Um modelo XGBoost é treinado para classificar o retorno em 5 dias como "Sobe", "Desce" ou "Neutro".
    3.  O backtest final (`itauquant3.py`) não apenas gera sinais de compra/venda, mas implementa um modelo híbrido:
        * A **volatilidade GARCH** define um orçamento de risco (posição-alvo).
        * A **probabilidade do XGBoost** (convicção do modelo) atua como um "acelerador", dimensionando a posição final de 0% até um máximo alavancado.

### 2. Estratégia Event-Driven (PEAD no NASDAQ-100)

* **Tese:** O mercado reage de forma incompleta aos anúncios de balanço (earnings). Gaps positivos (preço de abertura > fecho anterior) tendem a continuar subindo (drift/momentum), enquanto gaps negativos podem reverter.
* **Implementação (`itaun100.py`):**
    1.  Busca o calendário de balanços de todos os tickers do N100.
    2.  Mapeia a data do evento (T) para o dia de negociação (T+1).
    3.  Calcula o `Gap_Pct` = (Open[T+1] - Close[T]) / Close[T].
    4.  Simula a compra em gaps positivos e a venda em gaps negativos (teste de momentum), e o oposto (teste de reversão).
    5.  Analisa a performance (Sharpe Ratio) para diferentes períodos de holding (N=1, 2, 3, 5, 10, 20 dias).

### 3. Estratégia de Portfólio (MIS no S&P 500)

* **Tese:** É possível construir um portfólio de mínima variância altamente diversificado selecionando ativos de clusters que não possuam alta correlação entre si.
* **Implementação (`itaugrafo.py`):**
    1.  **Universo:** Constrói um universo histórico preciso do S&P 500.
    2.  **Filtragem:** Calcula os resíduos dos retornos contra o SPY para remover o fator de mercado (beta).
    3.  **Clustering:** Agrupa os ativos usando *Clustering Hierárquico* em seus resíduos, com uma matriz de covariância robusta (Ledoit-Wolf).
    4.  **Seleção (Magia do Grafo):**
        * Cria um **"meta-grafo"** onde cada *nó* é um *cluster* de ativos.
        * Cria uma *aresta* (conexão) entre dois nós se a correlação média entre eles for *acima* de um
