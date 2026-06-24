# Simulação Monte Carlo Risco de Crédito de Portfólio

Modelo quantitativo de risco de crédito que estima a distribuição de perdas de uma carteira via simulação Monte Carlo combinando regressão logística para PD, LGD segmentada por grade e o modelo de Vasicek one-factor para capturar correlação de defaults.

---

## Resumo Executivo

Instituições financeiras precisam manter capital para cobrir perdas inesperadas aquelas além da perda média (Expected Loss) que as provisões já cobrem. Este projeto constrói um modelo de risco de crédito de portfólio que estima a distribuição completa de perdas, permitindo calcular **Credit VaR**, **Expected Shortfall** e **Capital Econômico** em condições normais e sob estresse macroeconômico.

O modelo segue o mesmo arcabouço conceitual do **Basileia II IRB (Internal Ratings-Based approach)** e do **IFRS 9 ECL (Expected Credit Loss)** .

---

## Problema de Negócio

> *"Dada uma carteira de ~600 mil empréstimos não garantidos, qual a probabilidade de as perdas ultrapassarem um determinado nível? Quanto capital é necessário para permanecer solvente com 99% de confiança mesmo em uma recessão?"*

Essa é uma das perguntas centrais da gestão de risco bancário, diretamente ligada aos requisitos de capital regulatório (BACEN, Basileia III) e ao stress testing interno (ICAAP).

---

## Dataset

| Item | Detalhe |
|------|---------|
| Fonte | Kaggle [Lending Club Loan Dataset](https://www.kaggle.com/datasets/wordsforthewise/lending-club) |
| Registros brutos | ~2,26 milhões |
| Após deduplicação | ~928 mil |
| Após filtro de maturidade | ~598 mil contratos maduros |
| Exposição total | ~R$ 8,9 bilhões |

Apenas **contratos maduros** (Fully Paid, Charged Off, Default) são utilizados empréstimos ativos têm desfecho desconhecido e enviésariam as estimativas de PD.

---

## Metodologia

### Componente 1 Modelo de PD (Probabilidade de Default)

**Regressão logística** treinada com quatro features:

| Feature | Justificativa |
|---------|---------------|
| `fico_score` | Proxy da qualidade de crédito do tomador |
| `int_rate` | Sinal de precificação ajustado ao risco |
| `term` | Empréstimos de prazo maior apresentam maior PD observada |
| `grade_num` | Classificação ordinal de risco do originador |

Modelo validado com as métricas padrão da indústria:

| Métrica | Referência | Resultado |
|---------|-----------|-----------|
| AUC | > 0,65 | ✓ |
| Gini | > 0,30 | ✓ |
| KS | > 0,20 | ✓ |

### Componente 2 Modelo de LGD (Loss Given Default)

**Distribuição Beta** parametrizada por grade grades com maior risco de default também apresentam menores taxas de recuperação (maior LGD).

| Grade | LGD Média | Justificativa |
|-------|-----------|---------------|
| A | 35% | Tomadores de maior qualidade, melhor recuperação |
| C | 45% | Risco moderado |
| G | 65% | Subprime, baixa recuperação |

A LGD é **re-amostrada a cada cenário da simulação** para capturar a incerteza na severidade da perda.

### Componente 3 EAD (Exposure at Default)

Valor do empréstimo utilizado como proxy de EAD simplificação padrão para dados históricos de crédito ao consumidor.

### Componente 4 Simulação Monte Carlo

Dois modelos implementados e comparados:

#### Modelo A Independência (Benchmark)
Cada contrato entra em default de forma independente via Bernoulli(PD_i). Com 598 mil contratos, o Teorema Central do Limite força VaR ≈ EL isso **subestima o risco de cauda** e é apresentado como benchmark pedagógico.

#### Modelo B Vasicek One-Factor (Recomendado)
Baseado em Vasicek (1987) e no arcabouço IRB de Basileia II:

$$A_i = \sqrt{\rho_i} \cdot Z + \sqrt{1 - \rho_i} \cdot \varepsilon_i$$

- **Z** ~ N(0,1): fator sistêmico (estado da economia) compartilhado por todos os contratos
- **εᵢ** ~ N(0,1): fator idiossincrático independente por contrato
- **ρ**: correlação de ativo (parâmetro regulatório de Basileia, varia por grade)

O contrato i entra em default quando $A_i < \Phi^{-1}(PD_i)$.

Quando Z << 0 (recessão), todos os retornos de ativos caem simultaneamente → defaults correlacionados → cauda pesada na distribuição de perdas.

---

## Principais Resultados

### Validação do Modelo de PD

A regressão logística captura corretamente a relação monotônica entre grade de crédito e probabilidade de default verificação necessária antes de qualquer simulação.

### Comparação das Distribuições de Perda

| Métrica | Independência | Vasicek |
|---------|--------------|---------|
| EL (analítica) | ~R$ 1,1 bi | ~R$ 1,1 bi |
| VaR 95% | EL + buffer pequeno | EL + **buffer maior** |
| VaR 99% | EL + buffer mínimo | EL + **buffer significativo** |
| Capital Econômico | Subestimado | Realista |

O modelo Vasicek produz uma distribuição assimétrica com cauda direita mais pesada consistente com a evidência empírica de crises de crédito.

### Stress Testing

| Cenário | Fator Sistêmico Z | Interpretação |
|---------|-------------------|---------------|
| Base | 0,0 | Economia em estado normal |
| Adverso | −1,0 | Contração moderada |
| Severo | −2,0 | Recessão (~evento de 1 em 20 anos) |
| Extremo | −3,0 | Crise sistêmica (~evento de 1 em 740 anos) |

Os resultados de stress mostram como o capital necessário escala em condições macroeconômicas adversas insumo direto para os relatórios de ICAAP.

---

## Estrutura do Projeto

```
credit-risk-monte-carlo/
├── notebooks/
│   ├── 01_eda.ipynb                  # Análise exploratória + qualidade dos dados
│   ├── 02_feature_engineering.ipynb  # Modelo de PD + LGD + construção da carteira
│   └── 03_monte_carlo.ipynb          # Simulação + VaR + stress testing
├── data/
│   ├── raw/                          # Dados de origem (não versionados)
│   └── processed/                    # Arquivos parquet intermediários + parâmetros LGD
├── images/                           # Gráficos exportados
├── requirements.txt
└── README.md
```

---

## Tecnologias Utilizadas

| Ferramenta | Finalidade |
|------------|-----------|
| Python 3.10+ | Linguagem principal |
| pandas / numpy | Manipulação de dados |
| scikit-learn | Regressão logística e validação do modelo |
| scipy | Distribuição Beta e testes estatísticos |
| matplotlib / seaborn | Visualização |
| Jupyter Notebook | Desenvolvimento interativo |

---

## Como Executar

```bash
# 1. Clonar o repositório
git clone <repo-url>
cd credit-risk-monte-carlo

# 2. Instalar dependências
pip install -r requirements.txt

# 3. Baixar o dataset
# Colocar loan_default.csv em data/raw/
# Fonte: https://www.kaggle.com/datasets/wordsforthewise/lending-club

# 4. Executar os notebooks em ordem
jupyter notebook notebooks/01_eda.ipynb
jupyter notebook notebooks/02_feature_engineering.ipynb
jupyter notebook notebooks/03_monte_carlo.ipynb
```

---

## Referências Teóricas

- Vasicek, O. (1987). *Probability of Loss on Loan Portfolio*. KMV Corporation.
- Gordy, M. (2003). *A Risk-Factor Model Foundation for Ratings-Based Bank Capital Rules*. Journal of Financial Intermediation.
- Basel Committee on Banking Supervision. *International Convergence of Capital Measurement and Capital Standards* (Basileia II, 2004).
- McNeil, A., Frey, R., Embrechts, P. (2015). *Quantitative Risk Management*. Princeton University Press.

---

## Autora

**Leticia de Araujo**
Engenheira de Dados em transição para Ciência de Dados
[GitHub](https://github.com/Leticia0587)
