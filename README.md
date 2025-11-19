# 🏥 Pipeline MLOps para Previsão de Custos de Seguro Saúde

Este projeto implementa um pipeline MLOps ponta a ponta na plataforma **Databricks**, utilizando **Delta Lake**, **PySpark** e **MLflow**. O objetivo é prever os custos de seguro saúde (`charges`) com base em características dos pacientes, seguindo a arquitetura de camadas **Bronze, Silver e Gold**.

---

## ⚙️ Arquitetura e Ferramentas

| Ferramenta | Função |
| :--- | :--- |
| **Plataforma** | Databricks (Spark Connect) |
| **Armazenamento** | Delta Lake (Unity Catalog) |
| **Engenharia de Dados e ML** | PySpark / Spark MLlib |
| **MLOps** | MLflow (Tracking, Model Registry e Artifact Storage via Unity Catalog Volumes) |

---

## 1. Notebook 01: ETL e Ingestão de Dados (Bronze)

O primeiro notebook é responsável por carregar os dados brutos e salvá-los na camada **Bronze** (Raw Data) do Delta Lake.

### 📝 Descrição da Operação

* **Fonte:** Dados de seguro (idade, sexo, BMI, etc.).
* **Destino:** Tabela Delta `bronze_insurance_costs`.
* **Ação:** Leitura, inferência de *schema* e salvamento inicial.

### Saída da Execução

O *schema* do DataFrame Bruto confirma a estrutura dos dados:

Schema do DataFrame Bruto: root |-- age: integer (nullable = true) |-- sex: string (nullable = true) |-- bmi: double (nullable = true) |-- children: integer (nullable = true) |-- smoker: string (nullable = true) |-- region: string (nullable de = true) |-- charges: double (nullable = true)


**SUCESSO:** Dados brutos salvos na Tabela Delta: `dev_catalogue.staging_schema.bronze_insurance_costs`

---

## 2. Notebook 02: Treinamento e Registro do Modelo (Silver)

Este notebook implementa a Engenharia de Features e treina um modelo de Regressão Linear, registrando a Pipeline completa no MLflow.

### Camada SILVER: Pré-Processamento

Os dados da camada **Bronze** são lidos. A coluna alvo (`charges`) é renomeada para `label`, e os dados são salvos na camada **Silver** para servir como fonte de treino.

**Tabela: Estrutura da `silver_insurance_features`**

| Coluna | Descrição |
| :--- | :--- |
| `label` | Custo do Seguro (**variável alvo**). |
| `age`, `bmi`, `children` | Features numéricas. |
| `sex`, `smoker`, `region` | Features categóricas (brutas). |

Exemplo de dados lidos (Bronze):

| age | sex | bmi | children | smoker | region | charges |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| 19 | female | 27.9 | 0 | yes | southwest | 16884.924 |
| 18 | male | 33.77 | 1 | no | southeast | 1725.5523 |
| ... | ... | ... | ... | ... | ... | ... |


Exemplo de dados transformados (Silver), incluindo a variável alvo (`label`):

| label | age | bmi | children | sex | smoker | region |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| 16884.924 | 19 | 27.9 | 0 | female | yes | southwest |
| 1725.5523 | 18 | 33.77 | 1 | male | no | southeast |
| ... | ... | ... | ... | ... | ... | ... |

Camada SILVER salva com sucesso na Tabela Delta (Esquema Migrado): dev_catalogue.staging_schema.silver_insurance_features


### Treinamento e Registro (MLflow)

Uma **Pipeline completa do Spark ML** foi treinada e registrada, garantindo que o modelo de produção inclua todas as etapas de pré-processamento.

* **Modelo Registrado:** `Insurance_Cost_LR_Model` (Versão 3)
* **Métrica Chave:** RMSE (Root Mean Squared Error)

Tentando limpar variáveis antigas para evitar Model Cache Overflow... Registered model 'Insurance_Cost_LR_Model' already exists. Creating a new version of this model... Created version '3' of model 'workspace.default.insurance_cost_lr_model'.
✅ REGISTRO DA PIPELINE FINALIZADO!

RMSE: 5696.75 | A Pipeline completa está registrada na Versão mais recente.


O RMSE de **$5696.75$** indica a precisão média do modelo em dólares.

---

## 3. Notebook 03: Inferência em Lote e Camada Gold

Este notebook simula a inferência de novos dados, carrega a Pipeline registrada (com o Alias `Production`) e salva os resultados finais na camada **Gold**.

### Carregamento do Modelo e Inferência

O modelo foi carregado usando o Alias `Production`. A Pipeline é aplicada para gerar as previsões.

URI de carregamento: models:/Insurance_Cost_LR_Model@Production

Novos Dados de Entrada (Clientes para Previsão): 

| age | sex | bmi | children | smoker | region |
| :---: | :---: | :---: | :---: | :---: | :---: |
| 28 | female | 33.0 | 1 | no | northwest |
| 55 | male | 30.5 | 2 | yes | southeast |
| 19 | female | 25.0 | 0 | no | southwest |

✅ Modelo carregado com sucesso do MLflow Registry: models:/Insurance_Cost_LR_Model@Production


### Resultados Finais (Camada Gold)

As previsões são salvas na Tabela Delta **Gold** (`gold_insurance_predictions`), que está pronta para consumo.

| age | smoker | region | estimated\_insurance\_cost |
| :---: | :---: | :---: | :---: |
| 28 | no | northwest | 6510.24 |
| 55 | yes | southeast | 36546.76 |
| 19 | no | southwest | 249.78 |

Resultados da Inferência (Custo Estimado do Seguro):

| age | smoker | region | estimated\_insurance\_cost |
| :---: | :---: | :---: | :---: |
| 28 | no | northwest | 6510.24 |
| 55 | yes | southeast | 36546.76 |
| 19 | no | southwest | 249.78 |

✅ Camada GOLD (Inferência) salva com sucesso em: dev_catalogue.staging_schema.gold_insurance_predictions


O projeto demonstra um fluxo MLOps totalmente funcional, desde a ingestão de dados