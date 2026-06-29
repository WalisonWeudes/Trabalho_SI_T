# Detecção de Falhas em Compressor de Trem Metrô (MetroPT-3)

Este repositório contém um trabalho de faculdade focado no desenvolvimento de abordagens de Machine Learning para a **previsão de falhas (Manutenção Preditiva)** utilizando o conjunto de dados real **MetroPT-3**. O objetivo principal é monitorar o compressor de ar de um trem metropolitano (APU) e gerar alertas de vazamento de ar (Air leak).

---

## 📌 Sumário do Projeto

- [Objetivo do Trabalho](#-objetivo-do-trabalho)
- [Estrutura de Arquivos](#-estrutura-de-arquivos)
- [O Dataset MetroPT-3](#-o-dataset-metropt-3)
- [Abordagem 1: Pipeline Supervisionado (Random Forest)](#-abordagem-1-pipeline-supervisionado-random-forest)
- [Abordagem 2: Pipeline Não Supervisionado (K-Means Detalhado)](#-abordagem-2-pipeline-não-supervisionado-k-means-detalhado)
- [Como Executar](#-como-executar)

---

## 🎯 Objetivo do Trabalho

O compressor de ar de um trem é um componente crítico. Falhas repentinas podem causar atrasos severos na operação do metrô. 

Este repositório explora duas abordagens distintas para prever falhas:
1. **Supervisionada (`metro.ipynb`)**: Tenta classificar o estado de "Pré-falha" nas 24 horas anteriores ao evento. Devido ao desbalanceamento extremo de classes (~98% normal, ~2% pré-falha) e às variações térmicas sazonais, esse modelo apresenta baixa acurácia prática (falsos positivos no verão).
2. **Não Supervisionada (`metro_kmeans.ipynb`)**: Agrupa os dados dos sensores com K-Means para segmentar estados físicos de operação e monitora a taxa diária de estresse do compressor (tempo sob carga), fornecendo uma detecção robusta de anomalias sem sofrer com desvios térmicos.

---

## 📁 Estrutura de Arquivos

O projeto está organizado da seguinte forma:

* `metro.ipynb`: Jupyter Notebook contendo a modelagem supervisionada inicial (Random Forest).
* `metro_kmeans.ipynb`: Jupyter Notebook contendo a nova modelagem por agrupamento K-Means e detecção por indicador de saúde (Load Ratio).
* `metropt+3+dataset/Data Description_Metro.txt`: Descrição técnica original do dataset e mapeamento das falhas.
* `artifacts/`: Diretório que armazena os outputs e modelos serializados gerados pelo notebook.

---

## 📊 O Dataset MetroPT-3

Os dados foram coletados a uma frequência de **1Hz** (uma leitura por segundo) de fevereiro a agosto de 2020, totalizando mais de 15 milhões de registros. Eles monitoram o comportamento térmico, elétrico e pneumático de um compressor de ar por meio de 15 sensores principais (7 analógicos e 8 digitais).

O dataset registra **4 falhas do tipo Vazamento de Ar (Air leak)**:
1. **18/04/2020** (dia inteiro)
2. **29/05/2020** às 23:30 até **30/05/2020** às 06:00
3. **05/06/2020** às 10:00 até **07/06/2020** às 14:30
4. **15/07/2020** às 14:30 até **15/07/2020** às 19:00

---

## ⚙️ Abordagem 1: Pipeline Supervisionado (Random Forest)

Implementado no arquivo `metro.ipynb`:
1. **Rótulos**: Define `target_pre_failure = 1` correspondente às 24 horas que antecedem cada falha.
2. **Divisão Temporal**: Treina com as 3 primeiras falhas e testa na 4ª falha.
3. **Modelo**: Normalização seguido por `RandomForestClassifier` com pesos balanceados de classes.
4. **Métricas**: Revela as limitações da abordagem supervisionada com apenas 4 eventos de falha e forte desvio térmico estacional no conjunto de teste.

---

## 🧩 Abordagem 2: Pipeline Não Supervisionado (K-Means Detalhado)

Implementado no arquivo `metro_kmeans.ipynb`, este método adota uma estratégia analítica baseada na **física operacional do equipamento** e no comportamento de séries temporais. O pipeline detalhado é dividido em 6 etapas cruciais:

### 1. Pré-processamento e Agregação Temporal (Downsampling)
* **O que faz**: Reduz o volume do dataset de 1Hz (uma leitura por segundo) para intervalos de **1 minuto** (`'1min'`). 
  * Para os **sensores analógicos**, calcula a média de cada minuto.
  * Para os **sensores digitais**, calcula a taxa de ativação média no minuto.
* **Por que é feito**: O volume de dados brutos (~15 milhões de linhas) gera muito ruído de alta frequência e exige alto poder de processamento. A agregação minuto a minuto mantém a integridade dos padrões operacionais macro (degradações lentas) e otimiza severamente a computação.

### 2. Normalização dos Dados
* **O que faz**: Aplica o `StandardScaler` sobre as 15 variáveis de sensores agregadas.
* **Por que é feito**: Sensores diferentes medem grandezas diferentes (temperatura em °C, pressão em bar, corrente elétrica em ampères). Sem normalização, variáveis com valores absolutos maiores (como temperaturas próximas a 60°C) dominariam o cálculo de distância euclidiana do K-Means, anulando a influência de sensores críticos como correntes e pressões que operam em escalas menores (como pressões de bar a 9 bar).

### 3. Agrupamento K-Means com Estabilização Dinâmica
* **O que faz**: Divide o dataset em 4 grupos ($K=4$). Como o K-Means atribui rótulos de forma arbitrária (o que pode fazer os números dos clusters mudarem a cada execução), criamos um algoritmo de remapeamento. Ele ordena os clusters de forma crescente pelo consumo médio de corrente do motor (`Motor_current_mean`).
* **Mapeamento Físico Final Estável**:
  * **Cluster 0 (Desligado / Idle)**: Corrente do motor em ~0.13A, pressão de saída TP2 em ~0.04 bar. Representa o compressor totalmente desligado.
  * **Cluster 1 (Atípico / Manutenção)**: Leituras de pressões e temperatura extremamente baixas. Indica reinicializações ou períodos pós-manutenção prolongados.
  * **Cluster 2 (Transição / Vazio)**: Motor ligado com corrente moderada de ~3.6A, mas pressão TP2 ainda próxima de zero e válvula COMP ativa (compressor ligado, mas sem carga ativa).
  * **Cluster 3 (Sob Carga / Ativo)**: Motor operando com corrente elevada de ~5.5A e gerando pressão alta (TP2 de ~8.30 bar e TP3 pneumático de ~8.87 bar). **Este é o estado em que o compressor está de fato bombeando ar.**

### 4. O Conceito Físico da Falha e o Indicador "Load Ratio"
* **A Física do Vazamento**: Em condições saudáveis, o compressor trabalha (entra no Cluster 3), eleva a pressão pneumática nos tanques e desliga (volta para o Cluster 0/Idle). Esse ciclo dura apenas alguns minutos, fazendo com que ele passe apenas **10% a 15% do dia ativo sob carga**. 
* Porém, se há um **vazamento de ar (Air leak)**, o ar escapa continuamente. O compressor tenta bombear ar para elevar a pressão, mas não consegue atingir o limite programado devido à perda contínua. Ele fica **preso no estado de carga (Cluster 3)** trabalhando sem parar.
* **Criação do Indicador**: Criamos a métrica **Load Ratio 24h** que calcula a média móvel das últimas 24 horas (janela de 1440 minutos) da variável binária `is_loaded` (que vale 1 se `cluster == 3`, e 0 caso contrário). Esse indicador mede a porcentagem de tempo que o compressor passa gerando carga por dia.

### 5. Regra de Disparo de Alertas e Resultados
Definimos um limiar de alerta contínuo de **25%** (ou seja, se o compressor passar mais de um quarto do dia ativo sob carga, gera-se um alarme).
A avaliação comparativa com as 4 falhas reais demonstrou a eficácia preventiva e diagnóstica:
* **Falha #4 (Julho)**: O indicador começou a subir gradualmente 12 horas antes do evento de vazamento, cruzando a marca de 25% e disparando um **ALERTA PREVENTIVO 1.4 horas antes** da quebra física.
* **Falhas #1, #2 e #3**: O alarme foi ativado muito rapidamente após o início dos eventos (entre **3.4 e 6.2 horas** de operação em falha). O indicador nesses casos pulou rapidamente de ~15% para quase **100% de uso contínuo**, denunciando o vazamento grave.

### 6. Por que esta abordagem é superior para este Trabalho?
* **Robusta a Mudanças Climáticas (Sazonalidade)**: O K-Means não correlaciona diretamente o valor absoluto da temperatura do óleo com a falha (problema que afetava o Random Forest no verão de julho/agosto). Ele monitora o *tempo ativo de trabalho* do motor, que é uma assinatura puramente física do vazamento.
* **Sem Necessidade de Rótulos Complexos**: Funciona baseado em comportamento anômalo e princípios físicos, eliminando a dependência de termos grandes volumes de dados históricos de quebras para ensinar a máquina.

---

## 🚀 Como Executar

1. Certifique-se de que os dados do compressor estão baixados na pasta correta: `metropt+3+dataset/MetroPT3(AirCompressor).csv`.
2. Certifique-se de ter os pacotes Python necessários instalados. Você pode instalá-los executando:
   ```bash
   pip install pandas numpy scikit-learn matplotlib seaborn joblib
   ```
3. Abra e execute qualquer um dos notebooks (`metro.ipynb` ou `metro_kmeans.ipynb`) célula por célula na sua IDE favorita.
