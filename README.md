
# Sistema de Verificação de Folha Salarial

Este projeto identifica possíveis anomalias no processamento de folhas salariais de colaboradores. Ele detecta:
1. **Rendimentos inusitados**: Rubricas do tipo RENDIMENTO que não aparecem nos últimos 6 meses.
2. **Variações bruscas em descontos**: Rubricas do tipo DESCONTO que apresentam uma variação maior ou igual a 5% em relação à média dos últimos meses.

## Estrutura do Projeto

O projeto segue a **Arquitetura Limpa** (Clean Architecture), organizada em camadas para garantir modularidade e facilidade de manutenção.

### Diretórios e Arquivos

```plaintext
folha_verificacao/
├── domain/
│   ├── __init__.py
│   ├── models.py            # Entidades principais do domínio (Colaborador, Rubrica).
│   ├── exceptions.py        # Exceções personalizadas do domínio.
├── application/
│   ├── __init__.py
│   ├── use_cases.py         # Casos de uso para detecção de anomalias.
├── infrastructure/
│   ├── __init__.py
│   ├── data_loader.py       # Carregamento e preprocessamento do dataset.
│   ├── data_writer.py       # Exportação dos resultados (opcional).
├── interface/
│   ├── __init__.py
│   ├── cli.py               # Interface de linha de comando para interagir com o sistema.
├── tests/
│   ├── test_domain.py       # Testes unitários para entidades do domínio.
│   ├── test_use_cases.py    # Testes para os casos de uso.
│   ├── test_infrastructure.py
├── main.py                  # Ponto de entrada do sistema.
├── requirements.txt         # Lista de dependências do projeto.
└── README.md                # Documentação do projeto.
```

## Configuração do Ambiente

### Pré-requisitos
Certifique-se de que você possui:
- Python 3.8 ou superior.
- `pip` instalado para gerenciar pacotes Python.

### Instalando Dependências

No diretório do projeto, execute:

```bash
pip install -r requirements.txt
```

### Dependências Principais

- **Pandas**: Manipulação e análise de dados tabulares.
- **NumPy**: Operações matemáticas e estatísticas.
- **Datetime**: Manipulação de datas.

## Como Executar o Sistema

### Passo 1: Estrutura do Dataset

O sistema espera um arquivo CSV com a seguinte estrutura:

| Campo              | Tipo de Dados      | Descrição                                                         |
|--------------------|-------------------|-----------------------------------------------------------------|
| nome               | str               | Nome do colaborador                                              |
| matricula          | str               | Matrícula ou código do colaborador                               |
| cpf                | str               | CPF do colaborador                                               |
| sexo               | str               | Sexo do colaborador (M/F)                                        |
| cargo              | str               | Cargo do colaborador                                             |
| cargo_nivel        | str               | Nível do cargo do colaborador                                    |
| dataadmissao       | str (aaaa/mm/dd)  | Data de admissão do colaborador                                  |
| datarescisao       | str (aaaa/mm/dd)  | Data de rescisão (vazio se o colaborador estiver ativo)          |
| datanascimento     | str (aaaa/mm/dd)  | Data de nascimento do colaborador                                |
| tipo_rubrica       | str               | Tipo da rubrica (BASE, RENDIMENTO, DESCONTO)                     |
| codigo_rubrica     | str               | Código da rubrica                                                |
| valor              | float             | Valor da rubrica                                                 |
| tipo_calculo       | str               | Tipo do cálculo (FO, FE, F13)                                    |
| ano_calculo        | int               | Ano do cálculo                                                   |
| mes_calculo        | int               | Mês do cálculo                                                   |

### Estratégia para Resolver o Problema

Passo 1: Definir as Entidades no Domínio
No módulo domain/models.py, criamos classes que representam os dados e regras de negócio:

```python
from datetime import date

class Colaborador:
    def __init__(self, matricula, nome, cpf, data_admissao, data_rescisao=None):
        self.matricula = matricula
        self.nome = nome
        self.cpf = cpf
        self.data_admissao = data_admissao
        self.data_rescisao = data_rescisao

class Rubrica:
    def __init__(self, codigo, tipo, valor, quantidade, ano, mes):
        self.codigo = codigo
        self.tipo = tipo  # BASE, RENDIMENTO, DESCONTO
        self.valor = valor
        self.quantidade = quantidade
        self.ano = ano
        self.mes = mes

```

Passo 2: Implementar os Casos de Uso
No módulo application/use_cases.py, implementamos as funções que realizam a lógica de negócio:

```python
import pandas as pd
from datetime import datetime, timedelta

def detect_unusual_rendimentos(dataframe, current_year, current_month):
    anomalies = []
    for matricula, group in dataframe.groupby("matricula"):
        current_rendimentos = group[
            (group["ano_calculo"] == current_year) & 
            (group["mes_calculo"] == current_month) &
            (group["tipo_rubrica"] == "RENDIMENTO")
        ]["codigo_rubrica"].unique()

        # Histórico de 6 meses
        start_date = datetime(current_year, current_month, 1) - timedelta(days=6 * 30)
        historical = group[
            (group["ano_calculo"] > start_date.year) | 
            ((group["ano_calculo"] == start_date.year) & (group["mes_calculo"] >= start_date.month)) &
            (group["tipo_rubrica"] == "RENDIMENTO")
        ]["codigo_rubrica"].unique()

        new_rendimentos = set(current_rendimentos) - set(historical)
        if new_rendimentos:
            anomalies.append({"matricula": matricula, "novos_rendimentos": list(new_rendimentos)})
    return anomalies

def detect_variations_in_descontos(dataframe, current_year, current_month):
    anomalies = []
    for matricula, group in dataframe.groupby("matricula"):
        current_data = group[
            (group["ano_calculo"] == current_year) & 
            (group["mes_calculo"] == current_month) &
            (group["tipo_rubrica"] == "DESCONTO")
        ]
        for _, row in current_data.iterrows():
            historical = group[
                (group["codigo_rubrica"] == row["codigo_rubrica"]) &
                (group["tipo_rubrica"] == "DESCONTO") &
                ((group["ano_calculo"] < current_year) | 
                ((group["ano_calculo"] == current_year) & (group["mes_calculo"] < current_month)))
            ]["valor"]

            if not historical.empty:
                mean = historical.mean()
                if abs(row["valor"] - mean) / mean >= 0.05:
                    anomalies.append({
                        "matricula": matricula,
                        "codigo_rubrica": row["codigo_rubrica"],
                        "current_value": row["valor"],
                        "mean": mean
                    })
    return anomalies
```

Passo 3: Infraestrutura para Carregar os Dados

No módulo infrastructure/data_loader.py, implementamos funções para carregar e preprocessar dados:

```python
import pandas as pd

def load_data(filepath):
    df = pd.read_csv(filepath)
    for col in ["dataadmissao", "datarescisao", "datanascimento"]:
        df[col] = pd.to_datetime(df[col], format='%Y/%m/%d', errors='coerce')
    df["ano_calculo"] = df["ano_calculo"].astype(int)
    df["mes_calculo"] = df["mes_calculo"].astype(int)
    return df

```

Passo 4: Interface com o Usuário
No módulo interface/cli.py, criamos uma CLI simples:

```python
import argparse
from infrastructure.data_loader import load_data
from application.use_cases import detect_unusual_rendimentos, detect_variations_in_descontos

def main():
    parser = argparse.ArgumentParser(description="Sistema de verificação de folha salarial")
    parser.add_argument("filepath", help="Caminho para o arquivo CSV com os dados")
    parser.add_argument("year", type=int, help="Ano do cálculo atual")
    parser.add_argument("month", type=int, help="Mês do cálculo atual")
    args = parser.parse_args()

    df = load_data(args.filepath)
    rendimentos_anomalies = detect_unusual_rendimentos(df, args.year, args.month)
    descontos_anomalies = detect_variations_in_descontos(df, args.year, args.month)

    print("Anomalias de Rendimentos:")
    print(rendimentos_anomalies)
    print("Anomalias de Descontos:")
    print(descontos_anomalies)

if __name__ == "__main__":
    main()
```

### Passo 2: Executando a Verificação

Use a interface de linha de comando para executar o programa:

```bash
python main.py <caminho_para_arquivo_csv> <ano_atual> <mes_atual>
```

Por exemplo:

```bash
python main.py dataset.csv 2024 12
```

### Saída do Sistema

O programa exibirá anomalias detectadas na saída do terminal. Exemplos de saída:

#### Anomalias de Rendimentos
```json
[
    {
        "matricula": "12345",
        "novos_rendimentos": ["BONUS_ANUAL"]
    }
]
```

#### Anomalias de Descontos
```json
[
    {
        "matricula": "67890",
        "codigo_rubrica": "PLANO_SAUDE",
        "current_value": 300.00,
        "mean": 285.00
    }
]
```



