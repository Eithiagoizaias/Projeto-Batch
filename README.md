# 📦 Projeto 1 — Ingestão Batch de Dados Transacionais

Pipeline de ingestão batch orquestrado com **Apache Airflow**, processamento com **PySpark** e armazenamento em formato **Delta Lake**, seguindo a arquitetura medallion (Source → Bronze → Silver).

---

## 🏗️ Arquitetura

```
source/
└── transaction_system/
    └── transaction_data_YYYY_MM_DD.csv   ← Dados brutos por data

bronze/
└── transaction_data/                     ← Delta Table (dados brutos)

silver/
└── transaction_data/                     ← Delta Table (dados refinados)
```

O pipeline segue o padrão **Medallion Architecture**:

- **Source** → CSVs diários gerados pelo sistema transacional
- **Bronze** → Ingestão raw dos CSVs em Delta Lake, com suporte a reprocessamento
- **Silver** → Dados refinados com merge incremental via Delta Lake

---

## 🔄 Pipeline — DAG: `INGEST_TRANSACTION_DATA`

```
INGEST_DATA_FROM_SOURCE_TO_BRONZE  >>  INGEST_DATA_FROM_BRONZE_TO_SILVER
```

| Propriedade | Valor |
|---|---|
| DAG ID | `INGEST_TRANSACTION_DATA` |
| Schedule | `0 3 * * *` (diariamente às 3h) |
| Start Date | 2025-04-10 |
| Catchup | Habilitado |

### Etapa 1 — Source to Bronze
- Lê o CSV do dia correspondente à data de processamento
- Se a tabela Delta já existe: deleta os registros da data e faz append (idempotência)
- Se não existe: cria a tabela Delta com os dados do dia

### Etapa 2 — Bronze to Silver
- Lê a tabela Delta Bronze
- Se a tabela Silver já existe: filtra apenas registros mais recentes e executa merge (upsert por `transaction_id`)
- Se não existe: cria a tabela Silver com todos os dados do Bronze

---

## 📁 Estrutura do Projeto

```
projeto_ingestao_batch/
│
├── dags/
│   └── ingest_transaction_data.py    # DAG principal do Airflow
│
├── scripts/
│   ├── ingest_source_to_bronze.py    # Lógica de ingestão Source → Bronze
│   ├── ingest_bronze_to_silver.py    # Lógica de ingestão Bronze → Silver
│   └── mock_data_spliter.py          # Gerador de dados mock para testes
│
├── mock_data/
│   └── MOCK_DATA.csv                 # Dataset mock com 1000 transações
│
├── requirements.txt
└── README.md
```

---

## 🛠️ Tecnologias

| Tecnologia | Versão | Uso |
|---|---|---|
| Python | 3.12 | Linguagem principal |
| Apache Airflow | 2.10.4 | Orquestração do pipeline |
| PySpark | 3.5.1 | Processamento distribuído |
| Delta Lake | 3.1.0 | Formato de armazenamento |
| Pandas | latest | Geração dos dados mock |

---

## ⚙️ Pré-requisitos

- Windows com WSL 2 + Ubuntu
- Python 3.12
- Java 11 (necessário para PySpark)

---

## 🚀 Como executar

### 1. Clonar o repositório

```bash
git clone https://github.com/Eithiagoizaias/Projeto-Batch.git
cd projeto_ingestao_batch
```

### 2. Criar e ativar o ambiente virtual

```bash
python3 -m venv venv_projeto_1
source venv_projeto_1/bin/activate
```

### 3. Instalar dependências

```bash
pip install -r requirements.txt
pip install pyspark==3.5.1 delta-spark==3.1.0
```

### 4. Configurar o Java 11

```bash
sudo apt install openjdk-11-jdk -y
echo 'export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64' >> ~/.bashrc
echo 'export PATH=$JAVA_HOME/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

### 5. Configurar o DATALAKE_PATH

Nos arquivos `dags/ingest_transaction_data.py`, `scripts/ingest_source_to_bronze.py` e `scripts/ingest_bronze_to_silver.py`, defina o caminho do seu datalake:

```python
DATALAKE_PATH = '/caminho/para/seu/datalake'
```

### 6. Gerar os dados mock

```bash
python scripts/mock_data_spliter.py
```

Isso vai gerar 10 arquivos CSV em `source/transaction_system/`, um por dia nos últimos 10 dias.

### 7. Configurar e iniciar o Airflow

```bash
export AIRFLOW_HOME=$(pwd)
airflow db init
cp dags/ingest_transaction_data.py ~/airflow/dags/
airflow standalone
```

Acesse a interface em **http://localhost:8080** e ative a DAG `INGEST_TRANSACTION_DATA`.

---

## 📊 Dados Mock

O dataset `MOCK_DATA.csv` contém 1000 registros com a seguinte estrutura:

| Coluna | Tipo | Descrição |
|---|---|---|
| `transaction_id` | int | Identificador único da transação |
| `customer_id` | int | Identificador do cliente |
| `amount` | float | Valor da transação |
| `transaction_date` | date | Data da transação (adicionada pelo splitter) |

O script `mock_data_spliter.py` divide o dataset em 10 partes iguais, atribuindo uma data diferente para cada parte (últimos 10 dias), simulando um sistema transacional real.

---

## 📌 Observações

- O pipeline suporta **reprocessamento idempotente**: rodar a DAG para uma data já processada vai substituir os dados corretamente sem duplicações.
- O ambiente foi desenvolvido e testado em **WSL 2 com Ubuntu** no Windows.
- Para produção, recomenda-se substituir o SQLite pelo **PostgreSQL** como metadata database do Airflow.
