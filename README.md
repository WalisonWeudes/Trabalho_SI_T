# Detecção de Falhas em Compressor de Trem Metropolitano (MetroPT-3)

Este repositório contém um projeto focado no desenvolvimento de uma abordagem de Aprendizado de Máquina Não Supervisionado para a detecção de vazamentos de ar (Air leak) em compressores de trens metropolitanos (APU - Air Production Unit) utilizando o conjunto de dados real MetroPT-3.

---

## Sumário do Projeto

- [Objetivo do Trabalho](#objetivo-do-trabalho)
- [Estrutura de Arquivos](#estrutura-de-arquivos)
- [O Dataset MetroPT-3](#o-dataset-metropt-3)
- [Pipeline Não Supervisionado (K-Means e Load Ratio)](#pipeline-nao-supervisionado-k-means-e-load-ratio)
- [Como Executar](#como-executar)

---

## Objetivo do Trabalho

O compressor de ar de um trem é um componente crítico para o acionamento de sistemas pneumáticos, como freios de emergência e portas. Falhas repentinas podem paralisar a linha férrea e causar sérios transtornos operacionais.

Este projeto desenvolve uma modelagem preditiva baseada em detecção de anomalias:
* Agrupa os dados dos sensores usando K-Means para mapear os estados físicos do compressor de forma automatizada.
* Calcula o tempo de atividade diária da máquina (Load Ratio) em uma janela móvel de 24 horas.
* Fornece uma detecção robusta de vazamentos de ar que contorna flutuações térmicas climáticas sem a necessidade de rotulamento histórico de falhas no treinamento.

---

## Estrutura de Arquivos

O projeto está organizado da seguinte forma:

* `metro_kmeans.ipynb`: Jupyter Notebook contendo todo o pipeline não supervisionado (K-Means, cálculo do Load Ratio e avaliação).
* `metropt+3+dataset/Data Description_Metro.txt` e `Data Description_Metro.pdf`: Descrições técnicas oficiais do dataset e histórico de falhas da operadora.
* `artifacts/`: Diretório que armazena os dados processados e o modelo treinado exportado em formato binário.

---

## O Dataset MetroPT-3

Os dados foram coletados a uma frequência de 1Hz (uma leitura por segundo) de fevereiro a agosto de 2020, totalizando mais de 15 milhões de registros. Eles monitoram o comportamento térmico, elétrico e pneumático de um compressor de ar por meio de 15 sensores (7 analógicos e 8 digitais).

O dataset registra 4 falhas oficiais de vazamento de ar (Air leak):
1. **18/04/2020** (dia inteiro)
2. **29/05/2020** às 23:30 até **30/05/2020** às 06:00
3. **05/06/2020** às 10:00 até **07/06/2020** às 14:30
4. **15/07/2020** às 14:30 até **15/07/2020** às 19:00

---

## Pipeline Não Supervisionado (K-Means e Load Ratio)

Esta abordagem baseia-se na física de funcionamento do equipamento e é dividida em três fases principais:

### 1. Pré-processamento e Agregação
* Os dados brutos a 1Hz são agregados em janelas de 1 minuto (médias para analógicos, taxa de ativação para digitais) para mitigar ruídos de alta frequência e reduzir o custo computacional.
* Aplica-se a normalização z-score (StandardScaler) sobre os sensores analógicos.

### 2. Segmentação de Estados Físicos (K-Means)
* O K-Means divide as leituras em K=4 clusters.
* Os clusters são remapeados e ordenados de forma crescente pela corrente do motor:
  * **Cluster 0 (Desligado / Idle):** Corrente em ~0.13A, pressão zerada.
  * **Cluster 1 (Atípico / Manutenção):** Compressor inativo com baixa temperatura e pressão residual.
  * **Cluster 2 (Transição / Vazio):** Motor ligado com corrente de ~3.6A, mas sem carga de bombeamento (COMP ativo).
  * **Cluster 3 (Sob Carga / Ativo):** Motor em potência máxima (~5.5A) bombeando ar comprimido de forma ativa (TP2 em ~8.3 bar).

### 3. Indicador Load Ratio e Regra de Alerta
* Cria-se a variável binária `is_loaded` que assume valor 1 se o compressor opera no Cluster 3, e 0 caso contrário.
* O **Load Ratio 24h** calcula a média móvel das últimas 24 horas (janela de 1440 minutos) da atividade sob carga. O compressor saudável opera tipicamente entre 10% e 15% do dia.
* Define-se o limiar de alarme de segurança em **25%** da taxa diária sob carga. Se ultrapassado, o alarme preditivo é acionado.

### Resultados Obtidos:
* **Falha #4 (Desgaste Gradual):** O alarme preditivo disparou com **1.4 horas de antecedência** em relação ao colapso físico.
* **Falhas #1, #2 e #3 (Quebras Súbitas):** O alarme disparou em média **4.4 horas após o início** do vazamento, tempo considerado excelente para evitar a queima do motor devido ao esforço de trabalho contínuo.
* **Sazonalidade:** O modelo provou-se totalmente imune a alarmes falsos de calor no verão de julho e agosto.

---

## Como Executar

Siga os passos abaixo para executar a modelagem em seu ambiente local:

### 1. Preparação dos Dados
Faça o download do arquivo de dados bruto a partir do seguinte link do Google Drive:
[Dataset MetroPT-3 (Google Drive)](https://drive.google.com/drive/folders/1eCd9Y1lhOlDYk7t-0BEyQU6o9FG8SHnp?usp=sharing)

Certifique-se de extrair e salvar o arquivo de dados brutos exatamente no seguinte caminho do projeto:
`metropt+3+dataset/MetroPT3(AirCompressor).csv`

### 2. Instalação das Dependências
Instale as bibliotecas Python necessárias executando o comando abaixo em seu terminal:
```bash
pip install pandas numpy scikit-learn matplotlib seaborn joblib
```

### 3. Execução do Notebook
Abra qualquer uma das ferramentas compatíveis (como Jupyter Notebook, JupyterLab ou VS Code) e execute as células do arquivo de forma sequencial:
* Execute `metro_kmeans.ipynb` para visualizar e validar o modelo não supervisionado (K-Means).
