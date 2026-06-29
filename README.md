# Detecção de Falhas em Compressor de Trem Metrô (MetroPT-3)

Este repositório contém um trabalho de faculdade focado no desenvolvimento de abordagens de Machine Learning para a **previsão de falhas (Manutenção Preditiva)** utilizando o conjunto de dados real **MetroPT-3**. O objetivo principal é monitorar o compressor de ar de um trem metropolitano (APU) e gerar alertas de vazamento de ar (Air leak).

---

## 📌 Sumário do Projeto

- [Objetivo do Trabalho](#-objetivo-do-trabalho)
- [Estrutura de Arquivos](#-estrutura-de-arquivos)
- [O Dataset MetroPT-3](#-o-dataset-metropt-3)
- [Abordagem 1: Pipeline Supervisionado (Random Forest)](#-abordagem-1-pipeline-supervisionado-random-forest)
- [Abordagem 2: Pipeline Não Supervisionado (K-Means)](#-abordagem-2-pipeline-não-supervisionado-k-means)
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

* [metro.ipynb](file:///c:/Users/walis/OneDrive/Documentos/atividades_facul/Trabalho_SI_T/metro.ipynb): Jupyter Notebook contendo a modelagem supervisionada inicial (Random Forest).
* [metro_kmeans.ipynb](file:///c:/Users/walis/OneDrive/Documentos/atividades_facul/Trabalho_SI_T/metro_kmeans.ipynb): Jupyter Notebook contendo a nova modelagem por agrupamento K-Means e detecção por indicador de saúde (Load Ratio).
* [metropt+3+dataset/Data Description_Metro.txt](file:///c:/Users/walis/OneDrive/Documentos/atividades_facul/Trabalho_SI_T/metropt+3+dataset/Data%20Description_Metro.txt): Descrição técnica original do dataset e mapeamento das falhas.
* **artifacts/**: Diretório que armazena os outputs e modelos serializados gerados pelo notebook.

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

Implementado no arquivo [metro.ipynb](file:///c:/Users/walis/OneDrive/Documentos/atividades_facul/Trabalho_SI_T/metro.ipynb):
1. **Rótulos**: Define `target_pre_failure = 1` correspondente às 24 horas que antecedem cada falha.
2. **Divisão Temporal**: Treina com as 3 primeiras falhas e testa na 4ª falha.
3. **Modelo**: Normalização seguido por `RandomForestClassifier` com pesos balanceados de classes.
4. **Métricas**: Revela as limitações da abordagem supervisionada com apenas 4 eventos de falha e forte desvio térmico estacional no conjunto de teste.

---

## 🧩 Abordagem 2: Pipeline Não Supervisionado (K-Means)

Implementado no arquivo [metro_kmeans.ipynb](file:///c:/Users/walis/OneDrive/Documentos/atividades_facul/Trabalho_SI_T/metro_kmeans.ipynb):
1. **Modelagem**: Aplica `KMeans(n_clusters=4)` sobre as 15 variáveis base de sensores agregadas por minuto (`1min`).
2. **Mapeamento de Estados**: Mapeamento dinâmico ordenado pela corrente do motor para manter a estabilidade dos clusters:
   * **Cluster 0**: **Desligado / Idle** (corrente ~0.13A).
   * **Cluster 1**: **Estado Atípico / Manutenção** (pressões/temperatura muito baixas).
   * **Cluster 2**: **Transição / Vazio** (motor a ~3.6A sem vazão ativa).
   * **Cluster 3**: **Sob Carga (Ativo)** (motor a ~5.5A e pressão TP2 de saída a ~8.3 bar).
3. **Indicador Load Ratio (24h)**: Percentual de tempo diário que o compressor passa ativo sob carga (**Cluster 3**). Em operação normal, o valor fica entre 10% e 15%.
4. **Avaliação (Limiar 25%)**:
   * O indicador sobe para perto de 100% durante as falhas mecânicas.
   * **Falha #4**: Disparou **ALERTA PREVENTIVO com 1.4 horas de antecedência**.
   * **Falhas #1, #2 e #3**: Dispararam alertas de detecção poucas horas após o início do evento.

---

## 🚀 Como Executar

1. Certifique-se de que os dados do compressor estão baixados na pasta correta: `metropt+3+dataset/MetroPT3(AirCompressor).csv`.
2. Certifique-se de ter os pacotes Python necessários instalados. Você pode instalá-los executando:
   ```bash
   pip install pandas numpy scikit-learn matplotlib seaborn joblib
   ```
3. Abra e execute qualquer um dos notebooks ([metro.ipynb](file:///c:/Users/walis/OneDrive/Documentos/atividades_facul/Trabalho_SI_T/metro.ipynb) ou [metro_kmeans.ipynb](file:///c:/Users/walis/OneDrive/Documentos/atividades_facul/Trabalho_SI_T/metro_kmeans.ipynb)) célula por célula na sua IDE favorita.
