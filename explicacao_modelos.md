# Análise dos Modelos de Manutenção Preditiva - MetroPT-3

Este documento apresenta uma explicação detalhada e a comparação entre duas abordagens distintas (supervisionada e não supervisionada) desenvolvidas nos notebooks `metro.ipynb` e `metro_kmeans.ipynb` para prever falhas no compressor de ar de um trem metropolitano (MetroPT-3).

---

## 1. O Dataset (MetroPT-3)

Ambos os notebooks utilizam o dataset **MetroPT-3 (Air Compressor)**, que consiste em leituras de sensores coletadas ao longo de vários meses em 2020.

- **Volume de Dados**: O dataset original possui mais de 15 milhões de registros coletados em alta frequência (aproximadamente 1 Hz).
- **Variáveis**: Contém 15 colunas representativas dos sensores de um compressor de ar. Dentre as principais, destacam-se:
  - **Analógicas**: `TP2` (Pressão de saída para o sistema), `TP3` (Pressão no painel pneumático), `H1` (Pressão da válvula primária), `Oil_temperature` (Temperatura do óleo do compressor) e `Motor_current` (Corrente elétrica do motor em Ampères).
  - **Digitais (Estados)**: `COMP` (Válvula do compressor ativa), `Towers` (Torres de secagem) e `Pressure_switch` (Sinal de limite de pressão atingido).
- **Falhas**: Existem apenas **4 eventos de falha crítica reportados** (do tipo *Air leak* - Vazamento de ar) em todo o período.
- **Pré-processamento**: Devido ao alto volume e ruído dos dados em 1 Hz, ambos os modelos aplicam uma **agregação temporal por minuto**, calculando médias e taxas de ativação para estabilizar os dados e viabilizar o processamento.

---

## 2. Abordagem Supervisionada (`metro.ipynb`)

O notebook `metro.ipynb` foca na criação de um modelo de Machine Learning tradicional, treinado para classificar e prever explicitamente quando uma falha vai ocorrer com base no histórico conhecido.

### Como funciona o algoritmo

1. **Criação de Rótulos (Labels)**: Utilizando os horários documentados das 4 falhas, o notebook cria janelas de anotação. A variável alvo de maior interesse é a `target_pre_failure`, que marca como `1` (verdadeiro) as **24 horas imediatamente anteriores** ao início de uma falha.
2. **Engenharia de Atributos (Feature Engineering)**: Além das médias por minuto, o notebook enriquece os dados para capturar tendências ao longo do tempo. Ele calcula:
   - **Janelas móveis (Rolling Windows)**: Média e desvio padrão dos últimos 15 minutos e da última hora. Isso permite ao modelo "lembrar" se a pressão ou a temperatura vêm subindo ou descendo.
   - **Deltas**: A diferença entre a medição atual e a medição passada, explicitando taxas de variação.
3. **Treinamento do Modelo (Random Forest)**: Um classificador do tipo *Floresta Aleatória* foi escolhido devido à sua robustez a dados não lineares. Ele analisa as variáveis criadas na etapa anterior para classificar cada minuto do dataset como `0` (Normal) ou `1` (Pré-Falha). O modelo também nos fornece a "Importância das Variáveis", indicando matematicamente quais sensores mais influenciaram na decisão.
4. **Geração de Alertas**: O modelo gera uma **probabilidade de risco** contínua. Um limiar (threshold) é estabelecido (ex: probabilidade > 0.4) para disparar um alerta preditivo.

---

## 3. Abordagem Não Supervisionada (`metro_kmeans.ipynb`)

O notebook `metro_kmeans.ipynb` adota uma estratégia radicalmente diferente. Em vez de tentar "aprender" o padrão de uma falha a partir de apenas 4 exemplos, ele tenta mapear os estados normais de funcionamento da máquina e deduzir a falha por alterações drásticas nesse ciclo.

### Como funciona o algoritmo

1. **Normalização e Agrupamento (Clustering)**: Os dados originais operam em escalas muito diferentes (Temperaturas chegam a 80ºC, enquanto a corrente fica em torno de 5A). É aplicado um `StandardScaler` para normalizar as variáveis para que tenham a mesma importância. Em seguida, o algoritmo **K-Means** agrupa os dados matematicamente em `K=4` clusters distintos.
2. **Perfilamento Físico**: Ao analisar as médias das variáveis em cada cluster formado, percebe-se que o algoritmo isolou perfeitamente 4 estados físicos de operação do compressor:
   - *Cluster 0*: Desligado / Idle
   - *Cluster 1*: Atípico / Manutenção
   - *Cluster 2*: Transição / Vazio
   - *Cluster 3*: **Sob Carga (Ativo)** - estado de maior estresse (corrente alta e pressão alta).
3. **Indicador de Saúde (Load Ratio)**: A grande sacada deste algoritmo é a criação de um indicador simples e contínuo. Ele constrói uma variável binária `is_loaded` marcando apenas os momentos em que a máquina está no *Cluster 3*. Em seguida, aplica uma média móvel de 24 horas (1440 minutos) sobre isso, revelando a porcentagem exata de tempo que o compressor passa gerando pressão (trabalhando pesado) em um dia.
4. **Geração de Alertas**: Quando há um vazamento de ar, o compressor nunca atinge a pressão necessária para desligar e passa a trabalhar sem parar. O K-Means detecta que o tempo "Sob Carga" salta de 10-15% para quase 100%. Um limite simples (ex: > 25% de tempo sob carga no dia) dispara um alerta preventivo horas antes da parada do sistema.

---

## 4. Comparação entre as Aplicações

Embora ambas as abordagens consigam identificar que há algo errado com o trem antes da falha, elas possuem características, vantagens e desvantagens muito distintas para ambientes reais de manutenção preditiva.

### Modelo Supervisionado (Random Forest)

- **Vantagens**:
  - Avalia automaticamente a interação complexa entre dezenas de variáveis e derivadas temporais.
  - Retorna uma métrica probabilística direta (probabilidade de falha em %).
- **Desvantagens**:
  - **Extremo Desbalanceamento**: Como só existem 4 falhas em meses de dados, o modelo tem altíssimo risco de *overfitting* (decorar as falhas em vez de aprender o padrão).
  - **Sensibilidade Sazonal**: Modelos supervisionados em dados contínuos podem sofrer falsos positivos com mudanças de temperatura climática (ex: dias muito quentes no verão afetando a temperatura do óleo, confundindo o modelo).

### Modelo Não Supervisionado (K-Means)

- **Vantagens**:
  - **Não depende de rótulos de falha**: Pode ser implementado no "dia 1" da operação do equipamento, pois mapeia apenas o funcionamento do compressor, não o erro.
  - **Alta Robustez**: Como o modelo foca no *ciclo de trabalho* (tempo em que fica ativo) em vez dos valores absolutos das variáveis térmicas, ele é virtualmente imune a falsos positivos causados por sazonalidade climática.
  - **Explicabilidade**: A regra de alerta é fisicamente interpretável pelos engenheiros de manutenção ("O compressor está trabalhando tempo demais sem parar").
- **Desvantagens**:
  - **Exige Conhecimento de Domínio**: Requer que um especialista valide os clusters gerados pelo K-Means e defina qual o limite de segurança aceitável para o indicador criado (Load Ratio > 25%).
  - **Foco Específico**: Esta modelagem foi excelente para o tipo específico de falha reportada (Vazamento de Ar). Outros tipos de falhas silenciosas que não afetam o ciclo de trabalho poderiam passar despercebidas.

### Conclusão

Para este cenário específico (altíssima escassez de exemplos de falha e mecânica clara de funcionamento do compressor em ciclos), a **abordagem Não Supervisionada com K-Means provou ser vastamente superior, mais elegante e confiável para ir para produção**, como foi evidenciado pela clareza e separabilidade dos gráficos de alerta antes das falhas.
