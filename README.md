# 📊 BATE FATURAMENTO x CENSO DIÁRIO  
## Conciliação Assistencial – Internações SUS

---

## 🎯 Objetivo

Realizar a validação entre:

- **Planilha de Faturamento (AIH)**
- **Planilha de Censo Diário (Estatística / Movimentação)**

Identificando divergências relacionadas a:

- ✔ Caráter da internação (Eletiva / Urgência)  
- ✔ Tipo de Alta (Alta Hospitalar / Óbito / Transferência)  
- ✔ Clínica / Especialidade  
- ✔ Prontuários ausentes em uma das bases  

---

# 🏗️ Arquitetura do Processo

Fluxo estruturado em 8 etapas:

1. Importação de bibliotecas  
2. Definição de funções auxiliares  
3. Carga das planilhas  
4. Padronização de dados  
5. Normalização  
6. Filtro por competência  
7. Comparação das bases  
8. Geração do relatório Excel  

---

# 🧠 1. Bibliotecas

```python
import pandas as pd
from datetime import datetime
import xlsxwrite
```
🔧 2. Funções Auxiliares 
🔹 Conversão de Especialidade

```python
def especialidade(num):
    especialidades = {
        1: "C.CIRURGICA",
        2: "GO",
        3: "C.MEDICA",
        4: "Cuidados Prolongados",
        5: "Psiquiatria",
        6: "Tisiologia",
        7: "PEDIATRIA",
        8: "Reabilitação",
        9: "Hospital Dia"
    }
    return especialidades.get(num, num)
```

🔹 Conversão de Tipo de Alta
```python
def tipoAlta(num):
    motivoAlta = {
        "ALTA HOSPITALAR": [11,12,14,15,16,18,19,61,62,63,64,23],
        "ÓBITO": [41,42,43,65,67],
        "TRANSFERENCIA": [31]
    }

    if num in motivoAlta["ALTA HOSPITALAR"]:
        return "ALTA HOSPITALAR"
    elif num in motivoAlta["ÓBITO"]:
        return "ÓBITO"
    elif num in motivoAlta["TRANSFERENCIA"]:
        return "TRANSFERENCIA"
    else:
        return num
```
🔹 Conversão de Caráter de Internação
```python
def caraterInt(num):
    carater = {
        1: "E",  # Eletiva
        2: "U",
        5: "U",
        6: "U"
    }
    return carater.get(num, num)
🔹 Conversão de Datas
def converterData(data):
    try:
        dia, mes, ano = data.split("/")
        return datetime(int(ano), int(mes), int(dia)).date()
    except:
        return data
```
📂 3. Carga das Planilhas
```python
pf = pd.read_excel("faturamento.xls", engine="xlrd")
pm = pd.read_excel("censo_diario.xls", sheet_name="Janeiro 2024")
```
🔄 4. Padronização de Colunas
```python
pf = pf.rename(columns={
    "AIH_DT_SAI": "Saida",
    "AIH_DT_INT": "Entrada",
    "AIH_PRONT": "Prontuario",
    "AIH_MOT_COB": "Tipo de Alta",
    "AIH_CLINICA": "Clinica",
    "AIH_CAR_INT": "Carater Int",
    "AIH_NUM_AIH": "AIH"
})

pm = pm.rename(columns={
    "DATA.1": "Saida",
    "DATA INTERNAÇÃO": "Entrada",
    "PRONTUARIO": "Prontuario",
    "STATUS": "Tipo de Alta",
    "ESPECIALIDADE": "Clinica",
    "U": "Carater Int"
})
```
🔁 5. Normalização de Dados
```python
pf["Clinica"] = pf["Clinica"].map(especialidade)
pf["Tipo de Alta"] = pf["Tipo de Alta"].map(tipoAlta)
pf["Carater Int"] = pf["Carater Int"].map(caraterInt)

pf["Entrada"] = pf["Entrada"].map(converterData)
pf["Saida"] = pf["Saida"].map(converterData)

pm["Entrada"] = pm["Entrada"].map(converterData)
pm["Saida"] = pm["Saida"].map(converterData)
```
📅 6. Filtro por Competência
```python
inicio = datetime(2024, 1, 1).date()
fim = datetime(2024, 1, 31).date()

pf = pf[(pf["Saida"] >= inicio) & (pf["Saida"] <= fim)]
pm = pm[(pm["Saida"] >= inicio) & (pm["Saida"] <= fim)]
```

🔎 7. Identificação de Divergências
🔹 Estruturas utilizadas
```python
clinica = {}
tipoAlta = {}
carater = {}
ausentePm = {}
ausentePf = {}
```
🔹 Comparação Faturamento → Censo

```python
for prontuario in pf["Prontuario"].unique():

    dadosPf = pf[pf["Prontuario"] == prontuario]
    dadosPm = pm[pm["Prontuario"] == prontuario]

    if len(dadosPm) == 0:
        ausentePm[prontuario] = dadosPf.values.tolist()

    else:

        if dadosPf.iloc[0]["Carater Int"] != dadosPm.iloc[0]["Carater Int"]:
            carater[prontuario] = [
                prontuario,
                dadosPf.iloc[0]["Carater Int"],
                dadosPm.iloc[0]["Carater Int"]
            ]

        if dadosPf.iloc[0]["Tipo de Alta"] != dadosPm.iloc[0]["Tipo de Alta"]:
            tipoAlta[prontuario] = [
                prontuario,
                dadosPf.iloc[0]["Tipo de Alta"],
                dadosPm.iloc[0]["Tipo de Alta"]
            ]

        if dadosPf.iloc[0]["Clinica"] != dadosPm.iloc[0]["Clinica"]:
            clinica[prontuario] = [
                prontuario,
                dadosPf.iloc[0]["Clinica"],
                dadosPm.iloc[0]["Clinica"]
            ]
```
📈 8. Geração do Relatório Excel
```python
workbook = xlsxwriter.Workbook("Bate_Faturamento.xlsx")
worksheet = workbook.add_worksheet("Divergencias")

worksheet.write_row(0, 0, [
    "Prontuario",
    "Carater Fat",
    "Carater Estat",
    "Alta Fat",
    "Alta Estat",
    "Clinica Fat",
    "Clinica Estat"
])

linha = 1

for pront in carater:
    worksheet.write_row(linha, 0, carater[pront])
    linha += 1

workbook.close()
```
## 📊 Resultado Gerado
 O arquivo Excel contém:
- 🔍 Divergência de Clínica
- 🔍 Divergência de Tipo de Alta
- 🔍 Divergência de Caráter
- 🔍 Prontuários apenas no Faturamento
- 🔍 Prontuários apenas no Censo

## 🏥 Aplicação Hospitalar
 Esse processo permite:
- Garantir consistência entre AIH faturada e produção estatística
- Reduzir risco de glosa por divergência de caráter
- Validar motivo de alta antes do envio ao SIH/SUS
- Melhorar governança de dados assistenciais
- Detectar falhas de integração entre sistema hospitalar e planilhas paralelas

## 🚀 Possíveis Evoluções
- 🔹 Versão com merge() ao invés de loops
- 🔹 Integração direta ao Oracle via cx_Oracle
- 🔹 Automatização com agendamento (Task Scheduler / Cron)
- 🔹 Dashboard analítico para indicadores de divergência
